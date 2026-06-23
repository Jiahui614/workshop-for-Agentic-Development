# Meeting Bingo — Implementation Plan

**Derived from**: `meeting-bingo-prd.md`, `meeting-bingo-architecture.md`, `meeting-bingo-uxr.md`
**Target**: Browser-based 5×5 buzzword bingo with live Web Speech API auto-fill. Zero backend, zero cost.
**Build estimate**: ~90 min MVP (matches the workshop target). Note: tight given unwritten components — P1 persistence/rehydration items are the safe droppables if time runs short.

---

## Review Summary

Reviewed: 2026-06-23 | Reviewers: VP Product, VP Engineering, VP Design | Decision: **Approve all**

### Changes Applied

| # | Change |
|---|--------|
| C1 | G5 rewritten: gate `onend` restart on a `shouldListenRef`; remove side effect from `setState` updater (fixes runaway mic restarts). |
| C2 | G2 rewritten with concrete word→square `Map` + immutable square update + recompute count/bingo. |
| E3 | New G7 gate: immutable update helper for nested `squares[][]` (avoid stale UI). |
| P1 | Step 18 now specifies localStorage rehydration on mount + versioned key + game `id`. |
| P2 | Scope note + step 17 + acceptance criterion clarified: share = app URL only, no joinable game link. |
| D1 | New §7a Accessibility section (aria-pressed/labels, non-color listening + fill cues, aria-live). |
| P3 | G1 extended to WinScreen "X/24" (exclude FREE); acceptance criterion updated. |
| P4 | New G8 gate: `data/categories.ts` is single source of truth; mockup words illustrative only. |
| E6 | Step 1 pins Tailwind v3 (`init -p` absent in TW4); React 19 scaffold noted. |
| D2 | Step 13: one-shot `bounce-in` fill animation, not infinite `animate-pulse`. |
| D3 | Step 18: near-bingo hint surfaces missing word(s) + visual intensify (UXR Moment 2). |
| E5 | Step 19: `build.sourcemap: false` for public deploy. |
| E7 | Step 18: cap retained `transcript` state (~500 chars). |
| E8 | Step 19: secure-context note (HTTPS for LAN testing). |
| D5 | Step 14: privacy copy + Enable button before `recognition.start()`. |

### Notes
- E4 (closest-to-win fresh-card edge) works via FREE space — note only, no change.
- D4 (auto/manual visual distinction) addressed by D1's non-color icon cue.
- P5/P6 (sequencing/timeline) captured in the build-estimate note above.

---

## 1. Scope (locked)

**In (MVP):** card generation (5×5, 24 words + center free space), 3 category packs (Agile / Corporate / Tech), Web Speech API transcription, auto-fill on detection, manual tap fallback, BINGO detection (rows/cols/diagonals), win celebration (confetti), shareable result, localStorage persistence, mobile-responsive.

**Out (explicitly):** accounts/auth, multiplayer/real-time sync, custom buzzword creation, backend/DB, sound on by default, leaderboards, calendar integration, the "Custom" pack shown in UXR storyboards. Note: there is no joinable per-game link (no multiplayer/backend); any "share link" is the static app URL only.

---

## 2. Tech stack

| Layer | Choice |
|---|---|
| Framework | React 18 + TypeScript |
| Build | Vite |
| Styling | Tailwind CSS 3 |
| Speech | Web Speech API (`window.SpeechRecognition` / `webkitSpeechRecognition`) |
| Animation | CSS + `canvas-confetti` |
| State | `useState` + Context, persisted to `localStorage` |
| Deploy | Vercel (static, zero-config) |

> The architecture doc's code snippets assume this exact stack (e.g. `tailwind.config.js`). If using React 19 / Tailwind 4, the config model changes (CSS-first `@import "tailwindcss"`, no JS config file) and snippets need light adaptation.

---

## 3. Target structure

```
meeting-bingo/
├── index.html, package.json, tsconfig.json, vite.config.ts
├── tailwind.config.js, postcss.config.js
├── src/
│   ├── main.tsx, App.tsx, index.css
│   ├── components/
│   │   ├── LandingPage.tsx, CategorySelect.tsx
│   │   ├── GameBoard.tsx, BingoCard.tsx, BingoSquare.tsx
│   │   ├── TranscriptPanel.tsx, WinScreen.tsx, GameControls.tsx
│   │   └── ui/ (Button, Card, Toast)
│   ├── hooks/ (useSpeechRecognition, useGame, useBingoDetection, useLocalStorage)
│   ├── lib/  (cardGenerator, wordDetector, bingoChecker, shareUtils, utils)
│   ├── data/ (categories.ts)
│   ├── types/ (index.ts)
│   └── context/ (GameContext.tsx)
└── README.md
```

