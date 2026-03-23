# Speech Rocket — Project Context for AI Assistant

> This file is the single source of truth for continuing development of Speech Rocket.
> Read it fully before making any changes. Follow every constraint listed here strictly.

---

## Project Overview

**Speech Rocket** is an English speech pronunciation training game.
- Single-file web app: `index.html`
- Mobile-first (iOS Safari priority), also Chrome
- Built by **Inream** (inream.com) — proprietary, copyright stated in file header
- Target: deployed at `https://do.inream.com/speech-rocket.html`

---

## File Structure

```
speech-rocket/
└── speech-rocket.html     ← entire app: HTML + CSS + JS, all in one file
└── CLAUDE.md              ← this file
```

Do not split into separate CSS/JS files unless explicitly instructed.
Do not introduce a build system, bundler, or framework unless explicitly instructed.

---

## Tech Stack — Hard Constraints

| Concern | Solution                                   | Rule |
|---|--------------------------------------------|---|
| Speech recognition | Web Speech API (`webkitSpeechRecognition`) | Never replace with a third-party SDK |
| TTS | `window.speechSynthesis`                   | Never replace with a third-party SDK |
| Styling | Vanilla CSS with CSS variables             | No Tailwind, no CSS-in-JS |
| JS | Vanilla ES6+                               | No React, no Vue, no jQuery |
| Fonts | Google Fonts (Orbitron + Nunito)           | Keep these exact two fonts |
| QR code | `qrcodejs` via cdnjs CDN                   | No swap |
| Analytics | Google Analytics 4 (`gtag`)                | ID placeholder = `G-XXXXXXXXXX` |
| Payments | Embedded AllPay form (`allpay.to`)         | login=`pp1005009`, key=`ED56551809D83BBF612D0C7711BC0A11` |

---

## Architecture — Code Sections

The `<script>` block is divided into **13 named sections**. Keep them in order, keep the section headers.

```
SECTION 1:  CONFIG          — all game constants and tunable parameters
SECTION 2:  PHRASES         — word lists (singles / pairs / full)
SECTION 3:  STATE           — mutable game state object
SECTION 4:  SEQUENCE BUILDER
SECTION 5:  SPEECH RECOGNITION
SECTION 6:  GAME LOGIC
SECTION 7:  CLOUD ANIMATIONS
SECTION 8:  AUDIO (Web Audio API)
SECTION 9:  PAGES & UI
SECTION 10: ANALYTICS
SECTION 11: PAYMENT & SUPPORT
SECTION 12: TOAST & UTILS
SECTION 13: INITIALIZATION
```

**Rule:** All game parameters belong in `CONFIG` (Section 1). Never hardcode a magic number elsewhere.

---

## CONFIG Reference (Section 1)

```js
ACCURACY_THRESHOLD_EARLY: 50     // % — threshold for first EARLY_WORD_COUNT words
ACCURACY_THRESHOLD_NORMAL: 65    // % — threshold after first EARLY_WORD_COUNT words
EARLY_WORD_COUNT: 2              // how many words use the lower threshold
FALL_DURATION_INITIAL: 9500      // ms — first word fall time
FALL_DURATION_MIN: 3200          // ms — fastest possible fall
FALL_DURATION_DECREASE: 180      // ms — reduction per correct word
METERS_PER_WORD: 300             // score meters per correctly spoken word
FAKE_PLAYERS_MIN / MAX: 300–700  // random live player count range
TTS_RATE: 0.82                   // speechSynthesis rate
TTS_PITCH: 1.05
TTS_VOLUME: 0.9
CLOUD_DURATION_INITIAL: 9        // seconds per cloud cycle
CLOUD_DURATION_MIN: 2.2
CLOUD_SPEED_FACTOR: 0.06         // cloud speed increase per correct word
GAME_URL: 'https://do.inream.com/speech-rocket.html'
GA_ID: 'G-XXXXXXXXXX'           // replace with real GA4 ID before deploy
ALLPAY_LOGIN / KEY / CURRENCY
CRYPTO_ADDRS: { btc, bsc, poly, eth, ton }
ACHIEVEMENTS: array of [wordsNeeded, icon, title]
RANDOM_NAMES: pool of 20 names for leaderboard
```

---

## Pages

| ID | Route | Description |
|---|---|---|
| `#page-start` | default | Logo, rules, Start button, mic request |
| `#page-gameplay` | after start | Falling words, rocket, HUD, listening UI |
| `#page-result` | after crash | Score, failed word (spoken by TTS), stats |
| `#page-leaderboard` | after result | Top %, table, achievement, QR, next badge |
| `#page-support` | from result | Share link, AllPay donation, crypto addresses |

Navigation is done exclusively via `showPage(id)`. Never use `location.href` for page changes.

---

## Word List Logic

70 phrases total, split into 3 tiers:

- **Tier 1 — Singles** (`PHRASES.singles`, 12 phrases): each phrase is broken into **individual words** shown one at a time.
- **Tier 2 — Pairs** (`PHRASES.pairs`, 24 phrases): each phrase is broken into **2-word chunks**.
- **Tier 3 — Full** (`PHRASES.full`, 34 phrases): the **entire 4-word phrase** is shown as one unit.

Each tier is independently shuffled at game start. When the sequence is exhausted, it is extended by rebuilding (so the game is infinite).

---

## Speech Recognition — Critical Rules

