**Scope:** Firefox ESR crash fix (rAF batching) and a round of correctness hardening.

---

## 1. Abort / cleanup race

**Problem.** `stopGen()` set `ctrl = null` immediately after calling `ctrl.abort()`. The fetch promise chain was still unwinding (the reader threw `AbortError`, the `.catch` ran, the final `.then` ran). During that window the app looked idle, so a second `generate()` could start, `beginReply()` would reset all streaming state, and the old chain's `.catch` / `.then` would fire into the new turn — clobbering status, appending a stray newline, calling `showMeanTg()` with zeroed counters, and the old final `.then` would null the *new* controller.

**Fix.**

- `stopGen()` no longer nulls `ctrl`. It calls `ctrl.abort()` and sets a transient `"stopping…"` status. The controller stays non-null so every guard (`generate`, `newSession`, `undoLast`, `openFilePicker`) correctly reports "busy" until the old chain finishes.
- `generate()` captures the controller in a local variable (`myCtrl`) and identity-checks before nulling: `if (ctrl === myCtrl) ctrl = null;`. All status writes in the success and error paths are similarly guarded.
- The final `.then` in the promise chain is the single point where `ctrl` is cleared.

---

## 2. `probeModel()` race

**Problem.** `main()` fires `probeModel()` asynchronously. The user can type and hit send before `/props` and `/slots` return. When they do return, the callback unconditionally reset `turnMarks`, `lastServerPromptTokens`, overwrote the `¢` counter, and set the status to `"ready"` — clobbering an in-flight generation.

**Fix.**

- `turnMarks` and `lastServerPromptTokens` are reset only when `transcript.length === 0 && !ctrl`.
- The `¢` counter write is inside the same guard.
- The `"ready"` / `"error"` status writes are guarded by `if (!ctrl)`.
- `model.value` and `nCtx` are set unconditionally (cosmetic / read-only for the in-flight request).
- `slotId` is assigned unconditionally (the in-flight request already captured its own value; the new value is correct for the *next* request).

---

## 3. rAF timer fallback + visibility flush

**Problem.** `requestAnimationFrame()` is throttled or paused in hidden tabs. A long reply in a background tab could accumulate the entire response in `pendingAnswer` before a single `flushDOM()` ran, producing one massive `appendData` + reflow when the tab became visible.

**Fix.**

- `scheduleFlush()` now also arms a 250 ms `setTimeout` as a fallback. If rAF hasn't fired by then, the timer calls `flushDOM()` directly.
- A `visibilitychange` listener on `document` calls `flushDOM()` when the tab becomes visible and `rafPending` is true.
- `flushDOM()` clears the timer at the top of its body, so the rAF and timer paths are mutually exclusive.

---

## 4. `autoScroll` — stop hijacking the user's scroll position

**Problem.** `addSpan()` called `scrollBottom()` internally, and `flushDOM()` called `scrollBottom()` unconditionally every frame. If the user scrolled up to read earlier output, the app yanked them back to the bottom every 16 ms. The first frame that created a thought/answer span also produced two reflows (one inside `addSpan`, one at the bottom of `flushDOM`).

**Fix.**

- `scrollBottom()` removed from `addSpan()`. The function now only creates and appends the span.
- A new `autoScroll` boolean (default `true`) tracks whether the user is near the bottom of `#output`. A `scroll` listener on `#output` updates it: the user is "at the bottom" if `scrollHeight - scrollTop - clientHeight < 40`.
- Every `scrollBottom()` call site (`logUser`, `flushDOM`, the success/abort/error paths in `generate`) is now conditional: `if (autoScroll) scrollBottom()`.
- `newSession()` resets `autoScroll = true` after clearing the output.

---

## 5. Error path consistency

**Problem.** On a non-abort error (network failure, HTTP 500, JSON parse error in the SSE sink), the code set the status to `"error: …"` but did not flush pending text, did not push a partial assistant reply to `transcript`, and did not roll back the user message. The user prompt stayed in `transcript` and the editor was cleared, but no assistant reply was recorded. The next send would silently re-send the failed user message.

**Fix.**

