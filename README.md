# BASDN

```
██████╗  █████╗ ███████╗██████╗ ███╗   ██╗
██╔══██╗██╔══██╗██╔════╝██╔══██╗████╗  ██║
██████╔╝███████║███████╗██║  ██║██╔██╗ ██║
██╔══██╗██╔══██║╚════██║██║  ██║██║╚██╗██║
██████╔╝██║  ██║███████║██████╔╝██║ ╚████║
╚═════╝ ╚═╝  ╚═╝╚══════╝╚═════╝ ╚═╝  ╚═══╝
```

> **B**atch **A**uto-retry **S**treaming **D**ownload e**N**gine

A bulletproof CLI multi-downloader for Linux. Paste a wall of links, walk away. BASDN handles the rest — resuming broken downloads, retrying failed ones, and queueing everything with live progress.

**Dev** · `0xb0rn3` | `0xbv1`  
**GitHub** · [https://github.com/0xb0rn3/Basdn](https://github.com/0xb0rn3/Basdn)

---

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
  - [Interactive Mode](#interactive-mode-default)
  - [File Mode](#file-mode)
  - [Direct URL Mode](#direct-url-mode)
  - [Flags Reference](#flags-reference)
- [Multi-Link Paste Handler](#multi-link-paste-handler)
- [How Resume Works](#how-resume-works)
- [How Retry Works](#how-retry-works)
- [Download State](#download-state)
- [Output Structure](#output-structure)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Features

| Feature | Details |
|---|---|
| **Multi-link paste** | Paste a space or newline-separated block of URLs in one shot |
| **Auto resume** | Uses HTTP `Range` headers to continue exactly where a download stopped |
| **Auto retry** | Up to 10 retries per file with exponential backoff (5s → 10s → 20s … 2 min cap) |
| **File integrity check** | Verifies final file size against `Content-Length` — flags mismatches instead of silently saving corrupt files |
| **State persistence** | Progress saved to disk every 10 seconds — survives crashes, reboots, and `Ctrl+C` |
| **Skip completed** | Re-running BASDN skips files already fully downloaded |
| **Auto setup** | Detects Python, bootstraps pip if missing, installs all deps automatically |
| **PEP 668 aware** | Auto-detects externally-managed Python environments and applies `--break-system-packages` only when needed |
| **Zero config** | Single file — no install step, no config files required |
| **Live progress bars** | Per-file tqdm bars showing speed, ETA, and bytes transferred |
| **Queue summary** | Final report showing completed, skipped, and failed downloads |

---

## Requirements

| Requirement | Version |
|---|---|
| Linux | Any modern distro |
| Bash | 4.0+ |
| Python | 3.8+ |
| pip | Auto-bootstrapped if missing |

Python packages (`requests`, `tqdm`, `colorama`) are installed automatically on first run.

---

## Installation

```bash
# Clone the repo
git clone https://github.com/0xb0rn3/Basdn.git
cd Basdn

# Make executable
chmod +x basdn

# Run
./basdn
```

Or grab just the script:

```bash
curl -O https://raw.githubusercontent.com/0xb0rn3/Basdn/main/basdn
chmod +x basdn
./basdn
```

The first run automatically:
1. Detects your Python interpreter (3.8 → 3.14 supported)
2. Bootstraps `pip` if it isn't present
3. Installs `requests`, `tqdm`, and `colorama`
4. Writes the Python engine to `.ps4pkg_downloader.py`
5. Starts the download queue

Subsequent runs skip the setup step entirely (stamped via `.ps4pkg_deps_ok`).

---

## Usage

### Interactive Mode (default)

Run with no arguments to enter the **Multi-Link Paste Handler**:

```bash
./basdn
```

You'll see a prompt where you can paste URLs — one per line, or all at once space-separated.

---

### File Mode

Point BASDN at a text file with one URL per line:

```bash
./basdn -f links.txt
```

Lines starting with `#` are treated as comments and ignored. Blank lines are ignored.

**Example `links.txt`:**

```
# My download queue
https://example.com/file1.pkg
https://example.com/file2.zip
https://cdn.server.net/archive/bigfile.iso

# Commented-out link (skipped):
# https://example.com/skip-me.pkg
```

---

### Direct URL Mode

Pass URLs as arguments directly:

```bash
./basdn -u https://example.com/file1.pkg https://example.com/file2.pkg
```

---

### Flags Reference

| Flag | Long form | Description |
|---|---|---|
| `-f FILE` | `--file FILE` | Load URLs from a text file |
| `-u URL ...` | `--urls URL ...` | Pass one or more URLs directly |
| `-o DIR` | `--output DIR` | Set output directory (default: `./PS4_PKGs`) |
| `--reset` | | Clear saved state — re-downloads everything |
| `--setup-only` | | Install dependencies only, don't download |
| `-h` | `--help` | Show help and exit |

---

## Multi-Link Paste Handler

When run in interactive mode, BASDN presents a paste-aware prompt:

```
╔══════════════════════════════════════════════════════════════╗
║              MULTI-LINK PASTE HANDLER                        ║
╚══════════════════════════════════════════════════════════════╝

[0 links] Paste/type URL(s) or command:
```

**Paste methods supported:**

- **Single URL** — type or paste one URL and press Enter
- **Space-separated block** — paste all links on one line separated by spaces; BASDN splits and validates each one individually
- **Multi-line block** — paste multiple lines at once; each line is processed as it arrives
- **Mixed** — combine any of the above freely

**In-prompt commands:**

| Command | Action |
|---|---|
| `DONE` | Finish collecting and start downloading |
| `LIST` | Print all collected URLs so far |
| `CLEAR` | Remove all collected URLs and start over |
| `Ctrl+D` | Finish (same as `DONE`) |
| `Ctrl+C` | Cancel and exit |

BASDN validates each token against `http://` / `https://` / `ftp://` patterns. Non-URL tokens on their own line prompt a force-add confirmation. Non-URL tokens mixed inside a multi-URL paste are silently skipped.

---

## How Resume Works

BASDN uses the HTTP `Range` header standard (RFC 7233) to resume downloads:

1. On start, BASDN checks if a partial file exists on disk
2. If found, it sends `Range: bytes=<offset>-` in the request
3. The server responds with `206 Partial Content` and streams from the offset
4. If the server returns `200` instead (no resume support), BASDN restarts cleanly from zero without corrupting the existing file
5. Progress state (bytes downloaded, filename, total size) is saved to `.download_state.json` every 10 seconds

After any interruption — network drop, `Ctrl+C`, crash, power loss — simply re-run BASDN with the same arguments. It reads the state file and picks up where it left off.

---

## How Retry Works

Every download attempt is wrapped in a retry loop with **exponential backoff**:

| Attempt | Delay before retry |
|---|---|
| 1 | 5 seconds |
| 2 | 10 seconds |
| 3 | 20 seconds |
| 4 | 40 seconds |
| 5 | 80 seconds |
| 6–10 | 120 seconds (capped) |

On each retry:
- Current progress is saved to disk before sleeping
- The download resumes from the last saved byte offset
- The error is logged with the attempt number

After 10 failed attempts, the file is marked `"status": "failed"` in the state file and BASDN moves on to the next URL. Re-running the script will retry failed files (since they were never completed).

---

## Download State

BASDN writes a `.download_state.json` file inside the output directory. This file tracks every URL's progress:

```json
{
  "a1b2c3d4e5f6...": {
    "url": "https://example.com/bigfile.pkg",
    "filename": "bigfile.pkg",
    "total_size": 21474836480,
    "downloaded": 10737418240,
    "status": "in_progress",
    "last_updated": "2025-03-11T14:22:01"
  }
}
```

**Status values:**

| Status | Meaning |
|---|---|
| `in_progress` | Actively downloading or interrupted mid-download |
| `retrying` | Failed an attempt, waiting to retry |
| `done` | Fully downloaded and size-verified |
| `failed` | Exhausted all retries |

To force a completely fresh start (re-download everything):

```bash
./basdn --reset
```

---

## Output Structure

```
./PS4_PKGs/
├── .download_state.json     ← resume/state tracking (hidden)
├── .ps4pkg_deps_ok          ← dependency stamp (hidden)
├── .ps4pkg_downloader.py    ← auto-generated Python engine (hidden)
├── bigfile1.pkg
├── bigfile2.pkg
└── bigfile3.iso
```

Use `-o` to change the output directory:

```bash
./basdn -f links.txt -o /media/external/downloads
```

---

## Examples

**Download 12 files from a list:**

```bash
./basdn -f my_links.txt
```

**Paste links interactively and download to external drive:**

```bash
./basdn -o /mnt/usb/downloads
```

**Download two files directly:**

```bash
./basdn -u \
  https://example.com/game1.pkg \
  https://example.com/game2.pkg
```

**Fresh start — wipe state and re-download everything:**

```bash
./basdn -f links.txt --reset
```

**Setup only (useful for air-gapped prep):**

```bash
./basdn --setup-only
```

---

## Troubleshooting

**`externally-managed-environment` error**  
Handled automatically. BASDN detects PEP 668 environments and applies `--break-system-packages`. If it still fails, BASDN falls back to `--user` install, then virtualenv, then system package manager.

**Download stuck at 0 bytes**  
The server may not support `Range` requests. BASDN detects `200` vs `206` responses and restarts cleanly. If the server is rate-limiting, the retry backoff will space out attempts automatically.

**`pip not found`**  
BASDN auto-bootstraps pip via `get-pip.py` using `curl` or `wget`. If neither is available, run:
```bash
sudo apt install python3-pip
```

**Corrupt or size-mismatch file**  
BASDN compares the final file size against the server's `Content-Length`. If they differ, the file is flagged in the summary and marked `failed` in state — it will be retried on the next run.

**Resume not working after reboot**  
Make sure you run BASDN from the same directory as before, or pass the same `-o` output path so it finds `.download_state.json`.

---

## License

MIT License — see `LICENSE` for details.

---

*Built by `0xb0rn3` | `0xbv1` — [github.com/0xb0rn3/Basdn](https://github.com/0xb0rn3/Basdn)*
