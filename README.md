# knave-commandant

A [Knave](https://github.com/knave-lang/knave) library for building command line programs that follow the common conventions by default. See [docs/design.md](docs/design.md) for the design.

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
