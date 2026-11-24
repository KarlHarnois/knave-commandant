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

```clj
(use knave-commandant defcommand run-command)

(defcommand greet "Greets someone."
  :version "1.0.0"
  (who string :about "Who to greet")
  (:loud switch :short l :about "Greets in capitals"))

(defn main ()
  (let (options (run-command greet))
    (println "Hello, " (:who options))))
```

See [examples/serve](examples/serve/src/main.knv) for every kind of parameter.

`run-command` prints help, versions and usage errors, then exits. To print them yourself, for example as JSON, call `(parse-command greet (process-environment) (command-line-args))`. It returns `(ok greet)` for the typed struct, or `(err outcome)` for help, a version or a usage error.

## Development

Run each check before you push:

```sh
knave check
knave test
knave fmt --check
knave lint
knave task check-file-lengths
```
