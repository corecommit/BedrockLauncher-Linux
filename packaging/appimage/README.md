# BedrockLauncher Linux AppImage

This directory contains everything needed to package the launcher as a Linux
AppImage, plus the GitHub Actions workflow that builds it
(`.github/workflows/linux-appimage.yml`).

## What this actually is

BedrockLauncher is a **native Windows WPF application** (`net8.0-windows`).
It cannot run natively on Linux — WPF has no Linux implementation, and the
launcher's core job (registering/launching Minecraft for Windows GDK packages
through Windows Gaming Services, the Xbox app and the TokenBroker Win32 DLL)
is deeply Windows-only.

The AppImage therefore works as a **Wine bridge**: it bundles the Windows
build and runs it through the system's Wine installation, inside a dedicated,
isolated Wine prefix:

```
AppImage
├── AppRun                         # finds Wine, bootstraps the prefix, launches
├── bedrocklauncher.desktop
├── bedrocklauncher.png            # extracted from the app's own .ico
└── usr/
    ├── bin/                       # the Windows (win-x64) publish output
    │   ├── BedrockLauncher.exe
    │   ├── native/                # launchercore / gamelaunchhelper
    │   └── Runtimes/              # prebuilt TokenBroker (win-x86 / win-x64)
    └── share/dotnet/
        └── windowsdesktop-runtime-win-x64.exe   # .NET 8 Desktop Runtime
                                                 # offline installer
```

On first launch `AppRun`:
1. finds `wine` (or `$WINE`),
2. creates a dedicated 64-bit prefix at
   `~/.local/share/BedrockLauncher-Linux/prefix` (not `~wine`),
3. silently installs the bundled .NET 8 Desktop Runtime into that prefix,
4. starts `BedrockLauncher.exe` from `usr/bin`.

Subsequent launches skip straight to step 4.

## Requirements

- **Wine 9.x or newer** (not bundled — install it from your distribution or
  https://wiki.winehq.org/Download).
- ~1 GB free disk space (prefix + runtime + game data).
- A 64-bit system (the build is win-x64).

## Environment variables

| Variable            | Meaning                                                        |
|---------------------|----------------------------------------------------------------|
| `WINE`              | Path to a specific wine binary                                 |
| `WINEPREFIX`        | Use a custom prefix instead of the dedicated default            |
| `WINEDEBUG`         | Wine debug channels (default `-all`; set `+err` to debug)      |
| `BLL_FORCE_RUNTIME` | `1` = re-run the .NET runtime installer on next launch          |
| `BLL_NO_INSTALL`    | `1` = never run the .NET runtime installer                      |

Launcher data (profiles, installations, settings) lives inside the prefix at
`~/.local/share/BedrockLauncher-Linux/prefix/drive_c/users/<you>/AppData/Roaming/.minecraft_bedrock/`.

## What works / what does not

Works (expected, under recent Wine):
- The launcher UI itself (WPF rendering is well supported by modern Wine).
- Sign-in with a Microsoft account that owns Minecraft for Windows.
- Browsing, downloading and keeping multiple versions/installations.

Does **not** work under Wine (Windows infrastructure that Wine does not
provide):
- Installing/registering GDK packages (needs Windows Gaming Services / Xbox
  app infrastructure).
- Actually launching the game (needs the Windows GDK runtime).
- UWP-era package management (`Add-AppxPackage` and friends).

This matches the fork's stated purpose — *"testing a Linux version for
research purposes"* — it is a research vehicle, not a way to play Bedrock on
Linux. (For actually playing Bedrock on Linux, look at the separate
[mcpelauncher](https://mcpelauncher.readthedocs.io/) project, which runs the
Android build natively — an entirely different codebase.)

## Building

### Via GitHub Actions (recommended)

Push to `lite` (or run the workflow manually from the Actions tab):
`.github/workflows/linux-appimage.yml`.

Job 1 (`build-windows`) publishes the win-x64 build on a Windows runner.
Job 2 (`appimage`) assembles the AppDir and produces
`BedrockLauncher-Linux-x86_64-<version>.AppImage` as an artifact; pushing a
`v*` tag publishes it as a GitHub release.

> **Why is the build job on Windows?** The managed code cross-builds on Linux
> (`EnableWindowsTargeting` is already set), but **PostSharp** — which weaves
> the `[NotifyPropertyChanged]` aspects used by every WPF view model — must
> load the real WindowsDesktop runtime assemblies to weave, and those are
> Windows-only R2R binaries that cannot be loaded on Linux (verified:
> `WindowsBase`/`PresentationCore`/`PresentationFramework` all fail with
> `BadImageFormatException` on a Linux host). Building the weaved binary
> therefore requires a Windows host; everything after that is done on Linux.

### Locally

```sh
# 1. publish the Windows build (from a Windows machine, or CI)
dotnet publish BedrockLauncher/BedrockLauncher.csproj \
  -c Release -f net8.0-windows10.0.17763.0 -r win-x64 \
  --self-contained false -p:EnableWindowsTargeting=true \
  -o publish/win-x64

# 2. fetch the runtime installer
curl -L -o windowsdesktop-runtime-win-x64.exe \
  https://aka.ms/dotnet/8.0/windowsdesktop-runtime-win-x64.exe

# 3. assemble the AppDir (see the workflow's "Assemble AppDir" step) and run
#    appimagetool
```

## Making the payload self-contained

If you would rather not install the .NET runtime into the Wine prefix,
publish self-contained instead (`--self-contained true -r win-x64`, ~+170 MB)
and drop the runtime-installer step from `AppRun` (`BLL_NO_INSTALL=1` also
skips it).
