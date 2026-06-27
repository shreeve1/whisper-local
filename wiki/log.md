# Wiki Log

Append entries with this format:

## [YYYY-MM-DD] type | Title

- Actor: agent or human
- Inputs: paths or prompt summary
- Outputs: changed pages
- Notes: key decisions or unresolved questions

## [2026-06-19] setup | Initialize LLM Wiki

- Actor: agent (Claude Code, llm-wiki-setup skill)
- Inputs: project root `/Users/james/transcriblerr/`; defaults from `llm-wiki-setup/SKILL.md`
- Outputs: `wiki/README.md`, `wiki/index.md`, `wiki/log.md`, `wiki/ROUTING.md`, `wiki/CLAIMS.md`; subdirs `raw/`, `candidates/`, `sources/`, `entities/`, `concepts/`, `analyses/`, `assets/`; `.gitignore` `# LLM Wiki` block; `LLM Wiki` section appended to `CLAUDE.md` and `AGENTS.md`
- Notes: Fresh setup over empty `wiki/`. Domain inferred from `README.md` and `CLAUDE.md` as Tauri 2 macOS desktop app for offline speech-to-text using whisper.cpp (via `asr-core`/whisper-rs), with optional OpenAI-compatible transcription and summarization. Routing seeded with Project Overview, Architecture, Audio Pipeline, Transcription Backends, macOS Integration, Decisions, Research branches.
