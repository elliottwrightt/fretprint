# 🎸 Guitar Learning App — macOS Product Plan

---

> **Scope**: Personal use only — runs locally on a single MacBook. No App Store distribution, no multi-user accounts, no commercial licensing concerns.

## 1. Overview

A macOS desktop application that listens to a user's guitar playing via microphone or digital audio interface, guides them through songs section by section, and stitches together their best recorded attempts into a single cohesive performance.

---

## 2. Goals

- Make learning songs structured and measurable
- Provide real-time visual feedback with interactive chord/tab diagrams
- Automatically identify and save the best take of each section
- Produce a complete, user-played version of any chosen song

---

## 3. Technology Stack

### Primary Language: Swift
All UI, audio pipeline, and app logic is written in **Swift**, targeting macOS 13+. This gives native Core Audio access, low-latency AVFoundation integration, and a clean SwiftUI interface without cross-platform overhead.

### Python Sidecar
Two features require a Python subprocess — **Demucs** (stem separation) and **pitch detection** (CREPE/aubio). These run in a bundled, self-contained Python environment using `python-build-standalone` so there is no dependency on any system Python. Swift communicates with the sidecar over **stdin/stdout pipes** or a **local Unix socket**.

### What Not To Use
- ❌ Electron / web stack — loses direct Core Audio access, poor audio latency
- ❌ Python for the full app — fine for ML tasks but wrong for UI and real-time audio
- ❌ React Native / Flutter — unnecessary complexity for a single-user macOS app

### Full Stack Reference

| Layer | Technology | Notes |
|---|---|---|
| UI | SwiftUI | Native macOS, Light/Dark mode |
| Audio I/O | Core Audio + AVFoundation | Low-latency hardware access |
| Audio analysis | AudioKit (Swift) | Onset detection, waveform display |
| Stem separation | Demucs (Python sidecar) | Called via subprocess, results cached |
| Pitch detection | aubio or CREPE (Python sidecar) | Best accuracy for guitar |
| Tab scraping | URLSession + SwiftSoup | HTML parsing in Swift |
| Track assembly | AVMutableComposition (Swift) | Native stitching + crossfade |
| Local storage | SwiftData or SQLite via GRDB | File-based, no server needed |
| YouTube audio | yt-dlp (bundled binary) | Called from Swift via Process |
| Python runtime | python-build-standalone | Bundled venv, no system dependency |

### Recommended Build Order
Build in this order to de-risk the hardest parts first:

1. **Core Audio input** — get a signal from mic/interface into the app
2. **yt-dlp playback** — download and play a YouTube track via AVFoundation
3. **Python sidecar bridge** — establish Swift↔Python IPC and run Demucs on a test file
4. **Tab display** — fetch and render a tab with auto-scroll
5. **Section + recording loop** — the core learning flow
6. **Scoring + assembly** — analysis engine and final export
7. **UI polish** — chord overlays, stem toggles, settings

---

## 4. Target Platform

| Attribute | Detail |
|---|---|
| OS | macOS 13 (Ventura) and above |
| Architecture | Universal Binary (Apple Silicon + Intel) |
| Language | Swift 5.9+ |
| UI Framework | SwiftUI |
| Audio Engine | AVFoundation / Core Audio |
| Min RAM | 8 GB (16 GB recommended for Demucs) |
| Storage | ~500 MB app + user recordings + stem cache |

---

## 4. Core Features

### 4.1 Audio Input
- Connect via **system microphone**, **USB audio interface**, or **virtual audio device**
- Audio device selector in Settings
- Real-time level meter and latency compensation
- Support for common sample rates: 44.1 kHz, 48 kHz
- Low-latency input buffer using Core Audio Audio Units (AU)

### 4.2 Song Discovery & Import
- **YouTube Integration**: Search and download audio using `yt-dlp` (installed via Homebrew or bundled) — personal use only
- **Apple Music Integration**: Browse and play tracks via MusicKit on macOS (requires Apple Music subscription)
- **Local audio files**: Import your own MP3/WAV/M4A files directly as a song source
- Song search bar and recently played / saved song library

### 4.3 Stem Separation (Backing Track Generation)

Once a song's audio is loaded, the app can isolate individual stems using **Demucs** (Meta, MIT licence) — a local, offline ML model that splits a track into:

| Stem | Used for |
|---|---|
| Vocals | **Discarded** (or optionally kept) |
| Drums | ✅ Kept in backing track |
| Bass | ✅ Kept in backing track |
| Other (keys, pads, etc.) | ✅ Kept in backing track |
| Guitar (htdemucs_6s model) | Optionally discarded so only *your* playing is heard |

**Implementation:**
- Demucs is called via a **Python sidecar process** bundled with the app (using a embedded Python environment via `python-build-standalone` or a lightweight venv)
- Separation runs **fully locally** — no data leaves the machine
- Processing happens once per song and the stems are cached to `~/Library/Application Support/GuitarApp/stems/`
- The user can choose which stems to include in the backing mix via a simple toggle UI (e.g. turn bass off to practice bass lines yourself)
- Stem mixing uses `AVMutableComposition` with per-track volume controls

