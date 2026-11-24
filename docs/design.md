# knave-commandant: design notes

A Knave library for building command line programs that follow the
common conventions by default.

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
| Repeats | `-vvv` counts, `--tag a --tag b` collects | `count` and `(vec t)` value types |
| Choices | fixed set of values, listed in help and in the error | enum value types |
| Env vars | flag beats env var beats default, both shown in help | `:env` per option |
| Required | positionals are required, options are optional | warn when an option is required |
| Usage errors | message on stderr, one usage line, "try --help", exit 2 | default |
| Typos | "unknown option `--prot`, did you mean `--port`?" | default, for options and subcommands |
| Exit codes | 0 ok, 1 failure, 2 usage, 130 on Ctrl-C | usage code built in, app maps its own errors |
| Streams | data on stdout, messages on stderr | help to stdout, errors to stderr |
| Abbreviations | no guessing of subcommand prefixes | never, so new commands never break old scripts |
| Secrets | not passed as flag values | doc guidance, maybe a `:secret` lint |
| Extras | shell completions, man page or markdown help generated from the same spec | later, from the spec value |

## 2. Libraries worth borrowing from

| Library | Style | What to take |
| --- | --- | --- |
| clap (Rust) | builder API plus derive on structs and enums | A struct is a command, an enum is a set of subcommands, doc comments become help. Closest fit, since Knave compiles to Rust. |
| Cobra (Go) | tree of command structs | Persistent (global) flags inherited by subcommands, completion and doc generation from the tree. |
| Click and Typer (Python) | decorators, Typer reads the function signature | A command is a function and its parameters are the flags. This is what `deftask` already does. |
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

## 3. Proposed API

The writing-knave rules ask for data over functions over macros, with a
macro only where it binds names. That gives two layers.

### Layer 1: the spec is a value, parsing is a pure function

```knave
(defstruct command-spec
  :name string
  :about string
  :arguments (vec argument-spec)
  :options (vec option-spec)
  :subcommands (vec command-spec))

(defstruct option-spec
  :long string
  :short (option char)
  :about string
  :value value-kind
  :default (option string)
  :env (option string))

(defenum value-kind
  switch
  count
  (one string)
  (many string)
  (choice (vec string)))

(defenum outcome
  (parsed matches)
  (help string)
  (version string)
  (usage-error cli-error))
```

- `(parse spec arguments)` returns an `outcome`. It is pure and needs no IO,
  so every convention in section 1 gets a plain `deftest`.
- `(spec->help spec)`, `(spec->usage spec)`, later `(spec->completion spec
  shell)` are pure functions of the same value.
- The library adds `-h/--help`, `-V/--version`, `--` handling and the
  usage error format to every spec. Authors cannot forget them.
- `matches` holds strings by name. This layer is the escape hatch for
  dynamic CLIs, and the target that layer 2 compiles to.

### Layer 2: `defcommand` and `defcli` bind typed names

The macro is justified because it defines names: a struct, an enum, and a
conversion function. The parameter grammar extends `deftask`'s: a symbol is
a positional, a keyword is an option.

```knave
(defcommand serve
  "Serves a folder over HTTP."
  (root path :default "." :about "The folder to serve")
  (:port u16 :short p :default 8000 :env "SERVE_PORT"
   :about "The port to listen on")
  (:open bool :short o :about "Opens the browser once the server starts")
  (:verbose count :short v :about "Logs more, repeat for more detail")
  (:format output-format :default text :about "How to print requests"))
```

expands to:

```knave
(defstruct serve :root path :port u16 :open bool :verbose int
                 :format output-format)
(def serve-spec (->command-spec :name "serve" ...))
(defn matches->serve (matches) ...)   ; (result serve cli-error)
```

An application groups commands into an enum:

```knave
(defcli site
  "Builds and serves a static site."
  :version "1.4.0"
  (:color color-choice :default auto :env "SITE_COLOR")   ; global option
  (commands build serve (item (commands item-add item-list))))
```

expands to `(defenum site-command (build build) (serve serve) ...)`, a
`site-spec`, and `arguments->site`. `main` stays a normal Knave pipeline:

```knave
(defn main ()
  (match (cli/run site-spec arguments->site (command-line-args))
    ((build options) (build! options))
    ((serve options) (serve! options))))
```

`cli/run` is the one IO function. It prints help or version to stdout and
exits 0, or prints the usage error to stderr and exits 2. The rest of the
program only ever sees a typed, valid command, which is the "parse at the
edge" rule from writing-knave.

Value types go through a library trait, `cli-value`, with a
`from-text` constructor and a `value-names` method for help. The library
implements it for the built-in types by delegating to `parse`. User enums
cannot implement `from-str` (it is a marker), so
`(defchoice output-format text json)` emits the enum and its `cli-value`
impl. The spike in section 6 confirmed this design.

### Why not the other styles

- Pure builder functions with no macro: the result is a map of strings, and
  every read is a runtime lookup by name that can fail. That is what rmap
  writes by hand today.
- Applicative combinators: need closures generic over the result type and
  higher-kinded composition. Heavy in Knave, slow to compile, and hard to
  read in errors.
- Wrapping clap through `rust` interop: gives a mature parser, but help and
  errors stop being Knave data and tests get harder. `knave add` already
  adds crates, so this stays a real fallback if the pure parser stalls.

## 4. Where it would land first

### rmap

