**English** 

# WinSpotlight

A keyboard-driven launcher for Windows 11 in the style of macOS Spotlight. Press **Win+Space**, type, press Enter.

[![Rust](https://img.shields.io/badge/Rust-1.82%2B-orange?logo=rust&logoColor=white)](https://www.rust-lang.org)
![Platform](https://img.shields.io/badge/platform-Windows_11-0078D4)
![Arch](https://img.shields.io/badge/arch-x64-informational)
![Version](https://img.shields.io/badge/version-0.1.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Dependencies](https://img.shields.io/badge/dependencies-1-brightgreen)
![Telemetry](https://img.shields.io/badge/telemetry-none-success)
[![GitHub](https://img.shields.io/badge/GitHub-ENC3ll%2Fwin—spotlight-181717?logo=github&logoColor=white)](https://github.com/ENC3LL/win-spotlight)

## Overview
<img width="788" height="108" alt="image" src="https://github.com/user-attachments/assets/701913a6-560d-4ce5-a83d-212daec4de5d" />

<img width="803" height="428" alt="image" src="https://github.com/user-attachments/assets/6a8e257c-d869-474e-a3f1-3dd43d39938b" />

SpotlightWin runs in the background, opens a search window on a global hotkey and launches what you pick. It finds desktop applications, Microsoft Store (UWP) apps, and games from Steam and the Epic Games Launcher, and evaluates arithmetic expressions. It is a single portable executable of about 450 KB with no installer and no configuration files.

Design goals: low resource use, no network access, no telemetry, no administrator rights, and a small attack surface suitable for software that starts with the user session.

## Features

- Global hotkey **Win+Space** (fallback: **Ctrl+Alt+Space**).
- Search across Start menu apps (including Store/UWP), desktop and Start menu shortcuts, Steam and Epic Games titles, and programs registered in the Uninstall registry section.
- Fuzzy matching: prefix, word prefix, substring and subsequence.
- Calculator: `+ - * / × ÷ ^`, parentheses, decimal point or comma.
- Application icons loaded in the background.
- Glass-style interface with an animated expansion from a compact search field to a result panel.
- Interface languages: English and Russian, chosen from the Windows display language.
- Light and dark themes.

## How it works

1. **Hotkey.** Windows reserves Win+Space for switching the input language, so it cannot be registered with `RegisterHotKey`. A low-level keyboard hook (`WH_KEYBOARD_LL`) runs in its own thread, so slow UI work cannot stall keyboard input system-wide. The hook only inspects the Space key. When Win+Space is detected it consumes the key, injects a neutral key so releasing Win does not open the Start menu, and posts a single toggle message to the UI thread. Repeated presses are coalesced.
2. **Index.** A background thread builds the list of launchable items from the sources listed below. The index is refreshed in the background when it is older than five minutes (checked each time the window opens). Opening the window never waits for a scan.
3. **Window.** The window is a layered window with per-pixel alpha. When it opens, the area behind it is captured, blurred and tinted to produce the glass effect, then the panel shape, border and shadow are rendered in software. The height follows a damped spring, so the panel expands with a slight overshoot as results appear and collapses when the query is cleared. Capture and render buffers exist only while the window is open.
4. **Icons.** Three worker threads load icons through the shell's image factory with retries. At most 48 icons are cached.
5. **Launching.** The selected item is launched from a short-lived thread, and the window closes immediately.

Threads: UI, keyboard hook, scanner, three icon workers, watchdog, and a launcher thread that exists only while a launch is in progress.

### Search sources

| Source | Covers |
|---|---|
| `shell:AppsFolder` | Everything shown in the Start menu app list, including Store/UWP apps |
| Shortcuts | `.lnk` and `.url` on the user and public desktop; `.lnk` in the Start menu folders |
| Epic Games manifests | Installed Epic games, launched through the launcher URI |
| Uninstall registry keys | Programs with a valid `DisplayIcon` executable; Steam games (`Steam App <id>`) |

Uninstaller entries are filtered out. When several sources provide the same name, the first source in the order above wins.

## Installation

SpotlightWin is portable. Put `SpotlightWin.exe` in a permanent folder and run it.

Recommended location: `C:\Program Files\SpotlightWin\`. Copying there requires administrator rights, which also means ordinary processes of your user account cannot replace the executable. `%LOCALAPPDATA%\Programs\SpotlightWin\` works without elevation but offers weaker protection.

The binary is not code-signed. Windows SmartScreen may show an "unknown publisher" warning, and some antivirus products treat any process with a global keyboard hook with suspicion. Verify the file before running it:

```powershell
Get-FileHash .\SpotlightWin.exe -Algorithm SHA256
```

Compare the result with the hash published with the release.

### Building from source

Requirements: Rust (the minimum version of 1.82 is set by the `windows` crate, the published binary was built with 1.100.0) and, for the default MSVC target, the Visual Studio Build Tools.

```powershell
cargo build --release
```

`.cargo/config.toml` selects the `x86_64-pc-windows-msvc` target, so the output is `target\x86_64-pc-windows-msvc\release\spotlight-win.exe`. The configuration enables a static CRT, Control Flow Guard and CET compatibility. Note that the published binary was cross-compiled for `x86_64-pc-windows-gnu` and does not include those two hardening flags; the MSVC configuration has not been tested by the author.

## Usage

| Key | Action |
|---|---|
| Win+Space | Open or close the window |
| Typing | Search; the panel expands when there are results |
| Up / Down | Move the selection |
| Enter | Launch the selected item |
| Esc | Clear the query; press again to close |
| Ctrl+Backspace | Clear the query |
| Mouse | Hover to select, click to launch |

The window also closes when it loses focus. It opens on the monitor under the mouse cursor. The calculator row is informational: Enter on it does nothing.

## Starting, stopping and autostart

All control is through command-line arguments. Run these in PowerShell or a command prompt, using your own path.

| Command | Effect |
|---|---|
| `SpotlightWin.exe` | Start in the background. Does nothing if an instance is already running |
| `SpotlightWin.exe --quit` | Stop the running instance |
| `SpotlightWin.exe --install` | Add to autostart for the current user |
| `SpotlightWin.exe --uninstall` | Remove from autostart |

Example for a fresh setup:

```powershell
& "C:\Program Files\SpotlightWin\SpotlightWin.exe" --install --theme=light --lang=auto
& "C:\Program Files\SpotlightWin\SpotlightWin.exe"
```

Notes:

- `--install` does not start the program; it takes effect at the next sign-in. Start it manually the first time.
- Autostart is a single value `SpotlightWin` under `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`. No administrator rights are needed. It also appears in Task Manager under Startup apps.
- The registered path is the location of the executable when `--install` was run. If you move the file, run `--install` again.
- The path is stored in quotes, which prevents unquoted-path hijacking.
- To change options, run `--quit`, then `--install` with the new options, then start the program again. Options passed to a second instance are ignored while the first is running.

To remove SpotlightWin completely: `--quit`, `--uninstall`, delete the executable. The program writes nothing else to disk or to the registry.

### Options

| Option | Values | Default |
|---|---|---|
| `--theme=` | `light`, `dark`, `auto` (follow Windows) | `light` |
| `--lang=` | `en`, `ru`, `auto` (follow the Windows display language) | `auto` |

With `auto`, Russian is used when the Windows display language is Russian, and English otherwise. Options given together with `--install` are saved in the autostart entry.

## Security and privacy

- No network code, no telemetry, no logging, no configuration files.
- Runs as the current user. No elevation is requested.
- Only items produced by the program's own scanner can be launched, and each is validated again before launch: executables, `.lnk` and `.url` files must have an allowed extension and an absolute path without `..`; protocol links are accepted only in the exact Steam and Epic Games formats. The text typed into the search box never becomes part of a command line.
- A low-level keyboard hook is delivered every keyboard event by design. This program inspects only the Space key and passes every other event through untouched. Nothing is stored or transmitted. The source is short enough to audit: the hook is `kbd_hook` in `src/main.rs`.
- The DLL search path is restricted to System32.
- Only one instance runs per session.
- A watchdog thread checks private memory, GDI objects and handles every 30 seconds and terminates the process (exit code 3) if they exceed 150 MB, 400 objects and 2000 handles. Normal usage is far below these limits. A watchdog exit indicates a bug and should be reported.
- Input limits: queries up to 256 characters, expressions up to 128 characters with nesting depth up to 64, and caps on index size and directory depth.

## Limitations

- Win+Space no longer switches the input language while the program runs. Use Alt+Shift or Ctrl+Shift.
- The keyboard hook cannot see keystrokes while an elevated (administrator) window has focus. Ctrl+Alt+Space is provided as a fallback; its behavior over elevated windows has not been verified.
- The glass blur is a snapshot taken when the window opens, not a live effect.
- The hotkey is not configurable.
- It searches applications and games, not files.
- No tray icon.
- Interface fonts (Segoe UI Variable, Segoe Fluent Icons) are present on Windows 11 only.
- The program has not been tested on every Windows configuration. Please report issues with your Windows build and DPI scaling.

## Troubleshooting

| Symptom | What to check |
|---|---|
| Nothing happens on Win+Space | Confirm `SpotlightWin.exe` is in Task Manager. Try Ctrl+Alt+Space. Close other tools that hook Win+Space (launchers, keyboard utilities) |
| A game or app is missing | Wait a few seconds after start for the first scan. Games from Epic need to be fully installed. Report the install method of the missing item |
| An icon stays as a letter | The shell could not provide it; the program retries after a few seconds |
| The program exits on its own | Likely the watchdog (exit code 3). Please open an issue |
| Wrong language | Use `--lang=en` or `--lang=ru`, then `--install` again for autostart |

## Development

```powershell
cargo test
```

Unit tests cover the platform-independent modules: calculator, index and launch validation, animation, renderer and localization. Memory-growth assertions read `/proc` and are meaningful on Linux only.

| Module | Role |
|---|---|
| `main.rs` | Window, state, input, hook, watchdog, autostart |
| `scan.rs` | Background scanning of all sources (Windows only) |
| `icons.rs` | Background icon loading (Windows only) |
| `index.rs` | Index, search, target validation, shortcut walking |
| `render.rs` | Glass rendering, shape, shadow |
| `anim.rs` | Spring animation |
| `calc.rs` | Expression evaluator |
| `lang.rs` | Localization |

## License

MIT, as declared
