# Notesmith — Design Document

**Status:** Pre-implementation  
**Author:** Tyler Wennstrom  
**Last updated:** 2026-05-06

---

## Inspiration

Notesmith is a ground-up Swift rewrite inspired by **[notes-exporter](https://github.com/storizzi/notes-exporter)**, an AppleScript-based Apple Notes exporter that pioneered many of the ideas here: incremental exports, JSON tracking per notebook, bidirectional sync, batch Apple Event fetching, and AI-powered search.

The AppleScript implementation proved the concept and identified the real bottlenecks. Notesmith exists to push past the limits of what AppleScript can do — not to replace it, but to honor it by taking its ideas as far as they can go.

---

## Table of Contents

1. [Vision & Goals](#1-vision--goals)
2. [Performance Model](#2-performance-model)
3. [Architecture Overview](#3-architecture-overview)
4. [ScriptingBridge Integration](#4-scriptingbridge-integration)
5. [Data Models](#5-data-models)
6. [Module Specifications](#6-module-specifications)
7. [CLI Interface](#7-cli-interface)
8. [Concurrency Model](#8-concurrency-model)
9. [Error Handling & Recovery](#9-error-handling--recovery)
10. [Testing Strategy](#10-testing-strategy)
11. [Configuration](#11-configuration)
12. [File Layout on Disk](#12-file-layout-on-disk)
13. [Roadmap](#13-roadmap)

---

## 1. Vision & Goals

Notesmith is a high-performance Apple Notes exporter and bidirectional sync engine, written in Swift. It aims to be the fastest possible bridge between Apple Notes and the filesystem, targeting both power users and a future consumer Mac app.

### Goals

- **Incremental export in under 10 seconds** for any library size, when few notes have changed.
- **Full export in under 90 seconds** for 1,000 notes.
- **Zero shell forks.** All file I/O, JSON, and string work happens natively in Swift.
- **Minimal Apple Events.** Every IPC round-trip to Notes.app is counted and justified.
- **Correct.** Exports are byte-stable for unchanged notes; renamed notes are tracked; deleted notes are tombstoned.
- **Testable.** Core logic is dependency-injected so it can run without Notes.app.
- **Extensible.** Designed from the start to support a SwiftUI Mac app, bidirectional sync, and a search index.

### Non-Goals (v1)

- No GUI in v1 (CLI only).
- No iCloud API access (Apple Events only).
- No Markdown conversion (raw HTML + plaintext output only, conversion is a post-processing concern).
- No Windows/Linux port.

---

## 2. Performance Model

### Why AppleScript Was Slow

The predecessor script (`export_notes.scpt`) suffered from three categories of overhead:

| Source | Cost | Notes |
|---|---|---|
| Apple Events (IPC to Notes.app) | 20–80ms per round-trip | Unavoidable; Notes.app only speaks AE |
| `do shell script` forks | 30–150ms per invocation | Python for JSON, `mkdir`, `date` |
| AppleScript interpreter | 0.5–5ms per record field access | Adds up in O(N²) loops |
| O(N²) algorithms | Grows as N² | List copies, string `&` in loops |

A full export of 875 notes was ~7 minutes: mostly per-note AEs (name + body + plaintext + creation date = 4 AEs × 875 notes × 50ms = ~175s) plus Python/shell fork overhead.

### Apple Event Budget

Apple Events are the only irreducible cost. ScriptingBridge sends the same AEs as AppleScript — the win is eliminating all non-AE overhead.

**Batch property fetch via `SBElementArray.arrayByApplyingSelector`:**  
One call returns a property for every element in a single AE. This is the key primitive.

#### Incremental run (N folders, K folders with changed notes, M total changed notes)

| Operation | AE count | Notes |
|---|---|---|
| Get accounts | 1 | |
| Get all folders per account (one account typical) | 1 | Returns `SBElementArray` |
| Get all folder names | 1 | `arrayByApplyingSelector(.name)` |
| Get IDs of every note per folder | N | One AE per folder |
| Get mod dates of every note per folder | N | One AE per folder |
| Get names of changed notes' folders | K | Only folders with changes |
| Get bodies of changed notes' folders | K | Only folders with changes |
| Get plaintext of changed notes' folders | K | Only folders with changes |
| Get creation dates of changed notes' folders | K | Only folders with changes |

**Total:** `3 + 2N + 4K` AEs

For 172 folders, 0 changed notes: `3 + 344 + 0 = 347 AEs × 20ms = ~7 seconds`  
For 172 folders, 10 changed notes across 3 folders: `3 + 344 + 12 = 359 AEs × 20ms = ~7.2 seconds`

#### Full export (all N folders, all N have content)

**Total:** `3 + 6N` AEs  
For 172 folders: `3 + 1,032 = 1,035 AEs × 20ms = ~21 seconds`

### Performance Targets

| Scenario | Current (AppleScript) | Target (Swift) |
|---|---|---|
| Incremental, 0 changes | 41s | <10s |
| Incremental, 10 changes | ~45s | <12s |
| Full export, 1,000 notes | ~7 min | <45s |
| Full export, 5,000 notes | ~35 min (est.) | <3 min |

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         notesmith CLI                           │
│                     (ArgumentParser)                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                       ExportEngine                              │
│  • Orchestrates the export loop                                 │
│  • Drives incremental vs full decision per folder               │
│  • Computes statistics and progress                             │
└────────┬──────────────┬──────────────────────┬──────────────────┘
         │              │                      │
         ▼              ▼                      ▼
┌──────────────┐ ┌─────────────────┐ ┌──────────────────────────┐
│ NotesClient  │ │   DataStore     │ │      FileWriter           │
│              │ │                 │ │                           │
│ ScriptingBr- │ │ Reads/writes    │ │ Writes HTML + txt files   │
│ idge wrapper │ │ JSON tracking   │ │ Sets file dates           │
│              │ │ data per note-  │ │ Handles renames/deletes   │
│ Abstracts    │ │ book            │ │                           │
│ all Apple    │ │                 │ │ No Apple Events           │
│ Events       │ └─────────────────┘ └──────────────────────────┘
│              │
│ Batch fetch  │
│ via          │
│ SBElementArr │
│ ay           │
└──────┬───────┘
       │ Apple Events (IPC)
       ▼
┌─────────────────────┐
│    Notes.app        │
│  (com.apple.notes)  │
└─────────────────────┘
```

### Package Structure

```
Notesmith/
├── Package.swift
├── Sources/
│   └── notesmith/
│       ├── main.swift                  # Entry point, CLI setup
│       ├── CLI/
│       │   ├── ExportCommand.swift
│       │   ├── SyncCommand.swift
│       │   └── QueryCommand.swift
│       ├── Engine/
│       │   ├── ExportEngine.swift      # Main orchestration loop
│       │   ├── ExportConfig.swift      # All configuration
│       │   └── ExportStats.swift       # Metrics accumulation
│       ├── Notes/
│       │   ├── NotesClient.swift       # Protocol
│       │   ├── ScriptingBridgeClient.swift  # Real implementation
│       │   ├── NotesAccount.swift      # Value-type wrappers
│       │   ├── NotesFolder.swift
│       │   └── NotesNote.swift
│       ├── Storage/
│       │   ├── DataStore.swift         # JSON tracking DB
│       │   ├── NoteRecord.swift        # Codable model
│       │   └── NotebookData.swift      # Codable model
│       ├── Output/
│       │   ├── FileWriter.swift
│       │   └── FilenameGenerator.swift
│       └── Util/
│           ├── Logger.swift
│           └── StringExtensions.swift
├── Tests/
│   └── NotesmithTests/
│       ├── ExportEngineTests.swift
│       ├── DataStoreTests.swift
│       ├── FilenameGeneratorTests.swift
│       └── MockNotesClient.swift
└── DESIGN.md
```

---

## 4. ScriptingBridge Integration

### Generating the Swift Interface

ScriptingBridge generates a Swift-compatible Objective-C header from Notes.app's SDEF:

```bash
sdef /Applications/Notes.app | sdp -fh --basename Notes -o Sources/notesmith/Notes/
```

This produces `Notes.h`, which is imported via a bridging header or as a module. The generated header exposes `NotesApplication`, `NotesAccount`, `NotesFolder`, `NotesNote` as `SBObject` subclasses.

Key generated types:
```objc
@interface NotesApplication : SBApplication
- (SBElementArray<NotesAccount *> *)accounts;
@end

@interface NotesAccount : SBObject
@property NSString *name;
@property NSString *id;
- (SBElementArray<NotesFolder *> *)folders;
@end

@interface NotesFolder : SBObject
@property NSString *name;
@property NSString *id;
- (SBElementArray<NotesNote *> *)notes;
@end

@interface NotesNote : SBObject
@property NSString *name;
@property NSString *id;
@property NSDate *modificationDate;
@property NSDate *creationDate;
@property NSString *body;        // HTML
@property NSString *plaintext;
@end
```

### Batch Fetching with SBElementArray

`SBElementArray.arrayByApplyingSelector(_:)` sends a single Apple Event that retrieves a named property for every element in the array. This is the same primitive as `property of every element of container` in AppleScript.

```swift
// Get SBElementArray for all notes in a folder
let noteElements = folder.notes() as SBElementArray

// One AE: get all note IDs
let ids = noteElements.arrayByApplyingSelector(#selector(getter: NotesNote.id)) as! [String]

// One AE: get all modification dates
let modDates = noteElements.arrayByApplyingSelector(
    #selector(getter: NotesNote.modificationDate)
) as! [Date]

// Zip: no Apple Events, pure Swift
let pairs = zip(ids, modDates)
```

**Important constraint:** `SBElementArray` is only valid while Notes.app is open. The elements are lazy proxies — accessing individual `noteElements[i]` after building `ids`/`modDates` is one AE per access. Always use `arrayByApplyingSelector` for bulk reads.

### Apple Events Are Serial

Notes.app's AE server is single-threaded. Sending AEs concurrently from multiple Swift tasks does not parallelize them — they queue. Therefore:

- Folder processing **must** be serial from the AE perspective.
- File I/O and JSON reads **can** be parallelized after the AE phase.
- The inner loop structure is: fetch all IDs+dates for a folder (2 AEs), determine which notes changed, then optionally fetch content arrays (up to 4 AEs), then do all file I/O concurrently.

### ScriptingBridgeClient: Concrete Notes Interface

`ScriptingBridgeClient` wraps ScriptingBridge and exposes a clean value-type API, completely hiding `SBObject` from the rest of the codebase:

```swift
protocol NotesClientProtocol {
    func accounts() throws -> [AccountInfo]
    func folders(in account: AccountInfo) throws -> [FolderInfo]
    func noteMetadata(in folder: FolderInfo) throws -> [NoteMetadata]
    func noteContent(in folder: FolderInfo, ids: Set<String>) throws -> [String: NoteContent]
}

struct AccountInfo {
    let id: String
    let name: String
}

struct FolderInfo {
    let id: String
    let name: String
    let accountID: String
}

struct NoteMetadata {
    let id: String          // Short extracted ID
    let fullID: String      // Full Notes.app URL ID
    let modificationDate: Date
}

struct NoteContent {
    let id: String
    let name: String
    let body: String        // HTML
    let plaintext: String
    let creationDate: Date
}
```

`noteContent(in:ids:)` fetches the full content arrays for the entire folder (4 AEs), then returns only the notes requested by `ids`. This ensures we never issue individual per-note AEs.

### ID Extraction

Notes.app returns IDs in the form:
```
x-coredata://XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/ICNote/pXXXX
```

The short ID used for tracking is extracted as the last path component: `pXXXX` (or the full UUID-component suffix if that format varies). Implementation:

```swift
extension String {
    /// Extracts the short note identifier from a full Notes.app x-coredata URL.
    /// Input:  "x-coredata://UUID/ICNote/p1234"
    /// Output: "p1234"
    var extractedNoteID: String {
        (self as NSString).lastPathComponent
    }
}
```

---

## 5. Data Models

### NoteRecord

Tracks the export history of a single note. Persisted to JSON per notebook.

```swift
struct NoteRecord: Codable, Equatable {
    /// Short note ID extracted from the full x-coredata URI.
    let noteID: String
    /// Full x-coredata URI, used for bidirectional sync callbacks.
    let fullNoteID: String
    /// Filename on disk (without extension), as last written.
    var filename: String
    /// Note creation date as reported by Notes.app.
    let created: Date
    /// Note modification date as of last export.
    var modified: Date
    /// Date this note was first exported.
    let firstExported: Date
    /// Date this note was most recently exported.
    var lastExported: Date
    /// Total number of times this note has been exported.
    var exportCount: Int
    /// Set when the note no longer appears in Notes.app. nil if active.
    var deletedDate: Date?
}
```

### NotebookData

Holds all records for one notebook (one JSON file per folder).

```swift
struct NotebookData: Codable {
    /// Schema version for forward compatibility.
    var version: Int = 1
    /// All note records, keyed by short note ID for O(1) lookup.
    var notes: [String: NoteRecord]

    init() {
        self.notes = [:]
    }
}
```

**Key design choice:** The dictionary key is the short note ID. This gives O(1) lookup for `updateNoteRecord`, `markDeleted`, and incremental-check operations — replacing the O(N) linear scans in the AppleScript implementation.

### ExportConfig

All knobs for a single export run, constructed from CLI args and environment variables.

```swift
struct ExportConfig {
    var outputDirectory: URL
    var dataDirectory: URL
    var exportHTML: Bool = true
    var exportPlaintext: Bool = true
    var updateAll: Bool = false
    var filterAccounts: [String] = []
    var filterFolders: [String] = []
    var noteLimit: Int? = nil
    var noteLimitPerFolder: Int? = nil
    var modifiedAfterDate: Date? = nil
    var notePickProbability: Double = 1.0   // 0.0–1.0
    var useSubdirectories: Bool = true
    var filenameFormat: String = "{title}"
    var subdirFormat: String = "{account}/{folder}"
    var includeDeletedInTracking: Bool = false
    var scriptDirectory: URL                // Location of notesmith binary, for resolving data paths
}
```

### ExportStats

Accumulates counters across the full run.

```swift
struct ExportStats {
    var totalFolders: Int = 0
    var totalNotesExamined: Int = 0
    var totalNotesProcessed: Int = 0
    var totalNotesSkippedUnchanged: Int = 0
    var totalNotesSkippedOlder: Int = 0
    var totalNotesSkippedByFilter: Int = 0
    var startTime: Date
    var endTime: Date?

    var elapsedSeconds: Int {
        Int((endTime ?? Date()).timeIntervalSince(startTime))
    }

    var notesPerSecond: Double {
        guard elapsedSeconds > 0 else { return 0 }
        return Double(totalNotesExamined) / Double(elapsedSeconds)
    }
}
```

---

## 6. Module Specifications

### 6.1 ExportEngine

The central coordinator. Does not touch Notes.app or the filesystem directly — delegates to `NotesClientProtocol`, `DataStore`, and `FileWriter`.

**Algorithm — per folder:**

```
1. Load existing NotebookData for this folder (O(1) disk read, already in memory from batch load)
2. Fetch NoteMetadata for all notes (2 AEs: IDs + mod dates)
3. Compute currentNoteIDs = Set(metadata.map(\.id))
4. Determine latestExistingModDate = notebookData.notes.values
       .filter { $0.deletedDate == nil }
       .compactMap { $0.modified }
       .max()
5. Partition notes into changed/unchanged:
   - If updateAll: all notes → changed
   - Else: notes where modDate > latestExistingModDate → candidates
     - For candidates: look up in notebookData.notes[id]
       - If not found: changed (new note)
       - If found and record.modified < noteModDate: changed
       - Else: unchanged
6. If changed is non-empty:
   a. Fetch NoteContent for all notes in folder (4 AEs)
   b. For each changed note:
      - Generate filename
      - Handle rename (if filename differs from record)
      - Write HTML + plaintext files (concurrent)
      - Update notebookData.notes[id]
7. Mark deleted notes: for id in notebookData where not in currentNoteIDs → set deletedDate
8. Save NotebookData (only if step 6 or 7 changed anything)
```

**Key invariant:** Step 5 uses O(1) dictionary lookups. Steps 6–8 operate only on changed notes. The only O(N) work is step 4 (finding max date over all records) and step 7 (checking for deletions) — both unavoidable but O(N) not O(N²).

### 6.2 NotesClient (ScriptingBridgeClient)

Wraps all ScriptingBridge calls. All Apple Events go through this module.

**Critical implementation detail — `noteContent` strategy:**

When the engine requests content for a subset of notes in a folder, `noteContent` always fetches content arrays for the *entire folder* (4 AEs total), then filters in memory. This guarantees we never fall back to per-note AEs.

```swift
func noteContent(in folder: FolderInfo, ids: Set<String>) throws -> [String: NoteContent] {
    let noteElements = sbFolder(for: folder).notes() as SBElementArray
    let allIDs = noteElements.arrayByApplyingSelector(#selector(getter: NotesNote.id)) as! [String]
    let allNames = noteElements.arrayByApplyingSelector(#selector(getter: NotesNote.name)) as! [String]
    let allBodies = noteElements.arrayByApplyingSelector(#selector(getter: NotesNote.body)) as! [String]
    let allPlaintext = noteElements.arrayByApplyingSelector(#selector(getter: NotesNote.plaintext)) as! [String]
    let allCreated = noteElements.arrayByApplyingSelector(#selector(getter: NotesNote.creationDate)) as! [Date]

    var result: [String: NoteContent] = [:]
    for i in 0..<allIDs.count {
        let fullID = allIDs[i]
        let shortID = fullID.extractedNoteID
        guard ids.contains(shortID) else { continue }
        result[shortID] = NoteContent(
            id: shortID,
            name: allNames[i],
            body: allBodies[i],
            plaintext: allPlaintext[i],
            creationDate: allCreated[i]
        )
    }
    return result
}
```

**Notes.app availability check:**

Before starting, confirm Notes.app is running (or launch it):

```swift
func ensureRunning() throws {
    guard let app = SBApplication(bundleIdentifier: "com.apple.notes") else {
        throw NotesClientError.appNotFound
    }
    if !app.isRunning {
        app.activate()
        // Poll for up to 5s
        let deadline = Date().addingTimeInterval(5)
        while !app.isRunning && Date() < deadline {
            Thread.sleep(forTimeInterval: 0.1)
        }
        guard app.isRunning else { throw NotesClientError.appFailedToLaunch }
    }
}
```

### 6.3 DataStore

Manages the JSON tracking database. One JSON file per notebook (per folder).

**Batch load on startup:** All JSON files in the data directory are loaded once at startup into a `[String: NotebookData]` dictionary keyed by canonical file path. This replaces the per-folder Python subprocess from the AppleScript implementation.

```swift
class DataStore {
    private var store: [String: NotebookData] = [:]
    private let dataDirectory: URL
    private let encoder: JSONEncoder
    private let decoder: JSONDecoder

    func loadAll() throws {
        let files = try FileManager.default.contentsOfDirectory(
            at: dataDirectory,
            includingPropertiesForKeys: nil
        ).filter { $0.pathExtension == "json" }

        for file in files {
            let data = try Data(contentsOf: file)
            let notebook = try decoder.decode(NotebookData.self, from: data)
            store[file.path] = notebook
        }
    }

    func notebook(at path: URL) -> NotebookData {
        store[path.path] ?? NotebookData()
    }

    func save(_ notebook: NotebookData, at path: URL) throws {
        // Atomic write via temp file + rename
        let tmp = path.appendingPathExtension("tmp")
        let data = try encoder.encode(notebook)
        try data.write(to: tmp, options: .atomic)
        _ = try FileManager.default.replaceItemAt(path, withItemAt: tmp)
        store[path.path] = notebook
    }
}
```

**Atomic write:** Always write to a `.tmp` file then rename. This prevents data loss if the process is killed mid-write. `replaceItemAt(_:withItemAt:)` is atomic on the same volume.

**Date encoding:** Use ISO 8601 for human-readable JSON. Set `encoder.dateEncodingStrategy = .iso8601` and `decoder.dateDecodingStrategy = .iso8601`.

### 6.4 FileWriter

Handles all filesystem output. No Apple Events.

**Responsibilities:**
- Write HTML and plaintext note files
- Set file creation/modification dates to match Notes.app dates
- Handle note renames: delete old files if filename changed
- Create output directories on first use (no per-folder mkdir)

```swift
class FileWriter {
    private let fileManager = FileManager.default
    private var createdDirectories: Set<String> = []

    func ensureDirectory(_ url: URL) throws {
        let path = url.path
        guard !createdDirectories.contains(path) else { return }
        try fileManager.createDirectory(at: url, withIntermediateDirectories: true)
        createdDirectories.insert(path)
    }

    func write(html: String, plaintext: String, to paths: NotePaths, dates: NoteDates) throws {
        try ensureDirectory(paths.htmlPath.deletingLastPathComponent())
        try ensureDirectory(paths.textPath.deletingLastPathComponent())

        try html.write(to: paths.htmlPath, atomically: true, encoding: .utf8)
        try plaintext.write(to: paths.textPath, atomically: true, encoding: .utf8)

        // Set file dates to match Notes.app
        let attributes: [FileAttributeKey: Any] = [
            .creationDate: dates.created,
            .modificationDate: dates.modified
        ]
        try fileManager.setAttributes(attributes, ofItemAtPath: paths.htmlPath.path)
        try fileManager.setAttributes(attributes, ofItemAtPath: paths.textPath.path)
    }

    func handleRename(from oldName: String, to newName: String, in directory: URL) {
        guard oldName != newName && !oldName.isEmpty else { return }
        for ext in ["html", "txt"] {
            let old = directory.appendingPathComponent(oldName).appendingPathExtension(ext)
            let new = directory.appendingPathComponent(newName).appendingPathExtension(ext)
            if fileManager.fileExists(atPath: old.path) {
                try? fileManager.moveItem(at: old, to: new)
            }
        }
    }
}
```

### 6.5 FilenameGenerator

Pure function, no side effects. Maps a note + context to a filename string.

**Supported format tokens:**

| Token | Value |
|---|---|
| `{title}` | Note name, sanitized |
| `{id}` | Short note ID |
| `{account}` | Account name, sanitized |
| `{folder}` | Folder name, sanitized |
| `{account-id}` | Account ID |
| `{short-account-id}` | First 8 chars of account ID |
| `{date}` | Export date, YYYY-MM-DD |
| `{year}` | Export year |
| `{month}` | Export month, zero-padded |

**Sanitization rules (makeValidFilename):**
- Replace `/` with `-`
- Replace `:` with `-`
- Collapse multiple spaces/dashes into one
- Strip leading/trailing whitespace and dots
- Maximum 200 characters (truncate at word boundary if possible)
- If empty after sanitization, use note ID as fallback

```swift
struct FilenameGenerator {
    static func generate(
        format: String,
        title: String,
        noteID: String,
        accountName: String,
        folderName: String,
        accountID: String,
        exportDate: Date
    ) -> String {
        var result = format
        result = result.replacingOccurrences(of: "{title}", with: sanitize(title))
        result = result.replacingOccurrences(of: "{id}", with: noteID)
        result = result.replacingOccurrences(of: "{account}", with: sanitize(accountName))
        result = result.replacingOccurrences(of: "{folder}", with: sanitize(folderName))
        result = result.replacingOccurrences(of: "{account-id}", with: accountID)
        result = result.replacingOccurrences(of: "{short-account-id}", with: String(accountID.prefix(8)))
        // ... date tokens ...
        let sanitized = sanitize(result)
        return sanitized.isEmpty ? noteID : String(sanitized.prefix(200))
    }

    static func sanitize(_ input: String) -> String {
        var s = input
        s = s.replacingOccurrences(of: "/", with: "-")
        s = s.replacingOccurrences(of: ":", with: "-")
        // Collapse runs of whitespace and dashes
        while s.contains("  ") { s = s.replacingOccurrences(of: "  ", with: " ") }
        while s.contains("--") { s = s.replacingOccurrences(of: "--", with: "-") }
        s = s.trimmingCharacters(in: CharacterSet.whitespaces.union(.init(charactersIn: ".")))
        return s
    }
}
```

---

## 7. CLI Interface

Built with [Swift ArgumentParser](https://github.com/apple/swift-argument-parser).

### Subcommands

```
notesmith <subcommand>

Subcommands:
  export    Export notes from Apple Notes to the filesystem
  sync      (future) Bidirectional sync between Notes and filesystem
  query     (future) Natural-language search over exported notes
  stats     Print statistics about the last export run
```

### `export` Options

```
USAGE: notesmith export [<options>] <output-directory>

ARGUMENTS:
  <output-directory>    Root directory for exported files

OPTIONS:
  --data-dir <dir>      Directory for tracking JSON files [default: <output>/data]
  --update-all          Re-export all notes, not just changed ones
  --filter-accounts <a> Comma-separated account names to include
  --filter-folders <f>  Comma-separated folder names to include
  --note-limit <n>      Maximum total notes to export
  --per-folder <n>      Maximum notes per folder
  --since <date>        Only export notes modified after this date (ISO 8601)
  --probability <p>     Random sampling probability 0.0–1.0 [default: 1.0]
  --no-subdirs          Write all notes flat (no account/folder subdirectories)
  --filename-format <f> Filename template [default: {title}]
  --subdir-format <f>   Subdirectory template [default: {account}/{folder}]
  --no-html             Skip HTML output
  --no-plaintext        Skip plaintext output
  --verbose             Detailed progress output
  -h, --help            Show help
```

### Environment Variables

All options can also be set via environment variables (CLI args take precedence):

```
NOTESMITH_OUTPUT_DIR
NOTESMITH_DATA_DIR
NOTESMITH_UPDATE_ALL
NOTESMITH_FILTER_ACCOUNTS
NOTESMITH_FILTER_FOLDERS
NOTESMITH_NOTE_LIMIT
NOTESMITH_PER_FOLDER_LIMIT
NOTESMITH_MODIFIED_AFTER
NOTESMITH_PROBABILITY
NOTESMITH_USE_SUBDIRS
NOTESMITH_FILENAME_FORMAT
NOTESMITH_SUBDIR_FORMAT
```

### Output Format

Progress is written to stderr; structured output (stats JSON) is written to stdout on completion. This allows `notesmith export ... > stats.json` to capture the report.

```
[12:00:01] Loading 129 existing notebook files...
[12:00:01] Loaded 876 note records
[12:00:02] Scanning 172 folders...
[12:00:04]   iCloud: 172 folders, 875 notes
[12:00:08] 0 notes changed, 875 unchanged
[12:00:08] Export complete in 7s
```

---

## 8. Concurrency Model

### Apple Events Are Serial

Notes.app processes one Apple Event at a time. Concurrent AE dispatch queues behind the AE server. Therefore, **all AE calls must be on a single actor or serial queue**.

```swift
actor NotesActor {
    private let client: ScriptingBridgeClient

    func fetchMetadata(for folder: FolderInfo) async throws -> [NoteMetadata] {
        try client.noteMetadata(in: folder)
    }

    func fetchContent(for folder: FolderInfo, ids: Set<String>) async throws -> [String: NoteContent] {
        try client.noteContent(in: folder, ids: ids)
    }
}
```

### File I/O Can Be Parallelized

After fetching content for a folder, writing N notes to disk is independent. Use `withThrowingTaskGroup` to parallelize writes within a folder:

```swift
try await withThrowingTaskGroup(of: Void.self) { group in
    for (id, content) in changedContent {
        group.addTask {
            try self.fileWriter.write(html: content.body, plaintext: content.plaintext, ...)
        }
    }
    try await group.waitForAll()
}
```

**Folder processing remains serial** because:
1. AEs must be serial (see above)
2. The DataStore write after each folder must complete before the next folder modifies the same data

### Concurrency Architecture

```
Main Task (serial)
  └─ For each account (serial)
       └─ Fetch folder list (AE, serial)
       └─ For each folder (serial)
            ├─ Fetch metadata (AE, serial)    ← on NotesActor
            ├─ Fetch content if needed (AE)   ← on NotesActor
            └─ Write files (parallel)          ← TaskGroup
```

---

## 9. Error Handling & Recovery

### Error Taxonomy

```swift
enum NotesClientError: Error {
    case appNotFound
    case appFailedToLaunch
    case scriptingBridgeUnavailable
    case permissionDenied       // TCC — user has not granted Notes access
    case noteLazilyEvaluatedToNil(id: String)  // Note deleted mid-export
    case unexpectedType(expected: String, got: Any)
}

enum DataStoreError: Error {
    case directoryNotFound(URL)
    case readFailed(URL, underlying: Error)
    case writeFailed(URL, underlying: Error)
    case decodingFailed(URL, underlying: Error)
    case diskFull
}

enum FileWriterError: Error {
    case outputDirectoryNotWritable(URL)
    case writeFailed(URL, underlying: Error)
    case diskFull
}
```

### Recovery Policy

| Error | Behavior |
|---|---|
| Notes.app not running | Launch it, retry once, then abort with clear message |
| TCC permission denied | Abort immediately with message directing user to System Settings > Privacy > Notes |
| Note deleted mid-export | Skip note, log warning, continue |
| JSON decode failure (corrupt data) | Log warning, treat folder as having no existing data (re-export all notes in folder) |
| Disk full | Abort immediately, print partial stats |
| Individual file write failure | Log warning, skip note, continue |

### TCC (Privacy Permissions)

Apple Notes requires the user to grant the tool automation access. On first run, if access is denied:

```
Error: notesmith does not have permission to access Notes.

Grant access in:
  System Settings → Privacy & Security → Automation → notesmith → Notes ✓

Then re-run notesmith.
```

Detection: Catch the NSError with domain `com.apple.AppleEventsErrorDomain` code `-1743` (not authorized).

---

## 10. Testing Strategy

### Unit Tests (no Notes.app required)

Most logic is testable without Apple Events via the `NotesClientProtocol` abstraction.

```swift
// MockNotesClient.swift
struct MockNotesClient: NotesClientProtocol {
    var accountsToReturn: [AccountInfo] = []
    var foldersToReturn: [String: [FolderInfo]] = [:]  // keyed by account ID
    var metadataToReturn: [String: [NoteMetadata]] = [] // keyed by folder ID
    var contentToReturn: [String: [String: NoteContent]] = []

    func accounts() throws -> [AccountInfo] { accountsToReturn }
    // ...
}
```

**Unit test targets:**

| Test | What it covers |
|---|---|
| `FilenameGeneratorTests` | All format tokens, sanitization edge cases, empty strings, Unicode, path separators |
| `DataStoreTests` | Load/save round-trip, atomic write (simulate crash mid-write), corrupt JSON recovery, deleted note tracking |
| `ExportEngineTests` | Incremental skip logic, full-update logic, rename detection, delete tombstoning |
| `StringExtensionTests` | `extractedNoteID` with various input formats |

### Integration Tests (requires Notes.app, marked with `.integration` tag)

A small test notebook "Notesmith Test Notebook" with known content is used for integration tests. These are excluded from CI but run locally before releases.

### Performance Benchmarks

Use `XCTestCase.measure` to track regressions in hot paths:
- `FilenameGenerator.sanitize` on a 10,000-element batch
- `DataStore.loadAll` with 200 JSON files
- `ExportEngine.partitionChangedNotes` on 5,000 note records

---

## 11. Configuration

### Config File

Optional `~/.config/notesmith/config.json` for persistent defaults:

```json
{
  "outputDirectory": "~/Documents/Notes",
  "dataDirectory": "~/Documents/Notes/data",
  "filenameFormat": "{title}",
  "subdirFormat": "{account}/{folder}",
  "useSubdirectories": true
}
```

CLI args and environment variables always override the config file.

### Data Directory Layout

The data directory contains one JSON file per notebook:

```
data/
├── iCloud-Daily Notes.json
├── iCloud-Work.json
├── iCloud-Personal.json
├── export_stats.json       # Last run statistics
└── ...
```

The subdirectory format determines the JSON filename: `{account}-{folder}.json` in flat mode, or `{account}/{folder}.json` in subdir mode (with intermediate directories).

---

## 12. File Layout on Disk

### Default (subdirectory mode)

```
<output-dir>/
├── html/
│   └── iCloud/
│       ├── Daily Notes/
│       │   ├── my-first-note.html
│       │   └── another-note.html
│       └── Work/
│           └── meeting-notes.html
├── text/
│   └── iCloud/
│       ├── Daily Notes/
│       │   ├── my-first-note.txt
│       │   └── another-note.txt
│       └── Work/
│           └── meeting-notes.txt
└── data/
    ├── iCloud-Daily Notes.json
    └── iCloud-Work.json
```

### Flat mode (`--no-subdirs`)

```
<output-dir>/
├── html/
│   ├── my-first-note.html
│   └── meeting-notes.html
├── text/
│   ├── my-first-note.txt
│   └── meeting-notes.txt
└── data/
    ├── iCloud-Daily Notes.json
    └── iCloud-Work.json
```

---

## 13. Roadmap

### Phase 1: Core Export CLI (MVP)

**Goal:** Match all functionality of the AppleScript implementation, faster.

- [ ] ScriptingBridge header generation and integration
- [ ] `NotesClientProtocol` + `ScriptingBridgeClient`
- [ ] `DataStore` with Codable models
- [ ] `ExportEngine` with incremental logic
- [ ] `FileWriter` with rename/delete handling
- [ ] `FilenameGenerator` with all format tokens
- [ ] `notesmith export` CLI subcommand
- [ ] Compatibility with existing `data/*.json` files from AppleScript exporter
- [ ] Unit tests for all modules
- [ ] Performance: incremental <10s, full <45s for 875 notes

### Phase 2: Observability & Polish

- [ ] `notesmith stats` subcommand with per-folder breakdown
- [ ] `--verbose` mode with per-note logging
- [ ] Machine-readable JSON stats output (`--json`)
- [ ] launchd plist for scheduled background exports
- [ ] Homebrew formula for easy install

### Phase 3: Search Index

**Goal:** Full-text search over exported notes without re-reading files.

- [ ] SQLite database alongside data directory (`notesmith.db`)
- [ ] FTS5 virtual table over note content
- [ ] Index is updated incrementally alongside exports
- [ ] `notesmith query "text"` subcommand
- [ ] Returns matching note titles + file paths

### Phase 4: AI Search

**Goal:** Semantic search using embeddings.

- [ ] Embedding generation via local model (e.g., via `llm` CLI or CoreML model)
- [ ] Embedding stored per note in SQLite BLOB column
- [ ] `notesmith query --semantic "what was that thing about..."` subcommand
- [ ] Optional: OpenAI/Anthropic API for embedding generation

### Phase 5: Bidirectional Sync

**Goal:** Changes to exported files sync back to Notes.app.

- [ ] File watcher using FSEvents on the output directory
- [ ] Detect which exported file changed (diff against last known content)
- [ ] Write back via `tell application "Notes" to set body of note ... to ...`
- [ ] Conflict resolution: Notes.app timestamp vs filesystem mtime
- [ ] `notesmith sync` subcommand (long-running daemon mode)

### Phase 6: Mac App (SwiftUI)

**Goal:** Consumer-facing product.

- [ ] Menu bar app with sync status indicator
- [ ] Preferences window for output directory, format options
- [ ] Folder browser showing last-exported dates and note counts
- [ ] In-app search (wraps Phase 3/4 search)
- [ ] Onboarding flow including TCC permission grant
- [ ] App Store submission (requires hardened runtime, notarization)
- [ ] Sparkle auto-updater for direct distribution

---

## Appendix A: Why Not JXA?

JavaScript for Automation (JXA) is an alternative to AppleScript and ScriptingBridge. It shares the same Apple Events infrastructure but uses the JavaScriptCore engine.

Advantages of JXA over AppleScript:
- Familiar syntax
- Better string handling
- No `do shell script` needed (can use Node modules if run via `node`)

Disadvantages vs Swift:
- JXA is no longer actively developed by Apple (minimal documentation updates since 2014)
- Still interpreted (no compile-time type safety)
- File I/O and JSON still require either shell forks or JXA's limited built-in I/O
- Cannot use Swift concurrency or Swift Testing

Swift with ScriptingBridge is strictly better for a production tool.

## Appendix B: Notes.app SDEF Summary

Notes.app's scripting dictionary (accessible via Script Editor) defines:

```
Notes Suite:
  application
    ├── accounts (list of account)
  account
    ├── name (text)
    ├── id (text)
    └── folders (list of folder)
  folder
    ├── name (text)
    ├── id (text)
    └── notes (list of note)
  note
    ├── name (text, r/w)
    ├── id (text, r/o)
    ├── body (text, r/w)            — HTML
    ├── plaintext (text, r/o)
    ├── creation date (date, r/o)
    └── modification date (date, r/o)
```

`body` is the authoritative format. `plaintext` is a read-only derived property. Writing back to `body` is how bidirectional sync works.

## Appendix C: Compatibility with AppleScript Data Files

The Notesmith JSON schema is a strict superset of the notes-exporter format. Migration is automatic on first run: the DataStore decoder reads the old pipe-delimited fields if present and upgrades to the typed Codable format.

Old format (written by AppleScript):
```json
{
  "p1234": {
    "filename": "my-note",
    "created": "Wednesday, January 1, 2025 at 12:00:00 AM",
    "modified": "Wednesday, January 1, 2025 at 12:00:00 AM",
    "firstExported": "Wednesday, January 1, 2025 at 12:00:00 AM",
    "lastExported": "Wednesday, January 1, 2025 at 12:00:00 AM",
    "exportCount": 3,
    "fullNoteId": "x-coredata://UUID/ICNote/p1234"
  }
}
```

New format (Notesmith Codable):
```json
{
  "version": 1,
  "notes": {
    "p1234": {
      "noteID": "p1234",
      "fullNoteID": "x-coredata://UUID/ICNote/p1234",
      "filename": "my-note",
      "created": "2025-01-01T00:00:00Z",
      "modified": "2025-01-01T00:00:00Z",
      "firstExported": "2025-01-01T00:00:00Z",
      "lastExported": "2025-01-01T00:00:00Z",
      "exportCount": 3
    }
  }
}
```

The migration reads AppleScript date strings using `DateFormatter` with the locale-specific format and re-encodes as ISO 8601.
