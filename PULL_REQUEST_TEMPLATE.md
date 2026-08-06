## Pull request: ci: add workflow to build Debug APK and upload artifact

This pull request adds a GitHub Actions workflow at `.github/workflows/build-apk.yml` that builds a Debug APK and uploads it as an artifact on push and pull_request.

- Workflow: Build Debug APK
- Trigger: push, pull_request
- Steps:
  - Checkout
  - Set up JDK 11 (Temurin)
  - Cache Gradle
  - Run `./gradlew assembleDebug`
  - Upload artifact: `app/build/outputs/**/*.apk`

Notes:
- Produces an unsigned debug APK (installable)
- If your app module is not named `app`, update the artifact path in the workflow
- For signed release builds, add keystore and secrets and a signing task
