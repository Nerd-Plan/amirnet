# AmirNet Coach — Product & Engineering Audit

Audited version: **3.6** (live at https://nerd-plan.github.io/amirnet/, cache `amirnet-v13`), 10 Oct 2026.
Scope: the deployed single-file app (`index.html`, `sw.js`, `manifest.webmanifest`) and the build pipeline that produces it (`tools/build*.py`, `tools/patch_*.js`, data files, e2e tests).

**Line numbers** refer to the deployed `index.html` of v3.6.

**Status labels**
- **CONFIRMED**: reproduced from code or by measuring the data.
- **RISK**: plausible failure mode, not reproduced.
- **ASSUMPTION**: an unvalidated design premise.

---

## 0. Official reference (verified 10 Oct 2026)

Sources: NITE AMIRNET FAQ, NITE "Guidelines for Examinees", NITE score-meaning pages (EN + HE), Tel Aviv University placement table.

| Topic | Official / verified | App (v3.6) | Match |
|---|---|---|---|
| Question types | Sentence Completion, Restatement, Reading Comprehension | Same three | ✅ |
| Scored sections | 6 sections in a fixed order: SC, SC, RC, RS, RS, SC | `CHAPTERS` is identical | ✅ |
| Items / time per section | SC 4 q / 4 min; RC 5 q / 15 min; RS 3 q / 6 min. 23 items, 39 min | Identical | ✅ |
| Experimental sections | 7th–8th section: 2 experimental sections, or 1 writing task (12 min). Very little weight | Not simulated | ⚠️ gap (low) |
| Adaptivity | Section-level adaptive: after each section, the next section is chosen by estimated level | Simulation items are pre-selected at start; no between-section adaptation | ⚠️ fidelity gap |
| Navigation | Can move within a section, not back to earlier sections. May move on early **only if every question is answered** | Back-navigation ✅. Early exit allowed with blanks ❌ | ⚠️ minor |
| Unanswered items | Count as wrong; examinees are advised to guess | Counted wrong ✅ | ✅ |
| Score scale | 50–150, equated with Psychometric-English / AMIR / AMIRAM | Clamped to 50–150 | ✅ |
| Level cut-offs | **NITE publishes none.** Each institution decides. TAU example: Pre-Basic A 50–69, Pre-Basic B 70–84, Basic A 85–94, Basic B 95–99, Adv. A 100–119, Adv. B 120–133, Exempt 134+ | `band()` uses 85/100/120/134 with no disclaimer | ⚠️ must be labelled institution-specific |
| Official score formula | Not published. Scores are equated across adaptive forms | App shows a "ציון משוער" from its own Elo-style model | ❗ must be labelled uncalibrated |
| 2027 changes | NITE FAQ: the test "will change in stages, beginning in 2027", adding **listening** and **writing**, with speaking later; announcements to follow "with sufficient advance notice" | Not implemented | ✅ correct to wait. Track as versioned requirement `EXAM_FORMAT=2026` |

No official exam questions are copied into the bank. All items are original or AI-generated (see C-4).

---

## 1. Findings

### CRITICAL

**C-1 · The always-visible "readiness %" relies on invented vocabulary and accuracy requirements — CONFIRMED**
- **Where:** `goalState()`, `GOALS`, `goalBarHTML()`, `goalDetailHTML()` (v3.5 patch, ≈L3330–3400).
- **Evidence:**
  - `GOALS={100:{words:1200,acc:60},120:{words:2000,acc:72},134:{words:3000,acc:85}}`.
  - Readiness = 0.5·score + 0.3·words + 0.2·accuracy.
  - The word and accuracy thresholds were chosen by the developer. They have no official or empirical basis.
  - The score part also falls back to the default `base` (91) when no estimate exists.
  - An achievement, "הגעת למוכנות מלאה", fires when the % reaches 100.
- **Impact:** The most prominent number in the app presents made-up requirements as readiness for "פטור". Users may under- or over-prepare and trust a number with no validity.
- **Fix:**
  - The readiness ring reflects **only** the practice estimate vs. the chosen target, and only once an estimate exists. Before that: "אין עדיין אומדן".
  - Vocabulary becomes a **user-set personal goal**, labelled as such. Accuracy is shown as information without a threshold. Neither enters the %.
  - Level cut-offs get an "institution-specific" note.
  - The readiness achievement keys on estimate ≥ target.
- **Acceptance:**
  - No code path computes readiness from word counts or accuracy thresholds.
  - The UI shows the personal-goal label and the cut-off disclaimer.
  - A fresh user sees no readiness %.
- **Regression test:** `e2e_v37.js` → "readiness uses estimate only", "fresh user has no %", "personal vocab goal editable".

**C-2 · The score estimate is shown as a precise score, without disclosure that it is uncalibrated; the simulation uses an invented fallback formula — CONFIRMED**
- **Where:** `estimate()` L1290, `initTheta()` L2172, `finishSim()` L2389, `viewSim()` L2426, `viewStats()` L2463, practice summary L2281.
- **Evidence:**
  - The estimate is `113+15·weighted θ`, where θ is an Elo-style update seeded from `S.base`, with `DEFAULT.base=91`.
  - Item difficulty `bOf=(d−2)·1.1` comes from author-assigned labels. No validation data exists.
  - `finishSim()`: `sim.est = estimate() || Math.round(50 + raw/total*100)`. That linear formula is invented.
  - Texts read "הציון המשוער שלך: 112".
- **Impact:** Users read a practice heuristic as a predicted official score.
- **Fix:**
  - One shared label: "אומדן תרגול — לא ציון רשמי, לא מכויל מול ציוני מאל״ו".
  - Remove the fallback formula. With fewer than 15 reviewed answers, the simulation reports raw correct/total only.
  - Show the estimate with "~" and the number of answers it is based on.
- **Acceptance:**
  - Every place that shows the estimate includes the disclaimer.
  - `finishSim` never fabricates a score.
- **Regression test:** "short simulation without enough data shows no estimate", "estimate text carries disclaimer".

**C-3 · AI-generated (unreviewed) questions change the score estimate, skill statistics and accuracy — CONFIRMED**
- **Where:** `aiPractice()` → `startSet()` → `choose()` → `record()` L2236.
- **Evidence:**
  - `record()` updates `S.attempts`, `S.items`, `S.skills` and `S.theta` for any question, including ids `ai-*`.
  - Item difficulty `d` is self-reported by the model.
- **Impact:** Unvalidated content, with possibly wrong keys or difficulty, contaminates the estimate and skill analysis. This violates the separation between reviewed and generated content.
- **Fix:**
  - `record()` logs AI items to a separate `S.aiLog` and keeps daily activity counts, but does not touch `attempts/items/skills/theta`.
  - Migration v5 removes existing `ai-*` attempts and items, and recomputes `S.skills` from reviewed items.
- **Acceptance:**
  - Answering AI items leaves θ, `skills`, `attempts` and `items` unchanged.
  - The migration is idempotent and preserves all other progress.
- **Regression test:** "AI answers don't move θ/skills", "migration v5 strips ai attempts, keeps cards".

**C-4 · The vocabulary-mastery metric counts words seen once — CONFIRMED**
- **Where:**
  - `grade()` L1533: a first-sight "ידעתי" sets `box=2`, or `box=3` with a correct typed guess.
  - `masteredN()` L1714 and inline `box>=3` counters at L1377, L1508, L2077 and the "hard20" achievement.
- **Evidence:** A word typed correctly once, on first sight, is immediately "שולט/בשליטה", even though it has never been retrieved after a delay.
- **Impact:** The "mastered words" count, achievements and goal progress are inflated. That is a trustworthiness problem for a progress metric.
- **Fix:**
  - `isMastered(c) = box ≥ 3 AND ok ≥ 2`, meaning at least one successful later review. Use it in every mastery counter.
  - SRS scheduling is unchanged.
- **Acceptance:** A word known once on first sight is not counted as mastered until it passes a later review.
- **Regression test:** "first-sight known word is not mastered; after review it is".

### HIGH

**H-1 · Service worker caches every `*.googleapis.com` GET cache-first, including the Gemini API model list and error responses — CONFIRMED**
- **Where:** `sw.js`, fetch handler: `u.hostname.endsWith("googleapis.com")`.
- **Impact:** `GET generativelanguage.googleapis.com/v1beta/models` is served from cache forever, and a cached 401 survives a key fix. Responses to authenticated requests are stored in Cache Storage.
- **Fix:** Limit the runtime cache to `fonts.googleapis.com` and `fonts.gstatic.com`, and only cache `ok` responses.
- **Regression test:** `sw_test.js` (Node, fake SW globals) checks routing for font, API and same-origin URLs.

**H-2 · Answer-length cue: the correct option is the longest one in 61% of items — CONFIRMED (measured)**
- **Evidence:**
  - RS: 70 of 92 (76%). RC: 84 of 115 (73%). SC: 65 of 151 (43%). A neutral bank would be near 25%.
  - A test-wise strategy ("pick the longest") would score far above ability, which inflates practice accuracy and the estimate.
- **Fix:** Content pass to rebalance distractor length, plus a CI check that fails when more than 35% of a type has the correct option longest. Not in the critical batch because it is content work. **The plan includes it as the first content task.**
- **Acceptance:** Longest-correct rate is at most 35% per type.

**H-3 · Source code, data pipeline and tests are not in the repository — CONFIRMED**
- **Evidence:** The repo holds only built files. `tools/` (build scripts, 7,800-word data, 7 e2e suites) exists only in the developer workspace.
- **Impact:** Changes are not reviewable or reproducible, and losing the workspace means losing the pipeline.
- **Fix:** Commit `src/` and `tools/` (or a `dev/` folder) together with this audit.

**H-4 · Simulation state is lost on reload, backgrounding or crash — CONFIRMED**
- **Where:** `sim` is an in-memory variable (`startSim`).
- **Impact:** Mobile OSes kill backgrounded PWAs, so the user loses a 39-minute attempt.
- **Fix:** Persist `sim` (answers, chapter, deadline) in `S.simRun`, resume on load with the time remaining, and auto-finish expired chapters.

**H-5 · Simulation is not section-adaptive and allows leaving a section with blanks — CONFIRMED (fidelity)**
- **Fix:**
  - Choose each next section's difficulty band from the running section score.
  - Mirror the official rule: early "סיום הפרק" only when all items are answered. Timeout still moves on.

**H-6 · Score estimate has no validation and no uncertainty — ASSUMPTION**
- Even after C-2 labelling, the estimate is an untested model.
- **Plan:** Collect anonymised (opt-in) pairs of practice data and official score, then fit and report the error. Until then, keep the "uncalibrated" label.

### MEDIUM

| ID | Issue | Where | Fix |
|---|---|---|---|
| M-1 | Content is AI-authored and has not been reviewed by an expert. 7,812 words (1,441 translations changed in v3.6) and 358 items. Difficulty labels are not empirically derived. **ASSUMPTION** | data files | Expert sample review (e.g. 10% stratified), an in-app "report an error" button, and empirical difficulty from response data |
| M-2 | `persist()` swallows `QuotaExceededError` silently. **RISK** | `persist()` L1233 | Detect the failure, warn the user, offer a backup file |
| M-3 | No schema versioning since v4. New fields are added ad hoc (`ai`, `rg`, `gh`, `goalHit`…). **CONFIRMED** | `migrate()` | Bump to v5 in this batch. Every future field goes through `migrate()` with a test |
| M-4 | `importBackupText()` validates only `.cards` and replaces all state with no undo. **CONFIRMED** | backup | Keep a pre-import snapshot and validate the main shapes |
| M-5 | Network-first SW has no timeout, so the app shell can hang on a weak connection ("lie-fi"). **RISK** | `sw.js` | Race the network against the cache with a ~3 s timeout |
| M-6 | No cost transparency for AI (no per-call token estimate or spend). **CONFIRMED** | AI module | Show approximate tokens per action and a running counter |
| M-7 | Phrases/connectors overlap the word list (48 entries appear in two lists). **CONFIRMED** | data | Dedupe or cross-link |
| M-8 | `DEFAULT.base=91` is the original author's score. A new user who skips settings gets a θ prior of 91. **CONFIRMED** | `DEFAULT`, `initTheta` | Ask on first run, or use a neutral prior with a "not set" state |

### LOW

- **L-1:** `icon-maskable.png` is not in the SW precache.
- **L-2:** The level map lacks the Pre-Basic A/B and Basic A/B split used by some institutions.
- **L-3:** The game and some screens rely on colour (✓/✗ also have icons, so this is mostly fine).
- **L-4:** The legacy `connect()` (claude artifact db) path is dead code.
- **L-5:** Experimental and writing sections are not simulated. They are unscored or low-weight; add once the 2027 format is officially specified.

### Verified as OK

- Section structure and timing match NITE.
- Unanswered items count as wrong.
- Per-section deadline timer is wall-clock based (survives throttled intervals).
- Answer keys are shuffled per presentation (source keys are always index 0).
- No duplicate options.
- Every SC item has a blank.
- Every item has an explanation.
- All 7,812 words have an example sentence.
- The API key is stored only in `localStorage`, under a separate key, and is excluded from backups.
- No secrets are in the repository.
- `prefers-reduced-motion` is respected.
- Pinch-zoom is not disabled.

### Test coverage (before this batch)

Seven Playwright suites (≈200 assertions) cover UI flows, the game, AI mocks and the goal bar. **Missing coverage:**
- scoring and estimate disclosure
- migration
- service-worker routing
- AI-isolation
- mastery definition

---

## 2. Prioritised plan

| # | Item | Type | Batch |
|---|---|---|---|
| 1 | C-1 readiness from estimate only; personal vocab goal; cut-off disclaimer | metric | **now** |
| 2 | C-2 estimate labelling; remove invented sim fallback | metric | **now** |
| 3 | C-3 isolate AI items; migration v5 | data | **now** |
| 4 | C-4 mastery = box ≥ 3 and ≥ 2 successful reviews | metric | **now** |
| 5 | H-1 SW: cache fonts only, ok-only | infra | **now** (2-line fix) |
| 6 | H-3 commit source, tests and this audit | process | next |
| 7 | H-2 distractor-length rebalance + CI check | content | next |
| 8 | H-4 persist and resume simulations | reliability | next |
| 9 | H-5 section-adaptive sim, official early-exit rule | fidelity | next |
| 10 | M-2/M-4 storage-failure warning, import snapshot | reliability | next |
| 11 | M-1 expert content review + "report error" | content | ongoing |
| 12 | H-6 estimator validation study | validity | later |
| 13 | 2027 format (listening/writing) behind `EXAM_FORMAT`, only after the NITE spec is published | requirements | when announced |
