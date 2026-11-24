# knave-commandant: design notes

A Knave library for building command line programs that follow the
common conventions by default. This document records the conventions the
library follows and the decisions behind its shape. The code holds the
data shapes: start at `src/spec/shapes.knv` and the two programs under
`examples/`.

## 1. What a well-behaved CLI does

Sources: the POSIX Utility Syntax Guidelines, the GNU Coding Standards
(`--help`, `--version`, long options), and clig.dev. The library should do
everything in the "default" column without the author writing anything.

| Area | Convention | Library behaviour |
| --- | --- | --- |
| Help | `-h` and `--help` on every command and subcommand, plus `prog help <sub>` | default, cannot be turned off |
| Help output | goes to stdout, exits 0, shows description, usage, arguments, options, subcommands, examples, and each default and env var | default |
| Version | `-V` and `--version` at the root only | default when the app declares a version |
| Long options | kebab-case, `--name value` and `--name=value` | default |
| Short options | one character, `-p 80`, `-p80`, bundled switches `-abc` | opt in per option, `-h` and `-V` reserved |
| End of options | `--` ends option parsing, `-` is a positional meaning stdin | default |
| Ordering | options may come before or after positionals | default |
| Booleans | `--color` on, `--no-color` off when the default is on | default for switches whose default is true |
| Repeats | `-vvv` counts, `--tag a --tag b` and `--tag a,b` collect, `<FILE>...` repeats | `count` and `(vec t)` types, `:delimiter` |
| Conflicts | two options that cannot be given together are a usage error | `:conflicts-with` per option |
| Placeholders | values in capitals, as in `--port <PORT>` and `[PATH]` | default |
| Choices | fixed set of values, listed in help and in the error | enum value types |
| Env vars | flag beats env var beats default, both shown in help | `:env` per option |
| Required | positionals are required, options are optional | `:required true` on an option, an `option` type on an argument |
| Usage errors | message on stderr, one usage line, "try --help", exit 2 | default |
| Typos | "unknown option `--prot`, did you mean `--port`?" | default, for options and subcommands |
| Exit codes | 0 ok, 1 failure, 2 usage, 130 on Ctrl-C | usage code built in, app maps its own errors |
| Streams | data on stdout, messages on stderr | help to stdout, errors to stderr |
| Abbreviations | no guessing of subcommand prefixes | never, so new commands never break old scripts |
| Secrets | not passed as flag values | doc guidance, maybe a `:secret` lint |
| Extras | shell completions, man page or markdown help generated from the same spec | not built yet, the spec value allows it |

## 2. Libraries worth borrowing from

| Library | Style | What to take |
| --- | --- | --- |
| clap (Rust) | builder API plus derive on structs and enums | A struct is a command, an enum is a set of subcommands, doc comments become help. Closest fit, since Knave compiles to Rust. |
| Cobra (Go) | tree of command structs | Persistent (global) flags inherited by subcommands, completion and doc generation from the tree. |
| Click and Typer (Python) | decorators, Typer reads the function signature | A command is a function and its parameters are the flags. This is what Knave's `deftask` does. |
| Swift ArgumentParser | property wrappers `@Argument`, `@Option`, `@Flag` | A `validate` step after parsing, separate from the run step. |
| optparse-applicative (Haskell) | applicative parser combinators | Parsers compose and help is derived from the parser. Elegant, but needs higher-kinded types that Knave lacks. |
| tools.cli and babashka.cli (Clojure) | the spec is plain data, the result is a map | Spec as data, a dispatch table of `{:cmds [...] :fn f :spec ...}`. The closest Lisp model. |
| Racket `command-line` | one macro form | A single declarative form that reads like the help page. |
| docopt | the help text is the spec | Nice to read, fragile to maintain. Skip. |
| oclif (Node) | one file per command, discovered from a folder | Matches how `deftask` finds tasks under `tasks/`. |

The common winner across them: describe the CLI once, as data, and derive
the parser, the help, the errors, and the completions from that one
description. clap derive adds the part Knave needs most: the parse result is
a typed struct or enum, not a map of strings.

## 3. Two layers

Knave code prefers data over functions over macros, with a macro only
where it binds names. That gives two layers.

