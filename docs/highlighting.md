# Highlighting: one answer for the editor and the prompt

A Lingua language server colours a file with semantic tokens. The same answer
colours what is typed at a prompt. This guide covers how a token gets its
class, how a compiler refines that with what it knows, and how the two front
ends share it.

## Two steps

**1. By kind and spelling.** With a `tree` feature, every token of the lossless
tree is classified from the name the grammar gave its kind and from its text.
This needs nothing from the compiler.

**2. By what the compiler resolved.** With a `resolved` feature as well, the
compiler says what particular occurrences are, and those override step 1.

A spelling cannot say that a name is a function's. Name resolution can, and it
can say so of one occurrence and not another with the same spelling. This is
how Meadow's own language server colours a file, and the classes are its.

```meadow
lsp! {
  pub Server "miniml" for Session
    | text = setSessionSource
    | tree = treeOf
    | resolved = resolvedOf
    …
}
```

## The classes

In the order the protocol indexes them (`Lingua.Lsp.tokenTypes`):

`keyword`, `type`, `enumMember`, `function`, `variable`, `number`, `string`,
`operator`, `namespace`, `comment`

`classOf "function"` answers a class's index, or `-1` for a name that is not
one.

## Step 1: how a token is classified

`Lingua.Lsp.tokenClass kindName text` decides, first match wins:

| the kind's name contains…, or the text is…          | class      |
| --------------------------------------------------- | ---------- |
| `Whitespace` or `Error`, or the text is blank       | none       |
| `Comment`                                           | `comment`  |
| `Ident` or `Name`                                   | `variable` |
| `String` or `Str`                                   | `string`   |
| `Number`, `Int` or `Float`, or the text is all digits | `number` |
| the text is all letters and `_`                     | `keyword`  |
| anything else                                       | `operator` |

So the lexer's constructor names do the work: call an identifier token `Ident`
or `LowerIdent` and it is a `variable`; a token the grammar spells out as a
word, like `"let"`, is a `keyword`; one spelled in symbols is an `operator`.

Step 1 never produces `type`, `enumMember`, `function` or `namespace`. Those
come only from step 2.

## Step 2: `resolved`

```meadow
resolved : db -> Int -> [(Int, String)]
```

Each pair is the **byte a token starts at** and the class it resolved to:
`"function"`, `"type"`, `"enumMember"`, `"namespace"` or `"variable"`. A token
that starts at a listed byte takes that class; every other token keeps what
step 1 gave it. A class name that is not one of the ten is ignored.

`resolved` needs `tree`; a server with one and not the other is an error.

MiniML's reads the side table its name resolution pass filled. A binder whose
kind is `function` is a function where it is bound and at every use:

```meadow
fun resolvedOf db file =
  let binders = bindersOf db file in
  if V.isEmpty binders then written (treeOf db file)
  else
    V.concatMap
      (\(b : Binder) ->
        if b.kind != "function" then []
        else V.map (\m -> match m with | Meta a _ -> (a, "function")) (V.pushFront b.uses b.at))
      binders
```

For `let f = fun x -> x in f 1`, both occurrences of `f` are `function` and
both of `x` are `variable`, though all four are the same kind of token.

### Text that does not resolve yet

At a prompt, most of what is on screen is unfinished. `let foo a = a` with no
`in` yet is not a program, name resolution has nothing to say, and every name
would fall back to `variable`.

The fix is in the compiler's `resolved`, not in Lingua: when nothing resolved,
answer from what is written. MiniML's `written` treats as a function's name:

- the name of a `let` with parameters after it, `let foo a = …`;
- the name of a `let` whose value is a `fun`, `let g = fun x -> …`;
- the name of a `let rec`, which has parameters;
- any use spelled the same as one of those.

No scope is known in this fallback, so a name that shadows another is not told
from it. That is corrected as soon as the entry is whole and resolves.

A language with other binding forms writes its own version of this; the shape
to copy is "resolved if there is anything resolved, otherwise by syntax".

## What the server writes

A server with a `tree` writes:

```meadow
tokensServer : db -> Int -> [(Int, Int, Int)]
```

Each triple is a coloured token's start byte, end byte and class index, in
order. With `resolved` it is `refined (classified display tree) resolved`;
without, just `classified display tree`.

This is what the server encodes for `textDocument/semanticTokens/full`, and it
is public so that other things can ask the same question.

## The prompt asks the same question

`repl!` looks for a server declared for its database. When there is one, what
is typed is set as file `0` and coloured by `tokensServer db 0`, shown through
the current [theme](repl.md#themes). When there is none, the prompt runs step 1
on the tree itself.

Requirements for the shared path:

- the `lsp!` has a `tree` feature;
- it is declared **before** the `repl!` (MiniML has both in one module,
  [Editor.mw](../examples/MiniML/src/Editor.mw));
- both name the same database after `for`.

Nothing is configured on the prompt's side. The link is a compile-time binding
`lsp!` leaves and `repl!` reads.

## Using the pieces directly

| function (`Lingua.Lsp`)          | answers                                               |
| -------------------------------- | ----------------------------------------------------- |
| `tokenClass kindName text`       | a token's class from step 1, or `None`                |
| `classified name tree`           | every coloured token: start, end, class               |
| `refined classes resolved`       | those, with resolved occurrences overriding           |
| `tokensReply text classes`       | the protocol's encoding                               |

| function (`Lingua.Repl`)         | answers                                               |
| -------------------------------- | ----------------------------------------------------- |
| `painted theme text classes`     | `text` with each token shown as the theme shows its class |
| `coloured theme name tree`       | `painted` over `classified`                           |

`painted` colours a line at a time, so a token that runs over several lines
does not colour the start of the next.

## Formatting in the editor

The other feature added to `lsp!` alongside these is `format`:

```meadow
format : db -> Int -> String
```

It answers the file's text set out, normally a `format!` formatter applied to
the file's tree. The server answers `textDocument/formatting` with a single
edit replacing the whole file, or with no edits when the text is already what
the formatter would write. See [format.md](format.md).