1. `SpeechRecognition` instance is created **once** (`initRecognition()`), reused forever.
2. Mic permission is requested **once** via `navigator.mediaDevices.getUserMedia` on first Start press.
3. Recognition is `continuous: true`, `interimResults: true`.
4. **`STATE.listeningEnabled`** is the single gate. It is `false` during TTS and for 350ms after TTS ends. The `onresult` handler must check this flag first and return early if false.
5. Never stop/restart recognition between words — keep it running; only gate via `listeningEnabled`.
6. All transcription results must be logged to `console.log` with `[SpeechRocket]` prefix.
7. Accuracy uses **Levenshtein fuzzy matching** — max edit distance 1 for words ≤4 chars, 2 for longer.

---

## TTS — Critical Rules

`speakWord(text)` must:
1. Set `STATE.listeningEnabled = false` immediately.
2. Cancel any existing `speechSynthesis`.
3. Set `utter.onend` → wait 350ms → set `listeningEnabled = true` (only if word still active).
4. Set a **fallback timer** (`estDuration + 400ms`) in case `onend` never fires (iOS bug).
5. Never trigger recognition re-enable from anywhere else.

---

## Gameplay Loop

```
startGame()
  → buildSequence()
  → showPage('page-gameplay')
  → startRecognition()
  → nextWord()

nextWord()
  → set listeningEnabled = false
  → speakWord()          — mutes mic, speaks, then re-opens mic after TTS + 350ms
  → show bubble
  → after ttsDelay: add 'falling' class, start fallTimer

onresult (recognition)
  → if !listeningEnabled → ignore
  → calcAccuracy(transcript, STATE.currentTarget)
  → if accuracy >= threshold → wordCorrect()

wordCorrect()
  → increment score/meters
  → speed up fall & clouds
  → show accuracy badge
  → next word after 700ms + 300ms gap

wordMissed()   (fallTimer fires)
  → STATE.isPlaying = false
  → crash animation + sound
  → stopRecognition()
  → showResultPage() after 800ms
```

---

## CSS Architecture — Rules

- All colors via CSS variables in `:root` — never hardcode hex elsewhere.
- Pages use `position: fixed; inset: 0` with `display: none` / `display: flex`.
- `#page-gameplay` is `position: fixed` and acts as stacking context for all its children.
- Gameplay children (`gameplay-hud`, `word-area`, `listen-hud`, `rocket-area`) use `position: absolute` — **never `position: fixed`** — so they are contained within the gameplay page and don't leak over other pages.
- Result, leaderboard, support pages use `z-index: 30` to render above sky/cloud layer.
- Sky background (`.sky-bg`) and clouds (`.cloud-layer`) are `z-index: 0–2`, always behind pages.

---

## Design Constraints

- Fonts: `Orbitron` (display/headings) + `Nunito` (body). No substitutions.
- Color palette: navy `#0d1b4b`, blue `#1565C0`, sky `#42A5F5`, orange accent `#FF6B35`, gold `#FFC107`, green success `#69F0AE`, red danger `#FF5252`.
- Rocket SVG: inline, two instances (start page hero + gameplay). Both use `viewBox="0 0 70 140"`. Flame group at `translate(35,110)` — centered on rocket body bottom.
- Mobile-first: `max-width: 420px` cards, `clamp()` font sizes, no `user-select`, `-webkit-tap-highlight-color: transparent`.

---

## Analytics Events (GA4)

Every user action must fire `gaEvent(name, params)`:

| Event name | When | Key params |
|---|---|---|
| `page_view` | every page open | `page` |
| `game_start` | on Start press | `attempt` |
| `result` | on crash | `score, meters, correct_words, accuracy` |
| `leaderboard_view` | on LB open | `score, place, top_percent` |
| `support_page_open` | on Support open | `from` |
| `donate_card_click` | card pay btn | `amount, currency` |
| `crypto_toggle` | crypto btn | — |
| `crypto_copy` | address copy | `coin` |
| `share_link_copy` | link copy | — |

---

## Known Issues / Completed Fixes

- ✅ Gameplay overlays no longer bleed onto result/support pages (fixed: `position:absolute` not `fixed`)
- ✅ Rocket flame is centered (fixed: `translate(35,110)`, `viewBox 0 0 70 140`)
- ✅ TTS output no longer triggers recognition (fixed: `listeningEnabled` gate + 350ms buffer)

---

## Pending / Next Steps

- [ ] Replace `G-XXXXXXXXXX` with real Google Analytics 4 Measurement ID
- [ ] Test AllPay payment flow end-to-end (sandbox → production)
- [ ] iOS Safari: test `speechSynthesis.onend` reliability (fallback timer exists but verify)
- [ ] Desktop app wrapper: Electron `main.js` + `package.json` (mic permission via `setPermissionRequestHandler`, external links open in system browser)
- [ ] Optional: bundle Google Fonts + QRCode.js locally for offline support
- [ ] Optional: real backend leaderboard (currently fake/random)
- [ ] Optional: user accounts / persistent scores beyond `localStorage`

---

## External Links Used in App

| Purpose | URL |
|---|---|
| Terms of Use | `https://inream.com/terms_of_service` |
| Privacy Policy | `https://inream.com/privacy_policy` |
| Main product | `https://inream.com` |
| Game share URL | `https://do.inream.com/speech-rocket.html` |
| AllPay | `https://allpay.to/` |

---

## Copyright

© Inream (inream.com). All rights reserved.
This game may be played and shared freely between individuals.
Commercial use, resale, or rebranding requires written permission from Inream
and must credit Inream as the author on the title page.
This notice must be preserved in all derivative files.
