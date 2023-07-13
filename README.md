---
feature: Evaluation purity and caching builtins
start-date: 2023-05-10
author: Silvan Mosberger
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Make convenient pure evaluation and evaluation caching available to Nix expressions and the traditional CLI.

This allows detangling Flakes from Nix evaluation, making it possible to implement Flakes as a layer on top of a non-Flakes CLI.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

RFC 140 can use the pure eval

# Detailed design
[design]: #detailed-design

- Maybe enforce that builtins cannot be overridden. Could be done only in pure evaluation, though it probably doesn't hurt to do it always. Or maybe only if `builtins.pureEval` is `true`, it cannot be set to `false`, should be enough
- Add `builtins.allowedPaths`, some structure that can be queried for which paths are allowed
  By default set to all paths, impure, or maybe not set by default, along with `builtins.pure = false`
- Add:
  ```nix
  builtins.pureImport {
    allowedPaths = builtins.gitTracked ./.;
    allowedPaths = ...; # optional, if not passed, no paths are allowed to be accessed
    currentSystem = ...; # optional, if not passed the builtin is disabled
    nixPath = ...; # optional, if not passed no nix paths can be resolved
  } file
  ```

  Imports and evaluates `file` using the given pure environment configuration, exporting all of these variables under `builtins`, also setting `builtins.pure = true` or so
- Evaluation makes sure that `builtins.allowedPaths` is checked before accessing a path
- Add `builtins.cachedImport file args`, as described below, ensures that `builtins.pure = true` and uses all of the variable `builtins` values as a hash key, in addition to the `args`
- `builtins.lazyUpdate`

## New Nix Builtins



`builtins.allowedPaths`



### `builtins.lazyUpdate`

Type: `Attrs -> Attrs -> Attrs`

And maybe its cousin: `builtins.lazyListConcat`

`builtins.lazyUpdate a b` is semantically similar to `a // b`.
The difference is that when accessing an attribute of the result, if the attribute contained in `b`, `a` is not evaluated.
This works:

```nix
(builtins.lazyUpdate (throw "error") { x = null; }).x
```

This is effectively https://github.com/NixOS/nix/issues/4090 and has many applications.

### `builtins.cachedImport`

Type: `Path -> Any -> Any`

`builtins.cachedImport file args` is semantically similar to `import file args`.
This builtin can only be used when pure evaluation mode is enabled.
`args` and the result needs to be serializable.
The result of the evaluation is cached under a combined key of:
- The hash of all paths that can be accessed by pure evaluation mode, which will have to include `file` itself.
- The hash of the serialization of `args`

### `builtins.pureImport`

Type: `{ (optional) nixPath :: AttrsOf Path, allowedPaths :: [ Path ] } -> Path -> Any`

```nix
builtins.pureImport {
  nixPath.nixpkgs = ...;
  root = ./.;
  allowedPaths = [
    ./foo.nix
  ];
} file
```
is semantically similar to `import file`.
However the file is evaluated with pure evaluation restricted to the paths specified in `allowedPaths`.
This means that if `file` were to access `./bar.nix` it would fail, but `./foo.nix` would work.
Also, `./foo.nix` would evaluate to just `/foo.nix`.

### `builtins.gitTracked :: Path -> [ Path ]`

Like https://github.com/NixOS/nix/issues/2944

## Nix CLI changes

When pure evaluation mode is enabled, `nix-build` evaluates to something nice using both `pureEval` and `cacheEval`.

Look for `pure.nix` or so



```
let
  rev = "...";
  sha256 = "...";
  pkgs = import <nixpkgs> {};
  x = builtins.cache "x" [ pkgs ] pkgs.hello;
  # No point, we need to always evaluate the entire `pkgs` to figure out whether we hit the cache
  y = builtins.cache (
    (import (fetch { inherit rev sha256; }) {}).hello.outPath
  );
  # Cache key is composed of `rev`, `sha256` and the hash of the expression
  # Evaluation is performed such that the expression can't access any variables outside

  # The free variables become the cache key!

  z = builtins.cache (builtins.foldl' (acc: el: acc + el) 0 (range 0 1000000));
  # This would also work!!
in
```

`builtins.cache :: FileSet -> a -> a`

Evaluates `a`, caches it using the following as keys:
- The hash of all files in the file set
- All free variables in the second argument

All keys and the return value itself must be serializable

Serializable values are:
- `null`, booleans, integers, floats
- Strings (also with context), paths
- Attribute sets and lists of serializable values

Not serializable are:
- Functions

