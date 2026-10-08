# Lingua's documentation

[DESIGN.md](DESIGN.md) is the design: why each part is the way it is, from the
grammar to the build. Start there for the whole picture.

The guides below each cover one area in the detail needed to use it: what to
write, what gets generated, the defaults, the errors and the limits.

| guide                              | covers                                                                 |
| ---------------------------------- | ---------------------------------------------------------------------- |
| [drivers.md](drivers.md)           | `newSession`, `batchSession`, `parallelSession`, `sharedSession`: one set of queries as an editor's compiler or a batch compiler, and when threads are used |
| [build.md](build.md)               | `make!`: units, what the compiler is given, parallel scheduling and `parallel = false` |
| [languages.md](languages.md)       | `lang!`: names scoped by sort, `Core.Expr.Lam` with `with modules`, readers that read a program being written, `with set` and `with reflect` |
| [passes.md](passes.md)             | `pass!`: what the traversal does on its own, `Sort.Prod` cases, `node`, cases that answer several nodes, and `from` for a pass over part of a program |
| [format.md](format.md)             | `format!`: rules, layout within a width, indent-only mode, and the next line's indent |
| [repl.md](repl.md)                 | `repl!`: features, multi-line entries, keys, completion, the finder, themes and `:theme` |
| [highlighting.md](highlighting.md) | semantic tokens: how a token gets its class, `resolved`, and how the editor and the prompt share one answer |

Most examples in the guides are taken from tested code in
[examples/MiniML](../examples/MiniML) and [examples/Calc](../examples/Calc).

## Defaults at a glance

| choice                                   | default                     | to change it                          |
| ---------------------------------------- | --------------------------- | ------------------------------------- |
| a compiler's queries                     | sequential                  | `parallelSession db`                  |
| a batch's threads and their inputs       | a copy each                 | `sharedSession db`                    |
| a build's units                          | parallel                    | `\| parallel = false`                 |
| a language's setters                     | not written                 | `with set`                            |
| a language's names by module             | not written                 | `with modules`                        |
| a language's programs as `Datum`s        | not `Reflect`               | `with reflect`                        |
| a formatter                              | indents only, 2 spaces      | `width N`, `indent N`                 |
| a prompt's theme                         | `soft`                      | `\| theme = "…"`, or `:theme` at run time |

## Related packages

The prompt is built on three packages that work without Lingua:

- [LineEditor](https://github.com/meadow-lang/LineEditor): reading an entry, with
  history, completion and multi-line editing.
- [Fuzzy](https://github.com/meadow-lang/Fuzzy): ranking texts by a few typed
  letters.
- [Doodle](https://github.com/meadow-lang/Doodle): drawing in a terminal, a cell
  at a time.