The architecture doc already provides working code for: `types/index.ts`, `data/categories.ts`, `lib/cardGenerator.ts`, `lib/bingoChecker.ts`, `lib/wordDetector.ts`, `hooks/useSpeechRecognition.ts`, `App.tsx`, `BingoSquare.tsx`, `TranscriptPanel.tsx`. These can be copied largely verbatim.

---

## 4. Gaps & inconsistencies to resolve during build

| # | Issue | Resolution |
|---|---|---|
| G1 | `App.tsx` sets `filledCount: 1` (free space) but PRD shows "X/24"; `countFilled` also counts FREE. | Pick one convention (recommend: count only non-free filled squares) and apply **everywhere**, including the in-game counter AND the WinScreen "X/24" stat (else it shows 13/24). |
| G2 | Detection returns word *strings*; fill state lives in `squares[][]` keyed by `id` — no word→square mapping exists. | Build `Map<wordLower, square>` from `card.squares` once per card. On each final transcript: compute `alreadyFilled` (Set of filled words), call `detectWordsWithAliases`, then for each hit **immutably** update state — clone `squares` (new row/object arrays), set the matched square's `isFilled=true`/`isAutoFilled=true`/`filledAt=Date.now()`, recompute `filledCount` and `checkForBingo`. Never mutate squares in place (breaks React re-render). |
| G3 | Referenced-but-unwritten files: `lib/utils.ts` (`cn`), `GameBoard`, `LandingPage`, `CategorySelect`, `WinScreen`, `GameControls`, `shareUtils.ts`, `ui/*`. | Implement these. `cn` = `clsx` + `tailwind-merge`, or a tiny `filter(Boolean).join(' ')` helper. |
| G4 | Short/substring words ("API", "epic", "merge"). | `\b` word boundaries for single words + substring match for phrases (as in doc). Manual tap covers misses. |
| G5 | `useSpeechRecognition` `onend` calls `recognition.start()` inside a `setState` updater reading stale `isListening` (side effect in reducer; double-fires in StrictMode; runaway restarts after Stop). | Add a `shouldListenRef = useRef(false)`; set `true` in `startListening`, `false` in `stopListening`. In `onend`, restart only `if (shouldListenRef.current)`. Remove the `setState`-updater side effect entirely. |
| G6 | UXR "Custom" pack + multiplayer/leaderboard. | Out of scope; ignore for MVP. |
| G7 | Nested `squares[][]` mutated in place won't re-render. | Establish an immutable update helper that returns a new `BingoCard` with new array/object references for any changed square; use it for both auto-fill and manual toggle. |
| G8 | Mockups show JIRA/Hotfix squares and `WIP limit` that are absent from `categories.ts`. | Treat `data/categories.ts` (architecture doc) as the single source of truth; UXR/PRD card mockups are illustrative only. Do not rely on words shown in mockups existing on cards. |

---

## 5. Build phases

### Phase 1 — Foundation (~20 min)
1. Scaffold: `npm create vite@latest meeting-bingo -- --template react-ts`; install `canvas-confetti`. For Tailwind, **pin v3**: `npm i -D tailwindcss@3 postcss autoprefixer && npx tailwindcss init -p` (note: `init -p` does NOT exist in Tailwind 4). The doc's `tailwind.config.js` keyframes require the v3 JS-config model. If the scaffold pulls React 19, that's fine; do not adopt TW4 unless you also migrate keyframes to the CSS-first `@theme`/`@import "tailwindcss"` model.
2. Configure Tailwind (`content` globs, win animations/keyframes from doc).
3. `src/types/index.ts` — copy interfaces.
4. `src/data/categories.ts` — copy 3 word packs (48 words each).
5. `src/lib/utils.ts` — add `cn` helper.

### Phase 2 — Core game, no audio (~30 min)
6. `lib/cardGenerator.ts` + `lib/bingoChecker.ts` — copy from doc.
7. `BingoSquare.tsx` → `BingoCard.tsx` → `GameBoard.tsx` + `GameControls.tsx`.
8. `LandingPage.tsx`, `CategorySelect.tsx`; wire `App.tsx` screen state (`landing → category → game → win`).
9. Manual toggle → recount filled → `checkForBingo` → win transition. Apply G1 convention.
10. **Checkpoint:** fully playable by clicking; verify all 12 winning lines + regenerate card.