**Layer 1: the spec is a value, and parsing is a pure function.** A
`command-spec` describes a command: its arguments, its options and its
subcommands. `(parse spec environment words)` returns an `outcome`:
matches, help, a version or a usage error. Nothing in this layer reads
the process, so every case is a plain test. Programs that build their
command line at run time use this layer directly.

**Layer 2: `defcommand` and `defcli` bind typed names.** One declaration
gives a struct with a typed field per parameter, the spec and the
conversion from matches. `defcli` groups commands into an enum, with
global options and nested groups. The macros only write layer 1 values,
so anything they express can also be built by hand.

Two entry points sit on top. `run-command` reads the process, prints
help, versions and usage errors, and exits with 0 or 2. `parse-command`
does none of that and returns the typed value or the outcome, for a
program that prints its own way, such as JSON.

## 4. Layout

`src/` follows a command line through the library:

| Folder | Stage |
| --- | --- |
| `spec/` | what a command is, the built-in options, and the checks on a spec |
| `scan/` | words to tokens to the options and words found, descending into subcommands |
| `resolve/` | what was found to matches: arguments, defaults, env vars, conflicts |
| `values/` | matches to typed values, through the `cli-value` trait |
| `errors/` | the usage errors, their report and the "similar name" tips |
| `help/` | the usage line and the help page |
| `macros/` | `defcommand`, `defcli`, `defchoice` |

`parse.knv` joins the stages and `run.knv` is the only module that
touches the process. A stage uses the stages before it, never the ones
after. `tests/` mirrors `src/` file for file, and
`tests/e2e-tests.knv` goes through the public interface.

## 5. Decisions

1. **Help and version are built in and reserved.** Every command takes
   `-h` and `--help`, and `-V` and `--version` when it declares a version.
   A spec or a `defcommand` that defines one of those names is rejected.
   They live in one table, `spec/builtins.knv`.
2. **`help <command>` exists only in programs with subcommands**, next to
   `<command> --help`, as git, cargo and clap do. A command named `help`
   is rejected.
3. **Global options work before or after the command name.** Each
   subcommand's spec gains its parent's options, and scanning switches
   spec at the command word, so each command reads its own short options.
4. **No prefix matching** of options or commands. Adding a name would
   otherwise break scripts that relied on a prefix.
5. **`:env` takes one name or a list**, tried in order, after the flag
   and before the default.
6. **One `presence`** (`required`, `optional`, `defaulted`) serves
   arguments and options. Whether an argument repeats is a separate
   `repetition`.
7. **Spec mistakes are found before parsing.** `run-command` validates
   the spec first and exits with code 70 on a mistake, so a broken spec
   never looks like a user error.
8. **Help takes the width as a parameter.** `run.knv` reads `COLUMNS` and
   falls back to 80, so help text stays pure.
9. **`defcommand` keeps its own grammar.** It reads like `deftask`'s (a
   symbol is an argument, a keyword is an option) without accepting a
   `deftask` parameter list. A second grammar for the same parameter
   would cost more than rewriting a task once. `deftask` itself is
   untouched: the prelude cannot depend on a git library.
10. **`run-command` is a macro, not a generated function.** A generated
    `run-<name>` per command was unused in every `defcli` program and
    tripped the unreachable-definition lint.

## 6. Limits from the compiler

These shape the code today. Each has an issue on knave-lang/knave.

| Limit | Effect here | Issue |
| --- | --- | --- |
| A generic function bound by a trait with a method that takes no `self` must live in the trait's module | `cli-value` and its readers share `values/convert.knv` | #522 |
| User code cannot implement `from-str` | the library has its own `cli-value` trait, and `value-names` is picked with a typed `none` | |
| A module that imports `code` cannot define runtime functions of the same names | macro modules are separate from runtime modules | |
| Field inference looks at every struct in the program | functions that read a common field name, such as `:name`, declare their types, and an application may need to as well | #778 |
| A case pattern can resolve to the wrong enum when two share a case name | patterns on such enums are qualified | #777 |
| The prelude cannot read env vars, and `exit` returns unit | two `rust` forms in `run.knv` | #773, #774 |
| `fmt` needs fetched libraries to see their macro shapes | CI runs `knave check` before `knave fmt --check` | #769 |