**Models:**
- Default: `htdemucs` — fast, good quality, 4 stems
- Optional: `htdemucs_6s` — slower, separates guitar and piano as additional stems
- Model files (~80–320 MB) downloaded once on first use and cached locally

**UI:**
- "Generate Backing Track" button on the song detail screen
- Progress bar during separation (typically 1–3× song duration on Apple Silicon)
- Per-stem toggle switches + volume sliders before practice begins

### 4.4 Tabs & Chords
- Fetch tabs by scraping **Ultimate Guitar** or **Songsterr** for personal use (no redistribution)
- Support importing locally saved `.gp`, `.gpx`, or plain-text tab files
- Display options: **Guitar Tab view**, **Chord chart view**, or **Both**
- Auto-scroll synced to song playback position

### 4.5 Interactive Chord Diagrams
- Each chord name in the tab view is **tappable**
- On tap: pop-up overlay shows:
  - Fretboard diagram with numbered finger positions
  - Finger labels (index, middle, ring, pinky)
  - Optional: short animation showing transition from previous chord
- Chord images rendered as **SVG** (scalable, themeable) — stored locally for all standard chords; custom/unusual chords fetched on demand
- Supports standard, capo, and alternate tuning positions

### 4.6 Section-by-Section Learning Mode

Each song is divided into **sections** (Intro, Verse 1, Pre-Chorus, Chorus, Bridge, etc.) or alternatively **lick-by-lick** for lead guitar / solos.

**Section workflow:**
1. User selects a section to practice
2. App plays the reference audio for that section (looped, with adjustable tempo)
3. User plays along — recording begins automatically on first note detected
4. Attempt is captured and temporarily saved to a local session buffer
5. User can replay, re-record, or mark a take as "skip"
6. After N attempts (configurable, default: 3), the app auto-selects the best take

### 4.7 Attempt Analysis & Best Take Selection

Each recorded attempt is compared against the reference audio using:

| Signal | Method |
|---|---|
| Pitch accuracy | Pitch detection via YIN or CREPE algorithm |
| Timing accuracy | Onset detection, beat alignment |
| Dynamics/Volume | RMS envelope comparison |
| Completeness | Duration coverage vs. reference length |

A **composite score** is calculated per attempt. The highest-scoring take is flagged as the **Best Take** for that section. The user can override the selection manually.

All attempts are stored in a temporary session buffer (`~/Library/Application Support/GuitarApp/sessions/`) and cleared after 30 days unless promoted.

### 4.8 Song Assembly & Final Export

Once all sections are marked complete:

1. The best takes for each section are **stitched sequentially** using AVFoundation `AVMutableComposition`
2. A crossfade or clean splice is applied at section boundaries (user-configurable)
3. The assembled track is mixed with optional reference backing track at reduced volume
4. Exported as:
   - `.m4a` (default, lossless AAC)
   - `.wav` (uncompressed)
   - `.mp3` (optional, via bundled LAME encoder)
5. File is saved to `~/Music/GuitarApp/` and optionally shared to the macOS Share Sheet

---

## 6. App Architecture

```
GuitarApp/
├── App/
│   ├── AppDelegate.swift
│   └── MainWindowController.swift
│
├── Audio/
│   ├── AudioInputManager.swift       # Core Audio device handling
│   ├── PitchDetector.swift           # Bridges to Python sidecar (CREPE/aubio)
│   ├── AttemptRecorder.swift         # Records + buffers takes
│   ├── AttemptAnalyser.swift         # Composite scoring engine
│   └── SongAssembler.swift           # AVMutableComposition stitching
│
├── Sidecar/
│   ├── PythonSidecarManager.swift    # Launches + manages Python process lifecycle
│   ├── SidecarProtocol.swift         # JSON message schema over stdin/stdout
│   ├── demucs_runner.py              # Thin Python script: receives path, runs Demucs, returns stem paths
│   ├── pitch_runner.py               # Thin Python script: receives audio path, returns pitch/onset data
│   └── python/                       # Bundled python-build-standalone runtime + venv
│       └── (demucs, aubio, crepe, numpy, torch...)
│
├── UI/
│   ├── SongSearchView.swift
│   ├── SongPlayerView.swift
│   ├── StemMixerView.swift           # Per-stem toggles + volume sliders
│   ├── TabScrollView.swift           # Auto-scrolling tab/chord display
│   ├── ChordDiagramOverlay.swift     # Interactive chord pop-up (SVG)
│   ├── SectionListView.swift
│   ├── AttemptReviewView.swift
│   └── FinalExportView.swift
│
├── Models/
│   ├── Song.swift
│   ├── Section.swift
│   ├── Attempt.swift
│   └── ChordDiagram.swift
│
├── Services/
│   ├── YouTubeService.swift          # Wraps yt-dlp via Process
│   ├── AppleMusicService.swift       # MusicKit integration
│   ├── TabFetchService.swift         # Scrapes Ultimate Guitar / Songsterr
│   ├── StemService.swift             # Coordinates Demucs via sidecar
│   └── ChordLibraryService.swift     # Local SVG chord assets
│
├── Storage/
│   ├── SessionStore.swift            # Temporary attempt buffers
│   └── SongLibraryStore.swift        # Saved songs + completions (SwiftData)
│
└── Resources/
    ├── Chords/                       # SVG chord diagram assets
    └── yt-dlp                        # Bundled binary
```

