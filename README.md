Main repo: https://github.com/AstralVixen/GeForce-Infinity

## ARM64 test build

The released ARM64 Flatpak has no application executable. This build keeps version
1.2.2, runtime and BaseApp 25.08, Node SDK extension node22, and the existing sandbox
permissions. It stops on errors rather than exporting an empty application.

The `ARM64 Flatpak build` workflow builds on a native `ubuntu-24.04-arm` runner,
validates the payload, and exports an `arm64-test` bundle with its SHA-256 checksum.
An x86-64 development machine does not need ARM emulation. The workflow runs only
when started manually. It does not install the application or publish a release.

1. Enable GitHub Actions in the packaging fork if required.
2. Put `.github/workflows/arm64-diagnostic.yml` on the fork's default branch so
   GitHub can register its manual trigger. This does not require an upstream merge.
3. Open **Actions → ARM64 Flatpak build → Run workflow**. Select the build branch.
4. On success, download the `geforce-infinity-arm64-test` artifact. It contains the
   bundle, checksum, and source commit.
5. Build logs are uploaded as `arm64-build-logs`, including on failure. If a build
   fails, use the first error in the log to select the fix. Do not disable host
   security controls to resolve sandbox errors.

No application dependency patch is applied yet. The job attempts the full build
with the existing dependencies. A failure prevents bundle upload.

Validation requires the executable launcher target and a nonempty
`resources/app.asar`. It checks the main executable and all loose ELF files in the
application payload for AArch64 architecture, rejects non-ELF native libraries,
and lists the exported OSTree payload. A successful build does not verify launch,
graphics, or streaming on the Steam Frame.

### Native ARM64 build command

With Flatpak, flatpak-builder, and the Flathub user remote available, run:

```bash
set -euo pipefail
mkdir -p diagnostics
flatpak-builder \
  --user \
  --arch=aarch64 \
  --assumeyes \
  --install-deps-from=flathub \
  --default-branch=arm64-test \
  --repo=repo \
  builddir \
  io.github.astralvixen.geforce-infinity.yml \
  2>&1 | tee diagnostics/build-arm64.log
```

Use fresh build and repository directories. `--arch=aarch64` does not enable ARM
execution on an x86-64 host. The test branch is `arm64-test`; do not replace the
Steam Frame's system `stable` installation.
