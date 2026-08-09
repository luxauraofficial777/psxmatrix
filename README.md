# psxmatrix
PSXMATRIX HYRDA LOOP

# PSX Meta-Harness — Usage Guide for Builder Agents

**Date:** August 4, 2026  
**Location:** `DQLOSTTRANSLATION/PSXMatrix/meta_harness.py`

---

## What This Tool Does

The meta-harness automates a **9-test matrix** (3 emulators x 3 BIOS modes) for the Frankenstein disc. It:

- Launches each emulator with each BIOS configuration systematically
- Auto-configures DuckStation ini and StarPSX config.toml per BIOS mode (no manual edits)
- Streams live console output to your terminal while saving to log files
- Enforces a strict timeout (default 60s) to catch hangs and infinite loops
- Generates structured `_telemetry.json` per test + a master `matrix_summary.json`
- Supports `--fail-fast`, `--no-color`, `--tag` for agent automation

This replaces manual one-off emulator test runs with a single command.

---

## Quick Start

```powershell
cd C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\PSXMatrix

# Full 9-test matrix (all emulators, all BIOS modes, Frankenstein disc)
python meta_harness.py --clean --no-color

# Preview commands without executing
python meta_harness.py --dry-run --no-color

# Smoke test with DW7 disc (all emulators, all BIOS, short timeout)
python meta_harness.py --smoke-test --clean --no-color

# Run only Cybergrime + VHB BIOS
python meta_harness.py --cybergrime-only --vhb-only --clean --no-color

# Run only DuckStation tests
python meta_harness.py --duckstation-only --clean --no-color

# Run only StarPSX tests
python meta_harness.py --starpsx-only --clean --no-color

# Unlimited live output (don't truncate)
python meta_harness.py --max-output-lines 0

# Status badges only, no live emulator output
python meta_harness.py --quiet --clean --no-color

# Fail-fast: stop on first non-PASS result
python meta_harness.py --fail-fast --clean --no-color --tag vhb_rebuild_v3
```

---

## Emulators

| Emulator | Type | BIOS Config | Disc Format |
|---|---|---|---|
| **Cybergrime** | CLI (headless) | `--bios` CLI flag | `.bin` (auto-converted from `.cue`) |
| **DuckStation** | CLI (`-batch -nogui`) | `duckstation-qt.ini` (auto-written) | `.cue` |
| **StarPSX** | GUI (egui, `-a` auto-run) | `%APPDATA%/StarPSX/config.toml` (auto-written) | `.cue` |

All paths are pre-configured with absolute paths. No manual config edits required.

---

## BIOS Modes

| Mode | Description | BIOS File |
|---|---|---|
| `hle` | HLE / fastboot (no BIOS file) | None (Cybergrime/DuckStation) / SCPH fallback (StarPSX) |
| `vhb` | VHB Super BIOS | `VHB_SUPER_BIOS_V1.00A/FRANKENSTEIN.BIOS` |
| `scph` | Sony SCPH-1001 (US) | `bios/US/SCPH1001.BIN` (MD5: `924e392ed05558ffdb115408c263dccf`) |

---

## Discs

| Disc | Default? | CUE Path |
|---|---|---|
| `frankenstein` | **Yes** (default) | `build_staging/dq4_frankenstein_v99.cue` |
| `dw7` | Smoke test only | `DW7D1/DW7D1.cue` |

Use `--disc dw7` to add DW7, or `--smoke-test` to use DW7 with short timeout.

---

## CLI Reference

### Core Flags

| Flag | Default | Description |
|---|---|---|
| `--timeout N` | `60` | Per-test timeout in seconds |
| `--disc NAME` | `frankenstein` | Disc to test (repeatable: `--disc frankenstein --disc dw7`) |
| `--iso PATH` | (none) | Explicit disc .cue path (overrides --disc) |
| `--vhb-bios PATH` | `VHB_SUPER_BIOS_V1.00A/FRANKENSTEIN.BIOS` | VHB Super BIOS file |
| `--scph-bios PATH` | `bios/US/SCPH1001.BIN` | SCPH-1001 BIOS file |
| `--log-dir PATH` | `PSXMatrix/matrix_logs` | Where to write per-test logs |
| `--max-instr N` | `5000000` | Cybergrime instruction cap |
| `--max-output-lines N` | `50` | Cap live terminal output per test (`0` = unlimited) |
| `--dry-run` | — | Print all commands, don't execute |
| `--clean` | — | Wipe `matrix_logs/` before running (resilient to locked files) |
| `--quiet` | — | No live emulator output, just status badges |