### Swift ↔ Python IPC Pattern

Swift launches the Python sidecar once at app start and keeps it alive. Communication is JSON over stdin/stdout:

```swift
// Swift sends:
{ "command": "separate_stems", "input_path": "/path/to/song.m4a", "model": "htdemucs" }

// Python responds:
{ "status": "ok", "stems": { "drums": "/path/stems/drums.wav", "bass": "/path/stems/bass.wav", "other": "/path/stems/other.wav" } }
```

---

## 7. Data Flow

```
[User selects song]
        │
        ▼
[Fetch audio stream (YouTube / Apple Music)]
[Fetch tab / chords]
        │
        ▼
[Split song into sections]
        │
        ▼
[For each section:]
   [Play reference audio (looped)]
   [User plays → Audio captured]
   [Attempt saved to session buffer]
   [Score attempt vs. reference]
   [Repeat until section marked complete]
        │
        ▼
[All sections complete?]
        │ Yes
        ▼
[Retrieve best take per section]
[Stitch using AVMutableComposition]
[Export final track]
```

---

## 8. Third-Party Dependencies

| Library / Tool | Purpose | License |
|---|---|---|
| `yt-dlp` | YouTube audio extraction | Unlicense |
| MusicKit (Apple) | Apple Music playback | Apple SDK |
| YouTube Data API v3 | Song search | Google ToS |
| Ultimate Guitar / Songsterr | Tab data | Commercial / ToS |
| `Demucs` (Meta) | Stem separation / vocal removal | MIT |
| `python-build-standalone` | Embedded Python runtime for Demucs | MIT |
| `AudioKit` | Audio analysis helpers | MIT |
| `swift-pitch` or CREPE (Python bridge) | Pitch detection | MIT / Apache |
| LAME (bundled) | MP3 encoding | LGPL |

---

## 9. Permissions Required

| Permission | Reason |
|---|---|
| Microphone | Capture guitar input |
| Music Library | Access Apple Music |
| Network | Stream songs, fetch tabs |
| File System (`~/Music`) | Export final recordings |

---

## 10. Settings & Customisation

- **Audio device selector** — microphone or interface
- **Input gain & monitoring toggle**
- **Playback tempo** — slow down reference audio (50%–100%)
- **Section detection method** — automatic vs. manual markers
- **Min attempts before best-take selection** — default: 3
- **Splice type at assembly** — hard cut, 50ms crossfade, 100ms crossfade
- **Export format** — M4A / WAV / MP3
- **Theme** — Light / Dark / System

---

## 11. Milestones

| Phase | Deliverable | Est. Duration |
|---|---|---|
| 1 — Foundation | Core Audio input, device selector, basic recording | 3 weeks |
| 2 — Song Discovery | yt-dlp download + AVFoundation playback; Apple Music | 3 weeks |
| 3 — Python Sidecar | Swift↔Python IPC bridge; Demucs stem separation working end-to-end | 3 weeks |
| 4 — Tabs & Chords | Tab scraping, auto-scroll display, chord diagram overlays | 4 weeks |
| 5 — Section Learning | Section detection, looped practice, attempt recording | 4 weeks |
| 6 — Analysis Engine | Pitch/timing scoring via Python sidecar, best-take selection | 3 weeks |
| 7 — Assembly & Export | AVMutableComposition stitching, export options | 2 weeks |
| 8 — Polish | UI polish, stem mixer UI, settings, onboarding | 2 weeks |
| **Total** | | **~24 weeks** |

---

## 12. Open Questions / Risks

- **YouTube & tab scraping**: Since this is for personal use only and not distributed, `yt-dlp` usage and tab scraping sit in a practical grey area — just don't share the app or its outputs publicly.
- **Pitch detection accuracy**: Classical MIDI-style detection struggles with distorted guitar; consider training a lightweight ML model on guitar-specific audio, or using `aubio` / `librosa` via a Python sidecar.
- **Section auto-detection**: Reliable automatic section splitting requires ML-based music structure analysis. Manual section tagging should always be available as a fallback.
- **Latency**: Budget interfaces may experience input latency > 20ms; provide a manual latency compensation slider.
- **Apple Music DRM**: MusicKit streams protected audio — it cannot be recorded directly. Use Apple Music only for reference playback; record only the user's guitar input.

---

*Document version: 1.2 — Technology stack added; architecture updated with Python sidecar*
