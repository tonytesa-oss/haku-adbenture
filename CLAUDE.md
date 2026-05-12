# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

**ハクの大冒険 〜黄金のまたたびを求めて〜** ("Haku's Great Adventure: In Search of the Golden Catnip") is a single-file, browser-based Japanese cat RPG built entirely in one `index.html` with no build system or dependencies (aside from a Google Fonts CDN import).

To play or test: open `index.html` directly in a browser. There is no server, no npm, no compilation step.

A debug button ("🛠 最終バトルテスト") on the title screen skips directly to Stage 4 (the final boss) with all three party members — use this to test battle mechanics without replaying the full story.

## Architecture

The entire game lives in `index.html`, structured in three sections:

1. **CSS** (`<style>`) — all styling including per-screen layouts, card rarities, and every battle animation (`@keyframes`).
2. **HTML** — five `<div class="screen">` elements that are hidden/shown by toggling the `.active` class:
   - `#title-screen` → `#story-screen` → `#battle-screen` → `#result-screen` → (loop) → `#ending-screen`
3. **JavaScript** (`<script>`) — all game logic, data, and rendering. No modules; everything is in the global scope.

### Screen Flow

```
Title → Story scenes → [battleAfter triggers battle] → Battle
     ↑                                                       ↓
     └───── Result (retry on loss) ←─ [checkEnd] ──────────┘
                       ↓ (win)
              Story continues from [storyAfter index]
                       ↓ (after final stage)
                    Ending → Title
```

The `STORY_SCENES` array drives navigation: each scene object can include `battleAfter: <stageIdx>` (which triggers a battle on tap) and each stage in `STAGES` has `storyAfter: <sceneIdx>` (which resumes story after a win, or `-1` to go to the ending).

### Key Data Structures

- **`CATS`** — static base stats for the three cat characters (`haku`, `sen`, `kinu`), keyed by short name.
- **`STAGES`** — four stage definitions, each with an `enemy` object (name, sprite emoji, HP, attack, defense, background CSS, win message) and `drops`/`storyAfter` metadata.
- **`ALL_CARDS`** — all playable cards with `who` (character restriction: `'any'`, `'haku'`, `'sen'`, `'kinu'`), `cost`, `type` (`atk`/`skl`/`heal`/`spc`), and an `effect(gs)` function that mutates game state.
- **`STORY_SCENES`** — array of scene objects with background CSS, cat list, speaker, dialog text, optional `sfx`, `joinParty` (adds a cat to party), and `battleAfter`.
- **`gs`** — runtime battle state (enemy, activeParty with live HP, attackerIdx, mana, hand, playerTurn, busy).
- **`party`** — array of currently joined character keys (`['haku']` at start, grows to `['haku','sen','kinu']`).
- **`partyHP`** — HP persists between battles; initialized by `initParty()`, updated by `checkEnd()` on win, and fully restored on loss-retry.

### Cat Rendering

`makeCat(size, expr, type)` generates cat visuals. For the three real cats it returns an `<img>` tag pointing to the actual photo/art files (`haku.jpg`, `sen.png`, `kii.png`). For generic/SVG fallback it generates inline SVG. Attack animations swap `haku.jpg` → `haku hikkaki.png` momentarily; special moves trigger full-screen image overlays (`sengiri.png`, `haku inemuri.png`, `sen haku inemuri.png`).

### Battle Logic

- `initBattle(stageIdx)` — sets up `gs`, computes `maxMana` (3 + party size − 1, capped), calls `drawHand()`.
- `drawHand()` — filters `ALL_CARDS` by party membership, shuffles, picks 4, resets mana to `maxMana`.
- `useCard(idx)` — deducts mana, removes card from hand, plays SFX/animation, calls `card.effect(gs)`, then re-renders.
- `dealDamage(gs, power, from)` — applies defense reduction; 20% crit chance when damage ≥ 10 (multiplies by 1.5); triggers visual FX.
- `endTurn()` — advances `attackerIdx`, runs enemy AI (random alive target), then starts next player turn via `drawHand()`.
- `checkEnd()` — detects win (enemy HP ≤ 0) or loss (all party HP ≤ 0) and routes to `showResult()`.

### BGM / SFX

BGM is HTML `<Audio>` looped MP3/WAV from the `BGM/` folder. SFX uses the Web Audio API (`AudioContext`) via `sfx(type)` — created lazily on first use. BGM starts only after the first user click (browser autoplay policy).

BGM files: `title.mp3`, `sentou.mp3` (battle), `idou.mp3` (travel/result), `lastbattle.wav`, `ending.mp3`, `マサラタウンのテーマ.wav` (prologue).

## Conventions

- Japanese is used for all in-game text (dialog, UI labels, log messages). Variable/function names are English.
- Card `effect` functions receive `gs` and mutate it directly — no return values.
- Visual effects are created as transient DOM elements appended to `.battle-field` or `#battle-screen`, then removed via `setTimeout`.
- The `busy` flag on `gs` prevents action stacking during animations; always set `gs.busy=true` before async sequences and restore it after.
- Card rarity is derived purely from mana cost: cost 1 → `normal`, cost 2 → `rare`, cost 3 → `sr`.
- `.gitignore` excludes a large `BGM/PokemonRG_Music/` folder, some numbered MP3s, and the `.claude/` directory — don't add those files.
