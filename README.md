# Crystal overloading analyzer (WIP)

This is a tool for studying overload resolution behavior in [Crystal](https://crystal-lang.org/) as part of my licenciate thesis. 

Crystal allows defining multiple functions with the same name but different signatures. However, in many cases the result of the program depends on the order of the definitions, which is not a desired behaviour. There have been some attempts in the past to formalize and fix how ambiguity is solved but could not satisfy every case. 

The project has two components:

1. A permutation generator that produces all possible permutations of overloaded function definitions and runs them to compare the outputs.
2. (WIP) A static analyzer that parses Crystal (our minimal version of Crystal) code and, for a given callsite, decides if ambiguity exists between the available overloaded functions or returns directly the function that would be called.

## Prerequisites

- [GHC](https://www.haskell.org/ghc/) and [Cabal](https://www.haskell.org/cabal/) (tested with GHC 9.12.3 / Cabal 3.14)
- [Crystal](https://crystal-lang.org/install/) (latest)

> **Optional** A [devenv](https://devenv.sh/) / Nix-based setup is included (see `devenv.nix`) and will provide all dependencies automatically.

## Build

```sh
cabal build
```

## Usage

The tool exposes two subcommands.

### `parse`

Checks whether a `.cr` file is parseable by this tool and prints the parsed result:

```sh
cabal run crystal-parser -- parse <file.cr>
```

This is useful for verifying that your file falls within the [supported subset](#parser-limitations) of Crystal before running `rearrange`.

### `rearrange`

Generates all permutations of `@[Slot]`-annotated functions/methods in a file (or every `.cr` file in a directory), runs each permutation through the Crystal compiler, and saves the results.

**Single file:**

```sh
cabal run crystal-parser -- rearrange <file.cr>
```

**Directory:**

```sh
cabal run crystal-parser -- rearrange <directory/>
```

#### Marking overloads for rearrangement

Annotate the functions or methods you want to permute with `@[Slot]`:

```crystal
class Foo
  @[Slot]
  def bar(x : Int32)
    x * 2
  end

  @[Slot]
  def bar(x : String)
    x + x
  end
end
```

Only definitions annotated with `@[Slot]` are reordered, everything else stays in place.

## Output

For each input file `<name>.cr`, a directory `<name>/` is created containing:

- `1.cr`, `2.cr`, … one `.cr` file per permutation of the slots.
- `result.md` a summary report with exit codes, stdout, and stderr for each permutation, compiled once per crystal opts subset we are interested in.

## Samples

The `samples/` directory contains interesting Crystal cases used for analysis: programs where overload ordering has a noticeable effect on compiler behavior. To run all of them:

```sh
cabal run crystal-parser -- rearrange samples/
```

## Development

A `ghcid`-powered watch loop (reloads on file changes and runs the test suite) is available via:

```sh
make repl
```

## Parser Limitations

This tool only parses a small subset of Crystal, enough for the analysis it performs. Supported constructs include:

- **Classes** with an optional superclass, module inclusions and method definitions (`class Foo < Bar`)
- **Modules** with methods
- **Functions and methods** with typed/untyped arguments, default literal values, and `forall` type variables (with a very simple body)
- **Annotations** (e.g. `@[Slot]`)
- **Literals**: integers, strings (`"..."`), booleans (`true` / `false`)
- **Line comments** (`#`)

Function bodies are captured as raw text and are not parsed. Anything not recognized at the top level (modules, structs, macros, complex expressions, etc.) is passed through as-is and will not be rearranged.
`
