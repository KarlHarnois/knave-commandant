# knave-commandant

A [Knave](https://github.com/knave-lang/knave) library for building command line programs that follow the common conventions by default. See [docs/design.md](docs/design.md) for the design.

## Install

Add the library to the `libraries` section of your `project.knv`:

```clj
(defproject my-tool
  (libraries
    (knave-commandant :git "https://github.com/KarlHarnois/knave-commandant"
                      :rev "<commit>")))
```

Replace `<commit>` with the commit to pin.

## Usage

Declare a command with `defcommand`. The macro defines a struct with one typed field per parameter. It also defines `run-<name>`, which parses the process's command line into that struct:

```clj
(use knave-commandant defcommand)

(defcommand greet "Greets someone."
  :version "1.0.0"
  (who string :about "Who to greet")
  (:loud switch :short l :about "Greets in capitals"))

(defn main ()
  (let (options (run-greet))
    (println "Hello, " (:who options))))
```

`run-greet` answers `-h` and `--help` with the help page, and `-V` and `--version` with the version. It prints a usage error to stderr and exits with code 2. [examples/serve](examples/serve/src/main.knv) shows every kind of parameter.

## Development

Run each check before you push:

```sh
knave check
knave test
knave fmt --check
knave lint
knave task ledelint
```
