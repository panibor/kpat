<p align="center">
  <img src="logo.png" alt="KPat (KDE Patience) logo" width="320">
</p>

# KPat for Windows

**KPat for Windows** is an unofficial Windows installer for
[KPat (KDE Patience)](https://invent.kde.org/games/kpat), the free,
open-source solitaire collection from the KDE community. This fork
exists to ship a working Windows build (NSIS installer plus a portable
7-Zip archive) of KPat for users on Windows 10 and Windows 11, packaged
through KDE Craft from publicly available source.

> This is a community fork. **It is not endorsed by, affiliated with, or
> distributed by KDE e.V. or the KDE community.** The original, official
> upstream KPat repository is at
> [`invent.kde.org/games/kpat`](https://invent.kde.org/games/kpat).

## Download

Pre-built Windows installers are published on this fork's GitHub
Releases page:

> **[Download the latest KPat Windows installer ->](https://github.com/panibor/kpat/releases)**

Each release contains:

- `kpat-<version>-setup.exe`, the NSIS Windows installer.
- `kpat-<version>.7z`, a portable archive (extract anywhere and run
  `bin\kpat.exe`).
- `COPYING`, the full GPL v2 license text.
- `NOTICE.windows`, the modification notice and source links.

The installer is **not** code-signed. When you launch it for the first
time, Windows SmartScreen will warn that the publisher is unknown. Click
"More info" then "Run anyway" to proceed. See
[`README.windows.md`](README.windows.md) for the full details, including
how to build locally and how to cut a new Windows release.

## What is KPat?

KPat is the KDE community's collection of solitaire card games (also
known as patience games). It currently ships fifteen variants including:

- Klondike (the classic "Windows Solitaire" layout).
- FreeCell, with an integrated solver to suggest moves.
- Spider, Yukon, Forty and Eight, Golf, Mod3, Gypsy, Grandfather's Clock,
  Simple Simon, and several others.
- Importable [PySol](http://www.pysolfc.sourceforge.net/) saved games.

Card decks and table backgrounds are downloadable from inside the game
via the "Get New" actions (powered by KDE's KNewStuff framework).

<p align="center">
  <img src="previews/1.png" alt="KPat gameplay screenshot showing a solitaire layout"
       width="640">
</p>

## What this fork adds

Versus the upstream `invent.kde.org/games/kpat` source, this fork adds
only the files needed to produce and document a Windows build:

| File | Purpose |
| --- | --- |
| `.github/workflows/windows.yml` | GitHub Actions workflow that builds KPat for Windows via KDE Craft. |
| `.github/dependabot.yml` | Keeps the workflow's pinned action versions current. |
| `README.md` | This file. |
| `README.windows.md` | Build, install, and release documentation for the Windows binaries. |
| `NOTICE.windows` | GPL v2 section 2(a) modification notice. |

Two small Windows-only additions land in `src/main.cpp`:

1. A roughly 20-line palette fix guarded by `#ifdef Q_OS_WIN`, working
   around a Qt 6.7+ `windows11` style bug that made the labels of
   selected items in `KConfigDialog` sidebars unreadable.
2. A startup normaliser that relocates KNewStuff theme downloads from
   the Generic data directory into the App data directory, so the
   in-game "Get New Backgrounds" picker can actually find them on
   Windows.

Both source-level additions are no-ops on Linux and macOS. KPat itself,
its card-deck assets, game logic, solvers, translations, and
documentation are otherwise unchanged. See
[`NOTICE.windows`](NOTICE.windows) for the formal change list and source
links.

## License

KPat is licensed under **GPL-2.0-or-later**, with original code by Paul
Olav Tvete (1995) under a permissive GPL-compatible license. The full
text is in [`COPYING`](COPYING). KPat artwork and translation files
carry their own (GPL-compatible) headers under the original directories.

The complete corresponding source code for this Windows build is this
repository:

- Fork: <https://github.com/panibor/kpat>
- Upstream: <https://invent.kde.org/games/kpat>

## Reporting issues

If something is broken specifically in the Windows build, file a bug
here on this fork. If the bug reproduces on KPat for Linux as well,
please report it upstream at
[`invent.kde.org/games/kpat/-/issues`](https://invent.kde.org/games/kpat/-/issues)
so the KDE Games team can see it.
