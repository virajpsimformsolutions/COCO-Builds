# COCO-Builds — Guide for Other Apps

A step-by-step guide to replicate the COCO-Builds CI/CD system for any Expo/React Native app.

## Prerequisites

- Expo SDK 51+ project with `app.config.ts`
- Expo config plugins for native customizations
- `credentials.json` + `custom-signing.gradle` approach for Android signing
- GitHub repos: one for source code, one for build workflows (or use same repo)

---

## Step 1: Create Build Workflow Repository

Create a separate repo (e.g., `YourOrg/YourApp-Builds`) to hold workflows. This keeps your source repo clean.

```
YourApp-Builds/
├── .github/workflows/
│   ├── build-android-apk.yml
│   ├── build-android-aab.yml
│   └── build-ios-ipa.yml
└── README.md
```

---

## Step 2: Configure GitHub Secrets

Go to **Settings → Secrets and variables → Actions** in your builds repo.

### All apps need:

| Secret | How to get |
|--------|-----------|
| `COCO_PAT` | GitHub PAT with `repo` scope (needed to checkout source + create releases) |
| API URLs, Sentry, etc. | Your app's specific environment variables |

### Android apps need:

```bash
# Generate keystore (if you don't have one)
keytool -genkeypair -v -storetype PKCS12 -keystore release.keystore \
  -alias your-alias -keyalg RSA -keysize 2048 -validity 10000

# Base64 encode for secret
base64 -i release.keystore | pbcopy   # → KEYSTORE_BASE64
```

| Secret | Value |
|--------|-------|
| `KEYSTORE_BASE64` | Output of `base64 -i release.keystore` |
| `KEYSTORE_PASSWORD` | Keystore password |
| `KEY_ALIAS` | Key alias (e.g., `youralias`) |
| `KEY_PASSWORD` | Key password |

### iOS apps need:

```bash
# Export certificate from Keychain Access
# Right-click certificate → Export → .p12 format
base64 -i Certificates.p12 | pbcopy   # → IOS_DIST_CERT_BASE64

# Download provisioning profile from Apple Developer portal
base64 -i profile.mobileprovision | pbcopy  # → IOS_PROVISIONING_PROFILE_BASE64
```

| Secret | Value |
|--------|-------|
| `IOS_DIST_CERT_BASE64` | Base64-encoded P12 certificate |
| `IOS_DIST_CERT_PASSWORD` | P12 export password |
| `IOS_PROVISIONING_PROFILE_BASE64` | Base64-encoded mobileprovision |

---

## Step 3: Create Android APK Workflow

Copy `build-android-apk.yml` and customize these values:

### 3a. Change repository references

```yaml
# Line 31-32: Your builds repo
- name: Checkout YourApp-Builds
  uses: actions/checkout@v4
  with:
    repository: YourOrg/YourApp-Builds    # ← CHANGE THIS
    token: ${{ secrets.COCO_PAT }}

# Line 38-39: Your source repo
- name: Checkout your-app-source
  uses: actions/checkout@v4
  with:
    repository: YourOrg/YourAppSource     # ← CHANGE THIS
    ref: ${{ github.event.inputs.branch }}
    path: app
    token: ${{ secrets.COCO_PAT }}
```

### 3b. Update .env secrets

Match the secrets to your app's `.env` variables:

```yaml
- name: Write .env file
  working-directory: app
  env:
    ENV_API_URL: ${{ secrets.YOUR_API_URL }}          # ← CHANGE
    ENV_ENVIRONMENT: ${{ github.event.inputs.environment || 'development' }}
    # ... add/remove env vars matching your app.config.ts extra block
  run: |
    {
      echo "EXPO_PUBLIC_API_URL=${ENV_API_URL}"
      echo "EXPO_PUBLIC_ENVIRONMENT=${ENV_ENVIRONMENT}"
      # ... match your app's env var names
    } > .env
```

