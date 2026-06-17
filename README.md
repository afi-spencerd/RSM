# RSM Inventory

Design specifications and diagrams for RSM's inventory system — covering the
proposed transaction-based inventory schema, Formula Web (FW) feature requests,
and the FW ↔ QuickBooks integration.

The documentation is authored as an [mdBook](https://rust-lang.github.io/mdBook/)
and renders [Mermaid](https://mermaid.js.org/) diagrams (ER diagrams, flowcharts,
and sequence diagrams).

## Contents

The book sources live in [`src/`](./src):

| Page | Description |
| --- | --- |
| [`tx_table_proposal.md`](./src/tx_table_proposal.md) | Proposed `InventoryTx` / `Lot` / `Item` schema, inventory life-cycle flowcharts, goals, and open questions. |
| [`fw_requests.md`](./src/fw_requests.md) | Formula Web feature requests. |
| [`fw_qb_integration.md`](./src/fw_qb_integration.md) | Formula Web ↔ QuickBooks integration diagrams. |
| [`SUMMARY.md`](./src/SUMMARY.md) | Table of contents / book structure. |

`artifacts/` holds supporting reference material (exports, findings, source
documents) and is not part of the rendered book.

## Prerequisites

This project uses a [Nix](https://nixos.org/) flake to provide a reproducible
toolchain. Install Nix with [flakes enabled](https://nixos.wiki/wiki/Flakes),
then everything else comes from the dev shell — no global installs required.

## Development shell

The flake's `devShell` provides `mdbook` and `mdbook-mermaid`. Enter it with:

```sh
nix develop
```

To run a single command inside the shell without entering it interactively:

```sh
nix develop --command <cmd>
```

If a needed tool is missing, add it to the `packages` list in
[`flake.nix`](./flake.nix) and re-enter the shell.

## Building and viewing the book

From inside the dev shell (or prefixed with `nix develop --command`):

```sh
# Build the static site into ./book
mdbook build

# Serve with live reload at http://localhost:3000
mdbook serve --open
```

The generated `book/` directory is build output and is git-ignored.

## Version control

The repository is tracked with [Jujutsu](https://jj-vcs.github.io/jj/) on a Git
backend, so standard `git` tooling also works against it.
