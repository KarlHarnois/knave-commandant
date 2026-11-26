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

A command takes arguments and options:

```clj
(use knave-commandant defcommand parse-args)

(defcommand greet "Greets someone."
  :version "1.0.0"
  (who string :about "Who to greet")
  (:loud switch :short l :about "Greets in capitals"))

(defn main ()
  (let (options (parse-args greet))
    (println "Hello, " (:who options))))
```

`defcli` groups commands under one program, and `defchoice` names a fixed set of values:

```clj
(use knave-commandant defchoice defcli defcommand
                      parse-args)

(defchoice log-format text json)

(defcommand build "Builds the site."
  (dir string :default "." :about "The folder to build"))

(defcommand serve "Serves the site."
  (:port u16
         :short p
         :default 8000
         :env "SITE_PORT"
         :about "The port to listen on")
  (:format log-format
           :default text
           :about "How to print each request"))

(defcli site "Builds and serves a site."
  :version "1.0.0"
  (:verbose count :short v :about "Logs more with each -v")
  (commands build serve))

(defn main ()
  (match (:command (parse-args site))
    ((site-command/build options)
     (println "Building " (:dir options)))
    ((site-command/serve options)
     (println "Serving on port " (:port options)))))
```

Each program completes its own command line in bash:

```sh
source <(COMPLETE=bash site)
```

See [examples/serve](examples/serve/src/main.knv) for every kind of parameter, and [examples/site](examples/site/src/main.knv) for nested commands.

## Development

Run each check before you push:

```sh
knave check
knave test
knave fmt --check
knave lint
knave task check-file-lengths
```
