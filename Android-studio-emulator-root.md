
You are an automation agent. Your job is to ROOT an Android Studio Emulator running Android 16 (API 36) Google Play x86_64 using Magisk + rootAVD with the FAKEBOOTIMG workflow, end-to-end, with minimal user interaction.

Constraints / style:

- Prefer safe defaults: do not wipe emulator data unless I explicitly approve.
- Use copy/paste-safe commands and define variables once (ANDROID_HOME, ROOTAVD_DIR, AVD_NAME, RAMDISK).
- You can run shell commands and check outputs. You cannot reliably click UI, so pause and ask me ONLY when Magisk UI patching is needed.
- If you detect common failure modes (adb offline, lock files, snapshot mismatch, multiple emulator instance), apply the standard fixes and continue.

Goal state:

- `adb shell` then `su` drops to `#` root prompt.
- `id` shows uid=0.

Steps to execute:

0) Root check — always first

- Before any setup, download or patching, determine whether the emulator is already rooted.
  If it is, this entire procedure is wasted work: stop, report, and do nothing else.
- Resolve adb here, since preflight has not run yet:
  - ADB="${ANDROID_HOME:-$HOME/Android/Sdk}/platform-tools/adb"
- Run:
  - "$ADB" devices
  - "$ADB" shell "su -c 'id'"
- Interpret:
  - `uid=0(root)` (usually also `context=u:r:magisk:s0`) → already rooted. Stop and report
    the AVD name and Magisk version. Do not continue.
  - `su: inaccessible or not found` / `not found` / non-zero exit → not rooted. Continue to
    step 1.
  - `error: no devices/emulators found` → the emulator is not running. That is normal on a
    cold start: continue to step 1, boot it in step 3, and **re-run this check after boot
    completes and before step 4**.
- Only `uid=0` counts as rooted. A denied or missing `su` prompt is not "already rooted".

1) Preflight: confirm environment and paths

- Determine Android SDK path:
  - Default assumption: ANDROID_HOME="$HOME/Android/Sdk"
  - If that path doesn’t exist, ask me for the correct Android SDK path.
- Verify these exist and are executable:
  - "$ANDROID_HOME/platform-tools/adb"
  - "$ANDROID_HOME/emulator/emulator"
- Print emulator + adb versions.
- List available AVDs with:
  - "$ANDROID_HOME/emulator/emulator" -list-avds
- Ask me to choose the AVD name from the output (unless there is only one). Set:
  - AVD_NAME="(chosen name)"
- Set:
  - RAMDISK="$ANDROID_HOME/system-images/android-36/google_apis_playstore/x86_64/ramdisk.img"
- Validate RAMDISK exists:
  - ls -la "$RAMDISK"
  - If it fails, guide me to install the correct system image (API 36, Google Play, x86_64) and re-check.

2) Install rootAVD (if not present)

- If "$ROOTAVD_DIR/rootAVD.sh" doesn’t exist:
  - cd "$HOME"
  - git clone https://gitlab.com/newbit/rootAVD.git rootAVD
  - cd "$ROOTAVD_DIR"
  - chmod +x rootAVD.sh
- Set:
  - ROOTAVD_DIR="$HOME/rootAVD"

3) Boot emulator cleanly (avoid snapshot pitfalls)

- Start emulator in background (so the agent can keep working):
  - "$ANDROID_HOME/emulator/emulator" -avd "$AVD_NAME" -no-snapshot-load -no-snapshot-save &
- Wait for boot completion:
  - "$ANDROID_HOME/platform-tools/adb" wait-for-device
  - "$ANDROID_HOME/platform-tools/adb" shell 'while [ -z "$(getprop sys.boot_completed)" ]; do sleep 1; done'
- Confirm device is online:
  - "$ANDROID_HOME/platform-tools/adb" devices

4) First rootAVD run (generate fakeboot.img + install Magisk)

- cd "$ROOTAVD_DIR"
- Run:
  - ./rootAVD.sh "$RAMDISK" FAKEBOOTIMG
- If it prompts for Magisk version selection, choose the default stable/local stable (typically option 1) non-interactively if possible; otherwise, prompt me with the exact question and choices.

5) USER INTERACTION REQUIRED (Magisk UI patch)

- Tell me exactly what to do on the emulator screen:
  - Open Magisk app
  - Install → Select and Patch a File
  - Choose /sdcard/Download/fakeboot.img
  - Wait for “All done”
- Then you (agent) verify the patched file exists:
  - "$ANDROID_HOME/platform-tools/adb" shell ls -la /sdcard/Download/ | grep -i magisk_patched
- If not found, ask me what Magisk showed and retry guidance.

6) Second rootAVD run (detect magisk_patched, repack, inject)

- cd "$ROOTAVD_DIR"
- Run again:
  - ./rootAVD.sh "$RAMDISK" FAKEBOOTIMG
- Confirm output indicates it found magisk_patched and repacked/pulled/pushed successfully.

7) Reboot emulator and handle lock/offline issues automatically

- If emulator is still running but adb becomes offline or emulator refuses to start again, apply:
  - killall -9 qemu-system-x86_64 2>/dev/null || true
  - rm -f "$HOME/.android/avd/${AVD_NAME}.avd"/*.lock
- Start emulator again (background):
  - "$ANDROID_HOME/emulator/emulator" -avd "$AVD_NAME" -no-snapshot-load -no-snapshot-save &
- Wait for boot:
  - "$ANDROID_HOME/platform-tools/adb" wait-for-device
  - "$ANDROID_HOME/platform-tools/adb" shell 'while [ -z "$(getprop sys.boot_completed)" ]; do sleep 1; done'
- Confirm:
  - "$ANDROID_HOME/platform-tools/adb" devices

8) Magisk Additional Setup & Restart

- Tell me to open the Magisk app.
- Ask me to confirm any "Additional Setup" prompt inside Magisk, let it run, and let it restart the emulator automatically.
- Wait for the emulator to boot again:
  - "$ANDROID_HOME/platform-tools/adb" wait-for-device
  - "$ANDROID_HOME/platform-tools/adb" shell 'while [ -z "$(getprop sys.boot_completed)" ]; do sleep 1; done'
- Ask me to open the Magisk app one more time.

9) Verify root

- Open an adb shell and run su:
  - "$ANDROID_HOME/platform-tools/adb" shell
  - Inside the shell, run: su
- Ask me to approve the Superuser request inside the Magisk app.
- Finally, verify root identity:
  - run: id
- If it works, print the output and declare success.

Troubleshooting rules (apply automatically if encountered):

- If `adb devices` shows `offline`: restart adb server and re-wait:
  - "$ANDROID_HOME/platform-tools/adb" kill-server
  - "$ANDROID_HOME/platform-tools/adb" start-server
  - "$ANDROID_HOME/platform-tools/adb" wait-for-device
- If emulator says “Running multiple emulators with the same AVD…”:
  - Ensure no emulator/qemu is running; kill qemu; remove *.lock; retry.
- If snapshot mismatch appears:
  - Always run with -no-snapshot-load -no-snapshot-save. Do not use snapshots during rooting.

Output requirement:

- At the end, print a short “Done” summary including the exact AVD_NAME used and the verification command/output that proves root (su -c id).
