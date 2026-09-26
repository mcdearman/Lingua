# Lingua

Lingua is a library for writing compilers in Meadow as a specification: the
syntax as a grammar, the intermediate languages as declarations, and the passes
between them as the transformations that actually change something. Spans,
provenance, traversal of what a pass does not touch, caching and scheduling are
generated plumbing.

```
text ─Scythe─▶ tokens ─generated LL parser─▶ events ─▶ green tree (lossless)
                                                         │ red view (positions, parents)
                                                         ▼ typed AST (from ungrammar labels)
                                   lang L0 ─pass─▶ lang L1 ─pass─▶ … ─▶ output
                         every step a cached query: incremental and parallel
```

This document is the design. It is written ahead of the code, and each part
says what milestone builds it (see [Milestones](#milestones)).

## 1. One grammar: ungrammar

The syntax of a language is written once, in
[ungrammar](https://rust-analyzer.github.io/blog/2020/10/24/introducing-ungrammar.html)
— rust-analyzer's notation for the _shape_ of a concrete syntax tree:

```
File = Stmt*

Stmt = Let | ExprStmt
Let      = 'let' name:Name '=' value:Expr ';'
ExprStmt = Expr ';'

Expr = Literal | NameRef | Paren | Bin | Call
Literal = 'int_number'
NameRef = 'ident'
Paren   = '(' Expr ')'
Bin     = lhs:Expr op:('+' | '-' | '*' | '/') rhs:Expr
Call    = callee:Expr ArgList
ArgList   = '(' args:(Expr (',' Expr)*)? ')'
Name = 'ident'
```

Rules are `UpperCamel = rule`; `'…'` is a token; juxtaposition is a sequence,
`|` alternation, `*` and `?` repetition, `( )` grouping, `label:` names a child,
and `//` comments. A rule whose body is only an alternation of other rules
(`Stmt`, `Expr`) is an _enum_: it has no node of its own, and each alternative
is. Every other rule is a _node_.

Ungrammar says what trees look like, not how to read text into them — and
rust-analyzer pairs it with a parser written by hand. Lingua generates the
parser from the same grammar, which needs two things ungrammar leaves out,
written beside it in the same declaration:

- **What each token is.** `'ident'` and `'int_number'` are names for token
  kinds of the Scythe lexer; `'let'`, `'+'` are its fixed tokens. A `tokens`
  table maps the quoted names to the lexer's constructors.
- **How left recursion reads.** `Bin = lhs:Expr op:(…) rhs:Expr` and
  `Call = callee:Expr ArgList` start with the enum they belong to. A
  `precedence` table gives each binary operator its binding power and side,
  and names the postfix forms; those alternatives are parsed as a Pratt loop
  instead of by descent. A left-recursive alternative without an entry is an
  error, reported on its rule.
- **Application by juxtaposition** -- `f x y` -- is a postfix form whose
  argument is an atom: `App = func:Expr arg:Atom`, with `Atom` an enum of
  its own among `Expr`'s alternatives. The Pratt loop wraps the left side
  whenever an atom can start next, so `f x y` is `(f x) y` and `f x + 1` is
  `(f x) + 1`, as Meadow's own parser reads them (`examples/MiniML`).

```meadow
syntax! {
  pub Calc                            -- `pub`: what it writes is exported
  lexer Token                         -- the Scythe `@derive(Lexer)` type
  trivia { Whitespace, Comment }      -- its kinds the parser looks past
  tokens { "ident" = Ident, "int_number" = Number }
  precedence Expr {
    left "+" "-"
    left "*" "/"
    postfix Call
  }
  grammar r#"
    File = Stmt*
    // … the ungrammar above, exactly as written
  "#
}
```

The grammar is ungrammar verbatim, in a raw string. It cannot be written as the
macro's own tokens: those are lexed as Meadow, where `'let'` is a malformed
character and `//` an operator. A raw string holds it untouched — and later can
come from a `.ungram` file — and Lingua reads it with an ungrammar lexer of its
own. A raw string has no escapes, so a byte inside it is at a known distance
from the literal's `Loc`, and a mistake in the grammar is still reported at
the exact place it was written. The tables around it are Meadow tokens, read
with `Std.Macro.Parse`.

The declaration is a procedural macro. It reads the grammar and the tables,
checks them (below), writes the code of §2–§4, and leaves the grammar as a
compile-time binding, a `Datum`, for `lang` and `pass` to read (§5).

**Checks**, each reported at the rule it is about:

- every rule a rule mentions is defined, and every quoted token is in the
  lexer or the `tokens` table;
- the grammar is LL(1) once the precedence table has taken the left
  recursion out: the FIRST sets of an enum's alternatives are disjoint, and a
  `*` or `?` does not start with what may follow it. A conflict names the two
  alternatives and the token they share;
- no left recursion is left over.

## 2. Parsing: events

The generated parser follows matklad's
[resilient LL parsing](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html).
It never builds a tree. It reads the lexer's tokens — trivia filtered out —
and emits a flat list of events:

```meadow
data Event = Open Kind | Close | Advance | Error String
```

over a small runtime, written once in Lingua:

| operation                      | what it does                                                                        |
| ------------------------------ | ----------------------------------------------------------------------------------- |
| `open`                         | push a placeholder `Open`; answer a mark                                            |
| `close m k`                    | fill the placeholder with kind `k`; push `Close`; answer a closed mark              |
| `openBefore m`                 | a new `Open` before an already-closed node — how `a + b` wraps `a` after reading it |
| `at k`, `atAny ks`, `nth n`    | look without consuming; past the end is `Eof`                                       |
| `advance`, `eat k`, `expect k` | consume; `expect` records an error instead when it is not there                     |
| `advanceWithError msg`         | consume one token into an `Error` node                                              |

`nth` burns **fuel** and `advance` refills it, so a generated loop that stops
making progress fails loudly instead of hanging.

From the grammar:

- a **node** rule is a function: `open`, its elements in order, `close` with
  the rule's kind;
- an **enum** is a dispatch on the next token over its alternatives' FIRST
  sets, then — for an enum with a precedence table — the Pratt loop:
  `openBefore` the left side, read the operator, recurse with its binding
  power, `close` as `Bin`; a postfix form the same, without the recursion;
- a **token** is `expect`; a `?` is an `if at FIRST`; a `*` is a loop while at
  FIRST.

**Recovery** is by recovery sets, computed rather than written: a loop over
`X*` inside rule `R` gives up — breaking out without consuming — on a token in
FOLLOW(`R`) or in any enclosing loop's set, and skips anything else as an
error node. So `let x = ;` is one `Let` with a missing `Expr` and an error,
and the next statement still parses. Every token ends up somewhere in the
tree; nothing is dropped.

## 3. Trees: green and red

The events and the tokens are built into a **green tree**, the lossless one:

```meadow
data Green = Node Kind Int [Green]     -- kind, width in bytes, children
           | Token Kind String         -- kind and exact text
```

It has no positions, only widths, so a subtree is a value that can be shared
between versions of a file and compared by value — what makes a reparse cheap
and a query's early cutoff (§6) possible.

**Trivia** — whitespace and comments — are tokens like any other: the Scythe
lexer declares them as variants and leaves off `@skip`, so they are lexed
rather than dropped. The `trivia` line of `syntax!` names them. The parser
never sees them — it reads only the other tokens — and the tree builder puts
them back where they were: trivia before a node goes to its parent, before the
node opens, as rust-analyzer attaches it; trivia inside goes where it was.
Concatenating the tokens of a green tree gives back the source, byte for byte;
a test says so for every example. A lexer error is kept too: the text Scythe
stopped on becomes an error token, and the tree still covers every byte.

The **red view** is where positions and parents live: a cursor over the green
tree with its absolute offset and the path to the root, made on demand and
never stored.

```meadow
record Syntax = { green : Green, offset : Int, parent : Maybe Syntax }
kind, text, range, children, parent, ancestors, tokenAt, nodeAt
```

## 4. Typed AST, from the labels

Each **node** rule becomes a type wrapping a red node, and each **enum** a sum
of its alternatives, with a checked cast from `Syntax` and the way back:

```meadow
data CalcLet = CalcLet (Syntax Calc)
asCalcLet : Syntax Calc -> Maybe CalcLet
syntaxCalcLet : CalcLet -> Syntax Calc

data CalcExpr = Literal CalcLiteral | NameRef CalcNameRef | Bin CalcBin | …
asCalcExpr : Syntax Calc -> Maybe CalcExpr
```

**Names.** Everything `syntax! { Calc … }` makes is named after `Calc`, so a
rule may be called anything -- `Let`, as the lexer's token is, or `Int`, as a
built-in type is -- and meet nothing of the program's, or of another grammar's:

| what                  | named                                                                                     |
| --------------------- | ----------------------------------------------------------------------------------------- |
| the kinds             | `Calc`: `Calc.NodeLet`, `Calc.TokenLet`, and `Calc.ErrorNode`, `Calc.Eof`, `Calc.Unknown` |
| a rule's type         | `CalcLet`, `CalcExpr`                                                                     |
| its cast and way back | `asCalcLet`, `syntaxCalcLet`                                                              |
| an accessor           | `calcLetName`                                                                             |
| the entry points      | `parseCalc`, `astCalc`                                                                    |
| the parser's own      | `linguaCalc_<x>`, `x` lower case where a rule's is upper                                  |

A node rule and a token of one name are two kinds, and none of them is one of
the three a tree needs. A kind displays as its rule's or token's own name --
`Let` -- which is what a dump of the tree and an editor's colours read. The
other macros name what they make the same way, after what they declare:
`CoreExpr`, `metaCoreExpr`; `SessionKey`, `newSession`, `sessionTree`,
`linguaSession_<x>`; `askCli`; `serveServer`; `buildBuild`.

Accessors come from the rule's elements, named by their labels — ungrammar's
contract — or, unlabelled, by what the child is: `name` for a `Name`, `names`
for a `Name*`, `letToken` for a `'let'`. Meadow has no methods to hang them
on, so each is prefixed with its rule's type:

| in `Let`, `Bin`, `ArgList` | accessor                                               |
| -------------------------- | ------------------------------------------------------ |
| `name:Name`                | `calcLetName : CalcLet -> Maybe CalcName`              |
| `value:Expr`               | `calcLetValue : CalcLet -> Maybe CalcExpr`             |
| `'let'`                    | `calcLetLetToken : CalcLet -> Maybe Syntax`            |
| `lhs:Expr` … `rhs:Expr`    | `calcBinLhs`, `calcBinRhs : CalcBin -> Maybe CalcExpr` |
| `op:('+' \| …)`            | `calcBinOp : CalcBin -> Maybe Syntax`                  |
| `args:(Expr (',' Expr)*)?` | `calcArgListArgs : CalcArgList -> [CalcExpr]`          |

Every accessor is a `Maybe` or a list: the tree is lossless, so it holds
whatever was written, including what is missing. Two single children of one
type — `lhs` and `rhs` — are told apart by position, the first and the second;
two that would get the same name are an error in the grammar, asking for
labels. Two whose types overlap — `func:Expr arg:Atom`, where every `Atom` is
an `Expr` too — are told apart by position among all the node's children,
since counting `Atom`s would find `func` again whenever it is one.
`astCalc` casts a parse's root.

## 5. `lang` and `pass`

A language is a set of **sorts** — `Expr`, `Stmt` — each a choice of
**productions** — `Bin`, `Literal` — each a record of **fields**. They
cross between the macros as `Lingua.Lang` values, stored as compile-time
bindings.

`syntax!` leaves the language its grammar implies, under the grammar's name.
An enum rule is a sort whose productions are its alternatives; any other node
rule is a sort of one production. A production's fields are its typed-AST
slots, less the tokens that say nothing: a token is kept when it is labelled,
or carries text of its own (`Ident _`); `'let'` and `';'` are dropped. A child
under `?` is a `Maybe`, under `*` a list.

A **`lang!`** declares an intermediate language. The first is read off the
grammar; the rest are changes to one before them:

```meadow
lang! { pub Surface from Calc }

lang! {
  pub Core extends Surface
  Expr - Bin
  Expr + Prim { op : String, args : [Expr] }
}
```

`Sort - Prod` removes a production; `Sort + Prod { field : Type, … }` adds one
(to a new sort, if there is none by that name); `Sort * { field : Type, … }`
adds the fields to every production the sort has -- what an elaboration's
target wants, `Expr * { ty : Type }` being `Core` with a type on every
expression. A field every production of a sort has gets an accessor like
`meta`'s, `tyTypedExpr`. A language may take **type parameters** -- `lang! {
pub Inferring s extends Core … }` -- which its types take in turn, and a field
may be of a type applied to them, `MType s`; a language extending it takes
them too. A field's type is a sort of
the language, `[T]`, `Maybe T`, or any other type by name. Each language is
written out as ordinary `data`, a type per sort named with the language —
`CoreExpr`, `CoreStmt` — whose productions each hold an anonymous record: the
fields, and a `meta : Meta` nobody writes, the bytes of the source the node
came from. `metaCoreExpr` reads it. A language read off a grammar also gets its
conversion from the typed AST, `surfaceFromCalc : Green Calc -> Maybe
SurfaceFile` — `None` when the tree has something missing in it, since only an
error-free tree is a program.

A **`pass!`** is a function from one language to another, written as the
cases that change something:

```meadow
pass! {
  pub lower : Surface -> Core
  | Bin { lhs, op, rhs } -> Prim { op = op, args = [lhs, rhs] }
}
```

It reads both languages and writes the rest: a function per sort —
`lowerExpr`, `lowerStmt` — the children of every node translated before its
case sees them, and a copy of every production the target kept as it was. A
case names the fields it wants, bound already translated. In its body, a
production of the target written with a record — `Prim { … }` — is that
language's and holds the `meta` of the node the case replaces.

What cannot be written is said where it is: a case for a production the source
does not have, at the case; a field it does not have, at the field; a
production the target dropped or changed with no case, at the pass.

A case's body is **ordinary Meadow**, and may call any function — effects and
all. The functions a pass writes are left to inference, so a pass performs what
its cases do: one that counts with `Std.State` is run under `runState`. A
`match` in a body is parenthesised, since `|` begins the next case.

A pass may carry a **context** down the tree -- an inherited attribute, in the
old words -- and a case may take a child **`later`**:

```meadow
pass! {
  pub elaborate : Core -> Typed with env
  | Lam { param, later body } ->
      (let a = fresh () in
        let b = body ((param, Forall [] a) :: env) in
        Lam { param = param, body = b, ty = TFun a (tyTypedExpr b) })
  …
}
```

Every function the pass writes then takes `env` first and hands it to the
children it translates. A field taken `later` is not translated before the
case: it is a function from a context to the translated child, for the case to
call with the one the child is in -- under a binder, the environment with the
binder in it; a `let`'s value, one level deeper. This is what makes type
inference a pass rather than a function beside one. Without a context, a
`later` field is a function of `()`.

MiniML's inference is **Algorithm J** as nine cases, and shows what a case
body being ordinary Meadow buys. Its type variables are mutable cells --
`Std.St`'s -- so its cases perform `St s`, and fail with `Std.Exn`; `infer`
goes from `Core` to `Inferring s`, whose nodes hold types made of those cells,
and `settle` from there to `Typed`, reading each. Both run inside one
`runSt`, which takes the `St s` away: `typed` is pure, and runs inside a
query like anything else, though every step of it writes.

## 6. Queries: incremental and parallel

Every step above is a function of what came before it: a file's text gives its
tokens, its tree, its surface language; each pass's output is a function of
its input. Lingua runs them as **queries**, in the style of
[salsa](https://github.com/salsa-rs/salsa): each memoized on its key, each
recording what it read, re-run only when something it read has changed.

```meadow
database! {
  pub Session
  | input source (file : Int) : String
  | query tree (file : Int) : Green Calc = fst (parseCalc (querySource file))
  | query program (file : Int) : Maybe CoreFile = M.map lower (surfaceFromCalc (queryTree file))
  | query bindings (file : Int) : [(String, Int)] = evaluate (queryProgram file)
  | query sum (files : [Int]) : Int = V.foldl (\acc f -> acc + total (queryBindings f)) 0 files
}
```

- **Inputs** are set from outside: `setSessionSource db 1 text`. Setting one
  to something new starts a revision; setting it to what it holds does not.
- **Derived queries** are functions of a key -- the parameter -- whose bodies
  are ordinary Meadow, and read other queries by calling them, each as
  `query` and its name -- `queryTree file` -- so that a query never takes a
  name an ordinary function has.
  Each call performs `fetch` (the `Fetch` effect), and the handler that runs a
  query answers it and notes what was read.
- A query asked again **validates** before re-running: if it was checked this
  revision, its answer stands; otherwise each thing it read is brought up to
  date first, and if none of them changed since it was last checked, its answer
  still stands. If it does re-run and answers what it answered before -- equal
  as values, structurally, so any type will do -- it has not changed: **early
  cutoff**, and what read it is still right. A comment added to a file changes
  its tree, and the program whose `meta` covers the comment, but not what the
  file binds; the sum over the files is not run again.
- A query may perform no effect but `fetch`: it is a function of its key and
  of what it read, which is what makes its answer reusable. The type says so,
  and a body that prints does not compile.
- What can go wrong is a value, `QueryError`, raised through `Std.Exn` by
  `sessionTree db file` and the others that ask from outside: an input never
  set, or queries that read each other in a circle, named key by key. The
  database is usable after either.

**What is generated.** A database is `Lingua.Query.Db k v`: one type of key
and one of value, each a choice with a case per query -- `SessionKey`,
`SessionValue` -- since Meadow has no dynamic type to hold a table of
anything. `database!` writes both, a function per query for bodies to call,
the dispatch from a key to the body that computes it, `newSession`, a setter
per input and an asker per query. `Lingua.Query` is the engine underneath --
revisions, validation, cutoff, cycles, and a log of what ran, which is how the
tests say what an edit cost -- and can be used by hand the same way.

This is where the design first said that `syntax!` and each `pass` would make
queries of their own. They do not, and need not: the key and value types are
closed, so the queries of a database are declared in one place, and a query's
body is a call to what `syntax!` and `pass!` already wrote -- `parseCalc`,
`lower` -- one line each. A pass over a whole program is a query per file;
a pass per item waits on the open question below.

**Parallel** (milestone 5). `queryTreeEach files` -- the `fetchAll`
operation -- asks for several keys at once, each on a thread of its own
(`Std.Thread`), and `sessionTreeEach db files` does the same from outside.
Threads share nothing mutable, so the database is `TVar`s (`Std.Stm`): its
revision, its inputs, a slot per derived key, and a log. A slot is worked
out, _running_ -- claimed by one piece of work, a _chain_, so that a second
thread asking for the key waits for the first rather than computing it again
-- or read back from a snapshot and not yet checked. A cycle within a chain is
the key it is already computing; a cycle across threads is a wait that would
close a loop in the graph of which chains wait for which, and each is a
`Cycle` rather than a deadlock. A chain that fails gives back the keys it
claimed, so whoever waited sees why for themselves. The engine's own steps
answer a `Result` rather than raising, and `get` raises at the edge.

**Persistent** (milestone 5). A query marked `persisted` is kept between runs:
`saveSession db path` writes each of its answers checked this revision, with
every input it read -- through every query in between -- and a fingerprint of
each, the structural `hash` of its value, which is the same in every run and
on every machine. `loadSession db path`, after the inputs are set, reads them
back as unchecked slots: one stands, without running anything, if every
input it read has the fingerprint it had; one that does not is run again,
and what it answered before still cuts off what reads it. Keys and answers
cross as `Datum`s in JSON, so a persisted query's key and value, and the
inputs' keys, are `Reflect`.

## 7. Diagnostics

What a compiler says about a program is a value, `Lingua.Diagnostic`: how
serious, what it says, the `Meta` it is about -- the bytes every node carries
-- and further labelled spans and notes. It is drawn by
[Nettle](https://github.com/mcdearman/Nettle), the port of ariadne, only when
someone asks: `renderAll path text diagnostics`.

- **The parser**'s errors are offsets and messages; `parseErrors` makes each
  a diagnostic at the token found there, which is the one that was wrong or
  that something was missing before.
- **A pass** reports with `report` (the `Report` effect), and a `pass!` case
  names the node it is rewriting as `here`. Whoever runs the pass handles the
  effect; `collect` keeps what was said. A query may perform nothing but
  `fetch`, so a query that runs a pass collects inside its body and answers
  the diagnostics as part of its value -- which also means they are
  remembered, and cut off, like any other answer.
- **Anything else** -- a checker, an evaluator -- makes diagnostics the same
  way, from the `meta` of the node it is looking at.

## 8. Tooling: the command line, the editor, the build

A compiler is also the program that runs it, the server an editor talks to,
and the build that decides what to compile. Each is written from the same
queries the compiler already is, by a macro of its own.

**`cli!`** declares the command line: each command, with the fields it takes
-- positional, `Maybe` for one that may be left out, `[T]` for the rest,
`Flag` and `Option T` for flags -- and what help says of each. It writes a
constructor per command, holding what it was given, and `argsX ()`: the
command the process was started with. `--help`, `help` and `help <command>`
print help, `completions bash` a completion script, and a mistake is a
diagnostic like any other -- drawn by Nettle against the command line itself,
under the argument that is wrong.

**`lsp!`** declares a language server over the compiler's database: `text` is
how the editor's text becomes an input, and each other feature a function of
the database, a file and the byte the editor points at -- `diagnostics`,
`hover`, `definition`, `references` (which gives highlights and rename too),
`completion`, `symbols`, and `tree`, the lossless tree, from which semantic
tokens, folds and selection ranges are read here, the same for every
language. An edit sets an input and nothing more: what it changed is all
that is worked out again, so hovering after an edit reads an elaboration that
was only redone for the file that changed. The protocol is Lingua's -- its
framing, which needs Meadow's `Console.readExact`, its UTF-16 positions, the
documents open -- and the server says it can do exactly what it was given.

**Names** are resolved as a pass, for the same reason inference is: which `x`
a use means depends on what is around it. A pass may carry a **context**
(§5) -- an environment -- and `Resolved extends Surface` gives each name that
is used the place its binder was written. What a file binds, where each can
be seen, and every use are a query of their own, which going to a
definition, finding references, renaming and completion all read.

**`make!`** declares a build over **compilation units**: directories with a
manifest naming the units they depend on. Two kinds of incrementality meet
at a unit, and they belong to different owners:

- **Between units**, the build system's. A unit is compiled again only if
  its sources, or an interface it was compiled against, are not what they
  were; a change that leaves a unit's interface as it was compiles that unit
  and nothing that depends on it. Units that do not depend on each other are
  compiled at once, on threads of their own. What was built is kept in
  `target/lingua-build.json`, interfaces and all, which are `Reflect` so that
  they can be.
- **Inside a unit**, the compiler's. `compile` is given the unit's sources,
  the interfaces it depends on, and a directory of its own; how it does the
  work -- its own queries, its own threads, its own cache in that directory --
  is its business. MiniML's keeps a persisted session there, so a unit built
  before is read back and its files not typed again.

The compiler is held to that boundary by an effect handler: it runs under a
handler for `Fs` through which it can read its unit's sources and its own
directory and write only in the latter, so a read the build system did not
know about is an error, not a stale answer. The build's own graph is not
`Lingua.Query`'s: a query may perform only `Fetch`, and a compiler inside a
unit may want threads and a cache of its own.

## Milestones

1. **Trees and parsing.** The green tree and the red view; the event runtime
   of §2; the tree builder with trivia restored; `syntax!` reading ungrammar
   and the two tables, the checks of §1, and the generated parser. Tested on a
   small calculator grammar: every input — right, wrong and empty — gives a tree
   whose text is the input, and errors land where they belong.
2. **Typed AST** from the labels (§4).
3. **`lang` and `pass`** (§5), on the calculator: `Surface` from the grammar, a
   `Core` without binary expressions, a pass between them.
4. **Queries** (§6), in a session: inputs, derived queries, validation, early
   cutoff, cycles; `database!` declaring them. Done: `Lingua.Query`,
   `Lingua.Database`, and the calculator as a database of files.
5. **Parallel and persistent**: threads over independent keys, and the memo
   table written between runs. Done: `fetchAll`, `persisted`,
   `saveSession`/`loadSession`, both examples using them.
6. **Tooling** (§8), on MiniML: `cli!`, `lsp!`, name resolution as a pass
   with a context, and `make!` over compilation units. Done, on every Meadow
   runtime; the next language to get them is Meadow itself.

## Open questions

- **Where a `pass` may run.** Per item needs the language to say what an item
  is; the first cut runs a pass per file.
- **Names across units.** MiniML resolves within a file and imports a unit's
  interface whole; Meadow's modules, `use` and visibility will need the
  interface to say what it exports and the resolution to read it.
- **Operators whose precedence a program declares**, as Meadow's `infixl`
  does: parsed flat, and put in order by a pass once the declarations are
  known.
- **A build's units run by another compiler.** `compile` is a Meadow
  function; for the bootstrap it should also be able to be a process -- the
  compiler it replaces -- so that units compiled by each can meet in one
  graph.
