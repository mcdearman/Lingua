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
             every step a query: remembered for an editor, run once for a build
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
Let      = "let" name:Name "=" value:Expr ";"
ExprStmt = Expr ";"

Expr = Literal | NameRef | Paren | Bin | Call
Literal = "int_number"
NameRef = "ident"
Paren   = "(" Expr ")"
Bin     = lhs:Expr op:("+" | "-" | "*" | "/") rhs:Expr
Call    = callee:Expr ArgList
ArgList   = "(" args:(Expr ("," Expr)*)? ")"
Name = "ident"
```

Rules are `UpperCamel = rule`; `"…"` is a token; juxtaposition is a sequence,
`|` alternation, `*` and `?` repetition, `( )` grouping, `label:` names a child.
(Ungrammar itself quotes tokens `'…'`; in `syntax!` they are Meadow's string
literals.) A rule whose body is only an alternation of other rules
(`Stmt`, `Expr`) is an _enum_: it has no node of its own, and each alternative
is. Every other rule is a _node_.

Ungrammar says what trees look like, not how to read text into them — and
rust-analyzer pairs it with a parser written by hand. Lingua generates the
parser from the same grammar, which needs two things ungrammar leaves out,
written beside it in the same declaration:

- **What each token is.** `"ident"` and `"int_number"` are names for token
  kinds of the Scythe lexer; `"let"`, `"+"` are its fixed tokens. A `tokens`
  table maps the quoted names to the lexer's constructors -- its own token
  type, with no second one between it and the parser. An entry that says a
  token's text, `"self" as SelfWord = Ident "self"`, is a kind of its own,
  tried before the constructor's.
- **Where a token is.** Some kinds a token has only where it is: a word that
  is a keyword before a capital and a name anywhere else, or the names on a
  path `a.b.c!` that a `!` at its end makes a macro's -- which a parser
  looking a fixed number of tokens ahead cannot see. A `context` section says
  them, each a pattern and where the token has to be:

  ```meadow
  context {
    "mac_lower" = LowerIdent _ ending Bang via Period,
    "pattern" = LowerIdent "pattern" before UpperIdent _,
    "rec" = LowerIdent "rec" after Let before LowerIdent _
  }
  ```

  `before` and `after` test the neighbouring token that is not trivia;
  `ending … via …` a path of segments, each of a kind some entry with the same
  ending names, each pair joined by the separator. The first entry that holds
  gives the kind, and the tokens are read whole before any is kinded.

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
  grammar {
    File = Stmt*
    -- … the grammar above
  }
}
```

The grammar is Meadow tokens, as the tables around it are: a rule name is a
word, a token a string literal, a comment Meadow's own. Every token keeps the
place it was written, so a mistake is reported at the token it is about --
`no rule is called Nmae`, under `Nmae` -- and the code written from the
grammar is written _there_, too:

- a rule's name is where its type and cast are, so hovering `Let` shows
  `asCalcLet : Syntax Calc -> Maybe CalcLet`, and going to the definition of
  `CalcLet` anywhere in the program arrives at the rule;
- a label is where its accessor is: hovering `name:` shows
  `calcLetName : CalcLet -> Maybe CalcName`;
- a rule named inside another -- `value:Expr`, an enum's alternatives -- is
  where the cast to it is, so it hovers as that and goes to that rule.

A grammar may still be given as a string, `grammar r#"…"#`, in ungrammar's own
notation (`'let'`, `//` comments): read by an ungrammar lexer of Lingua's own,
its mistakes at their byte in the string. Written that way it has no places
for the editor to use.

The declaration is a procedural macro. It reads the grammar and the tables,
checks them (below), writes the code of §2–§4, and leaves the grammar as a
compile-time binding, a `Datum`, for `lang` and `pass` to read (§5).

**Checks**, each reported at the rule it is about:

- every rule a rule mentions is defined, and every quoted token is in the
  lexer or the `tokens` table;
