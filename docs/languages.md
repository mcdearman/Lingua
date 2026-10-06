# Intermediate languages: names and options

This covers two things about `lang!` beyond what
[DESIGN.md §5](DESIGN.md#5-lang-and-pass) introduces: how what it writes is
named, how its readers read a program that is still being written, and the
`with` options that ask it to write more.

## A production belongs to its sort

Two sorts may each have a production of the same name. A variable is an
expression and a variable is a pattern:

```meadow
lang! {
  Shapes with modules
  Expr + Var { name : String }
  Expr + Lam { param : Pat, body : Expr }
  Pat + Var { name : String }
  Pat + Wild {}
}
```

Neither needs a prefix to keep them apart, because everything written for a
production is named for its language, its sort **and** itself.

### Flat names

Every table language gets these, always:

| for                         | name                        | example               |
| --------------------------- | --------------------------- | --------------------- |
| writing a node              | `new<Lang><Sort><Prod>`     | `newShapesExprLam`    |
| reading a field             | `<lang><Sort><Prod><Field>` | `shapesPatVarName`    |
| the production as a pattern | `<Lang><Sort><Prod>`        | `ShapesExprLam`       |
| a sort's kind               | `<Lang><Sort>Kind.<Prod>`   | `ShapesPatKind.Var`   |

The sort is always in the name, whether or not another sort has a production
called the same. Adding a `Var` to a second sort later renames nothing.

One exception keeps names from stuttering: the single production of a sort
that shares the sort's name is that name once. A sort `Name` with one
production `Name` gives `newCoreName`, not `newCoreNameName`.

These flat names are what the code a `pass!` writes is written in.

### Names by module: `with modules`

A language declared `with modules` also has everything by module, the
language's and inside it one per sort:

```meadow
match (a, e) with
| Shapes.Expr.Var name -> [name]                    -- a production, as a pattern
| Shapes.Expr.Lam param body -> …

Shapes.Expr.newLam b noMeta param body              -- a node, written
Shapes.Expr.lamBody a e                             -- a field, read
Shapes.Pat.varName a p                              -- the pattern's `Var`, not the expression's
Shapes.Expr.kind a e                                -- which production a node is
Shapes.newBuilder (), Shapes.freeze b               -- the language's own
```

What each module holds:

| module        | names                                                                             |
| ------------- | --------------------------------------------------------------------------------- |
| `Core`        | `newBuilder`, `freeze`, `arena`, `root`, `startOf`, `endOf`                       |
| `Core.Expr`   | `kind`, `meta`, `view`, `sub`, and any field every production has (`ty`)          |
| per production| the pattern `Lam`, `newLam`, `newLamSpan`, and a reader per field, `lamBody`      |
| `with set`    | `setLamParam` for each field that can be rewritten                                |

Each dotted name is the flat name's other spelling: a pattern synonym of the
synonym, a function that calls the function. Once inlined it costs nothing, and
a `match` over a sort's dotted patterns is still one switch on the node's kind.

From another module, bring a sort in as any module:

```meadow
use MiniML.Desugar.Core.Expr as Expr

fun eval (a : CoreArena) env (e : CoreExpr) =
  match (a, e) with
  | Expr.Int n -> Value.Int n
  | Expr.Lam param body -> Value.Closure param body env
  …
```

Two limits:

- **Types stay flat.** `CoreArena`, `CoreExpr`, `CoreProgram`: Meadow does not
  yet write a type by a path of three segments.
- **Tree languages have no modules.** A tree language's node kind is its
  rule's, so it keeps one production per name and is named flat. Two sorts with
  a production of one name is an error there, and so is any `with` option.

`with modules` is off by default because it adds about a third to what a
language's declaration writes. Turn it on for a language a program names
things of by hand; leave it off for one only passes read and write.

### In a pass

Inside `pass!` neither spelling is written. See
[passes.md](passes.md#which-production-a-case-is-for).

## Reading a program while it is being written

A language's tables exist in two states: finished, an **arena**
(`CoreArena`), and being written, a **builder** (`CoreBuilder`) inside a
`runSt`. Every reader takes either. `kindCoreExpr a e`, `coreExprLamBody a e`
and `metaCoreExpr a e` read an arena when given one and a builder when given
one, and there is one function of each name.

Reading an arena is pure. Reading a builder performs the `St` of the `runSt`
it is being written in. The reader's type says so through the language's
`Rows` trait:

```meadow
trait CoreRows r {
  effect CoreReading r
  …
}

impl CoreRows CoreArena { effect CoreReading CoreArena = {} … }
impl CoreRows (CoreBuilder lst) { effect CoreReading (CoreBuilder lst) = { St lst } … }

-- coreExprLamBody : CoreRows r => r -> Int -> CoreExpr ! CoreReading r
```

`lang!` writes the trait and the two `impl`s. Nothing is asked for, and no
reader has a second spelling.

A pass uses this when a case reads nodes of the target it has just written.
MiniML's type inference does: it translates a child, then reads the type off
the node that came back.

```meadow
lang! {
  pub Inferring s extends Core
  Expr * { ty : MType s }
}

pass! {
  pub infer : Core -> Inferring with ctx
    | Lam { param, later body } ->
        (let a = fresh ctx in
         let b = body (bind ctx param a) in
         Lam { param = param, body = b, ty = MType.Fun a (tyInferringExpr b) })
    …
}
```

In a case a reader of the target is written with the node alone,
`tyInferringExpr b`: the pass supplies the builder it is writing. Outside a
pass, hand a reader the builder yourself: `tyInferringExpr out b`.

## Options: `with set, modules`

Options follow what the language extends or is read from, separated by commas:

```meadow
lang! { pub Core extends Resolved with modules … }
lang! { Shapes with modules … }                 -- written out whole
lang! { pub Surface from Mini with set }        -- read off a grammar
```

Each declaration states its own; a language does not inherit the options of
the one it extends. Anything other than `set` or `modules` is an error that
lists the two.

### `with set`: write a field again

`with set` writes `set<Lang><Sort><Prod><Field> b e v` for each field that
holds one thing: a node, a number, a truth, a text. A list or a `Maybe` field
cannot be rewritten in place.

It is for a pass that fixes up what it wrote, such as taking hygiene marks off
the names a macro expansion wrote once it is clear which are locals.

### Why they are opt-in

Few languages need setters, and `with modules` adds about a third to what a
declaration writes. Asking by name keeps generated code, and compile time,
down for every language that does not.

## What is written once for every language

Every table language keeps its rows the same way, so the code that reads and
writes rows lives in `Lingua.Lang` (`LangTables`, `LangWriting`, `slot`,
`listFrom`, `maybeFrom`, `node0`…`node4`), and a language's own functions are a
line each over it. `coreExprLamBody a e` is the second slot of `e` in `a`'s
rows.

A language that extends another is still written out whole, since its types
are its own, but what is written per production is little more than its name
and its place.
