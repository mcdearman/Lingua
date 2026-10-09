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
| `options { … }`| no       | what the build can be told; see [Options and profiles](#options-and-profiles) |
| `profile n { … }` | no    | a named set of options                                                   |
| `overrides`    | no       | a unit's own options, read from its manifest                             |
| `pinned`       | no       | a unit's own options that nothing overrides, the command line included   |
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

## Options and profiles

A build may say what it can be told. Each option has a name, a type, what it
is by default and a line of help; a profile is a named set of them.

```meadow
make! {
  pub Units "meadow"
    | sources = ".mw"
    | manifest = "Meadow.toml"
    | compile = compileUnit
    | link = linkProgram
    | options {
        opt : O0 | O1 | O2 | O3 = O1       "how hard to optimise"
        strict : Bool = False              "a match must be exhaustive"
        flags : [String] = []              "what `@cfg` holds of"
        entry : Maybe String = None        "a value run in place of main"
        runtime : Glade | Silo = Glade     "what the program runs on"     for link
        threads : Maybe Int = None         "threads the program runs on"  for nothing
      }
    | profile debug { }
    | profile release { opt = O2, strict = True }
    | overrides = profileOf
}
```

**Types.** `Bool`, `Int`, `String`, `Maybe Int`, `Maybe String`, `[String]`,
or a choice written in place as names with `|` between. A choice becomes a
data type named for the build and the option: `UnitsOpt` with `UnitsOpt.O0`
and the rest.

**Defaults** are ordinary Meadow expressions of the option's type, so one may
come from code: `os : String = hostOs ()`. A choice's name is written bare.

**What an option is for.** Unmarked, compiling depends on it. `for link`
means only the link does; `for nothing` means neither, such as how many
threads the built program runs on.

### What is written

For a build `Units`:

| name                  | what it is                                                     |
| --------------------- | -------------------------------------------------------------- |
| `UnitsOptions`        | a record with a field per option                               |
| `unitsDefaults ()`    | each option at its default                                     |
| `unitsProfiles`       | the profiles' names                                            |
| `unitsProfile name`   | a profile's options, or `None`                                 |
| `unitsSet o name text`| `o` with one option set from a text, or what is wrong          |
| `unitsWith o pairs`   | the same for several `(name, text)` pairs in turn              |
| `unitsOptionHelp`     | each option's name, what it may be, and its help               |
| `buildUnits profile pairs root` | the build                                            |
| `givenUnits profile root dir`   | what a unit was compiled from, as that profile       |

A text sets an option the way a command line or a manifest would give it:
`true`/`false` for a `Bool`, a number, a choice's name, `none` for a `Maybe`,
and texts with commas between for a `[String]`. What is wrong is said:

```text
`opt` is one of O0, O1, O2, O3, not `fast`
meadow has no option `speed`: it has `opt`, `strict`, `flags`, …
```

### What the compiler is handed

With options, `compile` and `link` take them first, as the typed record:

```meadow
fun compileUnit (o : UnitsOptions) (u : Given Interface) : Result String Interface = …
fun linkProgram (o : UnitsOptions) (l : Linking Interface) : Result String () = …
```

### Layers

A unit's options are worked out in this order, each over the one before:

1. the defaults in the declaration;
2. the profile the build was asked for;
3. `overrides`, if the build has one: a function of the root unit, the unit's
   directory, its manifest's text and the profile's name, answering
   `(name, text)` pairs. This is where a language reads its own manifest's
   profile section, and it is per unit: a standard library can be compiled
   the same way under every profile, so that one compile of it serves them
   all;
4. the pairs `buildUnits` was given, which is the command line's place;
5. `pinned`, if the build has one: the same kind of function as `overrides`,
   applied last. What it sets for a unit holds whatever the build was told,
   so a unit that must always be compiled one way is.

A build with options and no `profile` has one, `debug`: the defaults.

### What a change rebuilds

A unit's options are part of what it is compiled from. Its fingerprint takes
in the options compiling depends on, as that unit has them, so:

- changing a compile option recompiles the units whose own value of it
  changed, and through their interfaces whatever depends on them;
- changing a `for link` option reruns the link and compiles nothing;
- changing a `for nothing` option does nothing.

### Where a profile's build is kept

Each profile has a directory of its own, `target/<profile>/`, holding its
`lingua-build.json`, `interfaces/`, `units/` and `link/`. A build as one
profile leaves another's where it was, so going from debug to release and
back compiles nothing the second time. A build with no options keeps
everything directly under `target/`, as before.

### On the command line

`make!` writes no flags of its own. A compiler's `cli!` declares what it
likes, for instance `--profile`, `-O` and a repeated `--set name=value`, and
hands the profile's name and the pairs to `buildUnits`.

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
