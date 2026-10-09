# Building units: `make!`

`make!` declares a build over **compilation units**. It decides which units to
compile and when; what happens inside a unit is the compiler's business.

```meadow
make! {
  pub Build "miniml"
    | sources = ".ml"
    | manifest = "Unit"
    | compile = compileUnit
}
```

This writes `buildBuild root`, which builds the unit at `root` and every unit
it depends on, prints how each went, and answers whether all of them built. It
also writes `givenBuild root dir`: what the unit at `dir` was compiled from in
the last build, for a tool that wants to ask the compiler about a unit again.

## Settings

| setting        | required | what it is                                                               |
| -------------- | -------- | ------------------------------------------------------------------------ |
| `sources`      | yes      | the ending of a unit's source files, `".ml"`                             |
| `manifest`     | yes      | the file that makes a directory a unit, `"Unit"`                         |
| `compile`      | yes      | the compiler: a function of a `Given i` answering `Result String i`      |
| `parallel`     | no       | `false` to compile one unit after another; a build is parallel otherwise |
| `dependencies` | no       | how to read a manifest, for one that is more than a list of paths        |
| `files`        | no       | where a unit's sources are, when not beside the manifest                 |
| `link`         | no       | how the program is put together once every unit is built                 |

A setting the build does not have, a missing required one, or a `parallel` that
is neither `true` nor `false` is an error where it was written.

By default a manifest is a list of paths, one unit per line, relative to the
unit's own directory; blank lines and lines starting with `#` are skipped.
`dependencies` replaces that with a function of the root unit, the unit's
directory and the manifest's text, answering `Result String [String]`. `files`
is a function of a unit's directory answering its sources' paths.

## What the compiler is given

`compile` takes a `Lingua.Make.Given i`, where `i` is the unit's interface
type:

| function        | answers                                                    |
| --------------- | ---------------------------------------------------------- |
| `unitDir u`     | the unit's directory                                       |
| `unitSources u` | its sources, each a path and its text                      |
| `unitImports u` | the interface of each unit it depends on, by directory     |
| `unitOut u`     | a directory of its own to write into, under `target/units` |

It answers `Ok interface` or `Err report`. The interface type is `Reflect`, so
that it can be kept between runs, and it holds data alone: no function, `Ref`
or mutable array.

An interface is held once however many units import it. The build puts it in
a compact region (`Std.Compact`) as soon as its compiler answers; the units
that depend on it read that region where they are, and so does the thread
that writes it to its file. Nothing is copied from thread to thread.

The compiler runs under a handler for `Fs` that lets it read its unit's sources
and its own output directory, and write only in that directory. Anything else
fails with `… is not an input of this unit`, so a read the build did not know
about is an error rather than a stale answer.

## Linking

A language whose program is more than its units' interfaces -- one a back end
that is another program makes, from what every unit wrote -- says how it is put
together: `| link = linkProgram`, a function of a `Lingua.Make.Linking i`
answering `Result String ()`:

| function      | answers                                                                     |
| ------------- | --------------------------------------------------------------------------- |
| `linkRoot l`  | the root unit's directory                                                   |
| `linkUnits l` | every unit in dependency order: its directory, interface and `unitOut`      |
| `linkOut l`   | a directory of the link's own to write into, `target/link` (`linkDir root`) |

The link runs once every unit is built, and the build says `Linked`. It runs
again only if a unit was compiled from something else than it was the last time
the link ran -- its sources, or an interface it was compiled against -- or what
it wrote is gone; otherwise the build says `Fresh link` and what is in
`target/link` is still the program, so what an outside back end made is not
made again. A link that answers `Err report` fails the build, and runs again
the next time.

It runs under a handler for `Fs` that lets it read what any unit wrote and its
own directory, and write only in its own. A unit's compiler writes what the
link needs -- an object file, say -- into `unitOut`.

## When a unit is compiled

A unit is compiled again only if its sources, or an interface it was compiled
against, are not what they were. A change that leaves a unit's interface as it
was compiles that unit and nothing that depends on it.

What was built is kept under `target/`:

- `lingua-build.json` holds fingerprints only: for each unit, of its sources,
  of each interface it was compiled against, and of its own interface. It is
  small, and every build reads it.
- `interfaces/<unit>.json` holds one unit's interface, written on a thread of
  its own after the unit's answer has gone to the units waiting for it: they
  need the interface, not the file. The thread that compiled the unit has
  ended by then, and what it held with it. The build is not done until every
  such file is written, and a unit whose interface could not be written
  counts as failed.

Whether a unit is up to date is decided from fingerprints alone. An interface
is read back only when something needs it: a unit that depends on it and is
being compiled again, or the link step when it reruns. A build in which
nothing was compiled reads no interface and writes nothing. An interface file
that has gone missing makes its unit compile again.

A build kept by a different compiler binary, or by an older layout of these
files, is not trusted: every unit is compiled once more.

## Scheduling

**Parallel is the default.** Each unit runs on a thread of its own and is
started **as soon as the units it depends on are done**. It waits for those and
for no others.

That is finer than building in rounds. Take four units:

```text
a ──▶ b ──▶ d
a ──▶ c
```

In rounds, `d` would wait for the whole second round, `b` and `c`. Here `d`
starts the moment `b` finishes, even if `c` is still compiling. A slow unit
holds up only what needs it.

Each thread reports on a channel when its unit is done, and each report starts
whatever it was the last thing in the way of.

**`| parallel = false`** compiles the units one after another, in an order that
has each after the units it depends on. Such a build has no thread in it.

This is the opposite default from the compiler's own queries, which are
sequential unless the compiler asks for threads (see [drivers.md](drivers.md)).
The reasoning: a compiler may be used with no build system at all, and when
there is one, units are the natural place to divide the work.

Because the build already runs units at once, the usual choice inside a unit
is a plain `batchSession ()`. MiniML's unit compiler is one:

```meadow
fun compiling (u : Given Interface) =
  let sources = unitSources u in
  let files = V.range 1 (V.len sources + 1) in
  let db = batchSession () in
  let a = setSessionImports db 0 (V.concatMap (\i -> exports (snd i)) (unitImports u)) in
  …
```

## Why the build is not a query database

The build has its own graph and does not go through `Lingua.Query`. A query may
perform nothing but `Fetch`, and a compiler inside a unit may want threads, a
cache of its own and files to write. Keeping the two apart lets each have the
default that suits it.

## What it prints

One line per unit, in the order they finish, then a summary:

```text
   Compiled lib
   Fresh util
   Failed app
<the compiler's report>
   Blocked tool: app did not build
miniml: 1 compiled, 1 up to date, 2 failed
```

| line       | meaning                                      |
| ---------- | -------------------------------------------- |
| `Compiled` | the unit was compiled                        |
| `Fresh`    | nothing it was compiled from changed         |
| `Failed`   | `compile` answered `Err`; its report follows |
| `Blocked`  | a unit it depends on did not build           |

`buildBuild` answers `True` when nothing failed or was blocked. A unit that is
not one (no manifest), or units that depend on each other in a circle, stop the
build before anything is compiled, with an `error:` line.
