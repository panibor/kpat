# KPat on Windows (unofficial build)

This fork of [KDE KPat](https://invent.kde.org/games/kpat) adds a GitHub
Actions workflow that produces Windows binaries. **This is an unofficial
community build, not endorsed by KDE.** KPat itself is unmodified except
for two small Windows-only additions in `src/main.cpp`; everything else
is just packaging. See [`NOTICE.windows`](NOTICE.windows) for the full
GPL v2 section 2(a) modification notice.

## Download

Stable Windows binaries are attached to GitHub Releases on this fork:

> https://github.com/panibor/kpat/releases

Each release contains:

- `kpat-<version>-setup.exe`, the NSIS installer.
- `kpat-<version>.7z`, a portable archive. Extract and run `bin\kpat.exe`.
- `COPYING`, the full GPL v2 license text.
- `NOTICE.windows`, the modification notice and source links.

### SmartScreen warning

The installer is **not** code-signed. The first time you run it,
Windows will show a "Windows protected your PC" SmartScreen dialog
warning that the publisher is unknown. Click **More info** then
**Run anyway** to proceed. Code-signing requires an EV certificate
on a hardware token or HSM (~$400-700/year) which is out of scope
for this hobby build.

If you want to confirm the installer you downloaded is exactly
what this repository's CI produced:

- Every Release page lists SHA-256 hashes next to each file. Match
  them against `Get-FileHash kpat-*-setup.exe` locally.
- Every release artifact is also signed via a free
  [sigstore build-provenance attestation](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds)
  published to the public Rekor transparency log by the release job.
  Verify with the GitHub CLI:

  ```pwsh
  gh attestation verify kpat-26.07.70-win-setup.exe --owner panibor
  ```

  A successful verification proves the file came from this
  repository's public `windows.yml` workflow on the tagged commit.

## Building locally on Windows

The same KDE Craft setup that CI uses works on a local Windows box. From
an admin PowerShell:

```powershell
# 1. Install prerequisites: Python 3.11+, Git, and the Visual Studio 2022
#    "Desktop development with C++" workload (MSVC + Windows SDK).
# 2. Bootstrap Craft:
mkdir C:\CraftRoot
Invoke-WebRequest -UseBasicParsing `
  -Uri https://invent.kde.org/packaging/craft/-/raw/master/setup/CraftBootstrap.py `
  -OutFile $env:TEMP\CraftBootstrap.py
python $env:TEMP\CraftBootstrap.py --prefix C:\CraftRoot

# 3. (Optional) point Craft at a local kpat checkout instead of upstream:
#    Edit C:\CraftRoot\etc\BlueprintSettings.ini and append:
#
#      [kde/kdegames/kpat]
#      srcDir = C:/path/to/your/kpat/checkout
#      version = master
#
# 4. Enter the Craft environment and build:
. C:\CraftRoot\craft\craftenv.ps1
craft --install-deps kde/kdegames/kpat
craft kde/kdegames/kpat
craft --package kde/kdegames/kpat
# Output lands in C:\CraftRoot\tmp\
```

A full cold build takes several hours (it builds all of Qt 6, KDE
Frameworks 6, KDEGames 6, FreecellSolver, BlackHoleSolver, then KPat).
Subsequent incremental builds are fast.

## How the CI works

[`.github/workflows/windows.yml`](.github/workflows/windows.yml) runs on
`windows-2022` and uses KDE Craft. Notable details:

- All third-party actions are pinned to commit SHAs.
- The workflow uses least-privilege `permissions:`. The default is
  `contents: read`, escalated only where required.
- `pull_request` (not `pull_request_target`). PRs from forks run in the
  untrusted sandbox with no secret access.
- A watchdog stops the build at 5h20m, persists Craft state to the
  GitHub cache via `actions/cache/save` with `if: always()`, and
  re-dispatches the workflow with `resume=true`. Cold builds that need
  more than the 6-hour-per-job hard cap finish across multiple chained
  runs automatically (capped at 5 resumes).
- A weekly `cron` schedule keeps the cache from being evicted (GitHub
  evicts caches after 7 idle days).
- A post-install pass strips MinGW debug info from the bundled
  Qt and KF6 DLLs and removes Qt plugin directories KPat does not
  load (SQL drivers, QtMultimedia, QML tooling, sensors, etc.) so the
  installer stays small. Translation catalogues are kept intact.

## Cutting a Windows release

```bash
git tag v26.07.70-win
git push origin v26.07.70-win
```

The workflow's `release` job runs on `v*-win` tags and creates a **draft**
GitHub Release on this fork with the installer, portable archive,
`COPYING`, and `NOTICE.windows` attached. Review and publish from the
Releases page when ready.

The `-win` suffix on the tag namespaces this fork's Windows release
tags away from upstream KDE's release tagging convention.

## Recommended fork settings

To minimise warnings on the fork's home page and protect the workflow
from tampering, enable these in **Settings**:

- **Branches**, add a rule for `master`:
  - Require a pull request before merging.
  - Require status checks to pass before merging. Select
    `Windows (Craft) / build`.
  - Block force pushes.
- **Code security**, Dependabot: keep the default Dependabot alerts on.
  This repo already ships [`.github/dependabot.yml`](.github/dependabot.yml)
  to auto-PR action version bumps weekly.
- **Actions**, General, Workflow permissions: "Read repository contents
  and packages permissions" (the default). The workflow grants itself
  the writes it needs at the job level.

## Upstreaming

This workflow is GitHub-specific. Upstream KDE uses GitLab CI at
`invent.kde.org` and has its own Windows binary factory
(`binary-factory.kde.org`). Porting this CI to upstream would mean
rewriting it as a `.gitlab-ci.yml` job and opening an MR against
`invent.kde.org/games/kpat`. Not done here.
