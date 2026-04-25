---
name: nix
description: "Use when working on projects involving Nix expressions, NixOS modules, flakes, or Nix packaging."
---

# Nix

Guidance for writing Nix expressions, NixOS modules, flakes, packages, overlays, and system/user configuration.

## References

- [Nix Reference Manual](https://nix.dev/manual/nix/latest/)
- [Nixpkgs Manual](https://nixos.org/manual/nixpkgs/stable/)
- [NixOS Manual](https://nixos.org/manual/nixos/stable/)
- [Nix Pills](https://nixos.org/guides/nix-pills/)
- [nix.dev Tutorials](https://nix.dev/tutorials/)
- [flake-parts Documentation](https://flake.parts/)
- [Dendritic Pattern](https://github.com/mightyiam/dendritic)
- [hjem Documentation](https://github.com/feel-co/hjem)
- [Nix RFCs](https://github.com/NixOS/rfcs)

## Code Quality Checks

Run in order:

```bash
# 1. Format code (check project for formatter first; default to alejandra)
alejandra .

# 2. Run all checks (statix, deadnix, and custom checks should be wired into flake checks)
nix flake check

# 3. Build outputs
nix build

# 4. Security scan (scan build output, not the development system)
vulnix result/                          # Scan nix-build output + transitive closure
vulnix /nix/store/<drv>.drv             # Scan a specific derivation
```

`statix check .` and `deadnix .` can be run directly for faster local iteration, but `nix flake check` is the authoritative lint runner — wire all lints into `checks` in your `flake.nix`.

## Project Structure

Canonical flake-based layout:

```
project-root/
├── flake.nix
├── flake.lock
├── modules/
│   ├── nixos/
│   │   ├── default.nix
│   │   └── services/
│   └── common/
├── packages/
│   └── my-package/
│       └── default.nix
├── overlays/
│   └── default.nix
├── homes/              # hjem user configs
│   └── user/
│       └── default.nix
├── hosts/
│   └── hostname/
│       ├── default.nix
│       └── hardware.nix
├── lib/
│   └── default.nix
└── README.md
```

## Testing

### Evaluation Tests

```bash
nix eval .#myConfig.value          # Evaluate a specific attribute
nix eval --json .#myConfig         # JSON output for inspection
```

### Build Tests

```nix
passthru.tests.version = runCommand "version-test" { } ''
  ${myPackage}/bin/my-app --version | grep "${version}"
  touch $out
'';
```

### NixOS VM Tests

```nix
nixos-lib.runTest {
  name = "my-service-test";
  nodes.machine = { pkgs, ... }: {
    services.myService.enable = true;
  };
  testScript = ''
    machine.wait_for_unit("my-service")
    machine.succeed("curl -f http://localhost:8080")
  '';
}
```

Run with: `nix build .#checks.x86_64-linux.myTest`

## Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Attribute names | camelCase | `buildInputs`, `enableService` |
| Package names | lowercase with hyphens | `my-package`, `hello-world` |
| Module options | camelCase, dot-separated path | `services.myApp.enable` |
| Files | lowercase with hyphens | `my-module.nix`, `default.nix` |
| Flake outputs | camelCase | `nixosConfigurations`, `devShells` |
| Variables | camelCase | `pkgs`, `lib`, `config` |
| Functions | camelCase | `mkDerivation`, `mkOption` |
| Boolean options | `enable` prefix | `services.nginx.enable` |

## Idioms

- Use `let`/`in` blocks for local bindings — never `rec { }`.
- Avoid `with pkgs;` at top level. Use narrow `with` only when it clearly improves readability (e.g., inside a list of packages).
- Use `lib` helpers: `mkIf`, `mkMerge`, `mkDefault`, `mkOption`, `mkEnableOption`.
- Use `callPackage` pattern for package definitions — it handles dependency injection.
- Use [flake-parts](https://flake.parts/) for new flakes — eliminates per-system boilerplate and provides modular flake structure.
- Use the [dendritic pattern](https://github.com/mightyiam/dendritic) for multi-configuration flakes — every file is a flake-parts module, use `deferredModule` type for lower-level configs. See `reference/dendritic.md`.
- Pin all flake inputs via `flake.lock`. Run `nix flake lock --update-input <input>` for targeted updates.
- Prefer `pkgs.writeShellApplication` over `pkgs.writeShellScriptBin` — it runs shellcheck automatically.
- Use `overrideAttrs` for modifying existing packages.
- Use `passthru` for package metadata and tests.
- Prefer `lib.optional`/`lib.optionals` over `if/then/else` in lists.
- Use `lib.attrsets` functions (`mapAttrs`, `filterAttrs`, `genAttrs`) for attribute set manipulation.
- Use `builtins.readFile` for including external files.
- Use `pkgs.formats.*` (`pkgs.formats.toml { }`, `pkgs.formats.yaml { }`, etc.) for generating config files.

## Module Patterns

Always separate `options` and `config`:

```nix
{ config, lib, pkgs, ... }:
let
  cfg = config.services.myApp;
in
{
  options.services.myApp = {
    enable = lib.mkEnableOption "myApp service";

    port = lib.mkOption {
      type = lib.types.port;
      default = 8080;
      description = "Port to listen on.";
    };

    settings = lib.mkOption {
      type = lib.types.submodule {
        options.logLevel = lib.mkOption {
          type = lib.types.enum [ "debug" "info" "warn" "error" ];
          default = "info";
        };
      };
    };
  };

  config = lib.mkIf cfg.enable {
    systemd.services.myApp = {
      wantedBy = [ "multi-user.target" ];
      serviceConfig.ExecStart = "${pkgs.myApp}/bin/myApp --port ${toString cfg.port}";
    };
  };
}
```

Key patterns:
- `mkEnableOption` for service toggles.
- `mkOption` with proper `lib.types.*` (`str`, `int`, `port`, `bool`, `listOf`, `attrsOf`, `submodule`, `enum`, `nullOr`, `oneOf`).
- `mkIf cfg.enable` to gate all configuration.
- `mkMerge` for combining multiple conditional configs.
- `mkDefault` for overridable defaults.
- `imports` for composing modules.

### Dendritic Pattern

For multi-configuration flakes (NixOS + darwin + hjem), use the [dendritic pattern](https://github.com/mightyiam/dendritic):

- Every Nix file is a flake-parts module of the top-level configuration.
- Each file implements a single feature across all configurations it applies to.
- Lower-level configs (NixOS, darwin, hjem) are stored as `deferredModule` option values.
- Share values between files via top-level `config` — no `specialArgs` pass-through.
- Use [import-tree](https://github.com/vic/import-tree) to auto-import all modules.

See `reference/dendritic.md` for full examples.

## Packaging

### Standard Derivation

```nix
{ lib, stdenv, fetchFromGitHub }:

stdenv.mkDerivation {
  pname = "my-package";
  version = "1.0.0";

  src = fetchFromGitHub {
    owner = "example";
    repo = "my-package";
    rev = "v${version}";
    hash = "sha256-AAAA...";
  };

  meta = {
    description = "A short description";
    homepage = "https://example.com";
    license = lib.licenses.mit;
    maintainers = [ lib.maintainers.username ];
    platforms = lib.platforms.linux;
  };
}
```

- Always use `pname` + `version`, never bare `name`.
- Use language-specific builders: `buildGoModule`, `buildPythonPackage`, `buildRustPackage`, `buildNpmPackage`.
- Always set `meta` with `description`, `license`, `maintainers`, `platforms`.
- Use `passthru.tests` for package tests.
- Use `nix-update` for version bumps.

### callPackage Pattern

```nix
# In flake.nix or overlay
my-package = pkgs.callPackage ./packages/my-package { };
```

`callPackage` auto-injects dependencies from `pkgs` matching function parameter names.

## Overlays

Define in `overlays/default.nix`:

```nix
final: prev: {
  my-package = final.callPackage ../packages/my-package { };

  existing-package = prev.existing-package.overrideAttrs (old: {
    patches = old.patches or [ ] ++ [ ./fix.patch ];
  });
}
```

- Use `final: prev:` convention (not `self: super:`).
- `final` = the fixed point (use for dependencies). `prev` = the previous layer (use for overriding).
- Keep overlays minimal — prefer upstream contributions.
- Composition order matters: later overlays override earlier ones.
- Use overlays for: version pinning, patching, adding local packages.

Wire into flake:

```nix
{
  overlays.default = import ./overlays;

  nixosConfigurations.myHost = nixpkgs.lib.nixosSystem {
    modules = [{
      nixpkgs.overlays = [ self.overlays.default ];
    }];
  };
}
```

## Development Shells

Use `devShells` for reproducible project development environments. Enter with `nix develop`.

### Basic devShell (flake-parts)

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable";
    flake-parts.url = "github:hercules-ci/flake-parts";
  };

  outputs = inputs:
    inputs.flake-parts.lib.mkFlake {inherit inputs;} {
      systems = ["x86_64-linux" "aarch64-linux" "aarch64-darwin" "x86_64-darwin"];

      perSystem = {pkgs, ...}: {
        devShells.default = pkgs.mkShellNoCC {
          packages = with pkgs; [
            go
            gopls
            golangci-lint
          ];

          env.GOPATH = "${builtins.getEnv "HOME"}/go";
        };
      };
    };
}
```

- Use `mkShellNoCC` unless a C compiler is needed — lighter and faster.
- Use `mkShell` when `buildInputs` / `nativeBuildInputs` are required (e.g., C libraries, pkg-config).
- Use `env` attribute for environment variables (cleaner than `shellHook` exports).
- Use `shellHook` sparingly — only for side effects like printing or generating files.
- flake-parts `perSystem` handles the per-system attribute set — no manual `system` threading.

### Multiple shells

```nix
perSystem = {pkgs, ...}: {
  devShells = {
    default = pkgs.mkShellNoCC { packages = with pkgs; [ go gopls ]; };
    ci = pkgs.mkShellNoCC { packages = with pkgs; [ go golangci-lint ]; };
    docs = pkgs.mkShellNoCC { packages = [ pkgs.mdbook ]; };
  };
};
```

Enter non-default shells with `nix develop .#ci`.

### Composing with project packages

```nix
perSystem = {pkgs, self', ...}: {
  devShells.default = pkgs.mkShell {
    inputsFrom = [ self'.packages.my-package ];
    packages = with pkgs; [ gopls delve ];
  };
};
```

`inputsFrom` pulls in all build inputs from an existing derivation — keeps the devShell in sync with the package's dependencies. Use `self'` (from flake-parts) to reference the current system's outputs.

## Anti-Patterns

| Avoid | Do Instead |
|---|---|
| `with pkgs;` at module top level | Qualify names: `pkgs.git`, or narrow `with` in lists |
| `rec { }` attribute sets | `let`/`in` bindings |
| Impure fetches without hashes | Flake inputs or fixed-output derivations |
| `nix-env -i` | Declarative config |
| `builtins.fetchTarball` without hash | Add `sha256` or use flake inputs |
| Mutable state in `/etc` outside Nix | Manage via NixOS modules |
| Home Manager when hjem suffices | Use hjem (see below) |
| `nixpkgs.config.allowUnfree = true` globally | Scope to specific packages |
| Ignoring `--show-trace` | Always use when debugging errors |
| Legacy `nix-shell`, `nix-build` | `nix develop`, `nix build` (flakes) |

## Dotfiles & Config Management

### hjem (Preferred)

**Always use hjem for user-level dotfiles and config.** It is lightweight, fast, and maps directly to file placement.

```nix
{ pkgs, lib, ... }: {
  hjem.users.taylor = {
    files = {
      ".config/git/config".source = ./git/config;

      ".config/starship.toml".text = ''
        [character]
        success_symbol = "[›](bold green)"
      '';

      ".config/app/config.toml".generator = let
        format = pkgs.formats.toml { };
      in format.generate "config.toml" {
        database.host = "localhost";
        database.port = 5432;
      };
    };

    xdg.config.files = {
      "alacritty/alacritty.toml".source = ./alacritty.toml;
    };

    xdg.data.files = {
      "applications/my-app.desktop".source = ./my-app.desktop;
    };

    environment.sessionVariables = {
      EDITOR = "nvim";
      VISUAL = "nvim";
    };
  };
}
```

File types: `source` (symlink, default), `text` (inline), `generator` (via `pkgs.formats.*`).

### Home Manager (Last Resort)

Home Manager is bloated, significantly slows builds, and often lags behind native application config changes. Use it **only** when you need its module abstractions (e.g., complex program modules with deep NixOS integration that hjem cannot replicate).

- [Standalone docs](https://nix-community.github.io/home-manager/)
- [NixOS module docs](https://nix-community.github.io/home-manager/index.xhtml#sec-flakes-nixos-module)

## Documentation

- Every module option must have a `description`.
- Every package must have `meta.description`.
- Complex modules should have a README in their directory.
- Flake `description` should be set in `flake.nix`.

## Dependencies & Inputs

- All external dependencies come through flake inputs.
- Pin nixpkgs to a specific revision via `flake.lock`.
- Use `follows` to deduplicate shared inputs:
  ```nix
  inputs.hjem.inputs.nixpkgs.follows = "nixpkgs";
  ```
- Minimize the number of flake inputs — each adds evaluation overhead.
- Use flake-parts for new flakes — `perSystem`, `self'`, and `inputs'` eliminate manual system threading.

## Performance Considerations

- Minimize import chains — deep `import` trees slow evaluation.
- Use `lib.mkMerge` sparingly in hot paths.
- Avoid `builtins.fetchGit` in frequently evaluated expressions.
- Use `nix eval` to profile evaluation time.
- Binary caches: configure `nix.settings.substituters` and `trusted-public-keys`.
- Use `nix build --dry-run` to check what needs building before committing to a full build.

## Troubleshooting

| Problem | Solution |
|---|---|
| Cryptic error | Add `--show-trace` to the command |
| Infinite recursion | Check for circular `imports` or `rec` usage; use `let`/`in` |
| Attribute not found | Verify input wiring, check `with` scope, use `nix repl` to inspect |
| Type mismatch | Check `lib.types.*` in module options |
| Build failure | `nix log /nix/store/<drv>` for build logs |
| Store corruption | `nix store verify --all` |
| Flake lock conflicts | `nix flake lock --update-input <input>` |
| Printf debugging | `builtins.trace`, `lib.traceVal`, `lib.traceValSeq` |
| Interactive debugging | `nix repl` then `:lf .` to load current flake |

## Security

- Scan build output and its transitive closure: `vulnix result/`
- Scan a specific derivation: `vulnix /nix/store/<drv>.drv`
- Scan all passed derivations without following requisites: `vulnix -R /nix/store/*.drv`
- Scan the full NixOS system (if needed): `vulnix --system`
- JSON output for CI/post-processing: `vulnix --json result/`
- Use whitelists to suppress known false positives: `vulnix -w whitelist.toml result/`
- Review `meta.license` on all dependencies.
- Use `nixpkgs.config.permittedInsecurePackages` explicitly rather than blanket `allowInsecure`.

## Design Principles

1. **Declarative over imperative** — describe what, not how.
2. **Reproducibility above all** — same inputs must yield same outputs.
3. **Composition over inheritance** — combine modules, don't subclass.
4. **Minimal abstraction** — don't over-abstract; Nix is already a DSL.
5. **Pin everything** — `flake.lock` is your friend.
6. **KISS** — simple Nix is maintainable Nix.
7. **DRY** — extract shared logic into `lib/`.
8. **YAGNI** — don't add options or modules until needed.

## Checklist Before Completion

- [ ] Code is formatted: `alejandra .`
- [ ] No lint issues: `statix check .`
- [ ] No dead code: `deadnix .`
- [ ] Flake checks pass: `nix flake check`
- [ ] All outputs build: `nix build`
- [ ] Security scan: `vulnix result/`
- [ ] No `with pkgs;` at module top level
- [ ] No `rec { }` attribute sets
- [ ] All modules separate `options` and `config`
- [ ] All packages have `meta` attributes
- [ ] Overlays use `final: prev:` convention
- [ ] hjem used for dotfiles (not Home Manager)
- [ ] All flake inputs are pinned
