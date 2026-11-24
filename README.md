# knave-commandant

A [Knave](https://github.com/knave-lang/knave) library for building command line programs. A program describes its commands once, as data. The library derives the parser, the help text, the usage errors, and the exit codes from that description. Every program then follows the common conventions without extra code:

- `-h` and `--help` on every command
- `--version` at the root
- short and long options
- `--` to end the options
- usage errors on stderr with exit code 2

The library is in design. No release exists yet. [docs/design.md](docs/design.md) holds the conventions, the proposed API, and the results of the spike that tested it.

## Install

Add the library to the `libraries` section of your `project.knv`:

```knave
(defproject my-tool
  (libraries
    (knave-commandant :git "https://github.com/KarlHarnois/knave-commandant"
                      :rev "<commit>")))
```

Replace `<commit>` with the commit to pin.

## Development

Run each check before you push:

```sh
knave check
knave test
knave fmt --check
knave lint
knave task ledelint
```
