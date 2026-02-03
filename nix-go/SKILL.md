---
name: nix-go
description: Create Nix derivations for Go executables to include in system Nix config. Use when user asks about packaging Go apps with Nix, buildGoModule, or vendorHash.
allowed-tools: Bash(nix *), Bash(go *)
---

# Nix Derivations for Go Executables

Use `buildGoModule` to package Go applications as derivations for inclusion in system Nix (home-manager, NixOS config, etc).

## Basic Derivation

```nix
{ pkgs ? import <nixpkgs> {} }:

pkgs.buildGoModule {
  pname = "myapp";
  version = "0.1.0";
  src = pkgs.fetchFromGitHub {
    owner = "example";
    repo = "myapp";
    rev = "v0.1.0";
    hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
  };

  vendorHash = "sha256-BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=";
}
```

## Getting vendorHash

The `vendorHash` is the hash of fetched Go module dependencies.

1. Set `vendorHash = "";` or `vendorHash = pkgs.lib.fakeHash;`
2. Run `nix-build` or `nix build`
3. Copy the correct hash from the error message

If using vendored dependencies (`go mod vendor`), set `vendorHash = null;`.

## Common Options

| Option | Description |
|--------|-------------|
| `vendorHash` | Hash of dependencies (use `null` if vendored) |
| `subPackages` | List of packages to build (e.g., `["cmd/myapp"]`) |
| `ldflags` | Linker flags (e.g., `["-s" "-w" "-X main.version=${version}"]`) |
| `tags` | Build tags (e.g., `["netgo" "osusergo"]`) |
| `CGO_ENABLED` | Set to `0` to disable cgo |
| `doCheck` | Set to `false` to skip tests |
| `preBuild` | Commands to run before building |
| `postInstall` | Commands to run after installing |

## Advanced Example

```nix
{ pkgs ? import <nixpkgs> {} }:

pkgs.buildGoModule rec {
  pname = "myapp";
  version = "1.0.0";
  src = pkgs.fetchFromGitHub {
    owner = "example";
    repo = "myapp";
    rev = "v${version}";
    hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
  };

  vendorHash = "sha256-BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=";
  subPackages = [ "cmd/myapp" "cmd/mytool" ];

  ldflags = [
    "-s" "-w"
    "-X main.version=${version}"
  ];

  CGO_ENABLED = 0;

  nativeBuildInputs = [ pkgs.installShellFiles ];

  postInstall = ''
    installShellCompletion --cmd myapp \
      --bash <($out/bin/myapp completion bash) \
      --zsh <($out/bin/myapp completion zsh)
  '';
}
```

## Static Binaries

```nix
CGO_ENABLED = 0;
ldflags = [ "-s" "-w" "-extldflags=-static" ];
tags = [ "netgo" "osusergo" ];
```

## Cross-Compilation

```nix
GOOS = "linux";
GOARCH = "arm64";
CGO_ENABLED = 0;
```

## Troubleshooting

**vendorHash mismatch**: Dependencies changed. Re-run with empty hash to get new value.

**Module not found**: Ensure `go.mod` and `go.sum` are committed and up to date.

**CGO issues**: Set `CGO_ENABLED = 0` if not using cgo, or add required native deps to `buildInputs`.

**Tests fail**: Use `doCheck = false;` to skip, or add test deps to `nativeCheckInputs`.

**Private modules**: Use `GOPRIVATE` env var and ensure git credentials are available.
