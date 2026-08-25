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

Any change to `analyse`, `analyseInjury`, `diceDist`, the crit constants, or the injury bucket boundaries must be re-verified this way across several configurations, including negative dice modifiers, `sum > 0`, and every armour value. Brute force is only tractable up to about `N = 10`; the app itself supports up to 12 dice.

The engine functions are DOM-free and contiguous in the script (from `var FACT = [1];` down to the `/* биномиальное распределение серии */` comment), so they can also be sliced out of `index.html` and driven from Node against a brute-force enumerator — faster than clicking steppers when sweeping hundreds of configurations. The same trick works for the state block (`var LIMITS` down to the language section) with a stubbed `localStorage`, which is how the migration paths are checked.

## Domain model

These rules are project-specific and cannot be inferred from general Trench Crusade knowledge — they were defined by the user and several differ from a naive reading of the rulebook.

The roll shape is derived in `shape()` from three independent axes:

- `K = 2 + sum` — how many dice are added together
- `N = K + |dice|` — how many dice are actually rolled
- `mode` — `best` when `dice >= 0`, `worst` when `dice < 0`

The `sum` axis (labelled "кубик в сумму") is **not** part of the net modifier that picks best-vs-worst. It grants both an extra die and an extra slot in the sum, and it does not cancel out a negative `dice` value.

Crits are evaluated on the **final total** — counted dice **plus** the flat modifier: `>= CRIT_HI` (12) is an automatic success, `<= CRIT_LO` (2) an automatic failure. So a `+2` modifier turns a raw 10 into a crit. The `crit` flag in state disables both. With `K >= 3` and a non-negative `flat` a crit failure is unreachable — intended, not a bug; a negative `flat` brings it back.

### Two roll types

The app has two modes, switched by the tabs under the header and held in `app.mode`:

- `action` — the success roll described above (attacks, tests). Threshold `target`, binary outcome.
- `injury` — the injury roll. Same dice shape, but the total is `counted dice + flat − armour` and the result is looked up in a four-bucket table: `<= 1` nothing, `2..6` blood marker, `7..8` blood marker + prone, `>= 9` death. The ends are **open** — a total of 15 is still death, a total of −3 is still nothing.

Injury rolls have **no crits**: the toggle is hidden and `analyseInjury` never looks at `CRIT_HI`/`CRIT_LO`. Armour is a separate 0–3 control even though it is mathematically identical to a negative `flat` — the two are distinct things at the table (your weapon versus their armour) and are shown separately in the formula line, the roll breakdown, and the deltas panel. Note the mirror of the crit quirk: with `sum >= 1` the minimum total is 3, so the "nothing" bucket becomes unreachable without armour or a negative modifier. Intended, not a bug.

## Architecture

**Engine** (`diceDist` / `analyse` / `analyseInjury`). `diceDist` does not enumerate `6^N` outcomes. It enumerates face-count histograms — compositions of `N` into 6 buckets — and weights each by its multinomial coefficient, which keeps 12 dice at a few thousand iterations. Selecting the counted dice is a walk over faces from 6 down (or 1 up) taking `min(count, remaining)`. Results are memoized in `distCache` keyed `N:K:mode`, shared by both modes, so the deltas panel recomputing five to seven neighbouring configurations on every render is nearly free.

`deathChain(a, s, inj)` is the only place the two modes meet: it feeds the action result into an injury roll to produce the small line under the sticky footer in action mode. Success splits into the crit branch and the plain branch, because a crit success grants **+1 die to the injury roll**; the injury side always comes from `app.injury`, i.e. whatever the Injury tab is configured to. With `crit` off there is no split. It only reports — the actual injury roller does not add the die.

`analyse(s)` and `analyseInjury(s)` both take a **state object, not the global state** — the deltas panels depend on this to evaluate hypothetical states. Keep them pure. They are deliberately separate functions over the same `diceDist`: `analyse` is the verified success-roll path and should not grow a mode flag.

**State.** `app = { mode, action:{…}, injury:{…} }`, and the global `state` is a live reference to `app[app.mode]` — `setMode()` repoints it, and everything else mutates `state` in place. `LIMITS` holds the range of every numeric field across both modes and drives `clamp()` plus the disabled state of the steppers; `FIELDS` says which fields each mode actually owns (`target` is action-only, `armour` is injury-only) and `sanitize(s, mode)` drops everything else. Adding a control means adding both a `LIMITS` entry and a `FIELDS` entry, or it will be silently dropped on save. Persisted to `localStorage` under `tc-dice-state`, presets under `tc-dice-presets`, accordion open-state under `tc-dice-acc` — every access is wrapped in try/catch because `data:` and some `file://` contexts disable storage. `loadState()` migrates the old flat single-mode object by treating a stored object with no `action` key as an action state; presets do the same via `presetMode()`, defaulting to action.

