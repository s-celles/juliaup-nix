# juliaup-nix

Nix flake packaging [juliaup](https://github.com/JuliaLang/juliaup) — the official Julia version manager.

Juliaup is not in nixpkgs. This flake builds juliaup from source using `rustPlatform.buildRustPackage`, and provides pinned Julia binaries in the Nix store for fully reproducible deployments.

## What this provides

| Package | Binary | Description |
|---------|--------|-------------|
| `juliaup` (default) | `juliaup`, `julia` | Julia version manager CLI + `julialauncher` multiplexer |
| `julia` | `julia` | Standalone Stable (1.13.0) — pinned in the Nix store |
| `julia-lts` | `julia` | Standalone LTS (1.10.9) — pinned in the Nix store |
| `julia-1_13_0` | — | Raw Julia 1.13.0 binary (no wrapper) |
| `julia-1_10_9` | — | Raw Julia 1.10.9 binary (no wrapper) |

### About `juliaup` and `julialauncher`

Building `juliaup` from Rust source builds **both** binaries:
- `juliaup`: the version manager CLI.
- `julia`: the official `julialauncher` multiplexer.

Installing `packages.juliaup` provides both `juliaup` and `julia`. The launcher handles channel dispatching (`julia +1.10`, `julia +lts`, `julia +release`) dynamically based on `~/.julia/juliaup/juliaup.json`.

> **Note on standalone wrappers:** `julia` and `julia-lts` are standalone wrappers around store-pinned binaries (built with `autoPatchelfHook`). They are useful for `nix run` or purely declarative setups without `nix-ld`. If you use `juliaup`, install `packages.juliaup` directly (and avoid shadowing it with standalone wrappers).

## NixOS Setup (Juliaup + nix-ld)

Because Julia binaries downloaded dynamically by `juliaup add <version>` are standard dynamically linked Linux executables, NixOS requires `programs.nix-ld` to run them:

```nix
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    juliaup-nix = {
      url = "github:s-celles/juliaup-nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, juliaup-nix, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ({ pkgs, ... }: {
          # System packages: provides both `juliaup` and `julia` launcher
          environment.systemPackages = [
            juliaup-nix.packages.${pkgs.stdenv.hostPlatform.system}.juliaup
          ];

          # Dynamic linker for binaries downloaded by juliaup
          programs.nix-ld = {
            enable = true;
            libraries = with pkgs; [
              stdenv.cc.cc.lib
              zlib
              openssl
              curl
              libssh2  # Required for Julia 1.10 (LTS) LibCURL compatibility!
            ];
          };
        })
      ];
    };
  };
}
```

> **Important tip for Julia 1.10 (LTS):** If you include `curl` in `programs.nix-ld.libraries`, you **must** also include `libssh2`. Otherwise, Nixpkgs's `libcurl.so.4` will attempt to link against Julia 1.10's older bundled `libssh2.so.1`, leading to a symbol resolution error (`undefined symbol: libssh2_session_callback_set2`).

## Quick start

```bash
# Install channels
juliaup add release && juliaup default release
juliaup add lts

# Verify and switch versions
julia --version   # default release (1.13.0)
julia +1.10       # LTS (1.10.12)
julia +lts        # LTS (1.10.12)
```

## Standalone / Pure Nix Usage (no nix-ld)

You can run pinned Julia versions directly from the Nix store without installing Juliaup or enabling `nix-ld`:

```bash
nix run github:s-celles/juliaup-nix#julia      # latest stable (1.13.0)
nix run github:s-celles/juliaup-nix#julia-lts  # LTS (1.10.9)
```

## Updating Julia versions

Update the relevant `mkJuliaBin` call in `flake.nix` with the new version's URLs and hashes, then bump the flake in your configuration:

```bash
nix flake update juliaup-nix
```

## About `flake.lock`

The `flake.lock` pins `nixpkgs` and `flake-utils` versions for reproducible builds. Commit it alongside `flake.nix`.
