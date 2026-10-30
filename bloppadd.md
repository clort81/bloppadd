## Project: bloppadd

A single-file, zero-dependency HTML client for `llama-server` (llama.cpp). No frameworks, no build step, no node_modules. One `.html` file you open in a browser. The philosophy is explicit in the source: **one file, no frameworks, no build step.**

---

## Architecture at a Glance

```
┌─────────────────────────────────────────────┐
│  #output  (flex:1, own scrollbar)           │
│  Colored spans: .user / .thought / .answer  │
├─────────────────────────────────────────────┤
│  #bottom  (flex:none, pinned)               │
│  ┌──────────────────────────┬──────────────┐│
│  │ #edwrap > #editor        │ #settings    ││
│  │ (textarea, green text)   │ 13% wide     ││
│  └──────────────────────────┴──────────────┘│
└─────────────────────────────────────────────┘
```

- `html, body` are `height:100%; overflow:hidden`. The browser scrollbar never appears. Only `#output` and `#editor` scroll internally.
- The page is a flex column. `#output` stretches; `#bottom` is fixed-height.

---

## Data Flow (One Turn)

1. User types in `#editor`, hits **send** (or Ctrl+D).
2. `generate()` guards (busy? empty? no server?), then:
   - Appends a green `.user` span to `#output`.
   - Pushes `{role:"user", content:text}` onto `transcript[]`.
   - Computes `promptEst` (anchored to `lastServerPromptTokens` if available, else `chars/4` of transcript).
   - Calls `trimTranscript()` if over 75% of `nCtx`.
   - `POST /v1/chat/completions` with `stream:true`, `stream_options:{include_usage:true}`, `slot_id`, sampling params.
3. SSE chunks arrive via `reader.read()` → `makeSseSink` → `onObj`:
   - `delta.reasoning_content` → `appendThought()` (grey, small, `.thought` span).
   - `delta.content` → `appendAnswer()` (bright, `.answer` span).
   - `o.timings` → `showStats(pp, tg)` (final chunk only).
   - `o.usage` → snaps `¢` to `total_tokens`, stores `prompt_tokens` as new `lastServerPromptTokens`.
4. Stream ends → assistant reply pushed to `transcript`, status → `"done"`.
5. `ctrl` (AbortController) set to `null`.

---

## Key State Variables

| Variable | Purpose |
|---|---|
| `transcript[]` | `[{role, content}, ...]` — the conversation history sent to the server. **Does not include reasoning.** |
| `turnMarks[]` | Integer indices into `output.childNodes` where each user span begins. Used by undo to remove DOM nodes. |
| `lastServerPromptTokens` | Ground-truth token count from the last completed turn's `usage.prompt_tokens`. Zeroed on trim, abort, new, undo. |
| `slotId` | Pinned server slot (from `GET /slots`). Sent as `body.slot_id`. |
| `nCtx` | Model context window, from `/props`. |
| `ctrl` | `AbortController` for the in-flight request. `null` = idle. |
| `reply` | Accumulated answer text for the current turn (not in transcript until turn ends). |
| `thoughtChars` | Char count of streamed reasoning (for live ¢ estimate only; not stored in transcript). |
| `tgLastTime`, `tgLastTokens` | Rolling 800 ms window for live tokens/sec display. |

---

## Server Requirements

- `llama-server` with `--reasoning-format auto` (or `deepseek`) for out-of-band grey thoughts.
- Without that flag, reasoning arrives in-band in `delta.content` and renders as part of the answer.
- The server renders the chat template itself (Jinja in the GGUF). The client sends raw `messages[]` and trusts the server for stop tokens, turn boundaries, and template rendering.
- Endpoints used:
  - `POST /v1/chat/completions` (OpenAI-compatible, streaming)
  - `GET /props` (model name, `n_ctx`, `chat_template_caps`)
  - `GET /slots` (discover slot ID)
  - `POST /slots/{id}?action=erase` (wipe KV cache)

---

## Design Decisions & Rationale

### Trimming policy
- Budget: 75% of `nCtx`. Target after trim: 60%.
- Drop oldest messages in a batch (not one-per-overflow).
- Rationale: truncation invalidates the KV prefix from token 0 → one full re-prefill ("the cliff"). Pay it rarely, in big chunks.
- `chars/4` token estimate is intentionally crude. Good enough for the jump-or-not decision.

### Reasoning is not stored in transcript
- `transcript` holds only `{role, content}` with the answer text.
- Reasoning is display-only. The server re-derives nothing from it.
- Keeps history cheap and the wire format simple.

### Slot tracking
- `probeSlot()` runs once at startup, picks the first idle slot.
- `slot_id` is pinned in every request body.
- Erase targets the specific slot.
- **May be removed later** if multi-tab is not a requirement. For single-tab, hardcoding slot 0 is simpler.

