# Universal macOS Java Runtime

A GitHub Actions workflow that builds a **universal (Apple Silicon + Intel) Java runtime** for macOS from [Eclipse Temurin](https://adoptium.net/) releases.

Adoptium publishes separate macOS builds for `aarch64` and `x64`. If you bundle a Java runtime in a macOS app, you need one for each architecture or a single fat binary. This project makes the fat binary. It merges the two builds into one runtime bundle whose native executables and libraries contain both `arm64` and `x86_64` slices.

## What the workflow does

The workflow is in [`.github/workflows/build.yml`](.github/workflows/build.yml) and has two jobs.

1. **Download** (runs once per architecture)
   - Downloads the Temurin JDK for the selected release from the Adoptium API (`/v3/binary/version/<release>/mac/<arch>/jdk/hotspot/normal/eclipse`).
   - Extracts it and caches it under the key `<release>-<arch>`, so later runs skip the download.

2. **Create Universal Binary**
   - Restores both cached JDKs.
   - Runs `jlink` on each JDK to make a smaller runtime image (`--add-modules ALL-MODULE-PATH --strip-debug --no-man-pages --no-header-files --compress=2`). The image replaces `Contents/Home`.
   - Copies both bundles into one output directory with `ditto`, so files that exist in only one of them are kept.
   - Uses `lipo` to merge every Mach-O file under `Contents/MacOS`, `Contents/Home/bin` and `Contents/Home/lib` into a universal binary.
   - Changes `Contents/Home/release` so `OS_ARCH` reads `x86_64+arm64`.
   - Zips the result as `<release>-universal.zip`, uploads it as a workflow artifact, and creates a [build provenance attestation](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations) for it.

## Usage

1. Go to **Actions → Build Universal Java Runtime** in this repository.
2. Click **Run workflow** and fill in the inputs:
   - **JDK Release**: a Temurin release tag such as `jdk-21.0.9+10`.
   - **JDK Architectures**: a JSON array. The default is `["aarch64","x64"]`. Keep this order, because the merge step treats the first entry as arm64 and the second as x86_64.
3. When the run finishes, download the `<release>` artifact from the run summary. It contains `<release>-universal.zip`.

You can also start a build with the GitHub CLI:

```bash
gh workflow run build.yml -R iterate-ch/universal-macos-java-runtime -f release=jdk-21.0.9+10
```

### Verifying the build

Check the provenance attestation of a downloaded archive:

```bash
gh attestation verify jdk-21.0.9+10-universal.zip -R iterate-ch/universal-macos-java-runtime
```

Check that a binary contains both architectures:

```bash
lipo -archs jdk-21.0.9+10-universal/Contents/Home/lib/server/libjvm.dylib
```

## Adding a new JDK release

The selectable releases are a fixed list in the `release` input of the workflow. To add one, add its Temurin release name (as shown on [adoptium.net](https://adoptium.net/temurin/releases/), e.g. `jdk-25.0.3+9`) to `on.workflow_dispatch.inputs.release.options` in [`.github/workflows/build.yml`](.github/workflows/build.yml).

## Notes

- The output is a `jlink` runtime image, not the original JDK distribution. Debug symbols, man pages and header files are removed.
- The workflow does not code sign the merged bundle. If you embed it in a macOS application, sign it (and notarize the app) with your own Developer ID.
