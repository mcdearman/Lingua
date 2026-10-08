# The prompt: `repl!`

`repl!` declares an interactive prompt over a compiler's database. What is
being typed is a file of a session, and each feature is a function of the
database and that file, the same shape `lsp!` takes. Several can be the very
functions the language server uses.

```meadow
repl! {
  pub Prompt "miniml" for Session
    | text = setSessionSource
    | eval = shownIn
    | tree = treeOf
    | errors = errorsOf
    | completion = completionAt
    | declarations = declarationsOf
    | indent = indentStyle
    | prompt = "> "
    | history = ".miniml_history"
}
```

This writes `runPrompt ()`: entries are read and each printed as `eval` says,
until there are no more. `for Session` means the prompt makes its database with
`newSession ()`. The entry being typed is always file `0`.

## Features

Only `text` and `eval` are required. Each of the others turns something on.

| feature        | type                                         | what it gives                              |
| -------------- | -------------------------------------------- | ------------------------------------------ |
| `text`         | `db -> Int -> String -> ()`                  | how what is typed reaches the compiler     |
| `eval`         | `db -> Int -> String`                        | what entering an entry prints              |
| `tree`         | `db -> Int -> Green k`                       | colour, multi-line entries, indentation    |
| `errors`       | `db -> Int -> [Int]`                         | where the parser's errors are, in bytes    |
| `completion`   | `db -> Int -> Int -> [(String, String)]`     | Tab: names and what each is, at a byte     |
| `declarations` | `db -> Int -> [(String, String, String)]`    | Ctrl-F: name, what it is, what is said of it |
| `indent`       | `Green k -> String`                          | what a continuation line opens with        |
| `prompt`       | text                                         | what a line starts with; `"> "` if omitted |
| `history`      | text                                         | the file entries are kept in between runs  |
| `theme`        | text                                         | the theme it starts in; `soft` if omitted  |

`prompt`, `history` and `theme` are text in quotes; the rest are function
names. Mixing them up, naming a feature a prompt does not have, or leaving out
`text` or `eval` is an error where it was written.