- the grammar is LL(k) once the precedence table has taken the left
  recursion out: every choice -- an enum's alternatives, a `|`, whether a `*`
  goes on or a `?` is taken -- is decided by at most the next k tokens, k 4
  unless a `lookahead N` section says otherwise. The parser is an automaton
  worked out when the program is compiled: each choice a test of the tokens
  ahead, and nothing tried and taken back. Two alternatives no k tokens tell
  apart are an error naming both and the tokens they share; a `*` or `?` that
  could go on or stop takes what it can, as a greedy loop does;
- no left recursion is left over;
- no rule is named `Program`, `Arena` or `Builder`, which the language's
  tables are named with (§5).

How far ahead a decision looks is found by walking configurations -- an
alternative and the rules still to finish after it -- a token at a time,
through FOLLOW where a rule ends, with a budget of nodes per decision; FIRST
and FOLLOW are bitsets over the kinds. Where one alternative is a prefix of
another's, the grammar is written so the next tokens decide it: a pattern in
parentheses read as an expression and told apart after, a signature and a
definition read as one form.

## 2. Parsing: events

The generated parser follows matklad's
[resilient LL parsing](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html).
It never builds a tree. It reads the kinds of the lexer's tokens — trivia
filtered out — and emits a flat list of events:

```meadow
data Event = Open Kind | Close | Advance | Error String
```

kept not as a list of `Event`s but as tables (`Events`): what each event is,
the kind of each `Open`, and each `Error`'s message, a slot per event in
arrays that grow as the parse goes (`Lingua.Buf`). The lexer's output is
tables too — each token's kind, where it ends and whether it is trivia — so
a parse makes no value per token or event that is counted and freed one by
one. The runtime, written once in Lingua:

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
  FIRST. The generated code tests the next kind itself, with the kind type's
  `==` where that type is known, rather than calling `at` and `expect`, which
  are generic in it and would look its equality up at run time.

What a rule's function is made of is written once, beside it, and shared: a
choice is a function of its own -- a `match` on the token it looks at, an arm
a token, and where one token does not say, the choice the next one makes --
a `*` is a function that calls itself, and reading a token is one function
of the grammar's. Two rules that make the same choice, or read the same thing
any number of times, call the same one. So a rule's own function is a call
for each thing it reads, in turn. That is a third less written for a large
grammar, and it is the shape a back end wants that makes a function of every
point a call returns to and of every place two branches meet: a choice whose
every arm is an answer has no such place.

**Recovery** is by recovery sets, computed rather than written: a loop over
`X*` inside rule `R` gives up — breaking out without consuming — on a token in
FOLLOW(`R`) or in any enclosing loop's set, and skips anything else as an
error node. So `let x = ;` is one `Let` with a missing `Expr` and an error,
and the next statement still parses. Every token ends up somewhere in the
tree; nothing is dropped.

## 3. Trees: green and red

The events and the tokens are built into a **green tree**, the lossless one.
It is the text and a handful of tables, one slot per node or token in the
order they are met reading down and left to right:

```meadow
data Green k = Green String   -- the text it covers
  #[k]                        -- each slot's kind
  #[Int] #[Int]               -- where each starts and ends in that text
  #[Int]                      -- how many slots its subtree takes; 0 for a token
  #[Int]                      -- its parent's slot; -1 for the root
  #[Int] #[Int]               -- where each stands, when not where it is
```

A node's children follow it, each after the whole subtree of the one before,
so they are found by stepping from slot to slot. A tree of a hundred thousand
tokens is six blocks rather than a hundred thousand: passing it, walking it
and dropping it move one count, not one per node.

Positions are the tree's own — it starts at 0 whatever file it came from —
and `subtree` copies a node out as a tree of its own, with its text. So a
declaration read from two versions of a file is the same value however far it
moved, which is what a query's early cutoff (§6) needs. What the tables give
up is sharing: two versions of a file hold a copy each of what did not change,
rather than one node both point at.

