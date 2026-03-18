# Shift+Enter Fix for psmux (VS Code + Claude Code)

## What was changed

### 1. psmux source (branch: `fixes/shift-enter`)

**`src/platform.rs`** — Added `augment_enter_shift` function
- Polls `GetAsyncKeyState(VK_SHIFT)` to detect physical Shift key
- Remaps crossterm's misreported Alt+Enter back to Shift+Enter
- Needed because VS Code's xterm.js sends `\x1b\r` for Shift+Enter, which ConPTY interprets as Alt

**`src/client.rs`** — Calls `augment_enter_shift` + sends modifier with Enter
- `Event::Key(mut key)` instead of `Event::Key(key)`
- Sends `send-key S-Enter` instead of `send-key enter`

**`src/input.rs`** — Three changes:
- Fixed `parse_modified_special_key` S- modifier bit calculation (was a no-op bug)
- Added Enter/Return/CR to `parse_modified_special_key` (non-Windows path)
- Added `\x1b\r` handler in `send_key_to_active` for Shift+Enter on Windows
  - Sends ESC+CR to ConPTY, matching what VS Code sends
  - Round-trip: `\x1b\r` -> ConPTY Alt+Enter -> libuv `\x1b\r` -> Claude Code sees Shift+Enter

### 2. PowerShell profile (`$PROFILE` for pwsh)

Location: `C:\OneDrive\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`

Added:
```powershell
# Map Alt+Enter to AddLine (newline) -- ConPTY delivers Shift+Enter as Alt+Enter
Set-PSReadLineKeyHandler -Chord 'Alt+Enter' -Function AddLine
```

PSReadLine needs this because the `\x1b\r` round-trip produces Alt+Enter in the console input buffer, not Shift+Enter.

## How to reapply after upstream updates

```powershell
cd C:\1-Git\psmux

# 1. Fetch latest upstream
git fetch upstream

# 2. Update master
git checkout master
git merge upstream/master

# 3. Rebase shift-enter fixes onto new master
git checkout fixes/shift-enter
git rebase master

# 4. If conflicts, resolve them, then: git rebase --continue

# 5. Build and deploy
cargo build --release
taskkill /F /IM psmux.exe
cp target/release/psmux.exe "$LOCALAPPDATA/Microsoft/WinGet/Packages/marlocarlo.psmux_Microsoft.Winget.Source_8wekyb3d8bbwe/psmux.exe"
```

## After a PC format

1. Clone the fork: `git clone https://github.com/ToxMox/psmux.git C:\1-Git\psmux`
2. Add upstream remote: `git remote add upstream https://github.com/marlocarlo/psmux.git`
3. Checkout the branch: `git checkout fixes/shift-enter`
4. Build and deploy (step 5 above)
5. Add the PSReadLine keybinding to your pwsh `$PROFILE` (see section 2 above)
