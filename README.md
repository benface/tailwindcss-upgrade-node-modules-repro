# `@tailwindcss/upgrade` mutates files inside `node_modules/tailwindcss/` in a pnpm workspace

Minimal reproduction of a bug in [`@tailwindcss/upgrade`](https://www.npmjs.com/package/@tailwindcss/upgrade): when run from a **subpackage of a pnpm workspace**, the tool traverses `../../node_modules/…` and applies its `v3 → v4` migration transformation (`@tailwind utilities;` → `@import 'tailwindcss/utilities' layer(utilities);`) to files inside the installed `tailwindcss` package.

Robin Malfait shipped a fix in the closed [#19726](https://github.com/tailwindlabs/tailwindcss/issues/19726) thread that ignores gitignored files — that fix works for the simple flat-project case (see the "counter-tests" section below), but does not cover this workspace-subpackage variant.

## Reproduce

```sh
git clone https://github.com/benface/tailwindcss-upgrade-node-modules-repro
cd tailwindcss-upgrade-node-modules-repro
pnpm install
cd packages/css
pnpm dlx @tailwindcss/upgrade
```

Tool output includes:

```
│ ↳ Migrated stylesheet: `../../node_modules/.pnpm/tailwindcss@4.3.2/node_modules/tailwindcss/utilities.css`
│ ↳ Updated package: `tailwindcss`
```

The tool announces that it migrated the file inside `node_modules`. Verify against a pristine copy from the tarball:

```sh
curl -sL "$(pnpm view tailwindcss@4.3.2 dist.tarball)" | tar -xz -C /tmp package/utilities.css
diff /tmp/package/utilities.css ../../node_modules/.pnpm/tailwindcss@4.3.2/node_modules/tailwindcss/utilities.css
```

Output:

```diff
1c1
< @tailwind utilities;
---
> @import 'tailwindcss/utilities' layer(utilities);
```

The rewrite creates a self-referential import loop (`utilities.css` now imports itself).

## Counter-tests: cases where the fix works

For contrast, both these cases correctly skip `node_modules` (the tool never touches it):

- Non-workspace project with `node_modules` in `.gitignore`, run from project root
- Same, using an explicit-path import (`@import 'tailwindcss/utilities.css' layer(utilities) source(none)`)

The pnpm-workspace-subpackage layout is the specific gap.

## Why this matters

1. **`node_modules` is not a valid write target** — those files are shared across projects via the pnpm store, are immutable install artifacts elsewhere, and are out of scope for `git status`, so the corruption is invisible until it detonates.
2. **The rewrite creates a self-referential import** — the mutated file imports itself under a `layer(...)` wrapper, causing an infinite CSS resolution loop later.
3. **Symptoms look like a "cache corruption" bug** — the user sees infinite-recursion errors long after the upgrade run, from a completely unrelated command, with `git status` clean.
4. **Recovery requires nuking `node_modules` + reinstalling** (and on Bun, purging the install cache too, per #19726).

## Expected behavior

The upgrade tool should skip any file under `node_modules/`, regardless of how the path is reached (via `../` traversal, symlink resolution, or `@import` resolution). The gitignore-based filter would work if it also checked the ancestor `.gitignore` chain against absolute-resolved paths.

## Environment

- macOS (arm64), Node 24, pnpm 11.12.0
- `tailwindcss@4.3.2`
- `@tailwindcss/upgrade@4.3.2` (via `pnpm dlx`)
