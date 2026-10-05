# flutter_release_pipeline

A reusable GitHub Actions workflow that builds a Flutter app for iOS, Android,
macOS and Windows, tags and publishes a GitHub release, and ships to testers
through App Store Connect and Firebase App Distribution — all driven by
keywords in the commit message.

An app repository keeps a ~30-line caller workflow and nothing else. The
release logic, the guards and the platform quirks live here, once.

## Usage

Add `.github/workflows/release.yaml` to the app repository:

```yaml
name: "Build & distribute"

on:
  push:
    branches: [main]

# Never let two releases race for the same version, and never cancel one midway.
concurrency:
  group: build-version
  cancel-in-progress: false

jobs:
  pipeline:
    uses: crianpiro/flutter_release_pipeline/.github/workflows/release.yaml@v1
    # The release job pushes the version bump and the tag.
    permissions:
      contents: write
    with:
      artifact-name: myapp
    secrets:
      P12_BASE64: ${{ secrets.P12_BASE64 }}
      P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
      PROVISIONING_PROFILE_BASE64: ${{ secrets.PROVISIONING_PROFILE_BASE64 }}
      EXPORT_OPTIONS: ${{ secrets.EXPORT_OPTIONS }}
      RUNNER_KEYCHAIN_PASSWORD: ${{ secrets.RUNNER_KEYCHAIN_PASSWORD }}
      KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
      KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
      KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
      KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
      FIREBASE_ANDROID_APP_ID: ${{ secrets.FIREBASE_ANDROID_APP_ID }}
      FIREBASE_DISTRIBUTION_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_DISTRIBUTION_SERVICE_ACCOUNT }}
      APPLE_API_KEY_ID: ${{ secrets.APPLE_API_KEY_ID }}
      APPLE_API_KEY_ISSUER_ID: ${{ secrets.APPLE_API_KEY_ISSUER_ID }}
      APPLE_API_KEY_BASE64: ${{ secrets.APPLE_API_KEY_BASE64 }}
```

Secrets are passed one by one rather than with `secrets: inherit`, because
`inherit` only works between repositories of the same organisation or
enterprise, and this repository is not in the app's organisation.

The caller has to declare `permissions` and `concurrency` itself: a called
workflow can only narrow the permissions it is given, and the concurrency
guard is a property of the app repository.

## Commit-message keywords

Any push whose head commit carries neither `build-version` nor
`build-desktop-version` does nothing.

```
build-version                                    build, tag, release
build-version 1.4.0                              ...at a version you name
build-version distribute-android                 ...and ship the APK to Firebase
build-version distribute-ios                     ...and ship the IPA to App Store Connect
build-version distribute-ios distribute-android  ...ship both
groups: qa, management                           Firebase tester groups for this release

build-version build-desktop-version                  ...also attach macOS and Windows builds
build-version build-desktop-version desktop-macos    ...macOS only
build-version build-desktop-version desktop-windows  ...Windows only
build-desktop-version                                desktop only — no bump, tag or release
```

Put the keywords in the **pull request title**, so a squash merge carries them
onto the branch; the pull request description becomes the release notes. They
reach the GitHub release, Firebase testers and the TestFlight build's
"What's New". The subject line is used when there is no body.

### Versioning

`pubspec.yaml` holds the version (`X.Y.Z+N`) and stays the single source of
truth. A version named after `build-version` is written there; with none, the
patch is bumped. The build number increments either way, because App Store
Connect permanently refuses a build number it has seen. The bump is committed
back to the branch and tagged `vX.Y.Z` only after every artifact has built, so
a failed build leaves nothing behind.

A run fails early, before building anything, when:

- the version is already in `pubspec.yaml` or already tagged;
- a keyword names a version other than the one being built
  (`build-version 1.0.7 distribute-ios 1.0.5`);
- distribution is asked for a platform the app has opted out of;
- a release would contain no artifact at all;
- the `groups:` line is not a comma-separated list of aliases.

`build-desktop-version` alone builds what `pubspec.yaml` already says and
consumes no version number; its artifacts stay on the workflow run.

### Platforms

| Platform | On `build-version` | On `build-desktop-version` alone |
|---|---|---|
| iOS | built unless `ios: false` | not built |
| Android | built unless `android: false` | not built |
| macOS | built only with `build-desktop-version` too | built unless only Windows is named |
| Windows | built only with `build-desktop-version` too | built unless only macOS is named |

Mobile is opt-out per app, not per commit: a release always carries every
mobile platform the app has. Desktop is opt-in per commit. Distribution is
always opt-in.

## Inputs

