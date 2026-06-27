# AGENTS.md

Guidance for coding agents working in this repository.

## Project Summary

Transcriblerr is a Tauri 2 desktop app for Apple Silicon Macs. It captures microphone and system audio, transcribes locally with whisper.cpp via `whisper-rs`, and optionally uses OpenAI-compatible APIs for transcription and summarization.

## Repository Layout

- `apps/desktop/src/` - React/Vite frontend. Keep feature UI in `components/`, reusable state in `hooks/`, shared DTOs in `types.ts`, and frontend-only helpers in `utils/`.
- `apps/desktop/src-tauri/` - Tauri backend. `commands.rs` contains thin command wrappers; implementation lives in module functions, primarily `lib.rs` plus domain modules.
- `apps/desktop/src-tauri/src/audio/` - shared recording state, VAD processing, audio constants, and audio utilities.
- `apps/desktop/src-tauri/src/transcription/` - transcription worker and provider clients.
- `apps/desktop/src-tauri/src/summarization/` - OpenAI-compatible summary provider and persisted config.
- `apps/desktop/src-tauri/src/system_audio.rs` - macOS system audio capture bridge.
- `crates/asr-core/` - Rust whisper wrapper crate with no Tauri dependency.
- `vendor/whisper.cpp/` - git submodule required for local whisper builds.

## Setup And Commands

Run frontend commands from `apps/desktop/`:

```bash
pnpm install
pnpm dev
pnpm build
pnpm exec vitest run
pnpm tauri dev
pnpm tauri build
```

Run Rust commands from the repo root when the Rust toolchain is available:

```bash
cargo test
cargo check
cargo build
cargo build --features diarization
```

Initialize submodules before native builds:

```bash
git submodule update --init --recursive
```

## Implementation Conventions

- Keep frontend/backend communication through Tauri `invoke()` commands and backend-emitted events.
- Add new Tauri commands in `commands.rs` as thin wrappers only; put behavior in module-level `*_impl` functions.
- Keep backend state mutations behind the shared `RecordingState` mutex and avoid holding that lock while joining threads, blocking on I/O, or calling long-running work.
- Do not leak audio streams. Retain active streams in owned state and drop them deterministically on stop/replacement/shutdown.
- Treat the transcription worker as shared app infrastructure, not as mic-owned state.
- Do not reintroduce screen recording without wiring it end-to-end through Rust commands, Swift bridge status/error propagation, frontend controls, and tests/docs.
- Keep API keys backend-only. The frontend may receive booleans like `hasApiKey`, never raw secrets.
- Prefer small frontend components and hooks over growing `App.tsx`.
- Preserve the app’s current visual language unless explicitly asked for a redesign.

## Safety Notes

- `.env` files must remain untracked.
- `Cargo.lock` should be tracked for reproducible desktop app builds.
- Model files are large; do not commit downloaded Whisper models.
- `apps/desktop/src-tauri/tauri.conf.json` CSP should stay restrictive. Do not add broad permissions or plugins unless the frontend actually uses.
- The app is macOS-focused. Be careful with cross-platform assumptions in CPAL, ScreenCaptureKit, and Swift bridge code.
- After reinstalling or rebuilding the app, TCC permissions (Screen Recording, Microphone) may need to be reset with `tccutil reset ScreenCapture com.transcriblerr.app` and `tccutil reset Microphone com.transcriblerr.app`, then relaunch the app. Stale TCC entries from a previous install can cause "The user declined TCCs" errors even when permissions appear enabled in System Settings.

## Verification

Before finishing changes, run:

```bash
cd apps/desktop && pnpm exec vitest run
cd apps/desktop && pnpm build
```

If available, also run:

```bash
cargo test
cargo check
```

If a required tool is not available, report the blocked check explicitly in the final response.

## LLM Wiki

This project uses `wiki/` as an LLM-maintained knowledge base.

### Directories

- `wiki/raw/`: immutable source material; read but do not rewrite.
- `wiki/raw/sessions/`: curated session captures created by `/wiki-update` when conversation evidence needs citation.
- `wiki/candidates/`: generated pages awaiting review or promotion.
- `wiki/sources/`: promoted source summaries.
- `wiki/entities/`: promoted entity pages.
- `wiki/concepts/`: promoted concept pages.
- `wiki/analyses/`: promoted query outputs and syntheses.
- `wiki/raw/assets/`: source attachments clipped with raw material.
- `wiki/assets/`: generated or wiki-native images and attachments.

