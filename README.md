name: 🚀 Casabaia V3.2 - Build APK

on:
  workflow_dispatch:
  push:
    branches:
      - principal
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:

      # 1. Récupérer le dépôt
      - name: 📥 Télécharger le projet
        uses: actions/checkout@v4

      # 2. Installer 7-Zip
      - name: 📦 Installer 7-Zip
        run: |
          sudo apt-get update
          sudo apt-get install -y p7zip-full

      # 3. Extraire automatiquement le 7Z
      - name: 📂 Extraire le projet Casabaia
        run: |
          mkdir -p extracted

          ARCHIVE=$(find . -maxdepth 1 -type f -name "*.7z" | head -n 1)

          if [ -z "$ARCHIVE" ]; then
            echo "❌ Aucun fichier .7z trouvé"
            exit 1
          fi

          echo "Archive trouvée : $ARCHIVE"

          7z x "$ARCHIVE" -oextracted

          echo "Contenu extrait :"
          find extracted -maxdepth 3 -type f | head -100

      # 4. Trouver automatiquement le projet Flutter
      - name: 🔎 Trouver pubspec.yaml
        run: |
          PUBSPEC=$(find extracted -name pubspec.yaml -type f | head -n 1)

          if [ -z "$PUBSPEC" ]; then
            echo "❌ pubspec.yaml introuvable"
            exit 1
          fi

          PROJECT_DIR=$(dirname "$PUBSPEC")

          echo "Projet Flutter trouvé dans : $PROJECT_DIR"

          echo "PROJECT_DIR=$PROJECT_DIR" >> $GITHUB_ENV

      # 5. Installer Java
      - name: ☕ Installer Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      # 6. Installer Flutter
      - name: 🐦 Installer Flutter
        uses: subosito/flutter-action@v2
        with:
          channel: stable
          cache: true

      # 7. Vérifier Flutter
      - name: 🔧 Vérifier Flutter
        run: |
          flutter --version
          dart --version

      # 8. Préparer Android
      - name: 🤖 Préparer Android
        working-directory: ${{ env.PROJECT_DIR }}
        run: |
          flutter create \
            --platforms=android \
            --org ma.casabaia \
            .

      # 9. Installer les dépendances
      - name: 📚 Installer les dépendances
        working-directory: ${{ env.PROJECT_DIR }}
        run: |
          flutter pub get

      # 10. Appliquer les permissions GPS / caméra / Internet
      - name: 📍📷 Configurer les permissions
        working-directory: ${{ env.PROJECT_DIR }}
        run: |
          if [ -f "tool/patch_android_manifest.dart" ]; then
            dart run tool/patch_android_manifest.dart
          else
            echo "Script de permissions absent - on continue"
          fi

      # 11. Tests
      - name: 🧪 Tests Flutter
        working-directory: ${{ env.PROJECT_DIR }}
        run: |
          flutter test
        continue-on-error: true

      # 12. Analyse
      - name: 🔍 Analyse Flutter
        working-directory: ${{ env.PROJECT_DIR }}
        run: |
          flutter analyze
        continue-on-error: true

      # 13. FABRICATION DE L'APK
      - name: 🏗️ Construire APK Release
        working-directory: ${{ env.PROJECT_DIR }}
        run: |
          flutter build apk --release

      # 14. Vérifier que l'APK existe
      - name: ✅ Vérifier APK
        working-directory: ${{ env.PROJECT_DIR }}
        run: |
          ls -lh build/app/outputs/flutter-apk/
          test -f build/app/outputs/flutter-apk/app-release.apk

      # 15. Récupérer l'APK
      - name: 📦 Télécharger APK
        uses: actions/upload-artifact@v4
        with:
          name: collecte-casabaia-v3-2-APK
          path: ${{ env.PROJECT_DIR }}/build/app/outputs/flutter-apk/app-release.apk
          if-no-files-found: error
