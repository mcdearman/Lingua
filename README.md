# lingua

Compilers written as their specification, in Meadow.

The syntax is an [ungrammar](https://rust-analyzer.github.io/blog/2020/10/24/introducing-ungrammar.html);
lingua writes the parser from it and gives back a lossless tree — every byte
of the input, whitespace and comments included, and every error where it
belongs. Intermediate languages and the passes between them come next, as
declarations, with the plumbing generated. The whole design is in
[docs/DESIGN.md](docs/DESIGN.md).

```meadow
use Scythe (lexer!)
use Lingua (syntax!)

@derive(Lexer)
data Token
  = @regex("[ \t\r\n]+") Whitespace        -- no `@skip`: the tree keeps it
  | @token("let") Let
  | @token("=") Equals
  | @token(";") Semi
  | @regex("[0-9]+") Number String
  | @regex("[a-zA-Z_][a-zA-Z0-9_]*") Ident String

syntax! {
  pub Mini
  lexer Token
  trivia { Whitespace }
  tokens { "let" = Let, "=" = Equals, ";" = Semi, "ident" = Ident _, "int_number" = Number _ }
  grammar r#"
    File = Let*
    Let = 'let' name:Name '=' value:Literal ';'
    Name = 'ident'
    Literal = 'int_number'
  "#
}

-- parseMini : String -> (Green Mini, [(Int, String)])
```

## What there is

- **`syntax!`** reads the ungrammar and two tables beside it — which lexer
  token each quoted token is, and how left-recursive rules bind — checks the
  grammar (undefined rules, unknown tokens, alternatives that start alike, left
  recursion with no precedence), and writes a resilient LL parser that emits
  events, in the style of matklad's
  [resilient LL parsing](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html).
  A mistake in the grammar is reported at the byte of the string it is at.
- **A typed AST** from the grammar's labels: a type per rule, a cast from
  the tree, and an accessor per element — `MiniLet`, `miniLetName`,
  `miniLetValue` — each a `Maybe` or a list, since the tree holds whatever was
  written. `parseMini` is pure, and `astMini` casts its root. Everything
  `syntax! { Mini … }` makes is named after `Mini`, so a rule may have any
  name — `Let`, as a token is called, or `Int`, as a type is — without
  meeting one of the program's own.
- **`lang!`** declares an intermediate language — read off the grammar
  (`lang! { Surface from Calc }`) or as changes to another
  (`Core extends Surface`, `Expr - Bin`, `Expr + Prim { … }`,
  `Expr * { ty : Type }` for a field on every expression), with type
  parameters if it needs them (`Inferring s`) — written out as plain data
  whose every node carries where it came from.
- **`pass!`** is the cases that change something between two languages; the
  traversal, the copies of what did not change, and the provenance of what did
  are written for it. Case bodies are ordinary Meadow, effects and all. A pass
  may carry a context down the tree -- an environment -- and a case may take a
  child `later`, to translate it in a context of its own. MiniML's type
  inference is such a pass: Algorithm J, its type variables `runSt` cells,
  elaborating the program into one with every expression's type written on
  it.
- **`Lingua.Green`**, the lossless tree: kinds, widths and text, no positions.
- **`Lingua.Red`**, a view with offsets and parents: `range`, `children`,
  `parent`, `ancestors`, `tokenAt`, `nodeAt`.
- **`Lingua.Parser`** and **`Lingua.Build`**, the runtime a generated parser
  stands on, usable by hand.
- **`database!`** declares a compiler's steps as queries -- inputs set from
  outside, and derived queries whose bodies are ordinary Meadow reading the
  others by name -- and **`Lingua.Query`** runs them in the style of salsa:
  each remembered, re-run only when something it read has changed, and cut
  off where it answers what it did before. Several keys can be asked at once,
  each on a thread of its own, and a query marked `persisted` is kept between
  runs, standing for as long as the inputs it read are what they were. An
  input never set, or queries that read each other in a circle -- on one
  thread or across several -- are a `QueryError`.
- **`Lingua.Diagnostic`**, what a compiler says about a program and where,
  drawn by [Nettle](https://github.com/mcdearman/Nettle), the port of
  ariadne. A parse's errors become diagnostics at the tokens they were found
  at; a pass reports with the `Report` effect -- in a `pass!` case, `here` is
  the node being rewritten -- and `collect` gathers what it said, so a query
  can run a pass that reports and stay pure.
- **`cli!`** declares a command line -- commands, arguments, flags, help --
  and writes the parser, the help, a bash completion script, and its
  mistakes drawn as diagnostics against the command line.
- **`lsp!`** declares a language server over the compiler's database:
  diagnostics, hover, definition, references, rename, completion and
  symbols from the language's own queries, and semantic tokens, folds and
  selection ranges from the lossless tree.
- **`make!`** declares a build over compilation units: each compiled in
  dependency order, those that can be at once, and again only when its files
  or an interface it read changed -- what happens inside a unit being the
  compiler's own business, held to the unit by an `Fs` handler.

Two examples, in [`examples/`](examples):

- [`Calc`](examples/Calc), a calculator: statements, precedence, calls and
  recovery, a test that every input comes back byte for byte, a `Surface`
  to `Core` pass that is evaluated to the same answers, and all of it as a
  database of files -- where an edit re-runs only the file it is in, and a
  comment stops at what the file binds.
- [`MiniML`](examples/MiniML), Meadow's `examples/MiniML` -- a toy ML with
  `let`-polymorphism -- written as its specification: the grammar, a
  desugaring pass from `Surface` to `Core`, Hindley-Milner inference and an
  evaluator as queries, and every error, from the parser to the run, drawn
  where it is. It is also a command line (`check`, `run`, `repl`, `build`,
  `lsp`), a language server, and a build of units that import each other:

  ```text
  > 1 + true
  Error: cannot unify Bool with Int
     ╭─[ input.ml:1:5 ]
     │
   1 │ 1 + true
     │     ──┬─
     │       ╰─── cannot unify Bool with Int
  ───╯
  ```

## Next

All six milestones of the design are in, and pass on every Meadow runtime.
Next is bootstrapping: Meadow's own grammar in Lingua, its operators put in
order by a pass, and its front end emitting core for the compiler it
replaces.
