# BitChord for Windows — Build Plan

> Goal: a Windows desktop music player, reusing the Android app's code, matching its look
> and feel, with real Windows integration (media keys, tray, taskbar/volume flyout).

---

## 1. The finding that changes the plan

Thanks to a prior developer, **the Windows app already exists — it just isn't on `main`.**

| Fact | Detail |
|---|---|
| Repo | `github.com/kushagrasinghx/BitChord` (GPL-3.0) |
| `main` today | v1.9.1 — Android phone + TV + WearOS, plus KMP `shared` / `sharedUi` |
| `main` has no desktop module | Removed on **16 Oct 2026**: commit `e4dd63d8 "chore: remove the desktop app now that the web player replaces it"` |
| Where the desktop app lives | Tag **`v1.8`** (commit `32f4e7c5`, 5 Oct 2026) and branch **`desktop-build-30`** — same commit |
| Already released | v1.8 shipped `BitChord-1.8-beta1-windows-x64-setup.exe`, `.msi`, and a portable `.zip` |
| Size of the module | `desktopApp/` — 116 Kotlin files, ~31k LOC, 52 test files |

This is a **real, complete Compose Multiplatform desktop app**, not a stub. It already has:

- The phone's player screen, lyrics animation, queue, Home / Explore / Library / Search
- **Native Windows**: `bitchord_smtc.dll` (media keys + volume-flyout media card), `bitchord_window.dll` (Mica/Acrylic, native drag), tray icon, output-device picker, DPAPI crypto
- YouTube playback via the same InnerTubeX + BotGuard code (JavaFX WebKit replaces Android WebView)
- FFmpeg audio decoding (JavaCPP), Automix, Listen Together, downloads, scrobbling, Discord RPC
- MSI/EXE packaging via Compose Desktop's `jpackage`

**Conclusion:** do not port from scratch. Revive, build, and polish the existing module.

---

## 2. What is reused from Android, and what was replaced

| Android original | Windows counterpart | Reused? |
|---|---|---|
| `shared/` (InnerTube, YT Music, lyrics, addons, models) | same module, `jvm` target | ✅ same code |
| `sharedUi/` (player, lyrics, queue, pages) | same module, `jvm` target | ✅ same Compose code |
| Media3 / ExoPlayer | `DesktopAudioEngine` + FFmpeg + `DesktopAudioSink` | ❌ reimplemented |
| `MediaSession` / notification | `DesktopWindowsMedia` (SMTC) / `DesktopMpris` (Linux) | ❌ reimplemented |
| DataStore / SharedPreferences | `DesktopPersistence` | ❌ reimplemented |
| Android WebView (PoToken) | `DesktopPoTokenWebView` (JavaFX WebKit) | ❌ reimplemented |
| Foreground service | Tray + window lifecycle | ❌ reimplemented |

So "reuse the Android code" is already the architecture — the desktop module is a thin,
OS-specific shell around `shared` + `sharedUi`.

---

## 3. Choose a route

| | Route | Effort | What you get | Risk |
|---|---|---|---|---|
| **A** | Install the shipped v1.8 beta | ~15 min | A working Windows player today, zero code | None — but you don't own the build |
| **B** | Build `desktopApp` from tag `v1.8` | **1–2 days** | Same app, built by you, editable | Toolchain setup; a few known gaps |
| **C** | Rebase `desktopApp` onto `main` (v1.9.1) | 1–3 weeks | Desktop + all 1.9.x features | `main` refactored `shared`/`sharedUi` after removal; no upstream support |

### Recommendation

1. **Do A first** — confirms the app runs and the machine is fine, in 15 minutes.
2. **Then B** — make the source yours. This plan focuses on B.
3. **Only consider C** if you specifically need v1.9.1 features (see Phase 8).

---

## 4. Phase plan (Route B)

### Phase 0 — Validate the shipped build (30 min)

- [ ] Download `BitChord-1.8-beta1-windows-x64-setup.exe` (or the portable zip) from the v1.8 release
- [ ] Install / run it
- [ ] Check: playback, search, lyrics, media keys, tray, volume-flyout media card
- [ ] Note anything broken — that becomes your Phase 7 list

> This also proves the app is worth reviving before you spend a day on toolchain setup.

### Phase 1 — Toolchain

Current machine state:

| Tool | Status | Action |
|---|---|---|
| Git | ✅ present | — |
| MinGW `g++` | ✅ `C:\mingw64\bin` | — |
| Android SDK | ✅ `C:\Users\PARDHU\AppData\Local\Android\Sdk` | — |
| JDK | ⚠️ only **24** (`C:\Program Files\Java\jdk-24`) — build targets **21** | install JDK 21 |
| CMake | ❌ missing | install |
| Ninja | ❌ missing | install |
| WiX Toolset 3.x | ❌ missing | needed only for `.msi` / `.exe` |
| Gradle | ✅ via `gradlew.bat` (8.11.1) | — |

- [ ] Install **JDK 21** (Temurin 21) and set `JAVA_HOME` to it for builds
- [ ] `winget install Kitware.CMake`
- [ ] `winget install Ninja-build.Ninja`
- [ ] Optional (installer): `winget install WiXToolset.WiXToolset`
- [ ] Confirm: `cmake --version`, `ninja --version`, `g++ --version`, `java -version`
- [ ] If a Gradle daemon was started before installing these: `gradlew.bat --stop`

