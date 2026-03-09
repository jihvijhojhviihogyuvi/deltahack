# Plan: Auto-check for New Questions

## Goal
Automatically detect when a new DeltaMath question appears and trigger a scan without requiring the user to click the **Scan** button.

## 1) Define what counts as a “new question”
- Identify stable DOM markers that change when a question changes (e.g., question container text, prompt node, problem ID in URL, or assignment item attributes).
- Build a `getQuestionFingerprint()` helper that returns a normalized hash/string from:
  - Current page URL path/query (if it encodes problem identity)
  - Visible question prompt text
  - Optional metadata (question number, title, or problem ID attributes)
- Store `state.lastQuestionFingerprint` and ignore scans when the fingerprint is unchanged.

## 2) Add auto-check settings
- Add `state.autoCheck` (default `false`) and persist with localStorage key `math_scanner_auto_check`.
- Add `state.autoCheckIntervalMs` (default `1500`–`2500`) and persist with localStorage key `math_scanner_auto_check_interval`.
- Expose both options in Settings UI:
  - Toggle: **Auto-check for new questions**
  - Optional interval control (preset values to avoid aggressive polling)

## 3) Implement change detection engine
Use a hybrid approach for reliability:

### A. MutationObserver (primary)
- Create `startQuestionObserver()` that watches only relevant question containers.
- On mutations, schedule a debounced check (e.g., 300–500ms).
- Recompute fingerprint; if changed, call `onQuestionChanged()`.

### B. Fallback interval polling (secondary)
- Create `startQuestionPolling()` with `setInterval` using `state.autoCheckIntervalMs`.
- Poll fingerprint and compare against `state.lastQuestionFingerprint`.
- Start polling only when auto-check is enabled.

## 4) Add safe scan orchestration
- Add `state.isScanning` gate to prevent parallel requests.
- Add `state.lastAutoScanAt` and minimum cooldown (e.g., 2–3s) to avoid repeated scans from rapid DOM churn.
- Add `triggerScan(reason)` wrapper:
  - Validates API key exists
  - Skips when scan already in progress
  - Calls existing `scanPage()`
  - Logs reason (`observer`, `poll`, `manual`) for debugging

## 5) Lifecycle and cleanup
- On startup:
  - Initialize fingerprint once page is ready.
  - If auto-check enabled, start observer + polling fallback.
- On settings change:
  - Enable/disable observer and polling dynamically.
- On UI close/remove:
  - Keep background detection running (if desired) or tie to UI lifecycle explicitly.
- Add cleanup helpers:
  - `stopQuestionObserver()`
  - `stopQuestionPolling()`

## 6) UX improvements
- Show lightweight status text in panel, e.g.,
  - `Watching for new questions...`
  - `New question detected. Scanning...`
- Add a subtle cooldown indicator when changes are detected too quickly.
- Keep manual Scan button available as override.

## 7) Edge cases to handle
- Question text updates in-place during animations/loading skeletons.
- Non-question DOM changes triggering false positives.
- Navigation between tabs/assignments where URL changes but prompt temporarily empty.
- Same question reopened after wrong answer; avoid redundant scans unless content changed.

## 8) Validation checklist
- Auto-check disabled: no automatic scans.
- Auto-check enabled: scans exactly once per genuinely new question.
- Rapid question changes: no overlapping requests.
- API failures: retries only on next detected new question (no tight retry loop).
- Performance remains smooth (observer scoped, polling interval conservative).

## 9) Implementation order
1. Add state + persistence keys + settings UI controls.
2. Implement fingerprint helper and compare logic.
3. Add observer + polling fallback.
4. Add scan gate/cooldown wrapper and integrate with `scanPage()`.
5. Add cleanup + lifecycle wiring.
6. Test on real DeltaMath flows and tune debounce/cooldown/interval defaults.