**Trivia** — whitespace and comments — are tokens like any other: the Scythe
lexer declares them as variants and leaves off `@skip`, so they are lexed
rather than dropped. The `trivia` line of `syntax!` names them. The parser
never sees them — it reads only the other tokens — and the tree builder puts
them back where they were: trivia before a node goes to its parent, before the
node opens, as rust-analyzer attaches it; trivia inside goes where it was.
Concatenating the tokens of a green tree gives back the source, byte for byte;
a test says so for every example. A lexer error is kept too: the text Scythe
stopped on becomes an error token, and the tree still covers every byte.

**Parsing tokens.** Not every input is text. A macro writes tokens -- some
the call passed in, which stand where they were written, and some its
template wrote, which stand where the call is -- and they are to be parsed
there. So `syntax!` writes a second entry point beside `parseCalc`:
`parseCalcTokens`, given the lexer's tokens, each with the bytes it stands for
and its text. The parser and the builder are the same; what differs is the
tree. Its text is the tokens' texts, a space between each two, so that
reading it still reads them apart -- and each slot has an **origin**, the two
last tables: where a token stands, and a node from its first token's to its
last's, as a parser's spans do. The spaces between are trivia that stand
where they are. A tree parsed from text has no origin tables at all, and
costs what it did; a slot without an origin (-1) stands where it is. The
context rules (`context`) read the given tokens as they read lexed ones, and
an error is said where the token it was said at stands. `subtree` and
`Green.node` carry origins along, so a pass that rewrites a tree
(`Lingua.Rewrite`) keeps them.

The **red view** is where positions and parents live: a cursor over the green
tree, made on demand and never stored. The tables already say where each slot
starts and which is its parent, so a cursor is only a tree and a slot.
`offset`, `range` and what is found at a byte are where a slot stands; its
place in the tree's own text, which is what is spliced when a tree is
rewritten, is `textRange`.

```meadow
data Syntax k = Syntax (Green k) Int
kind, text, range, textRange, children, parent, ancestors, tokenAt, nodeAt
```

## 4. Typed AST, from the labels

Each **node** rule becomes a type wrapping a red node, and each **enum** a sum
of its alternatives, with a checked cast from `Syntax` and the way back:

```meadow
data CalcLet = CalcLetNode (Syntax Calc)
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
| the entry points      | `parseCalc`, `parseCalcTokens`, `astCalc`                                                 |
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

**Patterns.** Each node rule is also a pattern over the tree, named as its
type is: `CalcBin lhs op rhs` matches a `Bin` node and binds its children,
each a `Syntax Calc` to match further --

```meadow
match s with
| CalcLet n (CalcBin l op r) -> …
| CalcLet n v -> …
```

-- the abstract syntax read straight off the lossless tree, with nothing built.
A pattern binds what the language the grammar implies keeps (§5): every node
child, and every token that is labelled or has text of its own. A child that
is always there binds as itself, and a node missing it -- a tree with an
error in it -- does not match; a child that may be missing binds as a `Maybe`,
and one that may be several as a list. Each is a synonym with a view
(`pattern CalcBin l op r <- (view -> [(l, op, r);])`), so it is found,
imported and exported as any pattern is. MeadowBoot's parser reads Meadow's
tree this way into the dump its differential test compares.

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

A language that extends a grammar's -- or one that does -- is a **tree
language**: a program of it is a lossless tree, as the parser makes, whose
kinds are the language's own, and each production a pattern over it binding
its fields off a node's children:

```meadow
lang! {
  pub Grouped extends Surface
  Ops - Ops
  Ops + Infix { lhs : Ops, op : InfixOp, rhs : Ops }
  Ops + Leaf { operand : Operand }
}
```

`GroupedInfix l op r` matches an `Infix` node of a `Syntax Grouped`. A node's
fields are its children in order, trivia passed over, and so is any child no
field there or later could hold -- the brackets and commas a rule writes and
nothing names. A production a tree language adds holds nodes of sorts, or
tokens (`String`); no field is added to every production, since a node is
its children. This is how an AST stays the tree the parser made while the
passes after it change what it may hold: each language says what its trees
are, and the types say which pass a tree has been through.

A language may also be written out whole, from nothing -- `lang! { pub Core
Expr + Lam { param : Int, body : Expr } … }` -- its sorts all known before any
field's type is read, so a field may name a sort written further down. `Program`,
`Arena` and `Builder` are not sorts' names: the tables are named with them.

`Sort - Prod` removes a production; `Sort + Prod { field : Type, … }` adds one
(to a new sort, if there is none by that name); `Sort * { field : Type, … }`
adds the fields to every production the sort has -- what an elaboration's
target wants, `Expr * { ty : Type }` being `Core` with a type on every
expression. A field every production of a sort has gets an accessor like
`meta`'s, `tyTypedExpr`. A language may take **type parameters** -- `lang! {
pub Inferring s extends Core … }` -- which its types take in turn, and a field
may be of a type applied to them, `MType s`; a language extending it takes
them too. A field's type is a sort of
the language, `[T]`, `Maybe T`, or any other type by name.

A production is one of its **sort's**, and two sorts may each have one of a
name: a variable is an expression and a variable is a pattern, `Expr + Var`
and `Pat + Var`, with no prefix on either to tell them apart. A program names
what a language written `with modules` has **by module** -- the language's,
and in it one a sort:

```meadow
match (a, e) with
| Core.Expr.Lam param body -> …          -- a production, as a pattern
| Core.Expr.Var id -> …