### 3c. Update keystore details

```yaml
- name: Write credentials.json
  working-directory: app
  env:
    KS_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
    KS_ALIAS: ${{ secrets.KEY_ALIAS }}              # ← Your key alias
    KS_KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
  run: |
    printf '{"android":{"keystore":{"keystorePath":"release.keystore","keystorePassword":"%s","keyAlias":"%s","keyPassword":"%s"}}}' \
      "${KS_PASSWORD}" "${KS_ALIAS}" "${KS_KEY_PASSWORD}" > credentials.json
```

### 3d. Remove or customize Gradle patches

**If your app doesn't use DocuSign**, remove steps 10 and 12 (DocuSign AAR fix + stripped AAR addition).

**Keep these universal patches:**
- Step 11: JVM target alignment (safe for all RN apps)
- Step 13: Metaspace bump (recommended for large apps)

### 3e. Update the Release step

```yaml
- name: Create Release
  env:
    GH_TOKEN: ${{ secrets.COCO_PAT }}
  run: |
    TAG="build-apk-${{ github.event.inputs.environment || 'development' }}-$(date +%Y%m%d)-${SOURCE_SHA:0:7}"
    gh release create "$TAG" \
      --repo YourOrg/YourAppSource                  # ← CHANGE THIS
      --title "APK ${{ github.event.inputs.environment || 'development' }} $(date '+%Y-%m-%d')" \
      --notes "..." \
      app/android/app/build/outputs/apk/release/*.apk
```

---

## Step 4: Create Android AAB Workflow

Copy the APK workflow and make these changes:

1. **Change default environment** to `production`:
```yaml
environment:
  default: 'production'    # ← APK defaults to 'development', AAB to 'production'
```

2. **Change build command:**
```yaml
- name: Build AAB
  working-directory: app/android
  run: ./gradlew bundleRelease --parallel --build-cache -x lint
  # Note: NO -PreactNativeArchitectures — all archs for Play Store
```

3. **Update Release step:**
```yaml
TAG="build-aab-..."
# Upload .aab instead of .apk
app/android/app/build/outputs/bundle/release/*.aab
```

---

## Step 5: Create iOS IPA Workflow

Copy `build-ios-ipa.yml` and customize:

### 5a. Change repository references
Same as Android — update both checkout steps.

### 5b. Update .env secrets
Same as Android.

### 5c. Update signing configuration

```yaml
- name: Import Apple Distribution Certificate
  env:
    CERT_BASE64: ${{ secrets.IOS_DIST_CERT_BASE64 }}
    CERT_PASSWORD: ${{ secrets.IOS_DIST_CERT_PASSWORD }}
  run: |
    KEYCHAIN_PATH="$RUNNER_TEMP/build.keychain"
    KEYCHAIN_PASSWORD="yourapp-build-$(uuidgen)"
    # ... (keep same structure)
```

### 5d. Update provisioning profile filename

```yaml
- name: Install Provisioning Profile
  env:
    PROV_BASE64: ${{ secrets.IOS_PROVISIONING_PROFILE_BASE64 }}
  run: |
    mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
    echo "$PROV_BASE64" | base64 -d > ~/Library/MobileDevice/Provisioning\ Profiles/yourapp-release.mobileprovision
    #                                                    ↑ CHANGE THIS NAME
```

### 5e. Update Xcode signing

```yaml
- name: Archive
  working-directory: app/ios
  run: |
    xcodebuild archive \
      -workspace *.xcworkspace \
      -scheme '${{ steps.scheme.outputs.SCHEME }}' \
      -configuration Release \
      -archivePath "$RUNNER_TEMP/yourapp.xcarchive" \
      -allowProvisioningUpdates \
      CODE_SIGN_STYLE=Manual \
      DEVELOPMENT_TEAM=YOUR_TEAM_ID \
      #                        ↑ CHANGE THIS (Apple Developer Team ID)
      ONLY_ACTIVE_ARCH=NO
```

