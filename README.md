# BASDN

```
██████╗  █████╗ ███████╗██████╗ ███╗   ██╗
██╔══██╗██╔══██╗██╔════╝██╔══██╗████╗  ██║
██████╔╝███████║███████╗██║  ██║██╔██╗ ██║
██╔══██╗██╔══██║╚════██║██║  ██║██║╚██╗██║
██████╔╝██║  ██║███████║██████╔╝██║ ╚████║
╚═════╝ ╚═╝  ╚═╝╚══════╝╚═════╝ ╚═╝  ╚═══╝
         T U R B O   E N G I N E   v3.0
```

> **B**atch **A**uto-retry **S**treaming **D**ownload e**N**gine

A turbocharged CLI multi-downloader for Linux. Multi-segment parallel downloads (aria2c-style), concurrent file queue, real-time curses TUI, auto-resume, and exponential-backoff retry. Paste a wall of links, walk away.

**Dev** · `0xb0rn3` | `0xbv1`  
**GitHub** · [https://github.com/0xb0rn3/Basdn](https://github.com/0xb0rn3/Basdn)

---

## What's New in v3.0

| Change | Details |
|---|---|
| **Multi-segment downloads** | Each file split into N parallel segments (default 8), downloaded concurrently — aria2c-style acceleration |
| **Concurrent file queue** | Download M files simultaneously (default 3, configurable up to 10) |
| **Connection pooling** | Persistent HTTP sessions with urllib3 connection pooling and transport-level retry |
| **Curses TUI dashboard** | Real-time per-file progress bars, per-segment tracking, aggregate speed, ETA, scrollable log |
| **256 KiB read buffer** | Tuned chunk size for throughput (up from 1 MiB sequential reads) |
| **Removed tqdm dependency** | TUI handles all progress display — fewer deps |
| **`--no-tui` mode** | Plain text output for scripts and pipes |
| **Configurable parallelism** | `-j` (concurrent files) and `-s` (segments per file) flags |

---

## Performance Guide

| Profile | Command | Connections | Best For |
|---|---|---|---|
| **Maximum throughput** | `./basdn -f links.txt -j 5 -s 16` | 80 | Fast server, fat pipe |
| **Balanced (default)** | `./basdn -f links.txt` | 24 | General use |
| **Gentle** | `./basdn -f links.txt -j 1 -s 4` | 4 | Rate-limited servers |
| **Single-threaded** | `./basdn -f links.txt -j 1 -s 1` | 1 | Legacy (v2 behavior) |

The formula is simple: **total connections = jobs × segments**. Tune to your bandwidth and server tolerance.

---

## Features

| Feature | Details |
|---|---|
| **Multi-segment parallel** | Each file downloaded in N concurrent byte-range segments |
| **Concurrent file queue** | M files downloading simultaneously via thread pool |
| **Connection pooling** | HTTP keep-alive, pooled connections, transport-level retry |
| **Curses TUI** | Real-time dashboard with per-file progress, speed, ETA, scrollable log |
| **Auto resume** | Per-segment state tracking — resumes from exact byte offset |
| **Auto retry** | Up to 10 retries per file with exponential backoff (5s → 120s cap) |
| **File integrity** | Final size verified against `Content-Length` |
| **State persistence** | Atomic JSON state saves every 5 seconds |
| **Skip completed** | Re-running skips fully downloaded files |
| **Auto setup** | Detects Python, bootstraps pip, installs deps — PEP 668 aware |
| **Range detection** | Auto-detects server `Accept-Ranges` support; falls back to single-segment |
| **Zero config** | Single file — no install step, no config files |

---

## Requirements

| Requirement | Version |
|---|---|
| Linux | Any modern distro |
| Bash | 4.0+ |
| Python | 3.8+ |
| pip | Auto-bootstrapped if missing |

Python packages (`requests`, `colorama`) are installed automatically on first run.

---

## Installation

```bash
git clone https://github.com/0xb0rn3/Basdn.git
cd Basdn
chmod +x basdn
./basdn
```

Or one-liner:

```bash
curl -O https://raw.githubusercontent.com/0xb0rn3/Basdn/main/basdn && chmod +x basdn && ./basdn
```

---

## Usage

### Interactive Mode (default)

```bash
./basdn
```

Launches the multi-link paste handler. Paste URLs, type `DONE`, and the TUI takes over.

### File Mode

```bash
./basdn -f links.txt
./basdn -f links.txt -j 5 -s 16        # turbo mode
./basdn -f links.txt -o /mnt/external   # custom output dir
```

### Direct URLs

```bash
./basdn -u https://example.com/file1.pkg https://example.com/file2.iso
```

### Flags Reference

| Flag | Long form | Description |
|---|---|---|
| `-f FILE` | `--file FILE` | Load URLs from a text file |
| `-u URL ...` | `--urls URL ...` | Pass one or more URLs directly |
| `-o DIR` | `--output DIR` | Output directory (default: `./basdn_downloads`) |
| `-j N` | `--jobs N` | Concurrent file downloads (1-10, default: 3) |
| `-s N` | `--segments N` | Parallel segments per file (1-32, default: 8) |
| `--no-tui` | | Plain text output (no curses dashboard) |
| `--reset` | | Clear saved state — re-downloads everything |
| `--setup-only` | | Install dependencies only |
| `-h` | `--help` | Show help and exit |

---

## TUI Dashboard

The curses-based TUI shows:

```
╔══════════════════════════════════════════════════════════╗
║  BASDN v3.0 TURBO │ Multi-Seg │ Concurrent │ Resume     ║
╚══════════════════════════════════════════════════════════╝
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 3✔ 0✖ 0⏭ 2↓  │  45.2 MiB/s  │  1.2 GiB/3.8 GiB  │  0:02:15
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 ↓ ELDEN RING - [EU] [EN] [1.21].pkg [8seg]      23.4 MiB/s  ETA 12:30
   ███████████████████░░░░░░░░░░░░░░░░  42.3%  18.8 GiB/44.4 GiB
 ↓ game_update_v2.pkg [8seg]                       21.8 MiB/s  ETA 03:45
   ████████████████████████████░░░░░░░░  68.1%  2.1 GiB/3.1 GiB
 ✔ small_patch.pkg                                          DONE
   ████████████████████████████████████ 100.0%  45.2 MiB/45.2 MiB

─ LOG ─────────────────────────────────────────────────────
 [14:22:01] ▶ START ELDEN RING - [EU] [EN] [1.21].pkg (41.4 GiB) [8 seg]
 [14:22:01] ▶ START game_update_v2.pkg (3.1 GiB) [8 seg]
 [14:22:05] ✔ DONE  small_patch.pkg (45.2 MiB)

 [Q]uit  [↑↓]Scroll  │  BASDN v3.0 — 0xb0rn3 | 0xbv1
```

**Controls:**

| Key | Action |
|---|---|
| `Q` | Quit (saves progress — resume on re-run) |
| `↑ / ↓` | Scroll file list |
| `Ctrl+C` | Emergency stop (saves progress) |

---

## How Multi-Segment Works

1. BASDN sends a `HEAD` request to get `Content-Length` and check `Accept-Ranges: bytes`
2. If the server supports ranges, the file is split into N segments (default 8)
3. Each segment downloads its byte range in parallel using a thread pool
4. Segments are written to `.partN` temp files alongside the final destination
5. When all segments complete, BASDN merges them into the final file and removes parts
6. If the server doesn't support ranges, BASDN falls back to single-segment (v2 behavior)
7. Files under 2 MiB always use single-segment (splitting overhead isn't worth it)

---

## How Resume Works

Resume operates at the segment level:

1. Each `.partN` file tracks its own byte offset on disk
2. On restart, BASDN calculates `start_byte + existing_bytes` per segment
3. Each segment sends `Range: bytes=<offset>-<end>` to resume from its last position
4. State is atomically persisted to `.download_state.json` every 5 seconds

---

## How Retry Works

| Attempt | Delay |
|---|---|
| 1 | 5 seconds |
| 2 | 10 seconds |
| 3 | 20 seconds |
| 4 | 40 seconds |
| 5 | 80 seconds |
| 6-10 | 120 seconds (capped) |

On failure, all segment parts are cleaned up and the file retries from scratch. After 10 failed attempts, the file is marked `failed` and skipped.

---

## Output Structure

```
./basdn_downloads/
├── .download_state.json     ← resume/state tracking
├── .basdn_engine.py         ← auto-generated Python engine
├── .basdn_deps_ok           ← dependency stamp
├── bigfile1.pkg
├── bigfile2.pkg
└── bigfile3.iso
```

---

## License

MIT License — see `LICENSE` for details.

---

*Built by `0xb0rn3` | `0xbv1` — [github.com/0xb0rn3/Basdn](https://github.com/0xb0rn3/Basdn)*
