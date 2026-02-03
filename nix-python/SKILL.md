---
name: nix-python
description: Create Nix derivations for Python executables to include in system Nix config. Use when user asks about packaging Python apps with Nix, buildPythonApplication, buildPythonPackage, or uv2nix.
allowed-tools: Bash(nix *), Bash(uv *)
---

# Nix Derivations for Python Executables

Use nixpkgs builders to package Python applications as derivations for inclusion in system Nix (home-manager, NixOS config, etc).

## buildPythonApplication (for CLI tools)

```nix
{ pkgs ? import <nixpkgs> {}, python3Packages ? pkgs.python3Packages }:

python3Packages.buildPythonApplication {
  pname = "myapp";
  version = "1.0.0";
  src = pkgs.fetchFromGitHub {
    owner = "example";
    repo = "myapp";
    rev = "v1.0.0";
    hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
  };
  pyproject = true;

  build-system = [ python3Packages.setuptools ];
  dependencies = [
    python3Packages.click
    python3Packages.requests
  ];
}
```

## buildPythonPackage (for libraries)

```nix
{ pkgs ? import <nixpkgs> {}, python3Packages ? pkgs.python3Packages }:

python3Packages.buildPythonPackage {
  pname = "mylib";
  version = "1.0.0";
  src = pkgs.fetchFromGitHub {
    owner = "example";
    repo = "mylib";
    rev = "v1.0.0";
    hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
  };
  pyproject = true;

  build-system = [ python3Packages.setuptools ];
  propagatedBuildInputs = [
    python3Packages.numpy
  ];
}
```

## Key Differences

| Feature | buildPythonPackage | buildPythonApplication |
|---------|-------------------|------------------------|
| Modules in buildEnv | Yes | No |
| Python version prefix | Yes | No |
| propagatedBuildInputs inherited | Yes | No |
| Use case | Libraries | CLI executables |

## Common Options

- `pyproject = true` - Use pyproject.toml (PEP 517)
- `format = "wheel"` - Install from wheel
- `build-system` - Build-time deps (setuptools, hatchling, etc.)
- `dependencies` - Runtime deps (preferred over propagatedBuildInputs)
- `nativeCheckInputs` - Test deps (pytest, etc.)
- `doCheck = false` - Skip tests

## uv2nix (for uv-based projects)

For projects using `uv` with a `uv.lock` file, use uv2nix instead of manually specifying dependencies. It reads the lockfile directly and handles complex dependency trees automatically.

```nix
{
  description = "My Python application";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

    pyproject-nix = {
      url = "github:pyproject-nix/pyproject.nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    uv2nix = {
      url = "github:pyproject-nix/uv2nix";
      inputs.pyproject-nix.follows = "pyproject-nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    pyproject-build-systems = {
      url = "github:pyproject-nix/build-system-pkgs";
      inputs.pyproject-nix.follows = "pyproject-nix";
      inputs.uv2nix.follows = "uv2nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs =
    {
      nixpkgs,
      pyproject-nix,
      uv2nix,
      pyproject-build-systems,
      ...
    }:
    let
      inherit (nixpkgs) lib;
      forAllSystems = lib.genAttrs lib.systems.flakeExposed;

      workspace = uv2nix.lib.workspace.loadWorkspace { workspaceRoot = ./.; };

      overlay = workspace.mkPyprojectOverlay {
        sourcePreference = "wheel";
      };

      pythonSets = forAllSystems (
        system:
        let
          pkgs = nixpkgs.legacyPackages.${system};
          python = pkgs.python312;
        in
        (pkgs.callPackage pyproject-nix.build.packages {
          inherit python;
        }).overrideScope
          (
            lib.composeManyExtensions [
              pyproject-build-systems.overlays.wheel
              overlay
            ]
          )
      );

    in
    {
      packages = forAllSystems (system: {
        default = pythonSets.${system}.mkVirtualEnv "myapp-env" workspace.deps.default;
      });

      apps = forAllSystems (system: {
        default = {
          type = "app";
          program = "${pythonSets.${system}.mkVirtualEnv "myapp-env" workspace.deps.default}/bin/myapp";
        };
      });

      devShells = forAllSystems (
        system:
        let
          pkgs = nixpkgs.legacyPackages.${system};
          editableOverlay = workspace.mkEditablePyprojectOverlay {
            root = "$REPO_ROOT";
          };
          pythonSet = pythonSets.${system}.overrideScope editableOverlay;
          virtualenv = pythonSet.mkVirtualEnv "myapp-dev-env" workspace.deps.all;
        in
        {
          default = pkgs.mkShell {
            packages = [
              virtualenv
              pkgs.uv
            ];
            env = {
              UV_NO_SYNC = "1";
              UV_PYTHON = pythonSet.python.interpreter;
              UV_PYTHON_DOWNLOADS = "never";
            };
            shellHook = ''
              unset PYTHONPATH
              export REPO_ROOT=$(git rev-parse --show-toplevel)
            '';
          };
        }
      );
    };
}
```

### uv2nix Key Points

- Reads `uv.lock` directly - no manual dependency specification
- `sourcePreference = "wheel"` is more reliable than `"sdist"`
- Handles hatch-vcs and other build systems that need git history
- `workspace.deps.default` = runtime deps, `workspace.deps.all` = including dev deps

### uv2nix Usage

```bash
nix build              # Build package
nix run                # Run the app
nix develop            # Enter dev shell with editable install
```

### When to Use What

| Scenario | Approach |
|----------|----------|
| Project has `uv.lock` | uv2nix flake |
| Simple app, few deps in nixpkgs | buildPythonApplication |
| Library to publish | buildPythonPackage |
| Many deps not in nixpkgs | uv2nix or poetry2nix |

## Troubleshooting

**Missing pkg_resources**: Add `setuptools` to dependencies.

**No setup.py**: Set `pyproject = true` or generate in `preBuild`.

**Binary wheels**: Use `format = "wheel"` with `fetchPypi`.

**hatch-vcs version errors**: Use uv2nix, or set `env.SETUPTOOLS_SCM_PRETEND_VERSION = version;` in buildPythonApplication.
