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

## Getting the secrets

Everything below ends as a repository secret of the **app** repository
(Settings → Secrets and variables → Actions; see
[GitHub's guide](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)).
Piping a value straight into `gh secret set` is safer than copying it through
the clipboard, which can add or drop a trailing newline:

```bash
base64 -i certificate.p12 | gh secret set P12_BASE64
gh secret set EXPORT_OPTIONS < ios/ExportOptions.plist
gh secret set P12_PASSWORD          # prompts for the value
```

The `base64 -i` spelling is macOS's; on Linux use `base64 -w0 <file>`.

### iOS signing

You need a paid Apple Developer Program membership, the app's bundle id
registered under Identifiers, and its record created in App Store Connect
([Add a new app](https://developer.apple.com/help/app-store-connect/create-an-app-record/add-a-new-app))
before the first upload. GitHub's
[Installing an Apple certificate on macOS runners](https://docs.github.com/en/actions/use-cases-and-examples/deploying/installing-an-apple-certificate-on-macos-runners-for-xcode-development)
walks through the first three secrets end to end.

- **`P12_BASE64`, `P12_PASSWORD`** — an **Apple Distribution** certificate
  with its private key, exported as `.p12`.
  1. [Create a certificate signing request](https://developer.apple.com/help/account/certificates/create-a-certificate-signing-request)
     in Keychain Access.
  2. In [Certificates](https://developer.apple.com/account/resources/certificates/list),
     create an *Apple Distribution* certificate from it
     ([overview](https://developer.apple.com/help/account/certificates/certificates-overview)),
     download it and double-click it into your keychain.
  3. In Keychain Access → My Certificates, select the certificate together
     with its private key and
     [export it](https://support.apple.com/guide/keychain-access/import-and-export-keychain-items-kyca35961/mac)
     as `.p12`. The password you choose there is `P12_PASSWORD`; base64 of the
     file is `P12_BASE64`.
- **`PROVISIONING_PROFILE_BASE64`** — an
  [App Store provisioning profile](https://developer.apple.com/help/account/provisioning-profiles/create-an-app-store-provisioning-profile)
  for the bundle id, tied to that certificate. Base64 of the downloaded
  `.mobileprovision`. It expires yearly, and must be regenerated whenever the
  certificate changes.
- **`EXPORT_OPTIONS`** — the plain XML contents of an `ExportOptions.plist`
  that maps the bundle id to the profile above. The easiest way to get one is
  to archive once in Xcode (Product → Archive → Distribute App → App Store
  Connect → Export); the export folder contains it. Otherwise write it by hand:

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
  <dict>
    <key>method</key>
    <string>app-store-connect</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>signingStyle</key>
    <string>manual</string>
    <key>provisioningProfiles</key>
    <dict>
      <key>com.example.app</key>
      <string>Name of the App Store profile</string>
    </dict>
  </dict>
  </plist>
  ```

  `app-store-connect` needs Xcode 15.3 or later; older Xcode spells it
  `app-store`. The team id is under Membership details in your
  [developer account](https://developer.apple.com/account).
- **`RUNNER_KEYCHAIN_PASSWORD`** — not issued by anyone: any random string.
  It locks the temporary keychain the build creates.

  ```bash
  openssl rand -base64 32 | gh secret set RUNNER_KEYCHAIN_PASSWORD
  ```

### App Store Connect (`distribute-ios`)

- **`APPLE_API_KEY_ID`, `APPLE_API_KEY_ISSUER_ID`, `APPLE_API_KEY_BASE64`** —
  a team API key
  ([Creating API keys](https://developer.apple.com/documentation/appstoreconnectapi/creating-api-keys-for-app-store-connect-api)).
  1. In [App Store Connect → Users and Access → Integrations → App Store Connect API](https://appstoreconnect.apple.com/access/integrations/api),
     generate a **Team Key** with the **App Manager** role. Only the Account
     Holder can request API access the first time.
  2. The page shows the **Issuer ID** above the keys table and the **Key ID**
     in the key's row.
  3. Download the `AuthKey_<KEYID>.p8`. Apple offers it **once**; keep it
     somewhere safe.

  ```bash
  base64 -i AuthKey_XXXXXXXXXX.p8 | gh secret set APPLE_API_KEY_BASE64
  ```

### Android signing

- **`KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_PASSWORD`, `KEY_ALIAS`** — an
  upload keystore
  ([Flutter](https://docs.flutter.dev/deployment/android#create-an-upload-keystore),
  [Android](https://developer.android.com/studio/publish/app-signing#generate-key)):

  ```bash
  keytool -genkey -v -keystore upload-keystore.jks -storetype JKS \
    -keyalg RSA -keysize 2048 -validity 10000 -alias upload
  ```

  The store password and key password `keytool` asks for are
  `KEYSTORE_PASSWORD` and `KEY_PASSWORD`; the `-alias` is `KEY_ALIAS`; base64
  of the `.jks` is `KEYSTORE_BASE64`. Back the keystore up: an app already
  installed by testers only accepts updates signed with the same key.

### Firebase App Distribution (`distribute-android`)

The Android app has to be registered in a Firebase project, and App
Distribution opened once in the console for it.

- **`FIREBASE_ANDROID_APP_ID`** — in the
  [Firebase console](https://console.firebase.google.com/), Project settings →
  General → Your apps → the Android app's **App ID**, of the form
  `1:1234567890:android:0a1b2c3d4e5f67890`
  ([reference](https://firebase.google.com/docs/app-distribution/android/distribute-cli)).
- **`FIREBASE_DISTRIBUTION_SERVICE_ACCOUNT`** — the whole JSON key of a
  service account allowed to upload
  ([Authenticate with a service account](https://firebase.google.com/docs/app-distribution/authenticate-service-account)).
  1. In the Google Cloud console for the same project, IAM & Admin → Service
     accounts → create one, and grant it **Firebase App Distribution Admin**.
  2. On that account, Keys → Add key → JSON
     ([Create and delete keys](https://cloud.google.com/iam/docs/keys-create-delete)).
  3. `gh secret set FIREBASE_DISTRIBUTION_SERVICE_ACCOUNT < key.json`, then
     delete the local file.
- **Tester groups** are not a secret, but a build sent to a group alias that
  does not exist fails. Create the groups under App Distribution → Testers &
  Groups ([Manage testers](https://firebase.google.com/docs/app-distribution/manage-testers)):
  either one aliased `default`, or set `default-firebase-groups` to an alias
  you have.

### Private Git dependencies

- **`GIT_TOKEN`** — only when `pubspec.yaml` pulls packages from private
  GitHub repositories. A
  [fine-grained personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
  whose resource owner is the account or organisation that owns those
  repositories, limited to them, with **Contents: read-only**. An organisation
  may have to approve the token before it works.

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
