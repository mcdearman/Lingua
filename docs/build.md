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

| setting        | required | what it is                                                              |
| -------------- | -------- | ----------------------------------------------------------------------- |
| `sources`      | yes      | the ending of a unit's source files, `".ml"`                             |
| `manifest`     | yes      | the file that makes a directory a unit, `"Unit"`                         |
| `compile`      | yes      | the compiler: a function of a `Given i` answering `Result String i`      |
| `parallel`     | no       | `false` to compile one unit after another; a build is parallel otherwise |
| `dependencies` | no       | how to read a manifest, for one that is more than a list of paths        |
| `files`        | no       | where a unit's sources are, when not beside the manifest                 |

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

| function        | answers                                                     |
| --------------- | ----------------------------------------------------------- |
| `unitDir u`     | the unit's directory                                        |
| `unitSources u` | its sources, each a path and its text                       |
| `unitImports u` | the interface of each unit it depends on, by directory      |
| `unitOut u`     | a directory of its own to write into, under `target/units`  |

It answers `Ok interface` or `Err report`. The interface type is `Reflect`, so
that it can be kept between runs.

The compiler runs under a handler for `Fs` that lets it read its unit's sources
and its own output directory, and write only in that directory. Anything else
fails with `… is not an input of this unit`, so a read the build did not know
about is an error rather than a stale answer.

## When a unit is compiled

A unit is compiled again only if its sources, or an interface it was compiled
against, are not what they were. A change that leaves a unit's interface as it
was compiles that unit and nothing that depends on it.

What was built is kept in `target/lingua-build.json`, interfaces included.

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

| line       | meaning                                                  |
| ---------- | -------------------------------------------------------- |
| `Compiled` | the unit was compiled                                    |
| `Fresh`    | nothing it was compiled from changed                     |
| `Failed`   | `compile` answered `Err`; its report follows             |
| `Blocked`  | a unit it depends on did not build                       |

`buildBuild` answers `True` when nothing failed or was blocked. A unit that is
not one (no manifest), or units that depend on each other in a circle, stop the
build before anything is compiled, with an `error:` line.