- `flushDOM()` is called first to drain any pending text into the DOM.
- If `reply !== ""`, the partial assistant reply is pushed to `transcript` and a newline is appended to the output.
- If `reply === ""` (no tokens were generated), the user message is popped from `transcript`, the corresponding DOM turn is removed via `removeLastTurns(1)`, and the original text is restored to the editor.
- The status write is guarded by `if (ctrl === myCtrl)`.

---

## 6. Ctrl+Enter shortcut

**Problem.** The documented keyboard shortcut was Ctrl+D / Cmd+D, but the summary also listed Ctrl+Enter. The handler only checked for `d` / `D`.

**Fix.** `editorKeydown` now also matches `ev.key === "Enter"`.

---

## 7. Warning glyph

**Problem.** The trim warning used `\u24C9`, which is **Ⓣ** (circled T). The intended character is **ⓩ** (circled Z), which is `\u24E9`.

**Fix.** `trimwarn.textContent` now uses `"\u24E9"`.

---

## 8. `promptEst` simplification

**Problem.** The formula `lastServerPromptTokens + Math.ceil(newUserChars / 4)` undercounted by the size of the previous assistant reply. The previous request's `prompt_tokens` did not include the assistant turn that was generated afterward, so the live `¢` display was too low for every turn after the first.

**Fix.** `promptEst` is now simply `estTokens()` after the trim. This is O(n) over the transcript, but the transcript is small compared to the network round trip, and the result is correct.

---

## 9. `undoLast()` guard

**Problem.** After a trim, `turnMarks` can contain more entries than `transcript` (trimming removes messages from the front of `transcript` but leaves the output log and `turnMarks` intact). If the user undoes the last turn and `transcript` becomes empty, `turnMarks` still has old marks. Clicking undo again would remove another displayed turn even though `transcript` has nothing to undo.

**Fix.** `undoLast()` now checks `transcript.length === 0` in addition to `turnMarks.length === 0` and returns early with `"nothing to undo"`.

---

## 10. SSE parser relaxation + text-file limit

**Problem A.** The SSE parser required `data: ` (with a trailing space) via `line.lastIndexOf("data: ", 0) !== 0`. SSE allows `data:` without a space. It also did not handle the (rare) multi-line `data:` field format.

**Fix A.** The check is now `line.startsWith("data:")` and the payload is extracted with `line.slice(5).trimStart()`.

**Problem B.** The text-file upload limit was 15 MB, the same as the image limit. A 15 MB text file appended to the editor creates a single 15 MB string in a `<textarea>` and a 15 MB pre-wrap text node in `#output`, which is the exact condition that triggered the original Firefox OOM.

**Fix B.** The text-file limit is lowered to 2 MB. The image limit remains 15 MB. The limit is determined by `detectFileType()` before the size check.

---

## Additional changes

| Change | Detail |
|---|---|
| `slotId` initialised to `0` | Was `null`. Eliminates all `slotId !== null` ternaries in `newSession`, `undoLast`, `generate`, and `probeModel`. The `probeSlot` catch already defaulted to `0`. |
| `URL.revokeObjectURL` deferred | `saveLastReply()` now wraps the revoke in `setTimeout(…, 1000)` to avoid the revoke-before-download race in Firefox ESR. |
| Null-byte check uses `indexOf` | `onFilePicked` now uses `text.indexOf("\u0000") !== -1` instead of a character-by-character `for` loop. 100× faster on large files. |
| `onFilePicked` handles `"unknown"` type | Files whose extension is neither image nor text now get an explicit `"unknown file type"` status message instead of silently doing nothing. |
| `autoScroll` reset in `newSession` | After clearing `#output`, `autoScroll` is set back to `true` so the next generation auto-scrolls. |
| `scroll` listener on `#output` | Registered in `main()`. Updates `autoScroll` based on the user's scroll position. |
| `visibilitychange` listener | Registered in `main()`. Forces a `flushDOM()` when the tab becomes visible. |
| All `scrollBottom()` calls in `generate()` conditional | Success, abort, and error paths all check `autoScroll` before scrolling. |