Core.Expr.newLam b meta param body       -- a node of it, written
Core.Expr.lamBody a e                    -- a field of it, read
Core.Pat.varName a p                     -- and a pattern's `Var`, not this one
Core.Expr.kind a e, Core.newBuilder ()   -- the sort's, and the language's
```

From another module a sort's is brought in as any module is -- `use
MiniML.Desugar.Core.Expr as Expr`, and then `Expr.Lam param body`. The types
are named flat, `CoreArena` and `CoreExpr`, since a type is not yet written by
a path.

Underneath, everything is also written in one flat namespace, named for its
language, its sort and itself -- `newCoreExprLam`, `corePatVarName`, the
pattern `CoreExprLam` -- always, so adding a `Var` to another sort renames
nothing; the one production of a sort of its own name is that name once,
`newCoreName`. Those are what a pass's code is written in, and each name in a
module is the flat one's other spelling: a synonym of the synonym, a function
that calls the function, which costs nothing once inlined. A tree language
keeps one production of a name, since a node's kind is its rule's.

In a `pass!` neither is written. A case is its production's name and a node
is too -- `| Lam { … } -> Lam { … }` -- the node being the one of the sort
that is being written, if that sort has one of the name. Where two sorts have
a production of the name, the sort says which, as a constructor is said of
its type: `| Pat.Var { … }`, `Pat.Var { … }`; a bare name two sorts have is an
error saying both.

Each language is written out as **tables**, not as a type of node. A program
is a handful of arrays: for each node its production, the bytes of the source
it came from, and where its fields start in one table of slots. A node is the
number of its row -- `CoreExpr` is an `Int` -- and a field that is a node, a
number or a truth is one slot; a list, or a `Maybe`, is where its elements
start in the slots; any other type is kept in a column of its own. However
many nodes a program has, passing it, walking it and dropping it move no
count and free nothing per node, which is what a compiler written in a
reference-counted language needs of its trees. For `Core` the code has:

| written                                       | what it is                                                   |
| --------------------------------------------- | ------------------------------------------------------------ |
| `CoreProgram`, `coreArena`, `coreRoot`        | a program: its tables, and its root's row                    |
| `CoreBuilder`, `newCoreBuilder`, `freezeCore` | the tables being written, inside a `runSt`, and finished     |
| `newCoreExprLam b meta param body`            | a node, written: its row                                     |
| `CoreExprKind`, `kindCoreExpr a e`            | which production a node is -- a constructor with no fields   |
| `coreExprLamBody a e`, `metaCoreExpr a e`     | a field of a node, and where it came from                    |
| `CoreExprView`, `viewCoreExpr a e`            | a node as a value to `match` on, its nodes rows still        |
| `CoreExprLam param body`, matching `(a, e)`   | a production as a pattern: a synonym, of the kind and fields |
| `subCoreExpr a e`                             | a node and everything under it, as a program of its own      |

Two more families are written for a language that asks for them, after what
it extends or is read from -- `lang! { pub Typed extends Core with In, set
… }`. `with In` is each reading function again, suffixed `In`, reading a
builder while it is written: what a pass's case reads the nodes it has just
made with, so a pass whose cases read their target needs its target written
so, and is told where if it was not. `with set` is a field of one thing -- a
node, a number, a truth, a text -- written again on a node already written,
`setCoreExprLamParam b e v`: what a pass that fixes up what it wrote needs, such
as taking the hygiene marks off the names a macro's expansion wrote once it
is clear which names are locals. Most languages want neither, and between
them they were a third of what a language's declaration wrote. `with modules`
is its names by module, above: a third again on what is written, for a
language a program names things of by hand.

Every language keeps its rows the same way, so what reads and writes them is
written once, in `Lingua.Lang` -- a node's tag and where it came from, a
field's slot, a list's, a node written with its fields -- and a language's
own functions are a line each over those, with its own types: `coreExprLamBody a
e` is the second slot of `e` in `a`'s rows. A language that extends another
is written out whole all the same, since its types are its own, but what is
written for each production is its name and its place. Asking a node's kind and reading its fields makes nothing; a view
makes one value, to match on where that reads better. Each production is also
a **pattern synonym** over a node and its tables, and a sort's productions
together cover it, so the tables are matched as if they were a data type:

```meadow
fun eval (a : CoreArena) env (e : CoreExpr) =
  match (a, e) with
  | Core.Expr.Int n -> Value.Int n
  | Core.Expr.Lam param body -> Value.Closure param body env
  | Core.Expr.App func arg -> apply a (eval a env func) (eval a env arg)
  …
