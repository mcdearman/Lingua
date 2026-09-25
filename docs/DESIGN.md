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

Stmt = LetStmt | ExprStmt
LetStmt  = 'let' name:Name '=' value:Expr ';'
ExprStmt = Expr ';'

Expr = Literal | NameRef | ParenExpr | BinExpr | CallExpr
Literal   = 'int_number'
NameRef   = 'ident'
ParenExpr = '(' Expr ')'
BinExpr   = lhs:Expr op:('+' | '-' | '*' | '/') rhs:Expr
CallExpr  = callee:Expr ArgList
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
- **How left recursion reads.** `BinExpr = lhs:Expr op:(…) rhs:Expr` and
  `CallExpr = callee:Expr ArgList` start with the enum they belong to. A
  `precedence` table gives each binary operator its binding power and side,
  and names the postfix forms; those alternatives are parsed as a Pratt loop
  instead of by descent. A left-recursive alternative without an entry is an
  error, reported on its rule.

```meadow
syntax! {
  pub Calc                            -- `pub`: what it writes is exported
  lexer Token                         -- the Scythe `@derive(Lexer)` type
  trivia { Whitespace, Comment }      -- its kinds the parser looks past
  tokens { "ident" = Ident, "int_number" = Number }
  precedence Expr {
    left "+" "-"
    left "*" "/"
    postfix CallExpr
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
  power, `close` as `BinExpr`; a postfix form the same, without the recursion;
- a **token** is `expect`; a `?` is an `if at FIRST`; a `*` is a loop while at
  FIRST.

**Recovery** is by recovery sets, computed rather than written: a loop over
`X*` inside rule `R` gives up — breaking out without consuming — on a token in
FOLLOW(`R`) or in any enclosing loop's set, and skips anything else as an
error node. So `let x = ;` is one `LetStmt` with a missing `Expr` and an error,
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
data LetStmt = LetStmt (Syntax CalcKind)
asLetStmt : Syntax CalcKind -> Maybe LetStmt
syntaxLetStmt : LetStmt -> Syntax CalcKind

data Expr = Literal Literal | NameRef NameRef | BinExpr BinExpr | …
asExpr : Syntax CalcKind -> Maybe Expr
```

Accessors come from the rule's elements, named by their labels — ungrammar's
contract — or, unlabelled, by what the child is: `name` for a `Name`, `names`
for a `Name*`, `letToken` for a `'let'`. Meadow has no methods to hang them
on, so each is prefixed with its rule:

| in `LetStmt`, `BinExpr`, `ArgList` | accessor                                           |
| ---------------------------------- | -------------------------------------------------- |
| `name:Name`                        | `letStmtName : LetStmt -> Maybe Name`              |
| `value:Expr`                       | `letStmtValue : LetStmt -> Maybe Expr`             |
| `'let'`                            | `letStmtLetToken : LetStmt -> Maybe Syntax`        |
| `lhs:Expr` … `rhs:Expr`            | `binExprLhs`, `binExprRhs : BinExpr -> Maybe Expr` |
| `op:('+' \| …)`                    | `binExprOp : BinExpr -> Maybe Syntax`              |
| `args:(Expr (',' Expr)*)?`         | `argListArgs : ArgList -> [Expr]`                  |

Every accessor is a `Maybe` or a list: the tree is lossless, so it holds
whatever was written, including what is missing. Two single children of one
type — `lhs` and `rhs` — are told apart by position, the first and the second;
two that would get the same name are an error in the grammar, asking for
labels. `astCalc` casts a parse's root.

## 5. `lang` and `pass`

A language is a set of **sorts** — `Expr`, `Stmt` — each a choice of
**productions** — `BinExpr`, `Literal` — each a record of **fields**. They
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
  Expr - BinExpr
  Expr + Prim { op : String, args : [Expr] }
}
```

`Sort - Prod` removes a production; `Sort + Prod { field : Type, … }` adds one
(to a new sort, if there is none by that name). A field's type is a sort of
the language, `[T]`, `Maybe T`, or any other type by name. Each language is
written out as ordinary `data`, a type per sort named with the language —
`CoreExpr`, `CoreStmt` — whose productions each hold an anonymous record: the
fields, and a `meta : Meta` nobody writes, the bytes of the source the node
came from. `metaCoreExpr` reads it. A language read off a grammar also gets its
conversion from the typed AST, `surfaceFromCalc : Green CalcKind -> Maybe
SurfaceFile` — `None` when the tree has something missing in it, since only an
error-free tree is a program.

A **`pass!`** is a function from one language to another, written as the
cases that change something:

```meadow
pass! {
  pub lower : Surface -> Core
  | BinExpr { lhs, op, rhs } -> Prim { op = op, args = [lhs, rhs] }
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

## 6. Queries: incremental and parallel

Every step above is a function of what came before it: a file's text gives its
tokens, its tree, its surface language; each pass's output is a function of
its input. Lingua runs them as **queries**, in the style of
[salsa](https://github.com/salsa-rs/salsa): each memoized on its key, each
recording what it read, re-run only when something it read has changed.

- **Inputs** are set from outside: a file's text, by file id. Setting one
  bumps a revision.
- **Derived queries** are functions keyed by a value — a file, an item. A
  query reads other queries through an effect, `fetch`, whose handler records
  the dependency; a result remembers the revision it was computed at and what
  it read.
- A query asked again **validates** before re-running: if nothing it read has
  changed since, the old result stands. If it re-runs and its result equals the
  old one — cheap for green trees and generated languages, which are plain
  values — the queries that read it are still valid: **early cutoff**, so an
  edit inside a function body stops at that function.
- **Parallel**: keys that do not depend on each other run on separate threads
  (`Std.Thread`); the memo table is shared through `Std.Stm`, so two threads
  asking for the same key compute it once.

What is generated: `syntax!` makes the parse of a file a query; each `pass` a
query keyed by the item it runs on, so a pass over one function is cached per
function. A driver sets the inputs and asks for the last query's result.

Persisting the memo table between runs — an incremental build rather than an
incremental session — is last: results that are `Reflect` can be written as
`Datum`s and read back, keyed by the fingerprints of what they read.

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
   cutoff; `syntax!` and `pass` generating theirs.
5. **Parallel and persistent**: threads over independent keys, and the memo
   table written between runs.

## Open questions

- **Where a `pass` may run.** Per item needs the language to say what an item
  is; the first cut runs a pass per file.
- **Name resolution** between passes is not a pass's shape and wants a query of
  its own; the calculator will not exercise it, and the bootstrap will.
