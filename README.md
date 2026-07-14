# `@tailwindcss/upgrade` mutates files inside `node_modules/tailwindcss/`

Minimal reproduction for a bug in [`@tailwindcss/upgrade`](https://www.npmjs.com/package/@tailwindcss/upgrade): running it against a project that already uses Tailwind CSS v4 rewrites CSS files **inside `node_modules/tailwindcss/`** in place, silently.

The tool follows the project's `@import "tailwindcss"` back into `node_modules` and applies its `v3 → v4` migration transformation (`@tailwind utilities;` → `@import 'tailwindcss/utilities' layer(utilities);`) to the installed package's own source files.

This is likely the root cause behind [tailwindlabs/tailwindcss#19726](https://github.com/tailwindlabs/tailwindcss/issues/19726) ("Exceeded maximum recursion depth while resolving `tailwindcss/utilities`"), which was closed after the user cleared their install cache. The upstream cause — the upgrade tool writing to `node_modules` — was never fixed, so any subsequent user who runs the tool re-corrupts their install.

## Reproduce

```sh
git clone https://github.com/benface/tailwindcss-upgrade-node-modules-repro
cd tailwindcss-upgrade-node-modules-repro
pnpm install
git init -q && git add -A && git -c user.email=x@x -c user.name=x commit -qm init
pnpm dlx @tailwindcss/upgrade
```

Now compare the installed `tailwindcss/index.css` against a pristine copy from the tarball:

```sh
curl -sL "$(pnpm view tailwindcss@4.3.2 dist.tarball)" | tar -xz -C /tmp package/index.css
diff /tmp/package/index.css node_modules/.pnpm/tailwindcss@4.3.2/node_modules/tailwindcss/index.css
```

Output:

```diff
943c943
<   @tailwind utilities;
---
>   @import 'tailwindcss/utilities' layer(utilities);
```

Context in the pristine file (lines 942–944):

```css
@layer utilities {
  @tailwind utilities;
}
```

## Why this is problematic

1. **`node_modules` is not a valid write target.** Tools must never mutate installed packages — those files are shared across projects via the pnpm store, are immutable install artifacts elsewhere, and are out of scope for `git status`, so the corruption is invisible until it detonates.
2. **The rewrite is inside `@layer utilities { … }`.** `@import … layer(utilities)` inside a `@layer` block is a self-referential loop in the CSS resolution graph.
3. **Symptoms look like a "cache corruption" bug** — the reporter sees infinite-recursion errors long after the upgrade run, from a completely unrelated command, with `git status` clean.
4. **Recovery requires nuking `node_modules` + reinstalling** (and on Bun, purging the install cache too — see #19726). `pnpm install --frozen-lockfile` alone is not obviously the fix from the symptoms.

## Expected behavior

The upgrade tool should only migrate files under the project root (or explicitly-passed inputs) and never follow `@import` resolutions into `node_modules`.

## Notes

- The tool's own "Migrated stylesheet" output does not list the `node_modules` files it touched, so the mutation is silent.
- The bug also reproduces when the project's CSS imports a specific internal path (e.g. `@import 'tailwindcss/utilities.css' layer(utilities) source(none)`), in which case `utilities.css` (rather than `index.css`) is the file that gets rewritten.

## Environment

- macOS (arm64), Node 24, pnpm 11.12.0
- `tailwindcss@4.3.2`
- `@tailwindcss/upgrade@4.3.2` (via `pnpm dlx`)