### Agent Automation Flags

| Flag | Description |
|---|---|
| `--no-color` | Disable ANSI color output (for piped/agent use) |
| `--fail-fast` | Stop on first FAILED or TIMEOUT_HANG test |
| `--tag STRING` | Tag for this matrix run (embedded in summary JSON) |
| `--smoke-test` | Quick validation: DW7 + all emulators + all BIOS + 30s timeout + 1M instrs |
| `--report` | Generate BIOS-side and ROM-side reports from telemetry after run |
| `--wallclock-ms N` | Wall clock limit in ms for Cybergrime (0=no limit) |

### Per-Emulator Shortcut Flags

| Flag | Equivalent |
|---|---|
| `--cybergrime-only` | `--emulator cybergrime` |
| `--duckstation-only` | `--emulator duckstation` |
| `--starpsx-only` | `--emulator starpsx` |

### Per-BIOS Shortcut Flags

| Flag | Equivalent |
|---|---|
| `--hle-only` | `--bios-mode hle` |
| `--vhb-only` | `--bios-mode vhb` |
| `--scph-only` | `--bios-mode scph` |

---

## Test IDs

Each test has a deterministic ID: `{disc}__{emulator}__{bios_mode}`

Example: `frankenstein__cybergrime__vhb`

---

## Reading Results

### Per-Test Telemetry (`matrix_logs/{test_id}_telemetry.json`)

```json
{
  "test_id": "frankenstein__duckstation__vhb",
  "emulator": "duckstation",
  "bios_mode": "vhb",
  "status": "PASSED|FAILED|TIMEOUT_HANG",
  "exit_code": 0,
  "duration_seconds": 28.123,
  "stdout_line_count": 42,
  "stderr_line_count": 0,
  "stdout_file": "matrix_logs/frankenstein__duckstation__vhb_stdout.txt",
  "stderr_file": "matrix_logs/frankenstein__duckstation__vhb_stderr.txt",
  "command": ["duckstation-qt-x64-ReleaseLTCG.exe", "-batch", "-nogui", "-slowboot", "disc.cue"],
  "error_message": null
}
```

### Master Summary (`matrix_summary.json`)

```json
{
  "matrix_run_id": "20260804_100900",
  "tag": "vhb_rebuild_v3",
  "total_duration_seconds": 270.5,
  "configuration": {
    "timeout_seconds": 60,
    "discs": ["frankenstein"],
    "vhb_bios_path": "...",
    "scph_bios_path": "...",
    "max_instr": 5000000,
    "emulators": ["cybergrime", "duckstation", "starpsx"],
    "bios_modes": ["hle", "vhb", "scph"]
  },
  "summary": {
    "total_tests": 9,
    "passed": 3,
    "failed": 4,
    "timeout_hang": 2
  },
  "results": [ ... ]
}
```

### Status Meanings

| Status | Meaning |
|---|---|
| `PASSED` | Process exited with code 0 within timeout (Cybergrime exit 1 = boot_fail, also PASSED) |
| `FAILED` | Process exited with non-zero code, or executable not found |
| `TIMEOUT_HANG` | Process did not exit within timeout — killed |

---

## Generating BIOS-Side and ROM-Side Reports

After a matrix run, telemetry JSON files are in `matrix_logs/`. To generate reports:

### BIOS-Side Report (VHB BIOS validation)

Compare VHB vs SCPH vs HLE across all emulators:

```powershell
# Run VHB-only tests across all emulators
python meta_harness.py --vhb-only --clean --no-color --tag vhb_v3_validation

# Then check telemetry for each emulator
# Key indicators in stdout:
#   - StarPSX: [TTY]= lines show BIOS boot sequence
#   - DuckStation: FPS > 0 means game is running
#   - Cybergrime: VBlank count, BIOS call log, Pass_Status in telemetry JSON
```

### ROM-Side Report (Frankenstein disc validation)

Test the Frankenstein disc. 
> [!WARNING]
> **CyberGrime HLE Conflict:** The Frankenstein disc uses a custom `PC0` detour (`0x800BDF00`) for MISS-1 injection. CyberGrime's internal `load_thread_blob()` HLE hook conflicts with this and will hang. **Use DuckStation exclusively** for Frankenstein boot validation.

```powershell
# DuckStation Matrix with SCPH-1001 BIOS (Source of Truth)
python meta_harness.py --duckstation-only --scph-only --no-color --tag frank_scph_duckstation

# StarPSX fallback validation
python meta_harness.py --starpsx-only --scph-only --no-color --tag frank_scph_starpsx
```

