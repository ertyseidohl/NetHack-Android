NOTE: This plan was generated with Gemini 3.1 Pro (High). It may contain inaccuracies or hallucinations.

# NetHack-Android: Local Build, NetHack 5.0.0 Upgrade, and Play Store Publishing Plan

This document outlines the roadmap to:
1. Compile and run NetHack-Android locally on Linux Mint (x86_64).
2. Upgrade the engine from NetHack 3.6.7 to NetHack 5.0.0 (C99 compliance, Lua 5.4, updated windowport, tile index fallback).
3. Publish the application to Google Play as a free application complying with modern Android standards (Target API 34+, 16 KB page sizes, Scoped Storage, 64-bit ABIs, AAB format).

---

## 0. Technical Context: Issues with the Legacy Build

The original Gurr NetHack-Android codebase stopped working on modern Android devices due to:
1. **Target SDK Minimums (Android 14+)**: Android 14 blocks installation of apps targeting below Android 6.0 (`targetSdkVersion < 23`) with `INSTALL_FAILED_DEPRECATED_SDK_VERSION`. The legacy app targeted API 10–14.
2. **64-Bit-Only Devices**: Modern devices (Pixel 7+, Galaxy S24, Snapdragon 8 Gen 3) run 64-bit-only kernels and do not support 32-bit execution. The legacy app only bundled 32-bit `armeabi`/`armeabi-v7a` binaries.
3. **Scoped Storage (Android 11+)**: The legacy code wrote directly to `/sdcard/NetHack/` using `WRITE_EXTERNAL_STORAGE`. On Android 11+, external POSIX access without SAF returns `EACCES`, crashing file initialization.
4. **JCenter Shutdown**: The build dependencies relied on `com.tbd.forkfront:lib:1.1` hosted on JCenter, which was decommissioned in 2021.
5. **Google Play Delisting**: Google Play removes apps from discovery when their `targetSdkVersion` falls behind requirements (API 34+).

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                        NetHack-Android App                             │
├──────────────────────────────────┬─────────────────────────────────────┤
│      Android Frontend (Java)     │       Native Core (C / C99)         │
│  ┌────────────────────────────┐  │  ┌───────────────────────────────┐  │
│  │   ForkFront UI Framework   │  │  │   NetHack 5.0.0 Engine        │  │
│  │ (Activities, Input, Tiles, │  │  │ (Dungeon, Monsters, Objects,  │  │
│  │   Virtual D-Pad, Settings) │  │  │  Turn Engine, Save/Restore)   │  │
│  └─────────────┬──────────────┘  │  └───────────────▲───────────────┘  │
│                │                 │                  │                  │
│                ▼                 │                  │                  │
│  ┌────────────────────────────┐  │  ┌───────────────┴───────────────┐  │
│  │      JNI Bridge Layer      │◄─┼─►│   sys/android Windowport      │  │
│  │  (RunNetHack, Key events,  │  │  │ (winandroid.c, androidmain.c, │  │
│  │   Window/Tile callbacks)   │  │  │        window_procs)          │  │
│  └────────────────────────────┘  │  └───────────────▲───────────────┘  │
│                                  │                  │                  │
│                                  │  ┌───────────────┴───────────────┐  │
│                                  │  │   Embedded Lua 5.4 Runtime    │  │
│                                  │  │ (Level/Dungeon script engine) │  │
│                                  │  └───────────────────────────────┘  │
└──────────────────────────────────┴─────────────────────────────────────┘
```

---

## Phase 1: Local Build Setup on Linux Mint (x86_64)

### Step 1.1: System Packages & Toolchain
1. Install base build packages:
   ```bash
   sudo apt update
   sudo apt install -y build-essential gcc-multilib bison flex openjdk-17-jdk openjdk-21-jdk git curl unzip
   ```
2. Set and verify Java environment:
   ```bash
   export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
   export PATH="$JAVA_HOME/bin:$PATH"
   java -version
   ```
3. Install Android Command-line Tools, SDK, and NDK:
   ```bash
   mkdir -p ~/Android/Sdk/cmdline-tools
   cd ~/Android/Sdk/cmdline-tools
   curl -O https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
   unzip commandlinetools-linux-*_latest.zip
   mv cmdline-tools latest

   export ANDROID_HOME="$HOME/Android/Sdk"
   export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH"

   # NDK r27b supports 16 KB page alignment
   yes | sdkmanager --licenses
   sdkmanager --install "platforms;android-34" "build-tools;34.0.0" "ndk;27.1.12297006" "platform-tools"
   ```

### Step 1.2: Vendor ForkFront Library Source
1. Clone `ForkFront-Android` into `sys/android/forkfront`:
   ```bash
   git clone https://github.com/gurrhack/ForkFront-Android.git sys/android/forkfront
   ```
2. Update `sys/android/settings.gradle`:
   ```groovy
   include ':app', ':forkfront'
   project(':forkfront').projectDir = new File(settingsDir, 'forkfront/lib')
   ```
3. In `sys/android/app/build.gradle`, replace:
   ```groovy
   implementation 'com.tbd.forkfront:lib:1.1'
   ```
   with:
   ```groovy
   implementation project(':forkfront')
   ```
4. **JNI Package Constraint**:
   Keep the Java package name `com.tbd.forkfront` inside the ForkFront library unchanged. The C functions in `sys/android/winandroid.c` bind to this exact name:
   ```c
   void Java_com_tbd_forkfront_NetHackIO_RunNetHack(JNIEnv* env, jobject thiz, jstring path, jstring username);
   void Java_com_tbd_forkfront_NetHackIO_SaveNetHackState(JNIEnv* env, jobject thiz);
   ```
   Changing this package name requires renaming all corresponding JNI symbols in `winandroid.c`.

### Step 1.3: Configure Native Makefiles
1. **NDK Path Configuration in `sys/android/Makefile.src`**:
   Set `NDK` to the installed NDK directory:
   ```makefile
   NDK = $(HOME)/Android/Sdk/ndk/27.1.12297006
   NDK_TOOLCHAIN = $(NDK)/toolchains/llvm/prebuilt/linux-x86_64/bin
   ```
2. **Linker Configuration**:
   NDK r22+ removed GNU `ld`. Link shared objects with Clang directly using `-shared`.
   Add 16 KB page-size alignment to `LFLAGS`:
   ```makefile
   LFLAGS = -shared -Wl,-z,max-page-size=16384
   ```
3. **Build Target ABIs**:
   Compile `libnethack.so` for:
   - `arm64-v8a`
   - `armeabi-v7a`
   - `x86_64`

```bash
cd src
# 1. Build arm64-v8a (modern 64-bit ARM devices)
make spotless
make Sysandroid ABI=arm64-v8a
# 2. Build armeabi-v7a (32-bit ARM devices)
make spotless
make Sysandroid ABI=armeabi-v7a
# 3. Build x86_64 (emulator)
make spotless
make Sysandroid ABI=x86_64
```

4. **Data File Installation**:
   `make install` copies game data files (`dat/*`, `license`, `help`, `rumors`, `dungeon`, special levels) into `sys/android/app/assets/nethackdir/`. Run this in the top level dir, not /src.

### Step 1.4: Update Gradle Configuration
1. Update `gradle/wrapper/gradle-wrapper.properties` to Gradle 8.7:
   ```properties
   distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip
   ```
2. Update root `sys/android/build.gradle` to AGP 8.5+:
   ```groovy
   buildscript {
       repositories {
           google()
           mavenCentral()
       }
       dependencies {
           classpath 'com.android.tools.build:gradle:8.5.2'
       }
   }
   ```
3. Update `app/build.gradle`:
   - Set `compileSdkVersion 34`
   - Set `targetSdkVersion 34`
   - Set `minSdkVersion 21`
   - Set `namespace 'me.erty.NetHack'` (my domain name is erty.me)
   - Set Java 17 compile options
   - Set uncompressed native library packaging:
     ```groovy
     packaging {
         jniLibs {
             useLegacyPackaging = false
         }
     }
     ```
4. Clean permissions in `AndroidManifest.xml`:
   - Remove `android.permission.WRITE_EXTERNAL_STORAGE`
   - Remove `android.permission.INTERNET`
   - Add in application: android:requestLegacyExternalStorage="true"
   - Add in application: android:preserveLegacyExternalStorage="true"

### Step 1.5: Local Testing on Linux Mint
1. Build native binaries:
   ```bash
   cd sys/android
   sh setup.sh
   cd ../..
   make clean
   make ABI=arm64-v8a install
   make clean
   make ABI=x86_64 install
   ```
2. Build debug APK:
   ```bash
   cd sys/android
   ./gradlew assembleDebug # may need chmod +x first if freshly cloned.
   ```
3. Install and run on the Android Emulator (Linux Mint): see [Appendix A: Android Emulator Setup](#appendix-a-android-emulator-setup-linux-mint).
4. Verify:
   - Asset extraction into internal app storage
   - Character selection
   - Map rendering (tiles and ASCII)
   - Virtual controls
   - Save and restore

---

## Phase 2: Upgrade to NetHack 5.0.0

### Step 2.1: Git Workflow
1. Add official upstream remote:
   ```bash
   git remote add upstream https://github.com/NetHack/NetHack.git
   git fetch upstream --tags
   ```
2. Create upgrade branch:
   ```bash
   git checkout -b upgrade/nethack-5.0.0
   ```
3. Merge tag `NetHack-5.0.0_Released`:
   ```bash
   git merge NetHack-5.0.0_Released -m "Merge upstream NetHack 5.0.0"
   ```
   Resolve conflicts, retaining `sys/android/` files while updating common headers and game engine sources.

### Step 2.2: Lua 5.4 Cross-Compilation
NetHack 5.0.0 requires Lua 5.4 for levels and text processing:
1. Vendor or fetch Lua 5.4 sources via `make fetch-Lua`.
2. Compile Lua:
   - **Host Lua**: Compiled with host `gcc` for data preparation tools.
   - **Target `liblua.a`**: Cross-compiled using the NDK Clang for each ABI (`arm64-v8a`, `armeabi-v7a`, `x86_64`) with `-DLUA_USE_POSIX`. Ensure calls unavailable in Bionic are disabled or stubbed.
3. Link `liblua.a` into `libnethack.so` in `Makefile.src`.

### Step 2.3: Port `winandroid.c` to C99 and 5.0 Windowport Interface
1. Update function signatures from K&R / `FDECL` macros to standard C99.
2. Review `struct window_procs and_procs` against 5.0.0 `winprocs.h`:
   - Identify new callback hooks added in 5.0.0.
   - Assign valid `genl_*` stubs or custom handlers to prevent null pointer dereferences.
3. Update `status_update` and `bl_conditions` to match 5.0.0 player condition flags.
4. Save file handling: 5.0.0 saves are binary-incompatible with 3.6.x. Isolate the 5.0.0 save directory to avoid crashing on legacy saves.

### Step 2.4: Tileset Pipeline & Bounds Handling
1. Regenerate `tile.c` and `glyph2tile` using 5.0 tile compilation tools.
2. In ForkFront, add a bounds check for tile indices:
   ```java
   if (tileIndex >= currentTileset.getTotalTiles()) {
       tileIndex = fallbackDefaultTileIndex(glyph);
   }
   ```
   This prevents crashes when legacy tilesets (`dawnhack`, `nevanda`) encounter new 5.0 entities.

### Step 2.5: Build and Runtime Testing
1. Compile for all target ABIs (`arm64-v8a`, `armeabi-v7a`, `x86_64`).
2. Run test sessions:
   - Tutorial mode
   - Lua-driven special levels
   - Tile rendering across all bundled tilesets
   - Save / load cycles

---

## Phase 3: Publish to Google Play Store (Free)

### Step 3.1: Technical Play Store Requirements
1. **Target API**: Set `targetSdkVersion 34` in `app/build.gradle`.
2. **Architectures**: Include `arm64-v8a`, `x86_64`, and `armeabi-v7a`.
3. **16 KB Page Alignment**: Ensure `libnethack.so` has 16 KB segment alignment (`-Wl,-z,max-page-size=16384`) and `useLegacyPackaging = false`.
4. **Scoped Storage**: Use `context.getFilesDir()` for saves, bones, and config. Do not request external storage permissions.
5. **Application ID**: Set a unique `applicationId` in `app/build.gradle` (e.g., `org.nethack.android.classic`). Do not rename the Java package `com.tbd.forkfront`.
6. **Android App Bundle (.aab)**:
   ```bash
   ./gradlew bundleRelease
   ```
7. **Keystore Generation**:
   ```bash
   keytool -genkey -v -keystore nethack-release.jks -alias nethack-key -keyalg RSA -keysize 2048 -validity 10000
   ```

### Step 3.2: Legal and Policy Requirements
1. **Application Naming & Trademark**:
   "NetHack" is trademarked by the NetHack DevTeam. Use a descriptive name such as **"Port for NetHack (Classic)"**.
   Include the disclaimer in the store description:
   > "This is an unofficial, open-source community port licensed under the NetHack General Public License (NGPL). It is not developed, affiliated with, or endorsed by the NetHack DevTeam."
2. **License Compliance**:
   - Include the NGPL and ForkFront GPLv2 in the application's About/Credits screen.
   - Provide a link to the public GitHub repository.
3. **Privacy Policy**:
   Host `PRIVACY.md` publicly on GitHub Pages via HTTPS.

### Step 3.3: Play Console Account & Testing
1. Register a Google Play Developer account ($25 fee) and complete identity verification.
2. **Testing Track Requirements**:
   - Personal accounts created after November 13, 2023 require at least **20 testers opted in for 14 continuous days** in closed testing before production access can be requested.
   - Organization accounts (requiring D-U-N-S verification) do not have this requirement.

### Step 3.4: Store Listing Assets
1. **Graphics**:
   - App Icon: 512 x 512 px, 32-bit PNG.
   - Feature Graphic: 1024 x 500 px, PNG or JPEG.
   - Screenshots: Phone and tablet (7" and 10") showing gameplay and virtual controls.
2. **Questionnaires**:
   - IARC Content Rating: Complete questionnaire (Fantasy Violence).
   - Data Safety: Declare no user data collected or shared.
   - Pricing: Free ($0.00).

---

## Phase 4: CI/CD Pipeline

Update `.github/workflows/android.yml`:
1. Runner: `ubuntu-24.04`.
2. Setup Java 17 and Android SDK 34 / NDK r27b.
3. Multi-ABI build matrix (`arm64-v8a`, `armeabi-v7a`, `x86_64`).
4. Generate release `.aab` and `.apk` on Git tags.

---

## Milestone Checklist

| Milestone | Deliverables | Verification |
|---|---|---|
| **M1: Host Toolchain** | JDK 17, Android SDK, NDK r27b installed on Linux Mint | `sdkmanager` and `gcc` functional |
| **M2: ForkFront Vendoring** | ForkFront sources cloned into `sys/android/forkfront` | Library module compiles without JCenter |
| **M3: Native 16 KB Build** | `Makefile.src` configured with NDK r27b and 16 KB alignment | `libnethack.so` built for `arm64-v8a`, `armeabi-v7a`, `x86_64` |
| **M4: 3.6.7 Verification** | AGP 8 / Gradle 8 build with internal storage | NetHack 3.6.7 runs in emulator/device |
| **M5: 5.0.0 Upgrade** | Upstream 5.0.0 merged, Lua 5.4 linked, C99 windowport updated | NetHack 5.0.0 runs tutorial and special levels |
| **M6: Tileset Fallback** | Runtime bounds check in ForkFront | No crashes with legacy tilesets on new entities |
| **M7: Play Store Technical Prep** | Target SDK 34, unique `applicationId`, signed AAB | `./gradlew bundleRelease` passes Play Console checks |
| **M8: Production Release** | Store assets uploaded, closed testing completed | Production release published |

---

## Appendix A: Android Emulator Setup (Linux Mint)

Prerequisites: Android SDK and NDK already installed per Step 1.1, with `ANDROID_HOME=$HOME/Android/Sdk`.

### A.1 Enable KVM Hardware Acceleration

Your CPU has Intel VT-x (`vmx`). Load the KVM module and add your user to the `kvm` group:

```bash
sudo modprobe kvm_intel
sudo usermod -aG kvm $USER
# Log out and then back in
```

### A.2 Install the Emulator and System Image

```bash
export ANDROID_HOME="$HOME/Android/Sdk"
export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$PATH"

sdkmanager --install "emulator" "system-images;android-34;google_apis;x86_64"
```

### A.3 Create an Android Virtual Device (AVD)

```bash
avdmanager create avd \
  -n NetHackEmulator \
  -k "system-images;android-34;google_apis;x86_64" \
  -d "pixel_7"
# Answer "no" if asked about a custom hardware profile.
```

### A.4 Launch the Emulator

```bash
$ANDROID_HOME/emulator/emulator -avd NetHackEmulator &
# Wait ~60 seconds for the Android home screen to appear.
```

Verify it is visible to `adb`:
```bash
adb devices
# Expected: emulator-5554  device
```

### A.5 Install and Launch the NetHack APK

From the repo root (`~/workspace/NetHack-Android`):

```bash
adb install -r sys/android/app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n me.erty.nethack/com.tbd.forkfront.ForkFront
```

Or simply tap the NetHack icon in the emulator's app drawer.
