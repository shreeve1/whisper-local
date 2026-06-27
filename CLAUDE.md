# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Transcriblerr is an offline speech-to-text desktop app for Apple Silicon Macs. It uses whisper.cpp (via whisper-rs) for local transcription, captures both microphone and system audio, and runs as a Tauri 2 desktop application with a React frontend.

## Build & Dev Commands

```bash
# Install frontend dependencies (from apps/desktop/)
cd apps/desktop && pnpm install

# Run in development mode (from apps/desktop/)
pnpm tauri dev

# Production build (from apps/desktop/)
pnpm tauri build

# Build just the Rust backend (from repo root)
cargo build

# Build with speaker diarization support
cargo build --features diarization

# Check Rust code compiles
cargo check

# Frontend type-check
cd apps/desktop && pnpm build  # runs tsc && vite build
```

Git submodules are required (`vendor/whisper.cpp`):
```bash
git submodule update --init --recursive
```

## Architecture

### Workspace Layout

- **`crates/asr-core/`** — Core whisper.cpp wrapper crate. Provides `WhisperContext` for transcription with Metal GPU acceleration. Has no Tauri dependency — pure audio-in, text-out.
- **`apps/desktop/src-tauri/`** — Tauri 2 backend (Rust). All application logic: audio capture, VAD, transcription orchestration, system audio, summarization, diarization.
- **`apps/desktop/src/`** — React frontend (single-file `App.tsx` + CSS). Communicates with backend exclusively via Tauri `invoke()` commands and event listeners.
- **`vendor/whisper.cpp`** — Git submodule, built by `asr-core`'s `build.rs`.

### Backend Modules (`apps/desktop/src-tauri/src/`)

- **`lib.rs`** — App entry point, Tauri setup, all `*_impl()` functions that commands delegate to. Global statics (`APP_HANDLE`, `RECORDING_SAVE_PATH`, `DIARIZATION_MANAGER`).
- **`commands.rs`** — Thin `#[tauri::command]` wrappers that call `*_impl()` functions from `lib.rs`. All commands registered in `register()`.
- **`audio/`** — Audio pipeline: `state.rs` (shared `RecordingState` behind `parking_lot::Mutex`), `processing.rs` (VAD + session management), `constants.rs`, `utils.rs`.
- **`transcription/`** — Two backends: `worker.rs` (local whisper via `asr-core`), `llm_client.rs` (OpenAI-compatible API). `websocket_client.rs` for legacy WS mode. Mode selected at runtime via `transcription_mode` ("local" or "llm").
- **`summarization/`** — AI summarization via OpenAI-compatible API with local fallback. Config persisted to Tauri app config dir.
- **`diarization/`** — Speaker diarization (optional, behind `diarization` feature flag using `sherpa-onnx`).
- **`system_audio.rs`** — macOS system audio capture (loopback).
- **`whisper.rs`** — Model management (scan, download, delete models).

### Frontend ↔ Backend Communication

The frontend calls Tauri commands via `invoke()` (e.g., `invoke("start_recording")`). The backend emits events to the frontend: `transcription-segment`, `voice-activity`, `backend-error`. All state lives in the Rust backend.

### Key Patterns

- **Recording state**: Single `RecordingState` struct behind `parking_lot::Mutex`, accessed via `recording_state()` / `try_recording_state()`.
- **Runtime config persistence**: Audio settings (VAD threshold, transcription mode, partial interval) saved to `audio-runtime-config.json` in the Tauri app config directory.
- **Environment variables**: Backend loads `.env` via `dotenvy`. See `apps/desktop/.env.example` for LLM transcription and summarization config (`LLM_API_BASE_URL`, `LLM_SUMMARY_*`).
- **Transcription modes**: "local" (whisper.cpp via asr-core) or "llm" (OpenAI-compatible API). Aliases: "legacy_ws"→local, "api"/"openai"→llm.

## macOS TCC Permissions

The app requires Screen Recording and Microphone TCC permissions for system audio capture. When reinstalling or after a fresh build, macOS may not re-prompt if stale entries exist. Reset with:

```bash
tccutil reset ScreenCapture com.transcriblerr.app
tccutil reset Microphone com.transcriblerr.app
```

Then relaunch the app to get fresh permission prompts. If `tccutil` alone doesn't work, also manually remove the app from **System Settings → Privacy & Security → Screen Recording / Microphone** before relaunching.

When doing a full uninstall, remove these locations to clean all app state:

```bash
rm -rf ~/Library/Application\ Support/com.transcriblerr.app
rm -rf ~/Library/Application\ Support/local-whisper
rm -rf ~/Library/Application\ Support/transcriblerr
rm -rf ~/Library/Caches/com.transcriblerr.app
rm -rf ~/Library/Saved\ Application\ State/transcriblerr.app.*
rm -rf /Applications/transcriblerr.app
security delete-generic-password -s "com.transcriblerr.app" -a "llm-api-key"
tccutil reset ScreenCapture com.transcriblerr.app
tccutil reset Microphone com.transcriblerr.app
```

## Requirements

- Apple Silicon Mac (macOS 13+)
- Rust 1.70+, Node.js 18+, pnpm
- C++ compiler (for whisper.cpp native build)

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
