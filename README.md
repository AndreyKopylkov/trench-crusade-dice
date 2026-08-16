# Trench Crusade — Dice Calculator

Offline web app that computes exact roll probabilities for the tabletop game **Trench Crusade**.
Runs on phone and desktop, works without a connection, installs as a PWA.

**Open it:** https://andreykopylkov.github.io/trench-crusade-dice/

<p>
  <a href="https://andreykopylkov.github.io/trench-crusade-dice/">
    <img src="https://img.shields.io/badge/Open_the_app-c8a24a?style=for-the-badge&logo=googlechrome&logoColor=12100e" alt="Open the app">
  </a>
  <a href="https://t.me/Sheriff_Xlebywek">
    <img src="https://img.shields.io/badge/Telegram-@Sheriff__Xlebywek-229ED9?style=for-the-badge&logo=telegram&logoColor=white" alt="Telegram">
  </a>
  <a href="https://github.com/AndreyKopylkov">
    <img src="https://img.shields.io/badge/GitHub-AndreyKopylkov-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub">
  </a>
  <a href="mailto:andreykopylkov@gmail.com">
    <img src="https://img.shields.io/badge/Email-andreykopylkov@gmail.com-b4514a?style=for-the-badge&logo=gmail&logoColor=white" alt="Email">
  </a>
</p>

Made by **Kopylkov Andrey** — *Xlebniy_Samurai*.

## Roll rules

The base roll is 2d6; a **7 or higher** succeeds.

Modifiers:

| Modifier | Effect |
|---|---|
| ±1 die to the roll | roll `2 + \|net\|` dice |
| ±1 to the result | added to the dice total |
| +1 die into the total | adds both a die and a slot in the total |

- A **positive** net dice modifier counts the **highest** dice, a **negative** one the **lowest**.
- At least 2 dice are always rolled.
- "Extra die into the total" is a separate axis: `K = 2 + S` dice counted, `N = K + |D|` dice rolled.

Crits (switched off with the toggle):

- final total **12 or higher** — automatic success;
- final total **2 or lower** — automatic failure;
- the flat result modifier **is included** before the crit is checked.

## Injury rolls

The second tab uses the same dice shape, but the total is `counted dice + modifier − armour`
and the outcome is read off the injury table:

| Total | Outcome |
|---|---|
| ≤ 1 | no effect |
| 2–6 | blood marker |
| 7–8 | blood marker + prone |
| ≥ 9 | death |

Both ends are open — 15 is still death, −3 is still nothing. Injury rolls have no crits.

## Features

- Exact math over every outcome (enumeration, not simulation).
- Success percentage pinned to the bottom of the screen.
- Action mode also shows the crit-success chance and the resulting chance of killing the target,
  using the Injury tab settings — a crit success adds +1 die to that injury roll.
- "What one more modifier gives" panel — resulting percentage and the difference in pp.
- Series of rolls: chance of "N or more successes", average number of successes.
- A real dice roller that highlights the counted and the discarded dice.
- Named configuration presets.
- Total distribution and a "chance to roll X or higher" table.
- Russian and English interface, switched in the side drawer.

## Running it

Locally — just open `index.html` in a browser.

Installing to a phone home screen needs http(s). Local server:

```bash
python -m http.server 8000
```

Then open `http://<your-computer-IP>:8000` from a phone on the same network.

## Layout

- `index.html` — the whole app: math, markup, logic
- `manifest.webmanifest` — PWA metadata
- `sw.js` — service worker for the offline cache
- `icon-192.png`, `icon-512.png` — home screen icons
