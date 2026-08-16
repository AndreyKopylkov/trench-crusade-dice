# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Offline dice-probability calculator for the tabletop game Trench Crusade. A single static page — no build step, no dependencies, no package manager, no test runner. `index.html` contains all markup, styles, and logic; `manifest.webmanifest` and `sw.js` only exist to make it installable as a PWA.

Edits are made directly to `index.html` and take effect on reload.

## Commands

Serve locally (needed for the service worker and PWA install; `file://` works for everything else):

```bash
python -m http.server 8000
```

Deploy — GitHub Pages serves `main` at the repo root, so pushing is deploying:

```bash
git push
```

Check the Pages build after a push:

```bash
gh api repos/AndreyKopylkov/trench-crusade-dice/pages/builds/latest --jq '.status, .error.message'
```

## Verifying probability changes

There is no test suite. The engine is verified by cross-checking against an independent brute-force enumerator pasted into the browser console — recursively enumerate all `6^N` face combinations, sort, slice the counted dice, and apply the success rule directly. Compare its output against the app's displayed percentage for the same configuration, driven by clicking the steppers:

```js
document.querySelector('[data-step="dice:1"]').click();
document.getElementById('pct').textContent;
```

Any change to `analyse`, `diceDist`, or the crit constants must be re-verified this way across several configurations, including negative dice modifiers and `sum > 0`. Brute force is only tractable up to about `N = 10`; the app itself supports up to 12 dice.

## Domain model

These rules are project-specific and cannot be inferred from general Trench Crusade knowledge — they were defined by the user and several differ from a naive reading of the rulebook.

The roll shape is derived in `shape()` from three independent axes:

- `K = 2 + sum` — how many dice are added together
- `N = K + |dice|` — how many dice are actually rolled
- `mode` — `best` when `dice >= 0`, `worst` when `dice < 0`

The `sum` axis (labelled "кубик в сумму") is **not** part of the net modifier that picks best-vs-worst. It grants both an extra die and an extra slot in the sum, and it does not cancel out a negative `dice` value.

Crits are evaluated on the raw sum of the counted dice, **before** the flat modifier: `>= CRIT_HI` (12) is an automatic success, `<= CRIT_LO` (2) an automatic failure. With `K >= 3` a crit failure is unreachable — that is intended, not a bug. The `crit` flag in state disables both.

## Architecture

**Engine** (`diceDist` / `analyse`). `diceDist` does not enumerate `6^N` outcomes. It enumerates face-count histograms — compositions of `N` into 6 buckets — and weights each by its multinomial coefficient, which keeps 12 dice at a few thousand iterations. Selecting the counted dice is a walk over faces from 6 down (or 1 up) taking `min(count, remaining)`. Results are memoized in `distCache` keyed `N:K:mode`, so the deltas panel recomputing five neighbouring configurations on every render is nearly free.

`analyse(s)` takes a **state object, not the global state** — the deltas panel depends on this to evaluate hypothetical states. Keep it pure.

**State.** One flat object: `dice`, `flat`, `sum`, `target`, `rolls`, `crit`. `LIMITS` drives both `clamp()` and the disabled state of the stepper buttons; adding a numeric control means adding a `LIMITS` entry, or `sanitize()` will drop it. Persisted to `localStorage` under `tc-dice-state`, presets under `tc-dice-presets`, accordion open-state under `tc-dice-acc` — every access is wrapped in try/catch because `data:` and some `file://` contexts disable storage.

**Rendering.** `render()` is the only entry point: it rewrites every control, panel, and the sticky footer from scratch. Every mutation ends with a `render()` call rather than a targeted DOM update.

**Events.** A single delegated `click` listener on `document` dispatches on `data-step`, `data-close`, `data-preset-*` attributes and a few button ids. New interactive elements should use a `data-*` attribute rather than their own listener.

**Roller.** `rollOnce()` mirrors the engine's selection logic independently (sorted indices instead of histograms) — a change to the counting rules has to be applied in both places. It uses rejection-sampled `crypto.getRandomValues` to avoid modulo bias.

## Service worker gotcha

`sw.js` caches with a cache-first strategy under the constant `CACHE = 'tc-dice-v1'`. **Bump that version string on every content change**, otherwise returning visitors keep the old `index.html` indefinitely. Registration is skipped unless the protocol is http(s).

Paths in `manifest.webmanifest` and `sw.js` are relative (`./`) because the site is hosted on a GitHub Pages subpath, not at a domain root. Keep them relative.

## Hosting constraints

The repo is public, so a push publishes. Pages runs the legacy Jekyll build; a `.nojekyll` file at the root disables it, so files whose names start with `_` or `.` are published normally. Do not delete it.

`icon-192.png` and `icon-512.png` are generated, not hand-drawn — the GDI+ script that produces them lives outside the repo. Regenerate both at their exact sizes if the mark changes; Chrome on Android needs real raster icons at 192 and 512 to offer installation.
