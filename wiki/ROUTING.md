# Wiki Routing

Use this file after reading `index.md` when narrowing a wiki-backed question to likely branches.

## Project Overview

- Pages:
- Keywords: transcriblerr, Tauri, desktop app, Apple Silicon, offline transcription, README

## Architecture

- Pages:
- Keywords: workspace layout, crates/asr-core, apps/desktop/src-tauri, React frontend, Tauri commands, invoke, event emit, RecordingState, parking_lot Mutex

## Audio Pipeline

- Pages:
- Keywords: microphone capture, system audio, CPAL, ScreenCaptureKit, Swift bridge, VAD, voice activity detection, audio constants, RecordingState, partial interval

## Transcription Backends

- Pages:
- Keywords: whisper.cpp, whisper-rs, asr-core, Metal GPU, local transcription, LLM client, OpenAI-compatible, transcription_mode, websocket_client legacy, transcription worker, audio-runtime-config.json

## Summarization And Diarization

- Pages:
- Keywords: summarization, OpenAI-compatible API, local fallback, diarization feature flag, sherpa-onnx, DIARIZATION_MANAGER

## macOS Integration

- Pages:
- Keywords: TCC permissions, ScreenCapture, Microphone, tccutil reset, Screen Recording, com.transcriblerr.app, system_audio.rs

## Configuration And Secrets

- Pages:
- Keywords: .env, dotenvy, LLM_API_BASE_URL, LLM_SUMMARY_, Tauri app config dir, audio-runtime-config.json, keychain, llm-api-key

## Build And Tooling

- Pages:
- Keywords: pnpm, vite, tsc, cargo build, cargo check, cargo test, vitest, tauri dev, tauri build, diarization feature, git submodule whisper.cpp

## Decisions

- Pages:
- Keywords: ADR, design decision, scope change, contract change, terminology, deprecated path

## Research

- Pages:
- Keywords: investigation, exploration, external library notes, benchmark