### 5f. Update export options

```yaml
- name: Export IPA
  run: |
    cat > /tmp/export.plist << EOF
    <dict>
      <key>teamID</key>
      <string>YOUR_TEAM_ID</string>         # ← CHANGE THIS
    </dict>
    EOF
```

### 5g. Update Release step

```yaml
TAG="build-ipa-..."
# Upload .ipa instead of .apk/.aab
${{ runner.temp }}/ipa/*.ipa
```

---

## Step 6: Set Up Source Repo

Your source repo needs:

### `app.config.ts` — Environment-aware config

```typescript
export default ({ config }) => {
  const env = process.env.EXPO_PUBLIC_ENVIRONMENT || 'development';

  return {
    ...config,
    extra: {
      ...config.extra,
      environment: env,
      eas: { projectId: 'your-eas-project-id' },
    },
    android: {
      ...config.android,
      package: env === 'production'
        ? 'com.yourorg.yourapp'
        : env === 'preview'
          ? 'com.yourorg.yourapp.preview'
          : 'com.yourorg.yourapp.dev',
    },
    ios: {
      ...config.ios,
      bundleIdentifier: env === 'production'
        ? 'com.yourorg.yourapp'
        : env === 'preview'
          ? 'com.yourorg.yourapp.preview'
          : 'com.yourorg.yourapp.dev',
    },
  };
};
```

### `buildconfig.js` — ProGuard rules (if using custom signing)

If your app uses `credentials.json` + `custom-signing.gradle`:

```javascript
// buildconfig.js
module.exports = {
  // ... your existing config
  android: {
    // ... your android config
    extra: {
      // Add ProGuard rules for libraries that need them
      proguardFiles: [
        // RxJava, OkHttp, etc.
      ],
    },
  },
};
```

### `withCustomSigning.js` — Expo config plugin (if using credentials.json)

If you already have this from COCO setup, no changes needed. If creating fresh:

```javascript
// withCustomSigning.js
const { withAppBuildGradle } = require('expo/config-plugins');

module.exports = function withCustomSigning(config) {
  return withAppBuildGradle(config, (config) => {
    config.modResults.contents += `
      apply from: "custom-signing.gradle"
    `;
    return config;
  });
};
```

### Add to `app.config.ts` plugins:

```typescript
plugins: [
  // ... other plugins
  './withCustomSigning',
],
```

---

## Step 7: Test the Setup

### 7a. Push workflow files

```bash
cd YourApp-Builds
git add .
git commit -m "feat(build): add CI/CD workflows"
git push origin main
```

### 7b. First run — Android APK

1. Go to **Actions** tab in `YourApp-Builds`
2. Select **"Build Android APK (Release)"**
3. Click **"Run workflow"**
4. Select branch and environment
5. Watch the run — fix any errors

### 7c. First run — Android AAB

Same process, select **"Build Android AAB (Release)"**.

### 7d. First run — iOS IPA

1. Ensure secrets are set correctly
2. Select **"Build iOS IPA (Release)"**
3. Select branch, environment, export method
4. Watch the run

---

## Customizing Gradle Patches

### If your app has no DocuSign

Remove these steps from both Android workflows:
- **"Fix DocuSign AAR dependency"** (step 10)
- **"Add stripped DocuSign AAR to app build"** (step 12)

### If your app has different dependency issues

Add a new step after `expo prebuild` with `sed` patches:

```yaml
- name: Fix [YourIssue]
  working-directory: app
  run: |
    # Your sed commands here
    sed -i 's/old-pattern/new-pattern/' android/app/build.gradle
```

### If your app needs additional native modules

Add post-prebuild steps:

```yaml
- name: Patch [YourNativeModule]
  working-directory: app
  run: |
    # Add native code patches here
```

---

## Environment Variables Reference

Your `.env` file should contain (minimum):