### Required Files

- Read `wiki/index.md` first when answering wiki-backed questions.
- Use `wiki/ROUTING.md` after `wiki/index.md` to narrow large searches.
- Append every ingest, query, lint, and promotion to `wiki/log.md`.
- Track important factual claims in `wiki/CLAIMS.md`.

### Wiki-First Project Search

For any project-specific question, investigation, design task, bug hunt, or code search that requires looking up project context, check the wiki first.

1. Read `wiki/index.md` before searching broadly.
2. Use `wiki/ROUTING.md` to identify relevant promoted pages, candidates, and claim entries.
3. Read relevant wiki pages and `wiki/CLAIMS.md` entries before using general repository search.
4. If the wiki does not contain enough information, search the codebase, docs, or external sources as needed.
5. When non-wiki search reveals durable project knowledge, propose ingesting the source into `wiki/raw/`, creating or updating a page in `wiki/candidates/`, or promoting an existing candidate after James approves.
6. If external or codebase search was needed to answer a wiki-backed question, mention the wiki gap and proposed ingest or promotion path in the final answer.

### Session Update Workflow

Use `/wiki-update` during or after meaningful sessions to capture durable decisions, verified facts, root causes, follow-ups, and reusable context. Create curated raw session captures under `wiki/raw/sessions/` when conversation evidence is needed. Do not archive full transcripts, secrets, private material, or raw pasted user content without explicit approval. New or risky session-derived knowledge goes through `wiki/candidates/` and must update `wiki/index.md`, `wiki/ROUTING.md`, `wiki/CLAIMS.md`, and `wiki/log.md`.

### Maintenance Trigger

The wiki is a standing obligation, not an opt-in step. Before reporting any task complete, run the end-of-session wiki check. This is mandatory, not advisory.

A task produces durable project knowledge — and therefore requires a `/wiki-update` pass before it is reported done — when it includes any of:

- A decision that sets or reverses project direction, scope, or ownership.
- Accepted or changed terminology, naming, or domain concepts.
- A new or revised architecture, process, or contract that future sessions must honor.
- A verified fact, root cause, or fix that contradicts or supersedes existing wiki knowledge.

End-of-session check, every task:

1. Decide whether the task hit any trigger above.
2. If yes, run `/wiki-update` before reporting completion. If a full pass must be deferred, state the wiki gap and the proposed ingest, candidate, or promotion path in the final answer.
3. If no, state one line in the final answer confirming the wiki check ran and nothing qualified.

Mark superseded knowledge `superseded` in `wiki/CLAIMS.md` with a pointer to the newer claim; never delete it to clean up history.

### Ingest Workflow

1. Read the new source from `wiki/raw/`.
2. Summarize the source with citations to the raw path.
3. Discuss key takeaways or emphasis with James when the source is substantial, ambiguous, or likely to touch multiple pages.
4. Extract entities, concepts, contradictions, and atomic claims.
5. Create new pages in `wiki/candidates/` unless the edit is low-risk maintenance.
6. Update `wiki/index.md` candidate queue, `wiki/ROUTING.md`, and `wiki/CLAIMS.md` with cited candidate entries.
7. Append an entry to `wiki/log.md`.

### Query Workflow

1. Read `wiki/index.md` to identify relevant promoted pages and candidates.
2. Use `wiki/ROUTING.md` to narrow branches when the index is too broad.
3. Read only the relevant promoted pages and claim entries.
4. Answer with citations to wiki pages or raw sources.
5. If the answer produces durable synthesis, offer to save it as `wiki/candidates/<slug>.md`.

### Promotion Workflow

1. Review the candidate page for citations, confidence, and duplicates.
2. Move it to `sources/`, `entities/`, `concepts/`, or `analyses/`.
3. Set `status: promoted` and update timestamps.
4. Update `index.md`, `ROUTING.md`, `CLAIMS.md`, and `log.md`.

### Discard Workflow

When a candidate is rejected, remove its candidate index row, candidate-only routes, and candidate claim page references before deleting the candidate file. Append a discard entry to `wiki/log.md`.

### Lint Workflow

Check broken wikilinks, orphan pages, duplicate concepts, uncited claims, stale claims, claim content drift against cited sources, contradictions, missing concept pages, data gaps, stale candidate references, and missing index/routing entries. Report findings before making broad changes.
