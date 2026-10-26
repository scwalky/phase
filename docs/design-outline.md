# Commander — Design Outline (draft v0.2)

An application for playing Magic: The Gathering **Commander** (4-player free-for-all), loading real decks
from **Archidekt** and **Scryfall**, with **AI opponents** and (initially) **one human player**.

## Decisions so far (2026-10-26)

| Topic | Decision |
|---|---|
| Engine | **Build on [phase.rs](https://github.com/phase-rs/phase)** (MIT / Apache-2.0) |
| Our code | **TypeScript + React.** The phase.rs Rust engine is used through its WASM build. We write Rust only when we have to fix or extend the engine. |
| Deck sources | Archidekt URLs, **Scryfall deck URLs**, and pasted text lists. Scryfall is also the card and image source. |
| AI | **Heuristic first.** Search and LLM players come later. |
| Project shape | **Fork** — [scwalky/phase](https://github.com/scwalky/phase), `upstream` = phase-rs/phase. Our work lives on `commander/*` branches. |

Items marked **[DECIDE]** are still open.

---

## 1. Goals and non-goals

### v1 goals
- 1 human vs 1–3 AI opponents. Commander rules: 40 life, commander tax, commander damage (21),
  command zone replacement, color identity, singleton.
- Import a deck from an Archidekt URL, a Scryfall deck URL, or a pasted text decklist.
- Validate the deck (100 cards, color identity, banned list) and show which cards the engine can
  and can't play yet.
- Quick setup: pick your deck and 3 opponent decks (or "random precon"), choose AI difficulty, play.
- Games are deterministic and replayable (seeded RNG plus an action log).

### Non-goals for v1
- Online multiplayer with several humans. phase.rs has a server for this, so it comes later almost for free.
- Draft, sealed or other formats.
- Our own rules engine.

---

## 2. Prior art (GitHub research)

| Project | Stack | License | Commander / multiplayer | AI | Takeaway |
|---|---|---|---|---|---|
| **[phase.rs](https://github.com/phase-rs/phase)** ← our base | Rust → WASM + native, React/TS client, Tauri desktop | **MIT / Apache-2.0** | Commander supported; up to 4 players in the engine config | `phase-ai` (per-card logic, search, evaluation) **compiled into WASM**; `phase-llm` LLM opponents | Very active (last commit 2026-10-25). **88% overall / 93% Commander card coverage** according to its badges. Already imports Archidekt and Moxfield decks. |
| [XMage](https://github.com/magefree/mage) | Java, client/server | MIT | Up to 10 players | Rule-based | The most complete open engine (~32k cards, ~9k tests). A reference for tricky rules. |
| [Forge](https://github.com/Card-Forge/forge) | Java | **GPL-3.0** | Commander vs AI | Mature per-card AI hints | phase.rs has an *optional*, feature-gated bridge that reads Forge scripts. **Never bundle Forge data**, or our project becomes GPL. |
| [argentum-engine](https://github.com/rwaldron/argentum-engine) | Kotlin + Spring, React/TS | Not stated on the page | FFA 2–6 players | Alpha-beta search, optional LLM, RL/MCTS gym | A good model for layering AI tiers. |
| [mage-go](https://pkg.go.dev/github.com/benprew/mage-go) | Go | GPL-2.0 | 2-player | None | A small, readable engine for learning. |
| mage-bench ([Show HN](https://hn.svelte.dev/item/47049227)) | XMage + MCP | n/a | n/a | LLMs via tools | LLMs can drive a real engine through tool calls. |
| [deck2trice](https://pypi.org/project/deck2trice/), [archiTop](https://github.com/Julian-Brendel/archiTop), [tts-deckconverter](https://github.com/jeandeaual/tts-deckconverter) | Python / Go | various | n/a | n/a | Deck import references. |

**Main lesson:** almost all of the effort in these projects goes into card implementation and the
layer and replacement-effect system. phase.rs has already done most of that work, which is why we build on it.

---

## 3. What phase.rs already gives us (verified in the repo)

- **Engine (Rust):** Commander damage, commander tax and color identity are implemented. There's also an
  oracle-text parser plus an optional Forge-script bridge for cards it can't parse.
- **`engine-wasm` API** that our TS code calls directly, including:
  `load_card_database`, `initialize_game`, `get_legal_actions_for_viewer_js`, `submit_action`,
  `get_filtered_game_state(viewer)` (hidden info), `is_card_commander_eligible`, `classify_deck_js`,
  `evaluate_deck_compatibility_js` (how much of a deck is playable), and `estimate_bracket_for_deck`.
- **AI:** `phase-ai` is compiled into the WASM build. There's also `phase-llm` for LLM opponents.
- **Client:** React, TS, Zustand, Tailwind and Vite, plus a Tauri desktop shell.
  `client/src/services/deckUrlImport.ts` and `lobby-worker/src/import-deck.ts` already import from
  **Archidekt** (`archidekt.com/api/decks/{id}/`) and **Moxfield**.
- **Card data:** MTGJSON. Images come from Scryfall at runtime.

### The gaps we fill
1. **Scryfall deck URL import.** It isn't supported yet.
2. **A Commander-first experience:** a "1 human vs 3 AI" quick-start, a deck library, and opponent deck choice
   (random precons or your own decks).
3. **Commander-specific heuristic AI:** threat assessment across 3 opponents, who to attack, removal targeting,
   board-wipe awareness, and commander damage tracking. Check how much `phase-ai` already does
   before building anything.
4. **A coverage-aware deck flow:** after import, show which cards are unsupported or partly supported and
   offer substitutions.
5. **Bugs and missing cards found while playing our own decks.** These go upstream as Rust PRs.

**[DECIDE] after M0:** how much of this is already in phase.rs? Its UI may already cover gaps 2 and 4.
We confirm by running it.

---

## 4. Project shape **[DECIDE]**

| Option | Description | Pros | Cons |
|---|---|---|---|
| **A. Fork phase.rs** (recommended) | Fork it and work mostly in `client/` (TS). Rebase on upstream regularly. Send engine fixes and Scryfall import upstream as PRs. | Reuses the full UI, Tauri shell, AI and server. The smallest amount of new code. | Merges from a very active upstream (it gets ~daily commits). Our UI changes may conflict with theirs. |
| B. Separate app consuming `engine-wasm` | Our own React app, with the phase.rs engine as a git submodule or built WASM package. | Clean ownership of the UI. | We rebuild the board UI, prompts and targeting from scratch, which is a lot of work. |
| C. Contribute upstream only | No separate app. Everything goes into phase.rs. | Zero maintenance. | Their roadmap, not ours. |

**Recommendation: A**, keeping our changes isolated (our own routes and components, plus a `commander/`
feature folder). Anything generic (Scryfall import, engine fixes) goes upstream, so our fork's diff stays small.

---

## 5. Architecture (inside the fork)

```
┌─────────────── client/ (React + TS) ───────────────────────────────────────────┐
│ Existing phase.rs UI: board, stack, prompts, log                                │
│ + ours: Commander quick-start · deck library · import + coverage report         │
│ + ours: CommanderHeuristicAgent (TS) — optional, if phase-ai needs tuning        │
└──────────▲────────────────────────────────────────────────┬────────────────────┘
           │ EngineAdapter (WASM in browser / Tauri IPC)     │ actions
┌──────────┴────────────────────────────────────────────────▼────────────────────┐
│ engine-wasm: phase-engine (rules) + phase-ai (+ phase-llm later)                │
└──────────▲─────────────────────────────────────────────────────────────────────┘
┌──────────┴──────────────┐   ┌───────────────────────────────────────────────────┐
│ Card DB (MTGJSON-built) │   │ Deck import: Archidekt ✔ · Moxfield ✔ · Scryfall ✚ │
│ Scryfall images cached  │   │ · text ✔ → validate → coverage report              │
└─────────────────────────┘   └───────────────────────────────────────────────────┘
```

### Principles we keep
1. **The engine is pure and deterministic.** We never put game logic in TS. If a rule is wrong, fix it in Rust.
2. **One decision interface:** humans and AIs both answer `legal actions → choice`. This means AI-vs-AI
   simulation and tests use the same code path as real play.
3. **No cheating AI by default:** AIs only see `get_filtered_game_state(viewer)`.
4. **Replays:** every game is stored as `{decks, seed, actions[]}`.

---

## 6. Deck import

### Archidekt (already in phase.rs; verified live)
- `GET https://archidekt.com/api/decks/{id}/`: unofficial, public decks only, and **CORS blocks browser calls**.
  phase.rs routes the call through its lobby worker. In our desktop (Tauri) build we fetch from the Rust side.
- Commander = a card whose `categories` include `"Commander"`. Top-level `categories[].includedInDeck`
  marks which cards are in the deck rather than in the Maybeboard. `card.uid` is the Scryfall printing ID.

### Scryfall deck URLs (new; **[TODO] verify the endpoint**)
- Scryfall has a deck builder (`scryfall.com/@user/decks/{uuid}`) with an export feature, but it has
  **no documented API**. Before writing the importer, inspect a real public deck URL and record the
  export URL and format.
- If no stable export exists, fall back to a pasted text list (Scryfall's own "copy as text" output).

### Text lists (already in phase.rs)
- `1 Sol Ring` lines, with a `Commander` section header or `*CMDR*` marker.
  Names are resolved against the local card DB.

### After import
`resolve → validate (100 cards, singleton, color identity, banned list, partner/background) →
evaluate_deck_compatibility → coverage report → save to the deck library`

Unsupported cards can be (a) swapped for a basic land of the right color, (b) played as a vanilla
proxy with a warning, or (c) the import can be cancelled.

---

## 7. AI plan

| Tier | Approach | When |
|---|---|---|
| 0 | Random legal action (fuzzing: thousands of AI-vs-AI games looking for crashes) | M1 |
| 1 | **phase-ai as-is.** Measure it on Commander games first. | M1 |
| 1+ | **Commander heuristics:** threat scoring per opponent (board power, cards in hand, commander damage dealt), attack-target selection, removal targeting of the biggest threat, holding back against obvious board wipes. Built in Rust inside phase-ai where possible (fast, and upstream-able); in TS only for experiments. | M3 |
| 2 | Search (MCTS with determinization). phase-ai already does some tree search, so we tune it for 4 players. | Later |
| 3 | LLM opponents via `phase-llm`, with personalities and table talk (Claude API) | Later |

**Benchmarking:** a headless AI-vs-AI runner (Node + WASM) that plays N games with fixed seeds and reports
win rates per AI configuration. Every AI change gets measured, not eyeballed.

---

## 8. Milestones

1. **M0 — Evaluate (1–2 sessions).** ✅ The build and the first game work. Findings are in §9. Fork phase.rs, build it on Windows, then play 1 human vs 3 AI
   Commander games with 2–3 of your own Archidekt decks. Write down: what's broken, which cards are
   unsupported, how good the AI is, and what the UX lacks. **This produces the real backlog.**
2. **M1 — Harness:** a headless AI-vs-AI runner with seeds and replay files, plus a coverage report
   for your decks.
3. **M2 — Scryfall deck import** and a deck library screen. Upstream the importer as a PR.
4. **M3 — Commander quick-start + Commander heuristics** for the AI (threat assessment).
5. **M4 — Card and rules fixes** for the unsupported cards in your decks (Rust PRs upstream).
6. **M5+ —** better search AI, LLM opponents with personalities, multiple humans online.

---

## 9. M0 findings and backlog (2026-10-26)

M0 result: **the fork builds on Windows and plays a 4-player Commander game** (you on Krenko, Mob Boss vs 3 Medium AIs
on random precons). The first full build took about 25 minutes (card-data tool ~10 min, WASM ~12 min) plus about 1.5 GB of downloads.

### Already in phase.rs (removed from our scope)
- A Commander match setup with 2–4+ players, starting life, BO1/BO3, and who goes first.
- 10 starter Commander decks and 201 legal precons, each showing how many of its cards the engine supports (e.g. "67/68").
- An AI opponent pool with filters for archetype, bracket, minimum card coverage (default 90%) and difficulty, plus cEDH mode.
- An Import Deck button (Archidekt and Moxfield, through the local `lobby-worker` on :8787).
- In dev builds: a debug panel with AI decision view, turn checkpoints, state export and "Report a Card".

### Backlog

| # | Item | Type | Where | Priority |
|---|---|---|---|---|
| B1 | **Music plays by default, and the off switch is buried** (☰ → Settings → Audio → Music Volume). Add a one-click music toggle next to the sound icon, and/or default music off. | UX | `client/` (our fork) | High |
| B2 | **Scryfall deck URL import.** Find the export endpoint from a real deck URL, then add it alongside the Archidekt and Moxfield importers in `client/src/services/deckUrlImport.ts` and `lobby-worker/src/import-deck.ts`. | Feature | upstream PR | High |
| B3 | **Import and play your own Archidekt decks** and record which cards are unsupported or misplayed. | Test | — | High |
| B4 | **Judge AI quality in Commander** (threat assessment, attack targets, removal targeting) by watching the debug panel's "Show AI decisions" and the `ai-duel` skill or CLI for AI-vs-AI runs. | Evaluation | — | High |
| B5 | **Windows: `gen-scryfall-locale-images.sh` fails on CRLF output from jq.** The check prints `expected: de es fr …` and `actual: de es fr …` (identical apart from a hidden `\r`). Fix: `tr -d '\r'` the way `fetch-token-sets.sh` already does. Local workaround: `~/bin/jq` wraps `jq --binary`. | Bug | upstream PR | Medium |
| B6 | **`setup.sh` aborts before building WASM if any Scryfall sidecar fails**, even the optional locale map (non-English art). Make the locale map non-fatal, or move it after the WASM build. | Bug / DX | upstream PR | Medium |
| B7 | **The locale map step streams the 3 GB `all-cards` file through jq** and is extremely slow on Windows. We skipped it. Consider an opt-in flag. | DX | upstream | Low |
| B8 | **README says `npm i -g pnpm`**, which installs pnpm 12, but `client/` pins pnpm 9.15.9 and setup's preflight rejects the mismatch. Document `npm i -g pnpm@9` or corepack. | Docs | upstream PR | Low |
| B9 | **The build leaves generated files modified** (`crates/engine/data/mtgjson-vintage`, `oracle-subtypes.json`), so the working tree is dirty after setup. | DX | upstream issue | Low |
| B10 | **A narrow viewport makes setup awkward.** At pane width the match setup collapses and the player-count slider is fiddly. Check how it looks at laptop and tablet widths. | UX | `client/` | Low |
| B11 | A **Commander quick-start** ("Me + 3 AI, random precons, Medium") skipping the setup screen. Gap 2 is mostly covered already, so this is just a shortcut. | UX | `client/` | Low |

### Windows dev setup (as built)
- Rust via `rustup` with the MSVC host (Build Tools already installed). The toolchain is pinned by `rust-toolchain.toml`.
- `cargo install wasm-bindgen-cli@0.2.121` (must match `Cargo.lock`), pnpm 9.15.9, and jq via winget plus the `--binary` wrapper.
- Then `./scripts/setup.sh --no-tilt`. Run `gen-card-data.sh` and `build-wasm.sh` by hand if the sidecar step fails (B6).
- Run the dev servers with `.claude/launch.json`: `frontend` (:5173) and `lobby-worker` (:8787).

---

## 10. Open questions

1. ~~Fork vs separate app~~. Decided: fork.
2. **Which decks to use for M0?** Links to 2–4 of your Archidekt decks, ideally with different colors and
   strategies.
3. **A Scryfall deck URL** to reverse-engineer the export from.
4. **Desktop (Tauri) or browser first?** Desktop avoids the CORS proxy and makes local caching simpler.
5. **Rust comfort:** are you OK writing occasional Rust for engine fixes and AI heuristics, or should
   our work stay TS-only where possible?
