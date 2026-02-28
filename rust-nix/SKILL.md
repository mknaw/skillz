---
name: rust-nix
description: Create Nix derivations for existing Rust projects from GitHub or crates.io
argument-hint: [github-url or crate-name]
allowed-tools: Read, Write, Edit, Bash, WebFetch, Grep, Glob
---

Create a `default.nix` derivation for an existing Rust project using `buildRustPackage`.

## Gathering Project Info

If given a GitHub URL or repo reference:
- Extract owner, repo, and latest release/tag
- Check if there's a `Cargo.lock` in the repo (required)
- Look for native dependencies in `Cargo.toml` (openssl, sqlite, etc.)

If given a crate name:
- Look up the crate on crates.io to find the source repository
- Follow the GitHub URL from there

## Basic Template

```nix
{ pkgs ? import <nixpkgs> { } }:

pkgs.rustPlatform.buildRustPackage rec {
  pname = "PACKAGE_NAME";
  version = "VERSION";

  src = pkgs.fetchFromGitHub {
    owner = "OWNER";
    repo = "REPO";
    rev = "TAG_OR_REV";  # e.g., "v${version}" or just version
    hash = "";  # leave empty first - nix-build will report correct hash
  };

  cargoHash = "";  # leave empty first - nix-build will report correct hash

  meta = {
    description = "DESCRIPTION";
    homepage = "https://github.com/OWNER/REPO";
  };
}
```

## Hash Discovery Workflow

1. Leave `hash` and `cargoHash` as empty strings `""`
2. Run `nix-build default.nix`
3. Nix fails with: `hash mismatch ... got: sha256-XXXX`
4. Copy the correct hash into `fetchFromGitHub.hash`
5. Run `nix-build` again - it fails with the correct `cargoHash`
6. Fill that in and build succeeds

Alternative: use `pkgs.lib.fakeHash` instead of `""` for the same effect.

## Git Dependencies

If the project has git dependencies in `Cargo.toml`, use `cargoLock` instead of `cargoHash`:

```nix
cargoLock = {
  lockFile = "${src}/Cargo.lock";
  outputHashes = {
    # nix-build will tell you which git deps need hashes
    # "some-git-dep-0.1.0" = "sha256-XXXX";
  };
};
```

Do NOT set `cargoHash` when using `cargoLock`.

## Native Dependencies

Common patterns for projects that need system libraries:

```nix
nativeBuildInputs = with pkgs; [
  pkg-config      # almost always needed when buildInputs has libs
  cmake           # if project uses cmake
  perl            # sometimes needed for openssl
];

buildInputs = with pkgs; [
  openssl         # for networking/TLS
  sqlite          # for sqlite
  zlib            # compression
  libgit2         # git operations
  libssh2         # SSH support
] ++ lib.optionals stdenv.isDarwin [
  darwin.apple_sdk.frameworks.Security
  darwin.apple_sdk.frameworks.SystemConfiguration
];
```

## Skipping Tests

If tests require network or other unavailable resources:

```nix
doCheck = false;
```

## Example Output

For `https://github.com/BurntSushi/ripgrep`:

```nix
{ pkgs ? import <nixpkgs> { } }:

pkgs.rustPlatform.buildRustPackage rec {
  pname = "ripgrep";
  version = "14.1.1";

  src = pkgs.fetchFromGitHub {
    owner = "BurntSushi";
    repo = "ripgrep";
    rev = version;
    hash = "sha256-gyWnahj1A+iXUQlQ1O1H1u7K5euYQOld9qWm99Vjaeg=";
  };

  cargoHash = "sha256-9atn5qyBDy4P6iUoHFhg+TV6Ur71fiah4oTJbBMeEy4=";

  meta = {
    description = "Fast line-oriented regex search tool";
    homepage = "https://github.com/BurntSushi/ripgrep";
  };
}
```

## Including in System Config

```nix
# configuration.nix
{ pkgs, ... }:
let
  my-tool = pkgs.callPackage ./my-tool.nix { };
in {
  environment.systemPackages = [ my-tool ];
}
```

## Checklist

- [ ] Verify the project has a `Cargo.lock` committed
- [ ] Check for git dependencies in `Cargo.toml` (use `cargoLock` if present)
- [ ] Identify native library dependencies
- [ ] Add Darwin frameworks if needed for macOS
- [ ] Test with `nix-build default.nix`
- [ ] Verify binary runs: `./result/bin/BINARY_NAME --version`