### Live TG (tokens/sec)
- 800 ms rolling window. Resets on each `beginReply()`.
- Final chunk's `timings.predicted_per_second` overrides with the server's exact number.
- **Bug fixed this session:** was `Math.round(tg)` instead of `Math.round(rate)`. The `ReferenceError` propagated up the promise chain, killed the reader loop, and froze the display while the server kept generating.

### Undo
- Removes the last complete round (user + assistant + reasoning) from DOM, transcript, and server KV.
- Restores the user's prompt text into the editor with cursor at end.
- Walks back through transcript to find the user message (handles aborted turns where no assistant entry exists).
- Fires `?action=erase` on the slot.

---

## Things Learned / Bugs Fixed This Session

1. **`Math.round(tg)` vs `Math.round(rate)`** — a one-character typo that froze the entire UI. The `ReferenceError` was swallowed by the promise chain's `.catch(reject)`, which rejected the inner promise, which skipped the "clean end" `.then()`, landed in the outer `.catch()`, which saw `err.name !== "AbortError"` and set status to `"error: tg is not defined"`. The reader was never polled again. **Lesson:** in a recursive `pump()` with `.catch(reject)`, any uncaught exception in the sink kills the stream silently.

2. **`body.slot` vs `body.slot_id`** — llama.cpp reads `slot_id` from the JSON body. `slot` is silently ignored.

3. **`removeLastTurns` off-by-one** — was popping `turnMarks` *before* computing `startIdx`, so it removed one extra turn. Fixed: compute `startIdx`, remove nodes, *then* pop.

4. **`trimTranscript` and `turnMarks` desync** — trimming shifts the transcript but does NOT remove DOM nodes. `turnMarks` still points to the correct indices because the DOM is append-only (we never remove from the middle). This is fine as long as trimming only drops from the front of the transcript and the DOM keeps all rendered turns.

5. **`promptEst` anchoring** — after a trim, abort, undo, or new session, `lastServerPromptTokens` is zeroed. The next generate falls back to `estTokens()` (chars/4 of full transcript). After a successful turn, it re-anchors to the server's `usage.prompt_tokens`.

---

## The Three Buttons (Current State)

| Button | ID | Status |
|---|---|---|
| **undo** | `btn4` | ✅ Implemented (`undoLast()`) |
| **save** | `btn5` | 🚧 Stub (`saveTranscript()` → `"save: not yet"`) |
| **img** | `btn6` | 🚧 Stub (`exportImage()` → `"img: not yet"`) |

---

## Implementing `save`

### What it should do

Serialize the current conversation to a file the user can download. Two reasonable formats:

**Option A: JSON (machine-readable, round-trippable)**
```json
{
  "model": "qwen2.5-7b-instruct-q4_k_m",
  "server": "127.0.0.1:9931",
  "n_ctx": 8192,
  "saved_at": "2025-01-15T14:32:00Z",
  "transcript": [
    {"role": "user", "content": "Hello"},
    {"role": "assistant", "content": "Hi there!"}
  ]
}
```
- Loadable back into a fresh session (populate `transcript[]`, re-render `#output`).
- Compatible with other tools that read OpenAI chat format.

**Option B: Formatted text (human-readable, .md or .txt)**
```
# bloppadm session — qwen2.5-7b — 2025-01-15 14:32

## User
Hello

## Assistant
Hi there!

## User
How are you?

## Assistant
Great, thanks!
```
- No metadata header beyond model + timestamp.
- Easy to paste into a markdown editor.

**Recommendation:** Implement Option A (JSON). It's lossless, round-trippable, and the transcript array is already in the right shape. A `.md` export can be a trivial transform later.

### Implementation sketch

```javascript
function saveTranscript() {
    if (transcript.length === 0) { setStatus("nothing to save"); return; }

    var payload = {
        model: model.value,
        server: server.value,
        n_ctx: nCtx,
        saved_at: new Date().toISOString(),
        transcript: transcript
    };

    var blob = new Blob([JSON.stringify(payload, null, 2)],
                        { type: "application/json" });
    var url = URL.createObjectURL(blob);
    var a = document.createElement("a");
    a.href = url;
    a.download = "bloppadm_" +
        model.value.replace(/[^a-zA-Z0-9_-]/g, "_") + "_" +
        new Date().toISOString().replace(/[:.]/g, "-") + ".json";
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);

    setStatus("saved " + transcript.length + " messages");
}
```

### Considerations

