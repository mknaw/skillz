---
name: nix-npm
description: Create Nix derivations for npm executables to include in system Nix config. Use when user asks about packaging npm/Node.js apps with Nix, buildNpmPackage, or npmDepsHash.
allowed-tools: Bash(nix *), Bash(npm *), Bash(curl *)
---

# Nix Derivations for npm Executables

Use `buildNpmPackage` to package npm applications as derivations for inclusion in system Nix (home-manager, NixOS config, etc). The standard approach fetches the pre-built tarball from the npm registry, avoiding the need to build from source.

## Basic Derivation

```nix
{ buildNpmPackage
, fetchzip
, nodejs_20
,
}:

buildNpmPackage rec {
  pname = "myapp";
  version = "1.0.0";

  nodejs = nodejs_20;

  src = fetchzip {
    url = "https://registry.npmjs.org/myapp/-/myapp-${version}.tgz";
    hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
  };

  npmDepsHash = "sha256-BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=";

  dontNpmBuild = true;

  postPatch = ''
    cp ${./package-lock.json} package-lock.json
  '';
}
```

## Step-by-Step Workflow

### 1. Find the package version

Check the npm registry for the latest version and tarball URL:

```
https://registry.npmjs.org/<package-name>/latest
```

For scoped packages (e.g. `@org/pkg`), the tarball URL pattern is:

```
https://registry.npmjs.org/@org/pkg/-/pkg-${version}.tgz
```

### 2. Generate package-lock.json

`buildNpmPackage` requires a `package-lock.json`. Most npm tarballs don't ship one, so generate it from the tarball:

```bash
cd /tmp && rm -rf lockgen && mkdir lockgen && cd lockgen
curl -sL https://registry.npmjs.org/myapp/-/myapp-1.0.0.tgz | tar xz --strip-components=1
npm install --package-lock-only
```

Copy the resulting `package-lock.json` into the derivation directory alongside `package.nix`.

### 3. Get the fetchzip hash

```bash
nix-prefetch-url --unpack "https://registry.npmjs.org/myapp/-/myapp-1.0.0.tgz"
# Convert to SRI format:
nix hash convert --hash-algo sha256 --to sri <hash>
```

### 4. Get the npmDepsHash

Set `npmDepsHash = "";` and run `nix-build`. The error message will contain the correct hash:

```bash
nix-build -E 'let pkgs = import <nixpkgs> {}; in pkgs.callPackage ./package.nix {}'
```

Copy the `got: sha256-...` value from the error output.

### 5. Verify the build

Run `nix-build` again with both hashes filled in. Then test the binary:

```bash
./result/bin/myapp --version
```

## Key Concepts

| Option | Description |
|--------|-------------|
| `nodejs` | Pin Node.js version (`nodejs_20` recommended for Darwin sandbox compat) |
| `dontNpmBuild` | Skip `npm run build` — use when fetching pre-built tarballs from the registry |
| `npmDepsHash` | Hash of fetched npm dependencies (analogous to Go's `vendorHash`) |
| `postPatch` | Inject the generated `package-lock.json` before the build |
| `npmFlags` | Extra flags passed to npm commands |
| `makePkgConfigItem` | Build native dependencies |

## Custom Install Phase

If the default `npmInstallHook` doesn't correctly wire up the binary (e.g. for MCP servers or packages with unusual `bin` entries), use a custom `installPhase`:

```nix
installPhase = ''
  runHook preInstall

  mkdir -p $out/lib/node_modules/myapp
  cp -r . $out/lib/node_modules/myapp/
  cd $out/lib/node_modules/myapp

  npm ci --omit=dev --omit=optional --ignore-scripts

  mkdir -p $out/bin
  ln -s $out/lib/node_modules/myapp/dist/index.js $out/bin/myapp

  runHook postInstall
'';
```

This is needed when:
- The package has no `bin` field in `package.json`
- The `bin` entry points to a file that needs to be symlinked manually
- You need to run `npm ci` yourself to control dependency installation

## Scoped Packages

For scoped packages like `@upstash/context7-mcp`, the tarball URL follows this pattern:

```nix
src = fetchzip {
  url = "https://registry.npmjs.org/@upstash/context7-mcp/-/context7-mcp-${version}.tgz";
  hash = "...";
};
```

Note: the tarball filename does **not** include the scope prefix.

## Adding meta

```nix
meta = {
  description = "Short description";
  homepage = "https://github.com/org/repo";
  license = "MIT";
  maintainers = [ ];
  platforms = [ "x86_64-linux" "aarch64-linux" "x86_64-darwin" "aarch64-darwin" ];
  mainProgram = "myapp";
};
```

## Troubleshooting

**npmDepsHash mismatch**: Dependencies changed upstream. Re-run with empty hash to get the new value.

**No package-lock.json**: The tarball doesn't include one. Generate it with `npm install --package-lock-only` from the extracted tarball contents.

**Darwin sandbox errors**: Use `nodejs_20` explicitly — some Node versions have issues with macOS sandboxed builds.

**Binary not found after build**: Check the `bin` field in `package.json`. If it's missing or unusual, use a custom `installPhase` to create the symlink manually.

**Build step needed**: If the npm tarball doesn't contain pre-built `dist/`, remove `dontNpmBuild = true` and ensure build dependencies are available. This is rare for registry tarballs — they typically include build artifacts.

**pnpm/yarn projects**: When fetching from the npm registry, the tarball works with `buildNpmPackage` regardless of what package manager the project uses for development. The generated `package-lock.json` handles dependency resolution.