```

and cost what the kind and the accessors do: the synonyms are inlined, their
answers taken apart where they are made, and the arms become one switch on
the node's kind -- nothing is built to match on. `subCoreExpr` is the
tables' `Green.subtree`: the same part of a program is the same value whatever
is around it, which is what an early cutoff (§6) compares. A language read off
a grammar also gets its conversion from the typed AST, `surfaceFromCalc :
Green Calc -> Maybe SurfaceProgram` — `None` when the tree has something
missing in it, since only an error-free tree is a program.

A **`pass!`** is a function from one language to another, written as the
cases that change something:

```meadow
pass! {
  pub lower : Surface -> Core
  | Bin { lhs, op, rhs } -> Prim { op = op, args = [lhs, rhs] }
}
```

It reads both languages and writes the rest: a function per sort —
`lowerExpr`, `lowerStmt` — reading the source's tables and writing the
target's, the children of every node translated before its case sees them,
and a copy of every production the target kept as it was. A case names the
fields it wants, bound already translated: rows of the target. In its body, a
production of the target written with a record — `Prim { … }` — is that node
written into the target's tables, where the node the case replaces came from;
the target's reading functions — `metaCoreExpr`, `tyTypedExpr` — read what
has been written so far; and `out` is the target being written, for a
function the case calls to write nodes into or read them from. `lower` runs in
a `runSt` of its own and answers a program; `lowerSt` writes into the caller's.

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
`later` field is a function of `()`. A field of several children taken `later`
is a function each, and one that may be missing is a function if it is there:
each child is translated in the context it is in, so a `let` of several
bindings can give each the scope the ones before it made.

A case may also take a child **`raw`**: the row the source has for it, not
translated at all, for a case that has to look at what a child is before it
can say how to translate it -- `runSt`'s argument, say, whose body an
elaboration types in a state of its own, or a handler's clauses, whose
operations say what the body may perform before any clause is typed. In a
case `src` is the source's tables, to read such a row with, and the pass's
own functions -- `elaborateExpr src out env child` -- translate it when the
case is ready:

```meadow
pass! {
  pub lowerFolding : Surface -> Core
  | Bin { raw lhs, op, rhs } ->
      (if kindSurfaceExpr src lhs == SurfaceExprKind.Literal and surfaceLiteralNumber src lhs == "0" and op == "+" then rhs
        else Prim { op = op, args = [lowerFoldingExpr src out lhs, rhs] })
}
```

A context flows _down_ the tree; what a pass learns _across_ it -- every
binder it met, what each is called and where, every use of each -- is a
**side table** (`Lingua.Table`), and that is an effect rather than a
parameter, so no case passes it on:

```meadow
pass! {
  resolve : Surface -> Resolved with env
  | NameRef { ident } ->
      (match L.lookupAssoc ident env with
       | Just id -> (let u = amend id (\(b : Binder) -> { b | uses = V.pushBack b.uses here }) in NameRef { id = id })
       | None -> …)
  | Lam { params, later body } ->
      (let ids = V.map (\p -> enter (binderOf p)) params in
        Lam { params = ids, body = body (bindAll params ids env) })
  …
}
```

A pass may read a **tree** -- a grammar's, or a tree language's -- rather
than tables: a case is a production, its fields bound by the production's
pattern, a node child translated and a token its text. A field taken **`raw`**
is the child as the tree has it, for a case that looks at what was written.
Only the sorts the pass can come to from the root need a function: a concrete
grammar has many rules no pass translates on their own. And a pass from a
tree may write one, of the same language or another tree language: a node no
case is for is kept, its kind the target's of the same name -- nothing is
copied that nothing changed under -- and a case's `Infix { lhs, op, rhs }` is
a node built of its fields, with the tokens the source had between them, so
the tree written is as lossless as the one read (`Lingua.Rewrite`). A kind
the target lacks needs a case. In such a case `out` is the tree being read,
which `Lingua.Rewrite`'s functions take.

`enter` records what is known about something and answers its id -- its
place in the table, from 0 -- `entry` reads one back, and `amend` rewrites it
as more is learned. Whoever runs the pass installs the one handler,
`tabled`, and gets the table beside the tree. From then on the tree holds
only ids, and every later pass keys what it keeps by them: MiniML's
inference context and evaluation environment are by id, and its editor's
features are lookups in the table.

MiniML's inference is **Algorithm J** as nine cases, and shows what a case
body being ordinary Meadow buys. Its type variables are mutable cells --
`Std.St`'s -- so its cases perform `St s`, and fail with `Std.Exn`; `infer`
goes from `Core` to `Inferring s`, whose nodes hold types made of those cells,
and `settle` from there to `Typed`, reading each. Both run inside one
`runSt`, which takes the `St s` away: `typed` is pure, and runs inside a
query like anything else, though every step of it writes. A pass to or from a
language with type parameters writes its tables in that same `runSt` --
`inferSt`, `settleSt` -- since a function using the state of two at once is
not one Meadow can type.

## 6. Queries: for an editor, and for a build

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
  | query program (file : Int) : Maybe CoreProgram = M.map lower (surfaceFromCalc (queryTree file))
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

**A batch.** What the engine above remembers is for the edit after this one.
A compiler run once over its sources has no such edit to wait for, and pays
for the remembering all the same: on MiniML, a tenth to a seventh more than
the passes cost called one after another. So `database!` writes a second way
to run the same queries, `batchSession ()`: a key is worked out the first
time it is asked for and its answer kept, and that is all -- no revisions,
nothing noted of what was read, no answer compared with the one before, no
`TVar`. It is the passes, run once each in the order they are needed, and
costs what they do. A batch is set and asked with the functions a session
is -- `setSessionSource`, `sessionTyped` -- so a compiler is declared once
and driven two ways: its editor keeps a `newSession ()`, and its `check`,
its `run` and its REPL ask a batch. Setting an input of a batch forgets its
answers, and a snapshot of one keeps nothing. What the passes read and write
is the same either way: the lossless tree the parser made, and each pass's
side table.

**In turn, or parallel** (milestone 5). `queryTreeEach files` -- the
`fetchAll` operation -- asks for several keys at once, and `sessionTreeEach
db files` does the same from outside. They are worked out one after another:
a compiler is sequential until it says otherwise, and it says so with
`parallelSession db`, which is the database with each of those on a thread
of its own (`Std.Thread`). Nothing else reaches a thread, so a compiler that
does not ask has none in it -- which matters where a program that can spawn
pays for it everywhere, as one compiled all the way down does. In a batch, a
thread works a key out in a batch of its own, with a copy of the inputs, and
what it was asked for comes back. `sharedSession db` hands the threads the
inputs in one compact region instead, which saves the copies and is not the
default because of what it costs where Lingua is meant to run: a runtime
that counts references writes a count the threads share for every string or
record read out of a region, and they wait on each other for it. A copy each
is never very slow; a region of sources read from eight threads was. In
a session, threads share nothing mutable, so the database is `TVar`s
(`Std.Stm`): its revision, its inputs, a slot per derived key -- a `TVar`
each, so that two threads writing what they worked out do not meet -- and a
log. A slot is worked
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
`completion`, `symbols`, `format`, the file set out by its formatter, and
`tree`, the lossless tree, from which semantic
tokens, folds and selection ranges are read here, the same for every
language. A token's class is first what its kind and spelling say -- a
keyword, a name, a number -- and then, with `resolved`, what the compiler
knows of that occurrence: the byte a token starts at, and whether what it
names is a `function`, a `type`, an `enumMember`, a `namespace`. That is how
Meadow's own server colours a file, and the classes are its: a spelling
cannot say that a name is a function's, and name resolution can. An edit sets an input and nothing more: what it changed is all
that is worked out again, so hovering after an edit reads an elaboration that
was only redone for the file that changed. The protocol is Lingua's -- its
framing, which needs Meadow's `Console.readExact`, its UTF-16 positions, the
documents open -- and the server says it can do exactly what it was given.

**`repl!`** declares a prompt over the compiler's database, with the
server's features: what is being typed is a file of the session, and `text`,
`tree` and `completion` are the functions `lsp!` takes. `eval` is what
entering an entry prints. `errors` is where the parser found something
wrong: an entry whose only errors are at its end is right as far as it goes,
so Enter starts a new line of it rather than entering it -- which is all a
language has to say to have entries of several lines. `indent` is what that
line opens with, a formatter's `indent<Name>` (below), so an entry is
indented as it is typed the way it would be formatted. `declarations` is
what Ctrl-F finds among -- each a name, what it is, and what is said of it --
in a finder under the line: what is typed there narrows them to what it is
found in, the likeliest first, and Enter puts the chosen name where the
cursor was. What is typed is coloured as an editor colours a file: where the
database has a server (`lsp!`), by the server's own answer to what each token
is -- `tokensServer`, which is also what it tells the editor, name resolution
and all -- and where it has none, from the tree. How each class is shown is a
theme, of which a prompt has several built in; a line that starts with `:` is
for the prompt and not the language, and `:theme` lists them, each in its own
colours, while `:theme quiet` turns one on. What is written is `runRepl ()`.

The line, the finder's matching and its drawing are packages of their own,
which a compiler not written with Lingua can use as well:
[LineEditor](https://github.com/mcdearman/LineEditor),
[Fuzzy](https://github.com/mcdearman/Fuzzy) and
[Doodle](https://github.com/mcdearman/Doodle), over `Std.Terminal`. The
editor and the finder are each a state and a function of a key, so a prompt
is tested by giving it keys. Where the input is not a terminal, entries are
read a line at a time and nothing is drawn.

**`format!`** declares a formatter against a grammar's rules. A rule of the
formatter is a rule of the grammar written again as what a node of it holds,
in the grammar's order -- a token as it is written, a child by its label --
with how it is set out around and between them:

```meadow
format! {
  pub Style for Mini
  indent 2
  width 80
  | Let   = "let" recursive name params "=" group(indent(line value) line "in") newline body
  | Lam   = "fun" params "->" indent(line body)
  | If    = "if" cond indent(line "then" yes) indent(line "else" no)
  | Paren = "(" indent(cut expr) cut ")"
}
```

`indent(…)` is a level in from where the node began. `group(…)` goes on one
line if it fits and breaks together if not, and a rule is a group as a
whole. Between two things, `line` is a space or -- where its group breaks --
a line's end, `cut` nothing or a line's end, `newline` a line's end always,
and `tight` nothing; with none of them, a space. A rule or a child the
grammar does not have is an error where it was named. What is written is two
functions of a tree:

- `formatStyle`, its text laid out in the width: each node with a rule on
  one line if it fits and broken where the rule says if not, each child by
  its own rule in turn. A node no rule is for is written as it was, its
  breaks where they were, so a formatter is adopted a rule at a time. A
  comment keeps a line of its own before what it was written before. Only
  the space between tokens changes: the rules are matched against the
  lossless tree's children as they are met, and a child a rule did not name
  -- something the parser could not place -- is written where it was, so a
  broken program loses nothing. Given to `lsp!` as `format`, it is the
  editor's formatting. Lines are broken by
  [Pretty](https://github.com/mcdearman/MeadowPretty), Wadler's printer.
- `indentStyle`, what the line after an *unfinished* text opens with: a
  level for each line on which a node began that is still open at the end,
  in its indented part -- there and running to the end, `let x = 1 +`, or
  not there yet with nothing after it, `let x =`. This is what a REPL asks
  when Enter does not finish an entry, and it is the formatter's answer, so
  what is typed is indented as it would be formatted.

A formatter with no `width` only indents: lines stay broken where the
program broke them, and each is set in one level for each *line* on which a
node began whose indented part it starts in -- `let f = fun y ->` opens two
nodes on one line, and what follows is in by one level.

**Names** are resolved as a pass, the first after the grammar's: which `x` a
use means depends on what is around it, which the pass's **context** (§5)
carries down. `Resolved extends Surface` has an id wherever a name was -- a
binder's own, and a use its binder's -- and what the pass learned about
each binder is its side table (§5): what it is called, where, what it binds,
where it can be seen, and every use. That table is a query of its own, which
going to a definition, finding references, renaming, completion and symbols
all read.

**`make!`** declares a build over **compilation units**: directories with a
manifest naming the units they depend on. Two kinds of incrementality meet
at a unit, and they belong to different owners:

- **Between units**, the build system's. A unit is compiled again only if
  its sources, or an interface it was compiled against, are not what they
  were; a change that leaves a unit's interface as it was compiles that unit
  and nothing that depends on it. Units that do not depend on each other are
  compiled at once, on threads of their own, each started as soon as the
  units it depends on are done -- it waits for those and for no other, so a
  slow unit holds up only what needs it. A build is parallel unless it
  says otherwise, `| parallel = false`, which compiles them one after another
  and leaves no thread in it. That is the other way round from a compiler
  (§6), which is sequential until it asks: a compiler may be used with no
  build around it, and when there is one, the units are where the work
  divides. What was built is kept in
  `target/lingua-build.json`, interfaces and all, which are `Reflect` so that
  they can be.
- **Inside a unit**, the compiler's. `compile` is given the unit's sources,
  the interfaces it depends on, and a directory of its own; how it does the
  work -- its own queries, its own threads, its own cache in that directory --
  is its business. MiniML's is a batch (§6): the unit's files read once, the
  passes run straight through and its files typed in turn, since the build
  already runs its units at once, and nothing kept there -- a cache of a
  whole unit's answers stood only when nothing in the unit had changed,
  which is when the build does not ask.

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
   `saveSession`/`loadSession`, the calculator using them. And the batch:
   `batchSession`, with `parallelSession` the one way to a thread.
6. **Tooling** (§8), on MiniML: `cli!`, `lsp!`, name resolution as a pass
   with a context, and `make!` over compilation units. Done, on every Meadow
   runtime; the next language to get them is Meadow itself.
7. **Formatting and the prompt** (§8). `format!`: rules written against
   the grammar, laid out within a width or only indented, the editor's
   formatting and the indent a continuation line opens with -- done, on
   MiniML. `repl!`: a prompt over a session, with entries of several lines,
   the formatter's indentation, completion and a finder -- done, on MiniML.

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