**Mode-dependent UI.** `render()` stamps `data-mode` on `<body>` and CSS does the showing and hiding (`body[data-mode="action"] [data-only="injury"]`), so mode-specific markup is marked with `data-only` rather than toggled from JS. Elements whose *text* changes between modes (the three stat tiles `#k-1..3`, the roll button, the `#acc-cum-label` summary) carry no `data-i18n` — `render()` sets them, because `applyStatic()` would overwrite mode-aware text with the plain key.

**Localization.** `I18N` holds two complete dictionaries (`ru`, `en`); `D` points at the active one and is swapped by `setLang()`. Plain strings are keys, parametrized text is a function on the dictionary (`D.formula(a, s)`, `D.describe(sh, s)`, `D.sNote(...)`). Static markup carries `data-i18n` (textContent) and `data-i18n-title` (title + aria-label); `applyStatic()` rewrites them and runs first inside `render()`. **Every new user-visible string needs a key in both dictionaries** — a missing key silently leaves the Russian fallback text from the markup. Decimal separator is language-dependent via `dec()`, so use `pct`/`pct1`/`num` rather than `toFixed` directly. The default is **English** — `loadLang()` ignores `navigator.language` and only ever returns something else if the user has switched languages before, so the static Russian fallback text in the markup is never what a first-time visitor sees. Language lives outside `state` under `tc-dice-lang`, deliberately: presets store a snapshot of `state`, and applying one must not change the UI language.

**Drawer.** `#m-menu` holds the system actions: language toggle, `doInstall()` (fires the stashed `beforeinstallprompt` event — `#btn-install` is hidden by `render()` unless `installPrompt` is set, so it only appears in Chromium, which is the only engine that fires the event; the event is single-use, so it is cleared before `prompt()`), `doShare()` (the system share sheet on handhelds — gated on `isHandheld()` *and* `navigator.share`, since desktop Chromium also implements the API and the desktop behaviour is meant to be a clipboard copy — otherwise it copies the URL and toasts), `checkUpdate()` (calls `registration.update()` and watches `updatefound`), `clearCache()` (confirms, wipes Cache Storage + unregisters service workers, keeps localStorage), links, and a «Поддержать разработку» button that opens `#m-donate`. `APP_VERSION` and `LAST_UPDATE` are displayed at the bottom of the drawer and are updated by hand at release time.

**Donate modal.** `#m-donate` is a two-level view held in the module-level `donateView` (`'list'` | `'crypto'`) and drawn by `renderDonate()` into `#donate-body`. `#donate-back` pops one level: from `crypto` back to `list`, from `list` it closes the modal and reopens the drawer (the drawer has to be closed when the modal opens — `.drawer` sits at `z-index:60`, above `.modal`'s `50`). Platform URLs live in `DONATE`, wallet details in `CRYPTO`; crypto rows carry `data-copy`/`data-copy-label` and go through `copyText()`, which prefers the Clipboard API and falls back to `legacyCopy()` (hidden textarea + `execCommand`) both when the API is missing and when it rejects — an unfocused document rejects. The modal title is set by `renderDonate()` rather than `data-i18n`, so `render()` calls `renderDonate()` to keep it in sync on a language switch.

**Rendering.** `render()` is the only entry point: it rewrites every control, panel, and the sticky footer from scratch. Every mutation ends with a `render()` call rather than a targeted DOM update.

**Events.** A single delegated `click` listener on `document` dispatches on `data-step`, `data-close`, `data-tab`, `data-armour`, `data-lang`, `data-preset-*` attributes and a few button ids. New interactive elements should use a `data-*` attribute rather than their own listener.

**Roller.** `rollOnce()` and `rollOnceInjury()` mirror the engine's selection logic independently (sorted indices instead of histograms) — a change to the counting rules has to be applied in **all three** places. They use rejection-sampled `crypto.getRandomValues` to avoid modulo bias. `doRoll()` dispatches on the mode; both paths funnel their log line through `pushHistory()`, which keeps one shared 10-entry list tagged with the roll type.

## Service worker gotcha

`sw.js` caches with a cache-first strategy under the constant `CACHE = 'tc-dice-v1'`. **Bump that version string on every content change**, otherwise returning visitors keep the old `index.html` indefinitely. Registration is skipped unless the protocol is http(s).

Paths in `manifest.webmanifest` and `sw.js` are relative (`./`) because the site is hosted on a GitHub Pages subpath, not at a domain root. Keep them relative.

## Hosting constraints

The repo is public, so a push publishes. Pages runs the legacy Jekyll build; a `.nojekyll` file at the root disables it, so files whose names start with `_` or `.` are published normally. Do not delete it.

`icon-192.png` and `icon-512.png` are generated, not hand-drawn — the GDI+ script that produces them lives outside the repo. Regenerate both at their exact sizes if the mark changes; Chrome on Android needs real raster icons at 192 and 512 to offer installation.
