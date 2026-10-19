# Releasing

The APK users install is signed with a private key. **Back the key up**: if it
is lost, existing installs cannot be updated — Android rejects an APK signed
with a different key.

## What lives where

| File | Committed? | Purpose |
| --- | --- | --- |
| `exemple/release.keystore` | no (`*.keystore`) | signing key, PKCS#12, alias `glyphmatrix` |
| `exemple/keystore.properties` | no | passwords for the local build |
| `exemple/keystore.properties.example` | yes | template |
| `dist/` | no | locally built artifacts; upload them to a release |

Store a copy of `release.keystore` and its password somewhere safe (password
manager, offline backup). The key is not recoverable from the repo.

## Version

`exemple/app/build.gradle.kts`:

```kotlin
versionCode = 1        // integer, must increase with every public release
versionName = "0.1"    // what the About screen shows
```

The release workflow refuses a tag that does not match `versionName`, so bump
both before tagging.

## Local signed build

```powershell
$env:JAVA_HOME="C:\Users\talmo\.jdks\jbr-21.0.11"
$env:PATH="$env:JAVA_HOME\bin;$env:PATH"
cd exemple
.\gradlew.bat :app:testDebugUnitTest :app:assembleRelease :app:bundleRelease
```

Outputs:

- `exemple/app/build/outputs/apk/release/app-release.apk`
- `exemple/app/build/outputs/bundle/release/app-release.aab`

Verify the signature:

```powershell
& "$env:LOCALAPPDATA\Android\Sdk\build-tools\35.0.0\apksigner.bat" verify --print-certs `
  exemple\app\build\outputs\apk\release\app-release.apk
```

The certificate is `CN=Danil Evrikov, O=Glyph Matrix, C=RU`, SHA-256 digest
`10dbb02e33761739c7043a4e07658419daf15f8859cf23c7916126d21cd1344c`.

## Publishing a GitHub release

One-time setup — add four repository secrets under
**Settings → Secrets and variables → Actions**:

| Secret | Value |
| --- | --- |
| `KEYSTORE_BASE64` | `[Convert]::ToBase64String([IO.File]::ReadAllBytes("exemple\release.keystore"))` |
| `STORE_PASSWORD` | keystore password |
| `KEY_ALIAS` | `glyphmatrix` |
| `KEY_PASSWORD` | key password (same as the store password here) |

Copy the base64 output in one line. Then, for every release:

```powershell
git tag -a v0.1 -m "Glyph Matrix 0.1"
git push origin main
git push origin v0.1
```

`.github/workflows/release.yml` runs the tests, restores the key from the
secrets, builds the signed APK, checks that the tag matches `versionName`, and
attaches `GlyphMatrix-v0.1.apk` plus `SHA256SUMS.txt` to the GitHub release.

## If the key is compromised

Rotate the key and publish the new build under a **new applicationId** — or tell
users to uninstall and reinstall. There is no silent migration between signing
keys.
