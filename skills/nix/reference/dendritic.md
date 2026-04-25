# Dendritic Pattern

The [dendritic pattern](https://github.com/mightyiam/dendritic) is a Nixpkgs module system usage pattern for organizing multi-configuration flakes. It uses flake-parts as the top-level configuration, and every Nix file is a module of that top-level configuration.

## Core Principles

1. **Every file is a flake-parts module** — no separate NixOS/darwin/hjem module files.
2. **One file = one feature** — each file implements a feature across all configuration classes it applies to.
3. **File paths name features** — paths are meaningful to the author, not to the system. Files can be freely renamed, moved, or split.
4. **`deferredModule` for lower-level configs** — NixOS, darwin, hjem modules are stored as option values using `lib.types.deferredModule`.
5. **No `specialArgs` pass-through** — share values between files via the top-level `config`.

## Flake Entry Point

```nix
{
  inputs = {
    flake-parts = {
      url = "github:hercules-ci/flake-parts";
      inputs.nixpkgs-lib.follows = "nixpkgs";
    };
    import-tree.url = "github:vic/import-tree";
    nixpkgs.url = "github:nixos/nixpkgs/nixpkgs-unstable";
  };

  outputs = inputs:
    inputs.flake-parts.lib.mkFlake {inherit inputs;}
    (inputs.import-tree ./modules);
}
```

`import-tree` auto-imports every `.nix` file under `./modules/` as a flake-parts module.

## File Structure

```
project-root/
├── flake.nix              # Entry point — mkFlake + import-tree
├── flake.lock
└── modules/
    ├── meta.nix           # Shared options (username, hostname, etc.)
    ├── systems.nix        # Supported systems list
    ├── nixos.nix          # NixOS configuration option + flake output wiring
    ├── shell.nix          # User shell — across NixOS + darwin
    ├── editor.nix         # Editor config — across NixOS + hjem
    ├── networking.nix     # Networking — NixOS only
    └── desktop.nix        # Concrete NixOS configuration assembly
```

## Pattern Examples

### Shared Options

Declare options in the top-level config that any module can read:

```nix
# modules/meta.nix
{lib, ...}: {
  options.username = lib.mkOption {
    type = lib.types.singleLineStr;
    readOnly = true;
    default = "taylor";
  };
}
```

### Systems

```nix
# modules/systems.nix
{
  systems = [
    "x86_64-linux"
    "aarch64-linux"
    "aarch64-darwin"
  ];
}
```

### Lower-Level Module Storage

Use `flake.modules` (from `flake-parts.flakeModules.modules`) to store `deferredModule` values:

```nix
# modules/flake-parts.nix
{inputs, ...}: {
  imports = [
    inputs.flake-parts.flakeModules.modules
  ];
}
```

### Cross-Cutting Feature

A single file implements one feature across multiple configuration classes:

```nix
# modules/shell.nix
{config, lib, ...}: {
  flake.modules = {
    nixos.default = nixosArgs: {
      programs.fish.enable = true;
      users.users.${config.username}.shell = nixosArgs.config.programs.fish.package;
    };

    darwin.default = {
      programs.fish.enable = true;
    };
  };
}
```

### NixOS Configuration Wiring

Declare an option for NixOS configurations that maps to flake outputs:

```nix
# modules/nixos.nix
{lib, config, ...}: {
  options.configurations.nixos = lib.mkOption {
    type = lib.types.lazyAttrsOf (
      lib.types.submodule {
        options.module = lib.mkOption {
          type = lib.types.deferredModule;
        };
      }
    );
  };

  config.flake.nixosConfigurations =
    lib.mapAttrs
    (name: {module}: lib.nixosSystem {modules = [module];})
    config.configurations.nixos;
}
```

### Assembling a Configuration

```nix
# modules/desktop.nix
{config, ...}: let
  inherit (config.flake.modules) nixos;
in {
  configurations.nixos.desktop.module = {
    imports = [
      nixos.default
      # ...other nixos modules
    ];
    nixpkgs.hostPlatform = "x86_64-linux";
  };
}
```

## Key Types

| Type | Usage |
|---|---|
| `lib.types.deferredModule` | Store modules for later evaluation — supports value merging |
| `lib.types.lazyAttrsOf` | Named configurations without forcing evaluation of all values |
| `lib.types.submodule` | Structured option groups |

## Anti-Patterns

| Avoid | Do Instead |
|---|---|
| `specialArgs` for sharing values | Top-level `config` options |
| Separate directories per config class | One `modules/` directory, one file per feature |
| Manual imports lists | `import-tree` for automatic importing |
| Files that span multiple features | Split into one file per feature |

## References

- [Dendritic Pattern](https://github.com/mightyiam/dendritic)
- [import-tree](https://github.com/vic/import-tree)
- [flake-parts modules](https://flake.parts/options/flake-parts-modules.html)
- [`deferredModule` type](https://nixos.org/manual/nixpkgs/stable/)
- [Real examples](https://github.com/mightyiam/dendritic#real-examples)
