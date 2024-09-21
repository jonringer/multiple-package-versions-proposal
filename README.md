---
Title: Allow for many versions to be exposed by a package
Author: jonringer
Discussions-To: 
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2024-9-19
---

# Summary

Generally multiple versions are introduced by adding a suffix to similar package
(e.g. `ffmpeg_4`), and generally a default is exposed for the prefered package
version to be used across the package set. However, this is less than ideal
as you pollute a package scope with multiple variants which are treated as
separate packages in terms of overriding (e.g. overlays), deprecation policies,
and it is unintuitive to users how packages expose these versions (e.g. `ffmpeg_4` vs `nixVersions.nix_2_18`).

## Detailed Implementation

Expose `mkGenericPkg` which allows for constructing this version scheme in an uniform style.
`mkGenericPkg` in this fashion can be thought of as a partially applied function, where
version information is passed as one set of arguments before passing the nix expression
to callPackage.

Example PR: https://github.com/jonringer/core-pkgs/pull/5

```nix
{
  ... 
  # top-level.nix
  mkGenericPkg = callPackage ./build-support/mkgeneric { };
  ... 
}
```

```nix
# build-support/mkgeneric/default.nix

{ lib, config }:

{
# Intended to be an attrset of { "<exposed version>" = { version = "<full version>"; src = <path>; } }
# or a file containing such version information
# Type: AttrSet AttrSet
versions,

# Similar to versions, but instead contain deprecation and removal messages
# Only added when `config.allowAliases` is true
# This is passed the versions attr set to allow for directly referencing the version entries
# Type: AttrSet AttrSet -> AttrSet AttrSet.
aliases ? { ... }: { },

# A "projection" from the version set to a version to be used as the default
# Type: AttrSet package -> package
defaultSelector,

# Nix expression which takes version and package args, and returns an attrset to pass to mkDerivation
# Type: AttrSet -> AttrSet -> AttrSet
genericBuilder,
}:

# Some assertions as poor man's type checking
assert builtins.isFunction defaultSelector;

let
  versionsRaw = if builtins.isPath versions then import versions else versions;
  aliasesExpr = if builtins.isPath aliases then import aliases else aliases;
  genericExpr = if builtins.isPath genericBuilder then import genericBuilder else genericBuilder;

  aliases' = aliasesExpr { inherit lib; versions = versionsRaw; };
  versions' = if config.allowAliases then
      # Not sure if aliases or versions should have priority
      versionsRaw // aliases'
    else versionsRaw;

  # This also allows for additional attrs to be passed through besides version and src
  mkVersionArgs = { version, ... }@args: args // rec {
    # Some helpers commonly used to determine packaging behavior
    packageOlder = lib.versionOlder version;
    packageAtLeast = lib.versionAtLeast version;
    packageBetween = lower: higher: packageAtLeast lower && packageOlder higher;
    mkVersionPassthru = packageArgs: let
      versions = builtins.mapAttrs (_: v: mkPackage v packageArgs) versions';
    in versions // { inherit versions; };
  };

  # Re-call the generic builder with new version args, re-wrap with makeOverridable
  # to give it the same appearance as being called by callPackage
  mkPackage = version: lib.makeOverridable (genericExpr (mkVersionArgs version));
in
  # The partially applied function doesn't need to be called with makeOverridable
  # As callPackage will be wrapping this in makeOverridable as well
  genericExpr (mkVersionArgs (defaultSelector versions'))
```

## Openssl example

```diff
   # top-level.nix, pkgs/openssl/default.nix is called automatically

   # Aliasing from nixpkgs
-  openssl = openssl_3;
-  inherit (callPackages ./pkgs/openssl { })
-    openssl_1_1
-    openssl_3
-    openssl_3_2
-    openssl_3_3
-    ;
+  # Aliases for backwards compat
+  openssl_1_1 = openssl.v1_1;
+  openssl_3 = openssl.v3_0;
+  openssl_3_2 = openssl.v3_2;
+  openssl_3_3 = openssl.v3_3;
```

```nix
# pkgs/openssl/versions.nix
{
  v1_1 = {
    version = "1.1.1w";
    hash = "sha256-zzCYlQy02FOtlcCEHx+cbT3BAtzPys1SHZOSUgi3asg=";
  };
  ...
}
```

```nix
# pkgs/openssl/default.nix
{ callPackage
, mkGenericPkg
, ...
}@args:

# TODO: future proposal, make this less ugly
callPackage (mkGenericPkg {
  versions = ./versions.nix;
  aliases = ./aliases.nix;
  defaultSelector = (p: p.v3_3);
  genericBuilder = ./generic.nix;
}) args
```

New generic.nix:
```nix
# pkgs/openssl/generic.nix
{ version
, hash
, packageOlder
, packageAtLeast
, packageBetween
, mkVersionPassthru
, ...
}:

# Former packaging arguments:
{ lib
, stdenv
...
}@args:

stdenv.mkDerivation {
  ...
  passthru = (mkVersionPassthru args ) // {
    ... # existing args
  };
}
```

Usage of openssl from top-level:
```
nix-repl> openssl     
«derivation /nix/store/ayvv16slswqfps7c1zql8rml5nla6zky-openssl-3.3.1.drv»

nix-repl> openssl.v3_2 
«derivation /nix/store/77794w2h7xslv02k54xxc8x7krc1aazg-openssl-3.2.2.drv»

# Overriding still works
nix-repl> openssl.v3_2.override { withDocs = false; }
«derivation /nix/store/2196b3bj4vk2p6i7f3ai3p0n1m5im5iz-openssl-3.2.2.drv»

# To build all versions, need to pass `allowAliases = false;` to avoid eval error
nix-build -A openssl.versions
```

## Unresolved issues

- Argument passing is ugly
  - This is caused by us trying to satisfy mkGenericpkg and the package's arguments with the same arguments
  - This could be improved by importing default.nix with just mkGenericPkg, and passing the partially applied mkGenericPkg function to callPackage
    - E.g. `openssl = callPackage (import ./pkgs/openssl { inherit mkGenericpkg; }) { }; 
    - Similar to auto call-by-name, we could also create another directory which would just auto call generic packages, TBD in another proposal.

```nix
# pkgs/openssl/default.nix, with the result of this partially applied function
# being passed to callPackage
{ mkGenericPkg }:

mkGenericPkg {
  versions = ./versions.nix;
  aliases = ./aliases.nix;
  defaultSelector = (p: p.v3_3);
  genericBuilder = ./generic.nix;
}
```

- Instead of `passthru.versions`, should it be `passthru.variants`?
  - Although I use openssl's version as a parameter, build arguments could also be passed in versions.nix. E.g. `withDocs`
- Should you be able to recurse versions? E.g. `openssl.v3_2.v3_1` (returns v3.1)
- Add `.overrideVersion`?
  - Unsure if having yet-another way to `.overrideX` is worth it
  - In most cases, this should be the same as doing `drv.overrideAttrs { version = ...; src = ...; }`
  - However, this would apply version information before the expression gets called with package arguments
    - This allows for logic which references version in many locations to work without use of mkDerivation fixed points. E.g. `(finalAttrs: { ... })`

## Future work

- Mitigate argument passing awkwardness with mkAutoCalledGenericDir proposal (TBD).
- Provide updater-script for use with generic packages

## Meta concerns

- This potentially shifts a lot of logic (version and deprecation aliases) from what was previously "package sets concerns" into a "package concern".
- Hard divergent behavior from upstream nixpkgs, many expressions will be less interchangable