| Input | Default | |
|---|---|---|
| `ios` | `true` | Build the IPA on a release. `false` for an app with no iOS target. |
| `android` | `true` | Build the APK on a release. `false` for an app with no Android target. |
| `linux-runner` | `ubuntu-latest` | `runs-on` for the Linux jobs (resolve, Android, Firebase, release). |
| `macos-runner` | `macos-latest` | `runs-on` for the iOS, macOS and App Store Connect jobs. |
| `windows-runner` | `windows-latest` | `runs-on` for the Windows job. |
| `artifact-name` | repository name | Prefix of every artifact file: `<name>-1.4.0+12.ipa`, `<name>-macos-1.4.0+12.zip`. |
| `java-version` | `17` | JDK for the Android build. |
| `enforce-lockfile` | `true` | Install with `flutter pub get --enforce-lockfile`. Set `false` if the app does not commit `pubspec.lock`. Not applied to the iOS job, whose dependencies are installed inside `crianpiro/build_flutter_app`. |
| `refresh-cocoapods` | `true` | Run `pod repo update` before the macOS build. A stale index reports "could not find compatible versions for pod X", which reads like a dependency conflict. |
| `uses-non-exempt-encryption` | `false` | The export-compliance answer posted to App Store Connect. |
| `default-firebase-groups` | `default` | Firebase group aliases used when the commit has no `groups:` line. |

The runner inputs take either a plain label or a JSON object or array, for
self-hosted runners:

```yaml
with:
  linux-runner: '{"group": "my-runners", "labels": ["self-hosted", "Linux", "X64"]}'
  macos-runner: '["self-hosted", "macOS", "ARM64"]'
```

The workflow also exposes `version`, `build_number` and `release_requested` as
outputs, for a caller that chains further jobs after it.

## Secrets

All are optional at the interface, so an app that opts out of a platform
passes nothing for it. The job that needs a secret fails, naming it, when it is
missing.

| Secret | Needed for |
|---|---|
| `GIT_TOKEN` | `flutter pub get` fetching private Git dependencies on github.com (read access is enough). |
| `P12_BASE64`, `P12_PASSWORD` | iOS distribution certificate. |
| `PROVISIONING_PROFILE_BASE64` | iOS provisioning profile. |
| `EXPORT_OPTIONS` | Contents of `ios/ExportOptions.plist`. |
| `RUNNER_KEYCHAIN_PASSWORD` | Password for the temporary keychain the iOS build creates. |
| `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_PASSWORD`, `KEY_ALIAS` | Android release signing. |
| `FIREBASE_ANDROID_APP_ID`, `FIREBASE_DISTRIBUTION_SERVICE_ACCOUNT` | `distribute-android`. |
| `APPLE_API_KEY_ID`, `APPLE_API_KEY_ISSUER_ID`, `APPLE_API_KEY_BASE64` | `distribute-ios` (App Store Connect API key). |

`GIT_TOKEN` reaches git through a credential helper that reads it at call time,
so the token never appears in a remote URL, the process table, or the runner's
global `.gitconfig`. The release job pushes with the automatic `GITHUB_TOKEN`,
not this one.

## What the app has to provide

- **`pubspec.yaml`** with `version: X.Y.Z+N` and the Flutter version under
  `environment: flutter:`, which is where `subosito/flutter-action` reads it.
- **Android signing read from the environment.** The keystore path and
  passwords reach Gradle as environment variables, not through
  `key.properties`, because a shell and `java.util.Properties` both silently
  alter passwords containing `$`, backticks, backslashes or non-ASCII
  characters. In `android/app/build.gradle.kts`:

  ```kotlin
  signingConfigs {
      create("release") {
          storeFile = System.getenv("ANDROID_KEYSTORE_FILE")?.let { file(it) }
          storePassword = System.getenv("ANDROID_KEYSTORE_PASSWORD")
          keyAlias = System.getenv("ANDROID_KEY_ALIAS")
          keyPassword = System.getenv("ANDROID_KEY_PASSWORD")
      }
  }
  ```

- **The entry point at `lib/main.dart`.** Every platform builds the default
  target.

Desktop artifacts are not signed: the macOS `.app` is ad-hoc signed and
Gatekeeper refuses it until it is signed with a Developer ID and notarised;
the Windows `.exe` raises a SmartScreen warning.

## Releasing this repository

Callers pin a major tag (`@v1`). The workflow also references its own
`ensure-yq` action by that tag, because a called workflow's `./` is the
caller's checkout, not this repository. So:

- a compatible change moves the `v1` tag to the new commit;
- a breaking change bumps every `@v1` inside `.github/workflows/release.yaml`
  to `@v2` in the same commit, then tags `v2`.