- **No file input needed.** The browser download API (`Blob` + `createObjectURL` + `<a>.click()`) is sufficient. No `FileSaver.js`.
- **The transcript does not include reasoning.** If you want reasoning in the save file, you'd need to capture it per-turn (e.g., store `thoughtChars` worth of text in a parallel array, or add a `reasoning` field to the assistant entry). Currently it's display-only and lost after the turn ends. If the user wants it saved, that's a new state variable.
- **Multi-modal content (see `img` below):** once image-upload is implemented, `transcript` entries may have `content` as an array (OpenAI multi-part format) rather than a plain string. The JSON save will handle this naturally since it's already structured.
- **File naming:** use the model basename + timestamp so files are sortable and identifiable.

---

## The `img` Button: Image Upload (NOT Image Export)

**Critical clarification:** `btn6` is labelled `img` but it is **not** an image-export feature. It is an **image-upload** feature for **multimodal models**.

### What it should do

- Open a file picker (`<input type="file" accept="image/*">`).
- Read the selected image as a base64 data URL.
- Insert it into the current turn's message as a multi-part content block:
  ```json
  {
    "role": "user",
    "content": [
      {"type": "text", "text": "What's in this image?"},
      {"type": "image_url", "image_url": {"url": "data:image/png;base64,iVBOR..."}}
    ]
  }
  ```
- The image appears in the editor context (or as a thumbnail in `#output`) so the user can see what's being sent.
- The server (llama-server with a vision-capable GGUF, e.g. LLaVA, Qwen-VL) processes it.

### Implications for the codebase

- **`transcript` entries may have `content` as an array**, not a string. The `estTokens()` function and the `chars/4` estimate will need to handle both shapes:
  ```javascript
  function contentChars(msg) {
      if (typeof msg.content === "string") return msg.content.length;
      // multi-part: sum text parts; images are ~1000 tokens each (rough)
      var c = 0;
      for (var i = 0; i < msg.content.length; i++) {
          if (msg.content[i].type === "text") c += msg.content[i].text.length;
          else c += 4000; // ~1000 tokens * 4 chars/token
      }
      return c;
  }
  ```
- **`logUser`** will need to render a thumbnail or a `[image: filename]` placeholder in the green span.
- **The editor** stays as plain text. The image is attached separately (e.g., a small preview strip above the textarea, or a hidden file input).
- **`undoLast`** must handle multi-part content when restoring the user text (extract just the text parts for the editor; the image is lost on undo unless you cache it).
- **`saveTranscript`** (JSON) will naturally include the base64 data. File size will be large for high-res images. Consider downscaling to 768px max before encoding.

### Suggested UI

- Click `img` → hidden `<input type="file">` fires.
- On selection: read as `FileReader` → data URL → store in a per-turn variable (e.g., `pendingImage`).
- Show a small thumbnail (or filename) in the editor area or status line.
- On `generate()`: if `pendingImage` is set, build the multi-part `content` array for the user message. Clear `pendingImage` after.
- The image is NOT stored in `#editor`. It's a side-channel attachment.

---

## File Structure (Single File)

```
bloppadm.html
├── <style>
│   ├── SECTION 1: Global palette (CSS custom properties)
│   ├── SECTION 2: Page frame (flex column, no scroll)
│   ├── SECTION 3: #output styles
│   ├── SECTION 4: #editor styles
│   └── SECTION 5: #settings styles
├── <div id="output">
├── <div id="bottom">
│   ├── <div id="edwrap"><textarea id="editor">
│   └── <div id="settings">
│       ├── server input
│       ├── model input
│       ├── #nums (2×2 grid: n_predict, temp, top_p, repeat_penalty)
│       ├── #controls (send / stop / new)
│       ├── #controls2 (undo / save / img)
│       └── #statusbox (status line + pp/tg + ¢)
└── <script>
    ├── J1: Global state
    ├── J2: Small helpers (serverURL, $, setStatus, num, scrollBottom, addSpan)
    ├── J2b: showStats
    ├── J3: estTokens, trimTranscript
    ├── J4: logUser, appendThought, appendAnswer, updateCtxLive, beginReply
    ├── J4c: removeLastTurns
    ├── J5: makeSseSink
    ├── J6: stopGen, editorKeydown, newSession, undoLast, saveTranscript, exportImage
    ├── J7: probeSlot, probeModel
    ├── J8: generate
    └── J9: main (element wiring)
```

---

## Conventions & Style

- **`var`** everywhere, not `let`/`const`. Consistent with the "no build step" philosophy; targets any browser that supports `fetch` and `AbortController`.
- **Tabs** for indentation.
- **No semicolons at line starts** (they're at line ends).
- **Comments in `/* */`** with section headers. The "WHAT DIED AND WHY" block is sacred — update it when you remove or add major subsystems.
- **Status line** is the only user-facing feedback for async operations. No `alert()`, no `console.log` for user-visible errors (console is for debugging).
- **Guards first, side effects second** in every action function.
- **One `AbortController` per generation.** `ctrl === null` means idle.
- **The retention rule:** perf stats (pp, tg) are never zeroed between turns. Last nonzero values persist.

---