Without `tree` the prompt colours nothing from the tree, never indents, and
takes every entry as finished. (A database with a language server is still
coloured; see [Colour](#colour).)

## Entries of several lines

An entry is **unfinished** when the parser found something wrong with it and
every error is at its end. What is there is right as far as it goes.

| typed            | errors at       | Enter                    |
| ---------------- | --------------- | ------------------------ |
| `1 + 2`          | none            | enters it                |
| `let x =`        | the end         | starts a new line        |
| `let = 1 in x`   | inside          | enters it, as an error   |
| `let = 1 in`     | inside and end  | enters it, as an error   |

That is all a language has to supply to get multi-line entries: `tree` and
`errors`. No list of opening and closing brackets is needed.

The new line opens with what `indent` answers. Given a formatter's
`indent<Name>` (see [format.md](format.md#the-next-lines-indent)), an entry is
indented as it is typed the way the formatter would indent it:

```text
> let x =
    41 + 1 in x
42 : Int
```

Continuation lines are set in by the prompt's width, then by the indent.
Alt-Enter and Ctrl-J start a new line whatever the text.

## Keys

The line is [LineEditor](https://github.com/meadow-lang/LineEditor)'s.

| keys                                   | do                                                |
| -------------------------------------- | ------------------------------------------------- |
| Left, Right, Ctrl-B                    | move by a character                               |
| Home, End, Ctrl-A, Ctrl-E              | to the start or end of the line                   |
| Ctrl-Left, Ctrl-Right, Alt-B, Alt-F    | move by a word                                    |
| Backspace, Delete                      | delete a character                                |
| Ctrl-W, Ctrl-K, Ctrl-U                 | delete the word before, the rest of the line, its start |
| Up, Down, Ctrl-P, Ctrl-N               | between an entry's lines, then through history    |
| Tab                                    | complete as far as every candidate agrees         |
| Enter                                  | enter a finished entry, or start a new line       |
| Alt-Enter, Ctrl-J                      | start a new line                                  |
| Ctrl-F                                 | open the finder                                   |
| Ctrl-C                                 | give the entry up                                 |
| Ctrl-D on an empty entry               | end the prompt                                    |

Pasted text goes in as it is.

## Completion

Tab asks `completion` for everything that could be written at the cursor, and
offers what starts as the name before the cursor does. A name is letters,
digits, `_` and `'`.

`completion` is given a **byte** offset, which is what a compiler counts in.
The prompt converts from the cursor's character position.

## The finder

Ctrl-F opens a finder under the line, listing what `declarations` answered.

- Typing narrows the list to the declarations the letters are found in, the
  likeliest first. Matching is [Fuzzy](https://github.com/meadow-lang/Fuzzy)'s,
  against the name and what it is, so typing `String` finds a `lines` whose
  type mentions it.
- The pane beside the list shows the chosen declaration: its name, what it is,
  and what is said of it.
- Enter puts the chosen name where the cursor was, replacing the partial name
  before it. `to`, Ctrl-F, `tot`, Enter turns `to` into `total`.

| keys                         | do                    |
| ---------------------------- | --------------------- |
| any character                | add to the query      |
| Backspace                    | take one back         |
| Up, Down, Ctrl-P, Ctrl-N     | move the choice       |
| Enter                        | take the chosen name  |
| Escape, Ctrl-C, Ctrl-G, Ctrl-F | put the finder away |

It is drawn with [Doodle](https://github.com/meadow-lang/Doodle), twelve rows
high and up to a hundred columns wide.

## Colour

What is typed is coloured token by token, each token having a class: `keyword`,
`function`, `variable`, `type`, `number` and so on. Where the classes come from
depends on the database:

- **The database has a language server.** If an `lsp!` with a `tree` feature
  was declared for the same database **before** the prompt, the prompt asks
  the server's `tokens<Server>`. That is the same answer the server gives an
  editor, name resolution included, so a file and an entry are coloured alike.
- **It has none.** The prompt classifies the tree's tokens itself, by kind and
  spelling.

No setting chooses between them; `repl!` finds the server at compile time.
[highlighting.md](highlighting.md) explains the classes and how a compiler
refines them.

## Themes

How each class is shown is a theme. Six are built in:

| theme    | character                                                                  |
| -------- | -------------------------------------------------------------------------- |
| `soft`   | the default: pastel shades from the 256-colour palette, the same whatever the terminal's own colours are |
| `bright` | a colour for everything name resolution tells apart, variables included    |
| `quiet`  | few colours, keywords bold; reads as well on a light ground as a dark one  |
| `editor` | like an editor's dark theme: functions blue, types yellow                  |
| `warm`   | red keywords, yellow functions, numbers apart from both                    |
| `plain`  | nothing coloured                                                           |

A class a theme does not mention is shown as it is. `soft`, for instance,
colours a function's name and leaves an ordinary variable uncoloured.

Start in a particular one with `| theme = "quiet"`. A name no theme has falls
back to `soft`.

## Commands

A line that starts with `:` is for the prompt, not the language. It is not
coloured, never counts as unfinished, is not evaluated, and is not written to
the history file.

| command        | does                                                             |
| -------------- | ---------------------------------------------------------------- |
| `:theme`       | list the themes, each with a sample in its colours, the current one marked `*` |
| `:theme quiet` | switch to that theme for what is typed from then on              |

An unknown theme lists the ones there are. Anything else after `:` is answered
with `` `:x` is not something this prompt does: it has `:theme` ``.

## History

With `history`, each entered entry is appended to that file and loaded on the
next run. Up and Down walk through it. Add the file to `.gitignore`; Lingua's
own ignores `.*_history`.

## Without a terminal

When input is a file or a pipe, entries are read a line at a time, as many
lines as each takes to be finished, and nothing is drawn. A prompt run by a
script prints what it would have.

## Testing a prompt

The editor and the finder are each a state and a function of a key, so a prompt
is tested by giving it keys:

```meadow
fun typing (text : String) : [Key] = V.map (\c -> T.key (Character (C.toString c))) (S.chars text)

fun prompted (keys : [Key]) =
  snd (withOutput (\() -> T.withEvents (V.map (\k -> Event.Pressed k) keys) (\() -> runPrompt ())))

@test fun anEntryIsEvaluatedWhenItIsEntered () =
  assertEq
    (S.contains "\n\r3 : Int\n" (prompted (V.append (typing "1 + 2") [T.key Enter])))
    True
    "its value and type, on a line of their own"
```

[Editor.mw](../examples/MiniML/src/Editor.mw) has tests for evaluation,
continuation, completion, the finder, colour and themes written this way.

## Limits

- **A running entry cannot be interrupted.** Ctrl-C while `eval` is running
  ends the whole prompt. An entry that loops forever has to be killed with it.
  This waits on interrupt support in Meadow.
- **The theme is not remembered.** `:theme` lasts for the run; set the starting
  one with `| theme = …`.
- **`:theme` is the only command.**

## By hand

Everything `repl!` writes is a call into `Lingua.Repl`, which is usable
directly: `unfinished`, `candidates`, `painted`, `coloured`, `finder`,
`findKey`, `findDrawn`, `themes`, `themeNamed`, `command`, `editorOf`, `run`.
The three packages underneath work without Lingua at all, for a compiler not
written with it.
