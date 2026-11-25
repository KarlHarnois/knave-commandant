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
| Help output | goes to stdout, exits 0, shows description, usage, arguments, options, subcommands, and each default and env var | default |
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
| Typos | an unknown name gets a tip, as in "a similar option exists: `--port`" | default, for options and subcommands |
| Exit codes | 0 ok, 1 failure, 2 usage, 130 on Ctrl-C | 0 and 2 built in, and 70 for a broken spec. The app maps its own errors and signals |
| Streams | data on stdout, messages on stderr | help to stdout, errors to stderr |
| Abbreviations | no guessing of subcommand prefixes | never, so new commands never break old scripts |
| Secrets | not passed as flag values | `:env` lets a secret come from the environment. Nothing warns about an option that holds one |
| Completion | Tab completes commands, options and fixed values | default for bash: `source <(COMPLETE=bash prog)` |
| Extras | man page or markdown help generated from the same spec | not built yet, the spec value allows it |

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
subcommands. `(parse-line spec environment words)` returns the matches
or an `early-exit`: the text to print, the stream to print it on and
the exit code. Nothing in this layer reads the process, so every case
is a plain test. Programs that build their command line at run time use
this layer directly.

**Layer 2: `defcommand` and `defcli` bind typed names.** One declaration
gives a struct with a typed field per parameter, the spec and the
conversion from matches. `defcli` groups commands into an enum, with
global options and nested groups. The macros only write layer 1 values,
so anything they express can also be built by hand.

Two entry points sit on top. `run-command` reads the process, prints
help, versions and usage errors, and exits with 0 or 2. `parse-command`
does none of that and returns the typed value or the `early-exit`, for
a program that decides itself when to print and exit, or that tests its
command line. `(exit-early stop)` prints an `early-exit` and exits,
which is all `run-command` adds.

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
| `complete/` | what can complete the word under the cursor, and the bash script that asks |
| `macros/` | `defcommand`, `defcli`, `defchoice` |

`spec/`, `scan/`, `resolve/` and `values/` are the pipeline, in that
order. `errors/` names the errors each of them returns, and
`values/matches.knv` holds the matches `resolve/` builds. `help/`,
`complete/` and `macros/` read the pipeline and nothing in it reads
them. Two modules reach forward: `spec/checks.knv` takes the conflict
rule from `resolve/`, and `errors/report.knv` takes the usage line from
`help/`.

Five modules sit outside the folders. `parse.knv` joins the stages.
`command-line.knv` turns each outcome of a parse into a value or an
`early-exit`, and `run.knv` is the only module that touches the
process.
`environment.knv` holds the variables a parse reads, and `sequence.knv`
a vector helper two stages share. `tests/` mirrors `src/` file for
file, and `tests/e2e-tests.knv` goes through the public interface.

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
7. **Spec mistakes are found before parsing.** Every entry point
   validates the spec first, and a mistake is an `early-exit` with
   code 70, so a broken spec never looks like a user error.
8. **Help takes the width as a parameter.** `command-line.knv` reads
   `COLUMNS` from the environment value and falls back to 80, so help
   text stays pure.
9. **`defcommand` keeps its own grammar.** It reads like `deftask`'s (a
   symbol is an argument, a keyword is an option) without accepting a
   `deftask` parameter list. A second grammar for the same parameter
   would cost more than rewriting a task once. `deftask` itself is
   untouched: the prelude cannot depend on a git library.
10. **`run-command` is a macro, not a generated function.** A generated
    `run-<name>` per command was unused in every `defcli` program and
    tripped the unreachable-definition lint.
11. **The program completes its own command line.** With `COMPLETE=bash`
    in the environment, `prog` prints a bash script, and
    `prog -- <words>` prints what can complete the last word. The script
    is a fixed stub that calls the program on each Tab, so the logic is
    written once, in Knave, as a pure function of the spec. An
    environment variable is the trigger because a `completions` command
    would be read as the first argument of a program without
    subcommands. The request is answered by the pure parse, so a
    program that calls `parse-command` completes like one that calls
    `run-command`.
12. **Completion never fails on a half-written line.** It skips unknown
    words and options. A value or an argument without choices completes
    as a path, through bash's own completion. Only bash is supported:
    bash shows no descriptions, so candidates carry none.
13. **A group run without a command prints its help, on stderr, with
    exit code 2.** The line is incomplete, so it fails like every other
    usage error, and the help shows which commands exist. `--help`
    still prints on stdout with exit code 0. POSIX only asks for a
    non-zero status, and Click, clap and argparse all use 2 here.
14. **A parse that does not give a value gives one `early-exit`.** Help,
    a version, a usage error, a completion reply and a broken spec all
    come back as the same struct: `:text`, `:stream` and `:code`. The
    caller never works out which case it is, what to print or how to
    exit, so a new case inside the library breaks no program. clap's
    `Error`, argh's `EarlyExit` and bpaf's `ParseFailure` have the same
    shape. `examples/greet` uses this entry point, so CI runs it. The
    cost: the text is written for a terminal, and the caller cannot
    read which error it was. A program that needs the error as data,
    such as one that prints JSON, needs a new field on `early-exit`.

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
