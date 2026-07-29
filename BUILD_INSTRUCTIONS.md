# Tap Translate — Build & Distribution Guide

## Project Structure

```
android-translator/
├── app/
│   └── src/main/
│       ├── java/com/translator/app/
│       │   ├── MainActivity.kt                   ← Onboarding / permissions screen
│       │   ├── TranslationAccessibilityService.kt ← Captures selected text system-wide
│       │   ├── FloatingWidgetService.kt           ← Draws the overlay bar over other apps
│       │   ├── TranslationHelper.kt               ← Calls MyMemory EN→AR translation API
│       │   └── DictionaryHelper.kt               ← Calls Free Dictionary API
│       ├── res/
│       │   ├── layout/activity_main.xml           ← Onboarding UI
│       │   ├── layout/floating_widget.xml         ← Floating bar layout
│       │   ├── xml/accessibility_service_config.xml
│       │   └── values/  (strings, colors, themes)
│       └── AndroidManifest.xml
├── build.gradle
├── app/build.gradle
└── settings.gradle
```

---

## Prerequisites

| Tool                 | Version    | Where to get it                             |
|----------------------|------------|---------------------------------------------|
| Android Studio       | Hedgehog+  | https://developer.android.com/studio        |
| JDK                  | 17+        | Bundled with Android Studio                 |
| Android SDK          | API 34     | SDK Manager inside Android Studio           |
| Gradle               | 8.4        | Downloaded automatically by the wrapper     |

---

## Step 1 — Clone / Open the Project

1. Copy the `android-translator/` folder to your machine.
2. In Android Studio: **File → Open** → select the `android-translator/` folder.
3. Wait for Gradle sync to finish (first time takes 2–5 minutes for dependency download).

---

## Step 2 — Create a Signing Keystore (for Release APK)

Run once in your terminal. Remember the passwords — you need them every time you build.

```bash
keytool -genkeypair \
  -alias translator_key \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -keystore translator.keystore \
  -storepass YOUR_STORE_PASSWORD \
  -keypass  YOUR_KEY_PASSWORD \
  -dname "CN=Translator, OU=App, O=Dev, L=City, ST=State, C=US"
```

Copy `translator.keystore` into the `android-translator/app/` folder.

---

## Step 3 — Configure Signing in `app/build.gradle`

Uncomment and fill in the `signingConfigs.release` block:

```groovy
signingConfigs {
    release {
        storeFile     file("translator.keystore")
        storePassword "YOUR_STORE_PASSWORD"
        keyAlias      "translator_key"
        keyPassword   "YOUR_KEY_PASSWORD"
    }
}

buildTypes {
    release {
        ...
        signingConfig signingConfigs.release   // ← uncomment this line
    }
}
```

> ⚠️ Never commit your keystore or passwords to version control.

---

## Step 4 — Build the Release APK

### Option A — Android Studio GUI

1. **Build → Generate Signed Bundle / APK**
2. Choose **APK**
3. Select your keystore and fill in passwords
4. Choose **release** build variant
5. Click **Finish**
6. APK is saved to `app/release/app-release.apk`

Rename it:

```bash
mv app/release/app-release.apk translator.apk
```

### Option B — Command Line (faster for automation)

```bash
cd android-translator
./gradlew assembleRelease
# Output: app/build/outputs/apk/release/app-release.apk
cp app/build/outputs/apk/release/app-release.apk translator.apk
```

On Windows use `gradlew.bat assembleRelease` instead.

---

## Step 5 — Send via Telegram

### Sending (teacher side)

1. Open the Telegram chat with the student.
2. Tap the **📎 attach** icon → **File**.
3. Select `translator.apk` and send.
   - Telegram compresses images but sends APKs as raw files — no corruption.

### Receiving and Installing (student side)

1. Tap the file in the Telegram chat to download it.
2. Tap **Open** — Android will show the install screen.
3. If prompted: tap **Settings** → enable **Install unknown apps** for Telegram → go back.
4. Android may show a **Play Protect** warning:

   ```
   "App installed from outside the Play Store may be harmful."
   ```

   → Tap **More details** → **Install anyway**

5. The app installs and appears as **Tap Translate** in the launcher.

---

## Step 6 — First Launch: Grant Permissions

Open **Tap Translate**. The onboarding screen walks through two steps:

### Step 1 — Display Over Other Apps

- Tap **Grant Permission**
- Find **Tap Translate** in the list and enable the toggle
- Press Back

### Step 2 — Accessibility Service

- Tap **Enable Service**
- Find **Tap Translate – Text Selection** in the Installed Services list
- Tap it → enable the toggle → confirm the system dialog

> **⚠️ If the accessibility toggle is greyed out (sideloaded APK restriction)**
>
> Android 13+ may block accessibility settings for apps installed outside the
> Play Store. To unblock:
>
> 1. Press Back to leave Accessibility Settings.
> 2. Long-press the **Tap Translate** icon → tap **App Info** (ℹ️).
> 3. Tap the **⋮ three-dot menu** in the top-right corner.
> 4. Tap **Allow restricted settings**.
> 5. Return to **Tap Translate** → tap **Enable Service** again.

---

## How It Works

```
User selects text in any app
        │
        ▼
TranslationAccessibilityService
  (captures selection via TYPE_VIEW_TEXT_SELECTION_CHANGED)
        │
        ▼
FloatingWidgetService.onStartCommand()
  (renders TYPE_APPLICATION_OVERLAY bar via WindowManager)
        │
   ┌────┴─────────────────────┐
   │          │               │
   ▼          ▼               ▼
Translate  Dictionary       Speak
(MyMemory  (Free Dict       (Android
 EN→AR)     API)             TTS)
```

### APIs used (both free, no API key required)

| Feature     | API                                     | Limit                  |
|-------------|-----------------------------------------|------------------------|
| Translation | api.mymemory.translated.net             | 10 000 chars/day       |
| Dictionary  | api.dictionaryapi.dev                   | Unlimited              |
| TTS         | Android built-in `TextToSpeech` engine  | No limit, offline      |

---

## Troubleshooting

| Problem                          | Solution                                                     |
|----------------------------------|--------------------------------------------------------------|
| Floating bar doesn't appear      | Check Step 1 (overlay permission) is granted                 |
| Bar appears but shows no result  | Check internet connection; translation/dict need network     |
| Accessibility service is disabled after reboot | Normal on some OEMs; re-enable from Settings     |
| "App not installed" error        | Delete previous install first; package name conflict         |
| Gradle sync fails                | Check JDK 17+ is selected in Android Studio Preferences      |
| TTS speaks in wrong language     | Go to Android Settings → General Management → Text-to-Speech → install English (US) voice |
