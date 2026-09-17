# Platform Compatibility Checklist

> The Template must work on macOS and Linux. Check this list before committing.

## Windows

| Launch method | Status | Comment |
|---|---|---|
| WSL2 | Full support | This is Linux — follow the same path as the standard Linux checklist below |
| Git Bash | Best-effort, no guarantees | Hybrid environment: `bash`/GNU utilities from MSYS, but `python3` typically resolves to the native Windows interpreter, which does not understand MSYS paths (e.g., `/c/Users/.../IWE`) — known limitation (issue #799), will not be patched individually |
| Native Windows (cmd/PowerShell without Git Bash) | Not supported | Scripts are bash-only, no alternative |

Known issue: `update.sh` fails in Git Bash at the step where `python3` receives an MSYS path (issue #799). Workaround: use WSL2 to run `update.sh` and other Template scripts, even if you edit files via Git Bash/VS Code on Windows.

## Forbidden Constructs (without a wrapper)

| Construct | Problem | Replacement |
|-----------|---------|-------------|
| `sed -i '' ...` | GNU sed does not accept `''` | `sed_inplace` (defined in setup.sh, update.sh) |
| `date -v-Nd` | BSD-only (macOS) | `portable_date_offset N` (defined in role scripts) |
| `osascript` | macOS-only | `osascript \|\| notify-send \|\| true` |
| `launchctl` | macOS-only | Wrap in a `command -v launchctl` guard |
| `readlink -f` | BSD readlink does not support `-f` | `cd "$(dirname "$0")" && pwd` |
| `grep -P` | GNU-only (Perl regex) | `grep -E` (Extended regex) |
| `stat -c` / `stat -f` | GNU vs BSD | Avoid; use `wc`, `ls -l`, `find` |
| `mktemp -d -t` | Inconsistent behavior | `mktemp -d` (no template) |

## Wrappers (copy-paste to the top of the script)

### sed_inplace

```bash
if sed --version >/dev/null 2>&1; then
    sed_inplace() { sed -i "$@"; }
else
    sed_inplace() { sed -i '' "$@"; }
fi
```

### portable_date_offset

```bash
# portable_date_offset <days_back> [format]
portable_date_offset() {
    local days="$1"
    local fmt="${2:-%Y-%m-%d}"
    date -v-${days}d +"$fmt" 2>/dev/null || date -d "$days days ago" +"$fmt" 2>/dev/null
}
```

### notify (desktop)

```bash
notify() {
    local title="$1" message="$2"
    printf 'display notification "%s" with title "%s"' "$message" "$title" | osascript 2>/dev/null \
        || notify-send "$title" "$message" 2>/dev/null \
        || true
}
```

## Architectural Constraints

- **launchd / .plist** — macOS-only. On Linux, use cron or a systemd timer. Setup.sh skips step 5 on Linux.
- **~/Library/LaunchAgents** — macOS path. Role install scripts are currently macOS-only.
- **/opt/homebrew/bin** — Apple Silicon macOS. Substituted into the plist PATH by the Template, but not universal.
- **Sleep prevention** — scripts detect the OS automatically: `caffeinate -diu` (macOS) / `systemd-inhibit` (Linux). On macOS, the `-s` flag is **not used** — it is ignored when Optimized Battery Charging switches the power profile to battery.
- **Laptop wake** — macOS: `pmset repeat wakeorpoweron`, Linux: `rtcwake` / systemd timer `WakeSystem=true`, Windows: Task Scheduler. For macOS laptops, `pmset -b sleep 0` is recommended (disables idle sleep on the battery profile).
- **`scripts/kimi-whisper-safe.sh`** — optional dependencies `ffmpeg` + `openai-whisper` (Python package) are not included in the base Template installation. The script checks for `ffprobe`/`whisper` in PATH and exits with a clear error if they are missing — install as needed.

## How to Check

```bash
# Find all potential issues:
grep -rn "sed -i ''" --include="*.sh" .
grep -rn "date -v" --include="*.sh" .
grep -rn "osascript" --include="*.sh" .
grep -rn "launchctl" --include="*.sh" .
grep -rn "readlink -f" --include="*.sh" .
grep -rn "grep -P" --include="*.sh" .
```

---

*Last updated: 2026-03-16*