### Key Telemetry Indicators

| Indicator | Where to Look | Meaning |
|---|---|---|
| BIOS boot sequence | StarPSX stdout `[TTY]=` lines | BIOS initialized, SYSTEM.CNF parsed, EXE loaded |
| FPS > 0 | DuckStation stdout | Game is running, not stalled |
| VBlank count | Cybergrime telemetry JSON | Game loop is executing |
| CD-ROM reads | Cybergrime telemetry JSON, StarPSX stdout | Game is reading disc data |
| Pass_Status | Cybergrime telemetry JSON | `BOOT_OK` = EXE found, `BOOT_FAIL` = no EXE |
| CD-ROM command crash | StarPSX stderr | `not implemented: cdrom command XX` = emulator limitation |

---

## Agent Workflow: Parallel BIOS + ROM Edits

When making VHB BIOS edits and ROM edits in parallel:

1. **Build VHB BIOS** → place at `VHB_SUPER_BIOS_V1.00A/FRANKENSTEIN.BIOS`
2. **Build Frankenstein disc** → place at `frankenstein_pipeline/output/dq4_frankenstein.cue`
3. **Run validation matrix:**
   ```powershell
   python meta_harness.py --clean --no-color --fail-fast --tag parallel_build_$(Get-Date -Format yyyyMMdd_HHmm)
   ```
4. **Check results:**
   ```powershell
   # View summary
   Get-Content matrix_summary.json | ConvertFrom-Json | Select-Object -ExpandProperty summary
   
   # Check specific test
   Get-Content matrix_logs/frankenstein__starpsx__scph_stdout.txt
   Get-Content matrix_logs/frankenstein__cybergrime__vhb_cybergrime_telemetry.json
   ```
5. **If fail-fast triggered**, fix the issue and re-run only the failed combination:
   ```powershell
   python meta_harness.py --starpsx-only --vhb-only --no-color --tag fix_vhb_starpsx
   ```

---

## Auto-Configuration Details

### DuckStation (ini-driven)

The harness writes `duckstation-qt.ini` to **both** locations before each test:
- `duckstation/duckstation-qt.ini` (root)
- `duckstation/settings/duckstation-qt.ini` (settings subdir)

Settings written per BIOS mode:
- `BIOSPath` = VHB/SCPH path (or empty for HLE)
- `FastBoot` = true for HLE, false for VHB/SCPH
- `Enable8MBRAM` = false
- `Region` = NTSC-U
- `LogFileDir` / `LogFileName` = matrix_logs path

### StarPSX (config.toml-driven)

The harness writes `%APPDATA%/StarPSX/config.toml` before each test:
- `bios_path` = VHB/SCPH path (SCPH for HLE mode since StarPSX has no HLE)

### Cybergrime (CLI-driven)

No config files — all settings via CLI flags:
- `--bios PATH` for VHB/SCPH modes (omitted for HLE)
- `--verbose` for detailed logging
- Disc path auto-converted from `.cue` to `.bin`

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| StarPSX crashes with `not implemented: cdrom command XX` | Emulator limitation | Check psx-spx docs, patch `core/src/cdrom/mod.rs` |
| DuckStation no output | Running in background | Use `-batch -nogui` flags (harness handles this) |
| Cybergrime `BOOT_FAIL` | No SYSTEM.CNF / EXE in disc | Check disc image is valid |
| `TIMEOUT_HANG` for GUI emulator | Window didn't close | Increase `--timeout` or close window manually |
| Locked files on `--clean` | Emulator still running | Harness uses resilient rmtree, but kill lingering processes first |

---

## File Paths (All Absolute, Pre-configured)

```
Harness:     C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\PSXMatrix\meta_harness.py
Logs:        C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\PSXMatrix\matrix_logs\
Summary:     C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\PSXMatrix\matrix_summary.json

Cybergrime:  C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\cybergrime\psx_agent_runner.exe
DuckStation: C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\duckstation\duckstation-qt-x64-ReleaseLTCG.exe
StarPSX:     C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\study\starpsx\target\release\starpsx.exe

Frankenstein: C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\build_staging\dq4_frankenstein_v99.cue
DW7:          C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\DW7D1\DW7D1.cue
VHB BIOS:     C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\VHB_SUPER_BIOS_V1.00A\FRANKENSTEIN.BIOS
SCPH BIOS:    C:\LuxAura\VoidWalkers_Project\DQLOSTTRANSLATION\bios\US\SCPH1001.BIN
```

