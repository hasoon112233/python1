# BeastFiles App build v2
# The repo only holds this workflow. The step below downloads the app source
# tarball, builds the release APK, signs it and publishes it to the fixed
# Release tag "beastfiles-app" (stable download link that never changes).
name: Build BeastFiles App

on:
  workflow_dispatch:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Set up Gradle 8.7
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.7'

      - name: Download app source
        run: |
          curl -L "https://muse.ai/files/1309900792215384/1783345876308699/848jc7uqgadbqtfb3jty8l56/beastfiles-app-src-v2.tar.gz" -o src.tar.gz
          mkdir app-src
          tar -xzf src.tar.gz -C app-src
          ls app-src

      - name: Build release APK
        working-directory: app-src
        run: gradle assembleRelease --stacktrace

      - name: Locate APK
        run: |
          APK=$(ls app-src/app/build/outputs/apk/release/*.apk | head -n 1)
          echo "found: $APK"
          cp "$APK" app-release.apk

      - name: Verify / re-sign APK (fallback)
        continue-on-error: true
        run: |
          BUILD_TOOLS=$(ls -d $ANDROID_HOME/build-tools/* | sort -V | tail -n 1)
          echo "build-tools: $BUILD_TOOLS"
          "$BUILD_TOOLS/apksigner" verify --verbose app-release.apk || {
            echo "APK not signed as expected; signing with the bundled keystore"
            cp app-release.apk app-unsigned.apk
            "$BUILD_TOOLS/apksigner" sign \
              --ks app-src/release.jks \
              --ks-pass pass:beastfiles \
              --key-pass pass:beastfiles \
              --out app-release.apk app-unsigned.apk || true
          }
          "$BUILD_TOOLS/aapt" dump badging app-release.apk | head -n 3 || true

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: beastfiles-app-apk
          path: app-release.apk
          if-no-files-found: error

      - name: Publish to Release (fixed tag beastfiles-app)
        uses: softprops/action-gh-release@v2
        with:
          tag_name: beastfiles-app
          name: BeastFiles App
          body: |
            BeastFiles App - the Android app version of the BeastFiles personal file host.
            Install app-release.apk, press the start-server button, and the full BeastFiles
            site (uploads, AES-256-GCM encrypted storage, ZIP/APK inspection,
            AI-readable share links) runs inside the app on port 8787.
          files: app-release.apk
          make_latest: true