Problem: We can't really sell filesets to Nix

How about this instead:

```
builtins.withFilesystem {
  "flake.nix" = ./flake.nix;
  src = {};
} foo
```


`builtins.pureEval :: { nixPath :: AttrsOf Path, allowedPaths :: NestedAttrsOf Path, allowedURIs :: [ String ] } -> a -> a`
Evaluates the second argument with a filesystem according to the first argument

Could also allow ETAG header fetching to use as a key, probably also what flakes does

Expression must not have any free variables

The first argument must also be evaluated with pure eval mode.

How about having a file `pureEval.nix`. If it exists it will become the arguments to `builtins.pureEval`, and evaluate `default.nix`

It's a good thing to only be able to access local files

How about a `root.nix`. It indicates the root of the Nix flake. Only nested files can be accessed.

`builtins.cacheExpr :: Scope -> Expression -> Value`

```nix
builtins.cacheExpr { x = 10; } x
```

Can only refer to variables introduced in the scope

This requires being nested in a `builtins.withFilesystem`, because otherwise it would cache the entire filesystem, which is too much!

Aka, requires restricted eval

Also requires pure evaluation

```nix
builtins.pureEval {
  nixPath.nixpkgs = <nixpkgs>;
} (import ./pure.nix)
```

`builtins.pureEval :: a -> a`

Requires restricted eval
I guess this should require the expression to not have any free variables?

Should maybe work the same as `builtins.cache`: Free variables become part of the input
But how to evaluate those? In pure eval or not?

```
builtins.withFilesystem (builtins.gitTracked ./.) (
  let
    
  in
)
```


# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

This section illustrates the detailed design. This section should clarify all
confusion the reader has from the previous sections. It is especially important
to counterbalance the desired terseness of the detailed design; if you feel
your detailed design is rudely short, consider making this section longer
instead.

Example: Caching top-level derivations for a fetched nixpkgs:
```nix
# nixpkgs.nix
{ nixpkgsRev, nixpkgsSha256, system }:
let
  nixpkgs = fetchTarball {
    url = "https://github.com/NixOS/nixpkgs/tarball/${nixpkgsRev}";
    sha256 = nixpkgsSha256;
  };
in import nixpkgs { inherit system; }
```

```nix
# eval.nix
{ nixpkgsSpec, attribute }:
let
  pkgs = import ./nixpkgs.nix nixpkgsSpec;
in {
  # We don't want all attributes because that includes things like `buildInputs`
  inherit (pkgs.${attribute})
    type
    drvPath
    outPath
    outputs
    outputName;
}
```

```nix
# pure.nix
let
  nixpkgsSpec = {
    nixpkgsRev = "...";
    nixpkgsSha256 = "...";
    system = "...";
  };

  pkgs = import ./nixpkgs.nix nixpkgsSpec;

  cachedPkgs = mapAttrs (name: value:
    let
      cached = builtins.cachedImport ./eval.nix {
        inherit nixpkgsRev nixpkgsSha256 system;
        attribute = name;
      };
    in
    builtins.lazyUpdate
      value
      cached
  ) pkgs;
in cachedPkgs
```

```nix
# default.nix
# `builtins.cachedImport` is only allowed in pure evaluation mode
builtins.pureImport {
  # This is not strictly necessary, it limits access of ./pure.nix to just two files
  # But it does mean that changes to other files won't end up influencing whether it's a cache hit
  allowedPaths = {
    "eval.nix" = ./eval.nix;
    "nixpkgs.nix" = ./nixpkgs.nix;
  };
} ./pure.nix
```

Then calling `nix-build -A hello` will be very fast since it won't have to compute the `outPath`. However if `hello.meta` is used, it will be slow since `eval.nix` doesn't output `meta` for it to be cached. This could be added though.

The cache keys will be:
- The hash of `./eval.nix`, since they're the only files available to pure evaluation mode
- nixpkgsRev, nixpkgsSha256, system and attribute

It's not great how this requires multiple files. And using `builtins.writeFile "*.nix"` is weird, though it would work with a "single" file then.



Further ideas that could be implemented:
- Caching of all Nixpkgs packages within nixpkgs itself

- RFC 140

The `builtins.pureImport` primitive could provide an interface to local [lazy paths](https://github.com/NixOS/nix/pull/6530):
```nix
path: builtins.pureImport { root = path; allowedPaths = [ path ]; } (builtins.toFile "root" "/.")
```

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD or unknowns?

# Future work
[future]: #future-work

What future work, if any, would be implied or impacted by this feature
without being directly part of the work?
