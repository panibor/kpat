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
  gh attestation verify .\\kpat-26.07.70-win-setup.exe --owner panibor
  ```

  A successful verification proves the file came from this
  repository's public `windows.yml` workflow on the tagged commit.

## Building locally on Windows

The same KDE Craft setup that CI uses works on a local Windows box. From
an admin PowerShell:

```powershell
# 1. Install Python 3.11+ and Git, plus the dependencies for your chosen
#    Craft toolchain. GitHub Actions uses MinGW-w64 / GCC, not MSVC.
#    Use the GitHub Actions release procedure below for official fork builds.
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
- A manual **Run workflow** dispatch publishes a release only when
  **Publish a new Windows GitHub Release** is checked (the default).
  Ordinary pushes and pull requests build without publishing; tag pushes
  matching `v*-win` retain release support.
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

## Publishing a Windows release (one click)

Use the GitHub UI; there is no need to clone this repository, run
PowerShell, create a tag manually, or publish a draft release.

1. Open **[Actions > Windows (Craft)](https://github.com/panibor/kpat/actions/workflows/windows.yml)**.
2. Click **Run workflow** and select the `master` branch.
3. Leave **Publish a new Windows GitHub Release** **checked**
   (the default). Leave `resume` unchecked and `resume_count` at `0`;
   they are internal controls for checkpointed builds.
4. Click **Run workflow** and inspect the run's `build` and `release`
   jobs. On success, GitHub publishes the release with all four files:
   installer `.exe`, portable `.7z`, `COPYING`, and `NOTICE.windows`.
5. Open **[Releases](https://github.com/panibor/kpat/releases)**
   and confirm both Windows downloads appear in the new release.

The workflow determines the release version from
`RELEASE_SERVICE_VERSION_MAJOR`, `_MINOR`, and `_MICRO` in
`CMakeLists.txt`. It generates a tag in the form
`v<major>.<minor>.<micro>-win` and names the artifacts
`kpat-<major>.<minor>.<micro>-win-setup.exe` and
`kpat-<major>.<minor>.<micro>-win.7z`.
The tag is created only after the build succeeds and required files are
verified. A signed build-provenance attestation is generated before
publishing. The release is **published immediately**, not left as a draft.

**Version uniqueness:** the workflow refuses to replace an existing
version tag or release. If that version has already been tagged, wait
for a new upstream version or intentionally update the version in
`CMakeLists.txt` before triggering a new release. Do not delete or
move historical release tags to force a rebuild.

**Build only:** to run Windows CI without publishing, uncheck
**Publish a new Windows GitHub Release**. Regular pushes to `master`,
pull requests, and scheduled cache warm-ups also do not publish.

**Automatic resumes:** large Craft builds may reach the watchdog,
save a checkpoint, and dispatch a continuation automatically. The
continuation retains the original release/no-release intent. You
do not need to re-run the workflow manually. If a build or release
fails, inspect the Actions logs; no new release is published by a
failed release job. A failed job can leave a new tag behind if failure
occurs after tag creation, which requires investigation before retrying.

**Advanced alternative:** pushing a new `v*-win` tag still starts
the release path, provided the tag matches the version in
`CMakeLists.txt`. This is optional and not required for normal releases.

The `-win` suffix keeps the fork's Windows tags separate from upstream
KDE's tags.

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