### Phase 2 — Get the source into this folder

```powershell
cd C:\Users\PARDHU\fun\music
git remote add origin https://github.com/kushagrasinghx/BitChord.git
git fetch --depth 1 origin tag v1.8
git checkout -b windows-desktop v1.8
```

- [ ] Source checked out at `v1.8`
- [ ] Confirm `settings.gradle.kts` includes `:desktopApp`
- [ ] Commit the current state as your baseline

### Phase 3 — First run (development mode)

```powershell
.\gradlew.bat :desktopApp:run
```

- [ ] Window opens, audio plays
- [ ] If CMake/Ninja/g++ are missing, the build degrades gracefully (no Automix analyser) —
      check the log for `no usable cmake` / `Automix will not run`
- [ ] `:desktopApp:test` passes

### Phase 4 — Portable build (no installer, no WiX)

```powershell
.\gradlew.bat :desktopApp:windowsPortable
```

- [ ] Output: `desktopApp\build\distributions\BitChord-1.8-beta1-windows-portable.zip`
- [ ] The task self-checks the archive: Compose runtime, Windows Skiko, Windows FFmpeg,
      and whether `bitchord_analysis.dll` made it in
- [ ] Launch on a clean folder to confirm it isn't depending on your dev environment

### Phase 5 — Installer build

```powershell
.\gradlew.bat :desktopApp:packageMsi
.\gradlew.bat :desktopApp:packageExe
```

- [ ] Output in `desktopApp\build\compose\binaries\main\`
- [ ] Bump `desktopVersionCode` (currently hardcoded **30**) before each release build
- [ ] Note: unsigned → Windows SmartScreen warning

### Phase 6 — Verification checklist

- [ ] Search and play from YouTube Music
- [ ] Lyrics sync and animation
- [ ] Queue: reorder, shuffle, repeat
- [ ] **Media keys** (play/pause/next/prev) and volume-flyout media card
- [ ] Tray icon + tray menu; close-to-tray behaviour
- [ ] Mica/Acrylic window, drag, snap, maximize
- [ ] Output-device picker
- [ ] Local library import (folder scan, tags, artwork)
- [ ] Downloads
- [ ] Sign-in (browser cookie import / YouTube sign-in)
- [ ] Sleep timer, crossfade, equalizer
- [ ] Discord RPC / scrobbling (needs optional local keys)
- [ ] Listen Together (needs server URL)

### Phase 7 — Known gaps to expect

| Area | Issue | Fix |
|---|---|---|
| Media keys | `bitchord_smtc.dll` needs the WinRT Media headers; MinGW path (`smtc_abi.cpp`) may skip it | Install VS Build Tools and build with MSVC, or keep MinGW and verify the DLL loads |
| SmartScreen | No code signing | Buy a code-signing cert, or ship portable-only |
| Automix | Analyser DLL only builds with CMake + Ninja + MinGW/MSVC | Phase 1 |
| Config keys | `.msi` build reads `local.properties` for `LASTFM_API_KEY`, `LASTFM_SECRET`, `LISTEN_TOGETHER_SERVER`, `MODULE_INDEX_URL` | Add them, or those features stay off |
| Version | `desktopVersionCode = 30` is manual | Automate from git tag |

### Phase 8 — Optional: forward-port to `main` (v1.9.1)

Only if you need post-1.8 features. This is a real port, not a merge:

1. Branch from `main`, add `desktopApp/` from `v1.8`
2. Fix `shared` / `sharedUi` API drift (main already moved them to KMP with `jvm` targets —
   the direction is right, but signatures changed)
3. Re-run `:desktopApp:test` and reconcile the 52 parity tests
4. Rebuild the two native DLLs (`smtc`, `window`) against the new tree
5. Re-verify Phase 6 end to end

**Estimate: 1–3 weeks.** Keep `v1.8` as the shipping base until this passes Phase 6.

### Phase 9 — Distribution hygiene

- [ ] GPL-3.0: publish your source, keep the license and attribution
- [ ] BitChord is an unofficial YouTube Music client — no affiliation; distribution carries
      the same ToS risk the Android build does
- [ ] Add a desktop CI workflow (the branch has none — v1.8 installers were built by hand).
      GitHub Actions `windows-latest` + JDK 21 + CMake/Ninja/MinGW + WiX is enough
- [ ] Code-sign the installer if distributing publicly

---

## 5. Recommended next step

Run **Phase 0**, then **Phase 1 + 2 + 3**. If `:desktopApp:run` opens and plays, you have
a working, editable Windows player — and the rest is polish.

If you'd rather not wait on toolchain setup, Route A gets you the app in 15 minutes.

---

## Appendix — exact commands from a clean start

```powershell
# 1. toolchain
winget install EclipseAdoptium.Temurin.21.JDK
winget install Kitware.CMake
winget install Ninja-build.Ninja
# optional, for .msi/.exe
winget install WiXToolset.WiXToolset

# 2. source
cd C:\Users\PARDHU\fun\music
git remote add origin https://github.com/kushagrasinghx/BitChord.git
git fetch --depth 1 origin tag v1.8
git checkout -b windows-desktop v1.8

# 3. run
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-21..."
.\gradlew.bat --stop
.\gradlew.bat :desktopApp:run

# 4. package
.\gradlew.bat :desktopApp:windowsPortable
.\gradlew.bat :desktopApp:packageMsi
```