### Phase 3 — Speech (~25 min)
11. `hooks/useSpeechRecognition.ts` — copy; fix G5 restart logic.
12. `lib/wordDetector.ts` — copy (`detectWordsWithAliases`).
13. Wire `onResult(final)` → detect against unfilled card words → auto-fill squares (G2) → toast → bingo check. Fill animation must be a **one-shot** (`bounce-in` keyframe), not the infinite `animate-pulse` in the doc's `BingoSquare` (persistent pulse violates ambient/minimal Principle 1). Trigger animation only on the fill transition, then settle to the static filled state.
14. `TranscriptPanel.tsx` + mic-permission prompt with privacy copy ("processed locally, never recorded"). Show the in-app privacy explanation + an "Enable" button BEFORE calling `recognition.start()` (the native prompt fires on first `start()`).
15. Feature-detect `isSupported`; if false, show manual-only mode (Firefox path).

### Phase 4 — Polish & deploy (~15 min)
16. `WinScreen.tsx` + `canvas-confetti` (sound OFF by default — UXR Principle 3). Highlight winning line; show time-to-bingo, winning word, filled count, category.
17. `lib/shareUtils.ts` — clipboard text summary + the app URL (NOT a joinable game link); `navigator.share` on mobile. Share copy must not imply recipients join *this* game.
18. `useLocalStorage.ts` persistence. Define explicitly: persist (a) the in-progress `GameState` under a versioned key and (b) the last completed result. On `App` mount, rehydrate via `useEffect`: if a `playing` game exists, restore card + screen=`game`; otherwise start at `landing`. Add a game `id` (UUID) to `GameState`. Cap retained `transcript` state (e.g. last ~500 chars) to avoid unbounded growth. "One away!" hint via `getClosestToWin` — surface the missing **word(s)** for the closest line (compute unfilled words of that line), not just the line name (UXR Moment 2); intensify the line visually when one away. Responsive tweaks.
19. `npm run typecheck && npm run build` (set `build.sourcemap: false` for the public Vercel deploy); deploy to Vercel. Web Speech needs a secure context; localhost is OK, use HTTPS for LAN/device testing.

---

## 6. Acceptance criteria (success = all pass)

**Functional**
- [ ] Card has 24 unique words + center FREE space; regenerate produces a different card.
- [ ] Each category yields distinct cards.
- [ ] Manual tap toggles fill (and un-fill).
- [ ] BINGO fires on all 12 lines (5 rows, 5 cols, 2 diagonals); free space counts toward its lines.
- [ ] Win screen shows correct time, winning word, filled count (X/24, FREE excluded per G1), category.
- [ ] Share copies correct content incl. the app URL (no implied joinable game).

**Speech**
- [ ] Mic permission prompt appears with privacy messaging.
- [ ] Transcription begins < 1s after enable; continuous mode survives silence.
- [ ] Spoken buzzword auto-fills within ~500ms; toast shows detected word.
- [ ] Compound phrases detected ("code review", "scope creep").
- [ ] Same word twice fills once; multiple words in one sentence all fill.
- [ ] Graceful manual-only fallback when API unavailable.

**Non-functional**
- [ ] Card renders < 100ms; celebration doesn't jank.
- [ ] No audio recorded or transmitted (verify: nothing leaves the browser).
- [ ] Responsive on mobile portrait + landscape.
- [ ] `typecheck` + `build` clean.

---

## 7. Key UX principles to honor (from UXR)
1. **Ambient engagement** — minimal UI, peripheral-vision friendly.
2. **Earned delight** — per-word auto-fills build toward the BINGO payoff.
3. **Silent celebration** — satisfying but discreet; sound off by default (user is on mute in a meeting).
4. **Trust through transparency** — explicit local-processing privacy copy at the mic prompt.
5. **Minimal friction** — maximize automation; manual tap is the safety net, not the main path.

---

## 7a. Accessibility (required, not optional)
- Each square is a `<button>` with `aria-pressed={isFilled}` and an `aria-label` (e.g., `"Sprint, filled"` / `"Sprint, not filled"`); FREE space `aria-label="Free space"` and `aria-disabled`.
- Distinguish auto vs manual fill with a non-color cue (e.g. a ✨/✓ icon), not hue alone.
- Listening indicator must not be color-only: pair the red dot with visible text ("Listening" / "Paused") and an `aria-live="polite"` status.
- Detected-word toasts and the "one away!" hint announce via an `aria-live="polite"` region.

---

## 8. Risks & mitigations
| Risk | Mitigation |
|---|---|
| Web Speech API unavailable (Firefox) | Feature-detect; fall back to manual-only mode; core game works without speech. |
| Poor transcription accuracy | Manual tap always available; word aliases (CI/CD, ROI, MVP…). |
| Meeting audio not captured | User holds device near speaker; manual fallback. |
| Scope creep | Hard lock on §1; multiplayer/custom packs are post-MVP only. |

---

## 9. Post-MVP backlog (do not build now)
Custom word lists · multiplayer sync · achievements/streaks · meeting-type templates · sound toggle · dark mode · export history · PWA.