```bash
EXPO_PUBLIC_API_URL=https://api.yourapp.com
EXPO_PUBLIC_ENVIRONMENT=development
EXPO_PUBLIC_SENTRY_URL=https://xxx@sentry.io/xxx
EXPO_SENTRY_URL=https://xxx@sentry.io/xxx
EXPO_SENTRY_PROJECT_NAME=your-project
EXPO_SENTRY_ORGANIZATION=your-org
SENTRY_AUTH_TOKEN=xxx
EXPO_PUBLIC_API_ENCRYPTION_KEY=xxx
EXPO_PUBLIC_SOCKET_URL=wss://socket.yourapp.com
EXPO_PUBLIC_API_PIN_SHA256=xxx
EXPO_PUBLIC_SOCKET_PIN_SHA256=xxx
EXPO_PUBLIC_API_PIN_EXPIRES_AT=2026-01-01
EXPO_PUBLIC_SOCKET_PIN_EXPIRES_AT=2026-01-01
```

These are written from GitHub Secrets at build time. The `EXPO_PUBLIC_*` prefix is required by Expo to expose them at runtime.

---

## Workflow Input Customization

### Add new input

```yaml
on:
  workflow_dispatch:
    inputs:
      my-input:
        description: 'My custom input'
        required: true
        default: 'default-value'
        type: choice
        options:
          - option1
          - option2
```

Use in steps: `${{ github.event.inputs.my-input }}`

---

## Release Tag Patterns

Current patterns used:
- APK: `build-apk-<env>-<date>-<sha>`
- AAB: `build-aab-<env>-<date>-<sha>`
- IPA: `build-ipa-<env>-<date>-<sha>`

To change, modify the `TAG=` line in the Release step:

```yaml
TAG="release-v1.2.3-$(date +%Y%m%d)"
# or
TAG="build-$(date +%Y%m%d-%H%M%S)"
```

---

## Common Issues & Fixes

### "Permission denied" when creating release
→ `COCO_PAT` needs `repo` scope and write access to the source repo.

### "No such file: *.apk"
→ Build failed. Check the "Build APK" step logs. Common causes:
- Gradle OOM → bump metaspace
- Dependency resolution → check patch steps
- Signing error → verify keystore secrets

### iOS archive fails with signing error
→ Verify:
- P12 was exported without OpenSSL 3.x issues (use `-legacy` flag or Keychain Access)
- Provisioning profile matches the bundle ID
- Team ID is correct

### Yarn install fails
→ Check `yarn.lock` exists and `--frozen-lockfile` works. If using npm, change to `npm ci`.

### Environment not applied
→ Ensure `app.config.ts` reads `EXPO_PUBLIC_ENVIRONMENT` and applies it to bundle IDs.

---

## Minimum File Structure

For a new app, you need at minimum:

```
YourApp-Builds/
├── .github/workflows/
│   ├── build-android-apk.yml
│   ├── build-android-aab.yml
│   └── build-ios-ipa.yml
└── README.md

YourApp-Source/
├── app.config.ts
├── buildconfig.js          (if using custom signing)
├── withCustomSigning.js    (if using custom signing)
├── app/
├── package.json
└── yarn.lock
```

---

## Checklist for New App

- [ ] Source repo has `app.config.ts` with environment-aware bundle IDs
- [ ] Source repo has `buildconfig.js` with ProGuard rules (if needed)
- [ ] Source repo has `withCustomSigning.js` (if using credentials.json)
- [ ] Source repo has `.env.development`, `.env.preview`, `.env.production` (local only, gitignored)
- [ ] Builds repo created with workflow files
- [ ] `COCO_PAT` secret configured in builds repo
- [ ] Android secrets configured (keystore, passwords)
- [ ] iOS secrets configured (certificate, provisioning profile)
- [ ] Test run of all three workflows passes
- [ ] Release created with artifact on source repo
