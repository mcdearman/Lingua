# Running a compiler: sessions, batches and threads

`database!` declares a compiler's queries once. The same declaration can then
be driven four ways, and which one a program picks is the difference between
an editor's compiler and a command line's.

| made with                        | what it is                                              | for                          |
| -------------------------------- | ------------------------------------------------------- | ---------------------------- |
| `newSession ()`                  | incremental: remembers what each answer read            | an editor, a prompt          |
| `batchSession ()`                | a batch: each key worked out once, nothing else kept    | `check`, `run`, a build unit |
| `parallelSession db`             | `db`, with every `…Each` on threads                     | many independent files       |
| `sharedSession db`               | the same, a batch's threads sharing one copy of inputs  | large inputs, see below      |

The names follow the database's: `database! { pub Session … }` writes
`newSession`, `batchSession`, `parallelSession` and `sharedSession`; a database
called `Project` writes `newProject` and so on.

All four are a `Lingua.Query.Db`, and all four are set and asked with the same
generated functions. Nothing else about the compiler changes with the driver.

```meadow
database! {
  pub Session
  | input source (file : Int) : String
  | query parsed (file : Int) : (Green Mini, [(Int, String)]) = parseMini (querySource file)
  | query typed (file : Int) : Result [Diagnostic] String = …
}

setSessionSource db 1 text      -- an input, set
sessionTyped db 1               -- a query, asked
sessionTypedEach db [1, 2, 3]   -- several, asked at once
```

## A session

`newSession ()` is the engine described in [DESIGN.md §6](DESIGN.md#6-queries-for-an-editor-and-for-a-build):
each answer is kept with what it read, setting an input starts a revision, and
a query asked again is re-run only if something it read changed. A re-run that
answers what it answered before stops there (early cutoff).

Use it wherever the same database outlives an edit: `lsp!` and `repl!` both
make theirs with `newSession ()`.

## A batch

`batchSession ()` runs the same queries with nothing remembered for a next
edit. A key is worked out the first time it is asked for and its answer kept;
that is all. There are no revisions, nothing is recorded about what a query
read, no answer is compared with a previous one, and nothing is a `TVar`.

In effect it is the passes called one after another, in the order they are
needed. On MiniML an incremental session costs a tenth to a seventh more than
that for a single run, which is what a batch saves.

```meadow
-- MiniML's `run`: one file, read once.
fun runFile (path : String) =
  let db = batchSession () in
  match load db [path] with
  | Err why -> (let u = println why in False)
  | Ok files ->
      match sessionResult db 1 with
      | Ok out -> (let u = println out in True)
      | Err ds -> (let u = writeOutput (renderToOutput path (sessionSource db 1) ds) in False)
```

What stays the same in a batch:

- **The data.** The parser still makes the lossless tree and every pass still
  writes its tables and side tables. A batch changes what is remembered between
  queries, not what a query computes.
- **The interface.** `setSessionSource`, `sessionTyped`, `sessionTypedEach` and
  the rest are the functions a session has.
- **Errors.** An input never set, or queries that read each other in a circle,
  raise `QueryError` (`Unset key`, `Cycle keys`) through `Std.Exn`.

What differs:

- Setting an input **forgets every answer**. A batch is for inputs set once,
  before anything is asked.
- `saveSession` on a batch writes nothing worth reading back: a batch records
  no dependencies, and a `persisted` answer is only reusable with them.

## Threads are opt-in

A compiler is sequential until it says otherwise. `sessionTypedEach db files`,
and `queryTypedEach files` inside a query body, ask for several keys at once.
By default they are worked out one after another.

`parallelSession db` answers the same database with each key of an `…Each`
worked out on a thread of its own. It wraps either kind:

```meadow
-- MiniML's `check`: a batch, its files typed on threads.
let db = parallelSession (batchSession ()) in
…
let typed = sessionTypedEach db files in
```

`…Each` is the only place threads come from. A compiler that never calls
`parallelSession` or `sharedSession` has no thread in it, which matters on a
runtime where a program that can spawn pays for that everywhere.

This is the opposite default from [`make!`](build.md), which compiles units in
parallel unless told not to. A compiler may be used with no build system around
it; when there is one, units are where the work divides.

What a thread sees depends on the database underneath:

- **In a batch**, each thread works its key out in a batch of its own, with a
  **copy of the inputs**. Only what was asked for comes back. Two threads that
  both need the same intermediate query each compute it.
- **In a session**, threads share the database. Every slot is a `TVar`
  (`Std.Stm`), so two threads asking for one key compute it once and the second
  waits for the first. A wait that would close a loop between threads is a
  `Cycle`, not a deadlock.

Inputs are set between asks, not during one.

## Sharing the inputs instead of copying them

`sharedSession db` is `parallelSession db` with one difference for a batch: the
threads read the inputs from a single compact region (`Std.Compact`) instead of
taking a copy each.

Copying is the default because of where Lingua is meant to run. On Silo, which
counts references, every read of a string or record out of a shared region
writes a count that all the threads share, and they wait on each other for it.
A copy per thread is never very slow; a region of sources read from eight
threads was.

Reach for `sharedSession` only when the inputs are large enough that copying
them dominates, and measure on the runtime you ship on. Two restrictions come
with it:

- no input may hold a function, a `Ref` or a mutable array, which a compact
  region refuses;
- on a session that is not a batch it is the same as `parallelSession`, since
  a session already shares what it holds.

## Letting go of a database

A database is a value, and what it holds goes when nothing refers to it any
more. `closeSession db` says so sooner: it lets go of every input and every
answer at the point a compiler is done with them. The database is then as it
was made, and may be set and asked again.

Call it when a database's answers are large and the program goes on after:
a unit's compiler under `make!`, for instance, once it has its interface.
On Silo it also makes certain of what would otherwise wait on the runtime
noticing that a thread which has ended refers to nothing.

## Choosing

| the program                                   | use                                     |
| --------------------------------------------- | --------------------------------------- |
| a language server, a prompt                   | `newSession ()`                         |
| compiles its inputs once and exits            | `batchSession ()`                       |
| the same, over many files that do not depend on each other | `parallelSession (batchSession ())` |
| a unit's compiler under `make!`               | `batchSession ()`: the build already runs units at once |

MiniML does exactly this: its editor and prompt keep a session, `check` is a
parallel batch, and `run` and its unit compiler are plain batches
([Lib.mw](../examples/MiniML/src/Lib.mw), [Units.mw](../examples/MiniML/src/Units.mw)).

## By hand

`database!` is a layer over `Lingua.Query`, which can be used directly:

| function                 | what it does                                          |
| ------------------------ | ----------------------------------------------------- |
| `newDb compute`          | an incremental database over `compute : k -> Maybe v` |
| `newBatch compute`       | a batch over the same                                 |
| `parallel db`, `shared db` | the two thread-taking wrappers                      |
| `set db key value`       | set an input                                          |
| `get db key`, `getAll db keys` | ask, raising `QueryError`                       |
| `takeLog db`             | the keys computed since the log was last taken        |

`takeLog` is how the tests say what an edit cost: set an input, ask, and compare
the log with the keys that should have re-run.