About 2,000 lines under `src/cli/`. Most of it is what the library would own:

| rmap module | Replaced by |
| --- | --- |
| `routes`, `commands`, `flag-specs`, `route-matching`, `arguments`, `flags` | `defcli` with nested `commands`, typed option structs |
| `usage`, `help`, `help-text` | `spec->help`, `spec->usage` |
| `environment-variables` | `:env` on `--actor` and `--json`. The `RMAP_ACTOR` then `USER` chain needs `:env` to accept a list. |
| `exit-codes` (usage case) | built-in exit 2, app keeps its own codes |
| `tasks/sync-readme-help.knv` | still useful, could call `spec->help` directly instead of running the binary |

What rmap needs that the design must cover: multi-word subcommands (`item
add`, `dep rm`), global options shared by every command, choice values
(`graph-format`, `status`), required options (`new --manager`), and a JSON
output mode. Output formatting and the reply envelopes stay in rmap.

### ledelint

`src/arguments.knv` is 86 lines for one optional positional and `-h`. It
becomes:

```knave
(defcommand ledelint
  "Checks that each Knave file reads top-down, like a newspaper."
  (path string :default "." :about "The directory to lint"))
```

Fixes it gets for free: usage errors exit 2 instead of 1, `--version`,
`--` handling, "did you mean" on unknown options.

These two are the acceptance tests for the API: if both migrate cleanly and
shrink, the design holds.

## 5. Should it plug into `deftask`?

`deftask` lives in the prelude (`rslisp/prelude/task.knv`). Its parser is a
block of inline Rust inside the macro. It already supports positionals,
`--name value`, `--name=value`, switches, `--`, `--help`, exit 2 on errors.
It lacks short options, per-option help text, `-h`, env vars, repeats,
choices, `--no-x`, and "did you mean".

Options:

1. Leave `deftask` alone. The library is for shipped binaries, tasks are
   for project chores.
2. Share the parameter grammar. `defcommand` accepts every `deftask`
   parameter list unchanged, then adds `:short`, `:about`, `:env`. A task
   can graduate into a real command by renaming the form.
3. Rebuild `deftask` on top of the library. The prelude cannot depend on a
   git library, so this means moving layer 1 into Knave's standard library.

Recommendation: do 2 now, keep 3 as the goal once the API has settled
through rmap and ledelint. Do not make the library depend on `deftask`
internals, since they are private to the prelude and change with the
compiler. The gaps listed above are worth raising against rslisp on their
own, starting with `-h` as an alias for `--help`.

## 6. Spike results

The spike is a library project and an app project that uses it via
`:path`, on knave 0.21.2.

| Question | Answer |
| --- | --- |
| Can a library export a `defmacro` that emits `defstruct`, `defenum`, `impl` and `defn` into the caller's module? | Yes. `(defcommand serve (root string ".") (:port u16 "8000") (:format output-format "text"))` produced `serve`, `matches->serve`, and parsed to `(ok (->serve :root "." :port 8000 :format output-format/json))`. |
| Does generated code need the caller to import the library's runtime names? | No. Names in a macro's template resolve in the library, so the app imported only `defcommand` and `defchoice`. |
| Does type inference pick the value type from the struct field? | Yes. One generic `read-value` returned `u16`, `string` and a user enum depending on the field. |
| Can a macro read a `defchoice` from another form through module notes? | Yes, through `add-module-note` and `(module-notes module key)` over `(program-modules)`. A module sees the notes of its own forms (even ones below the call) and of modules it depends on, not of modules that depend on it. The final design does not need it, see the trait below. |
| Can user enums plug into `parse`? | No, `from-str` is a marker that user code cannot implement. A library trait `cli-value` with `(from-text (fn ((ref str)) (option self)))` works, implemented for both `u16` and user enums. |
| Can a trait method with no `self` (for listing choice names) be called? | Not directly (L0456). Passing a typed `none`, `(value-names (ref (the (option output-format) none)))`, picks the impl. The macro knows the type, so it can emit that. |

Constraints the spike found:

1. **Compiler bug, to report to rslisp.** A generic function bound by a
   trait from another module, calling a trait method without `self`,
   passes `knave check` but fails in rustc with `E0405 cannot find trait`.
   The generated Rust does not import the trait. Workaround: keep
   `cli-value` and every generic function that uses it in one module.
   Minimal repro: three files, `shape.knv` with the trait and an impl for
   `u16`, `reader.knv` with `(defn read (text) (from-text text))` under a
   `where` bound, `main.knv` calling `(read "x")`.
2. A module that imports from `code` for macro helpers cannot also define
   runtime functions that use the same names (L0412, `first` exists only at
   compile time). Split each layer into macro modules and runtime modules.
3. A re-export needs the constructor imported too: `(use values ->entry
   entry)` before `(export ->entry entry)`.

Design questions, answered:

1. **`:env` takes one name or a list**, tried in order. rmap's actor
   becomes `(:actor string :env ("RMAP_ACTOR" "USER"))`. A list costs
   little and removes a whole module from rmap.
2. **`help <sub>` is added only to apps with subcommands**, next to
   `<sub> --help`, as git, cargo and clap do. The word `help` is then
   reserved, and `defcli` rejects a command named `help`.
3. **`spec->help` takes the width as a parameter.** `cli/run` reads
   `COLUMNS` and falls back to 80, so help text stays pure and testable.
