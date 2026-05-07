# Notesmith

A high-performance Apple Notes exporter and sync engine, written in Swift.

**Status:** Design phase — see [DESIGN.md](DESIGN.md) for the full spec.

## Goals

- Incremental export in **< 10 seconds** for any library size
- Full export in **< 45 seconds** for 1,000 notes
- Zero shell forks — native Swift file I/O and JSON throughout
- Foundation for a Mac app, bidirectional sync, and AI search

## Why Swift?

The predecessor ([notes-exporter](https://github.com/tcw7/notes-exporter-fork)) was written in AppleScript and took 41 seconds for an incremental run with no changes. The bottleneck was AppleScript interpreter overhead on top of Apple Events — not the Apple Events themselves.

Swift + ScriptingBridge sends the same Apple Events to Notes.app but eliminates all per-iteration interpreter overhead, shell forks (Python for JSON, `mkdir`, `date`), and O(N²) string/list patterns.

## Roadmap

1. **Phase 1 (MVP):** Core export CLI — feature-parity with AppleScript exporter
2. **Phase 2:** Observability, launchd scheduling, Homebrew
3. **Phase 3:** Full-text search index (SQLite FTS5)
4. **Phase 4:** Semantic/AI search (embeddings)
5. **Phase 5:** Bidirectional sync
6. **Phase 6:** SwiftUI Mac app

See [DESIGN.md](DESIGN.md) for detailed architecture, data models, and implementation plan.
