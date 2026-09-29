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

### Electron upgrade experiment

The Steam Frame test with Electron 37.10.3 reported no hardware video decoder.
This build tests stock Electron **44.4.3** before a custom V4L2 build.
`files/electron-44.patch` updates the upstream `package.json` and `yarn.lock`. The
manifest applies that patch and sets electron-builder's runtime version from the
exact dependency pin. It keeps Infinity 1.2.2, the ARM64 Zink override, the existing
decoder flags, and sandbox permissions unchanged.

`generated-sources.json` is generated from the patched lockfile. It includes the
matching Electron archives, checksums, cache links, and installer dependencies for
offline Yarn installation. Dependencies no longer used by Electron are removed.
Unrelated dependency versions remain unchanged. The generator also updates the
node-gyp cache marker and fixes its esbuild cache links.

Both `@electron/rebuild` and `electron-rebuild` resolve `node-abi` to **3.96.0**.
The previous nested version, 3.87.0, cannot identify Electron 44 during native
module rebuilds. Version 3.96.0 maps Electron 44 to ABI 149 and satisfies both
rebuild tools' existing 3.x dependency ranges. The app's separate `node-abi` 4.x
dependency is unchanged.

Electron 44 requires build-time Node.js 22.12.0 or newer and has no ARMv7 binary;
Electron sources cover ARM64 and x86-64 only. The existing Node 22 SDK extension
remains in use. Keep the dependency patch, generated sources, and CI version
assertion in sync for future upgrades.

To reproduce the source list, start in a fresh upstream `1.2.2` checkout. Set
`PACKAGING_DIR` to this packaging checkout's absolute path, then run:

```bash
git apply "$PACKAGING_DIR/files/electron-44.patch"
uv tool run --exclude-newer 2026-09-22 \
  --from 'git+https://github.com/flatpak/flatpak-builder-tools.git@41c20aa10819cdb2a4f3ca171758a96d1955c018#subdirectory=node' \
  flatpak-node-generator yarn yarn.lock -o "$PACKAGING_DIR/generated-sources.json"
```

This regenerates sources only. It does not build or run the application.

Validation requires the executable launcher target and a nonempty
`resources/app.asar`. CI runs the packaged binary in Node mode inside the Flatpak
build environment, records its component versions, and requires Electron 44.4.3
on ARM64. It also checks the main executable and all loose ELF files in the
application payload for AArch64 architecture, rejects non-ELF native libraries,
and lists the exported OSTree payload. A failure prevents bundle upload.

A successful build does not verify application launch, graphics, or streaming on
the Steam Frame. Check decoder profiles and actual H.264 hardware decoding on the
Frame before testing GeForce NOW. VA-API build support alone does not establish
that a compatible VA-API driver is available for the Frame's V4L2 decoder.

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
