# Task: root a Google Pixel 6a (bluejay) — persistent Magisk root, no data loss

You are driving a Pixel 6a over `adb` and `fastboot`. Your goal is persistent Magisk root
with **zero user-data loss**.

Work through the steps in order. Each step has a gate. Do not advance past a gate that has
not passed. Do not skip the root check, Setup, or Step 0.

**No UI interaction is required for the rooting itself.** Everything is driven from the
shell, including the boot-image patching — which runs on-device through Magisk's own
`boot_patch.sh`, not through the Magisk app.

There is exactly **one** possible tap: on the first `su`, MagiskSU may show a *Superuser
request* for the `shell` user (see Step 5). If it appears, ask the user to tap **Grant**.
Nothing else in this procedure needs the screen.

---

## Hard rules

These override everything else, including a request to "just make it work".

1. **Never touch `vbmeta`.** Do not run `fastboot flash vbmeta`, and never pass
   `--disable-verity` or `--disable-verification`. This is the only known path to
   unrecoverable `/data`.
2. **Never wipe.** No `fastboot -w`, no `erase userdata`, no factory reset.
3. **Never write a boot image that does not match the running build.** A mismatch either
   boots with no `su`, or bootloops.
4. **Never write a partition before `fastboot boot` has proven the image.** `fastboot boot`
   writes nothing — it is your test.
5. **The only partition you may write is `boot`, on the active slot.**
6. **Never guess a download URL or a hash.** Derive the URL, then verify the file against
   the vendor-published SHA-256 before using it.
7. **Query the device for facts. Never assume them.** Build, slot and lock state change
   without warning via OTA.
8. If the bootloader is **locked**, stop and report. Unlocking wipes the device and is the
   user's call, not yours.

---

## First — check for existing root (do this before anything else)

Before setup, before downloading anything, before patching: establish whether the device is
**already rooted**. If it is, this entire procedure is wasted work. Stop and say so.

```bash
adb devices -l
adb shell "su -c 'id'"
```

| Result | Meaning | Action |
|---|---|---|
| `uid=0(root)` (usually also `context=u:r:magisk:s0`) | already rooted | **Stop.** Report the build, slot and Magisk version, and do nothing else. |
| `su: inaccessible or not found`, `not found`, or a non-zero exit | not rooted | continue to *Known-good combination* |
| `error: no devices/emulators found` | not connected | connect and authorise the device, then re-run this check |

Only `uid=0` counts as rooted. A denied prompt, an absent `su`, or any error is **not**
"already rooted" — do not report success on those.

---

## Known-good combination

- **Device:** Pixel 6a, codename `bluejay`. It has **no `init_boot` partition**, and its
  ramdisk may live in `vendor_boot` rather than `boot` — see the Step 4 note.
- **Verified working:** Android 17 (SDK 37), Magisk **v30.7** (versionCode 30700), build
  `CP2A.260705.006`, active slot `_a`.
- **Re-derive the build and slot on every run.** The build string above is an example of a
  combination that worked, not a constant to hardcode.

---

## Setup — host toolchain and preflight (once per machine)

Do this before Step 0. A missing tool discovered halfway through a flash is a self-inflicted
failure, so fail fast here.

### Define the variables once

Set these in your shell and reuse them for the rest of the task. Every later command is
copy/paste-safe as written.

```bash
export ANDROID_HOME="${ANDROID_HOME:-$HOME/Android/Sdk}"
export ADB="$ANDROID_HOME/platform-tools/adb"
export FASTBOOT="$ANDROID_HOME/platform-tools/fastboot"
export WORK="$HOME/pixel6a-root"          # host scratch directory
export PATH="$ANDROID_HOME/platform-tools:$PATH"
mkdir -p "$WORK"
```

`PATH` is exported deliberately: later blocks call bare `adb` and `fastboot` for
readability. Without this line they only work if platform-tools happens to already be on
your `PATH` — the `ls` check below uses the variables and would pass anyway.

If the SDK lives somewhere else, set `ANDROID_HOME` explicitly before continuing.

### Verify the Android tools

```bash
ls -l "$ADB" "$FASTBOOT"            # both must exist and be executable
"$ADB" version
"$FASTBOOT" --version
```

If either is missing, install Android SDK Platform-Tools and re-check. Do not continue with
an unknown system-packaged `adb`.

### Verify the host tools

```bash
for t in curl unzip strings sha256sum; do
  command -v "$t" >/dev/null || echo "MISSING: $t"
done
```

### Get the Magisk APK onto the host

```bash
MAGISK_VER=v30.7
curl -L -o "$WORK/Magisk-$MAGISK_VER.apk" \
  "https://github.com/topjohnwu/Magisk/releases/download/$MAGISK_VER/Magisk-$MAGISK_VER.apk"
sha256sum "$WORK/Magisk-$MAGISK_VER.apk"
```

Compare against the SHA-256 published on that release page. **Do not proceed on a mismatch.**

### Confirm USB debugging authorisation

```bash
"$ADB" devices -l
```

**Gate:** the device is listed as `device`, not `unauthorized`.

| Output           | Fix                                                                     |
| ---------------- | ----------------------------------------------------------------------- |
| `unauthorized` | accept the USB debugging prompt on the phone, then re-run               |
| nothing listed   | check the cable; confirm USB debugging is enabled in Developer options  |
| `offline`      | `"$ADB" kill-server && "$ADB" start-server && "$ADB" wait-for-device` |

### Safety checkpoint

Confirm both of these before going any further:

- The bootloader is **already unlocked** (`ro.boot.flash.locked=0`). If it is locked, **stop
  and ask** — unlocking wipes the device and is the user's decision.
- You will not touch `vbmeta`, and you will not wipe. See *Hard rules*.

### Device-class facts you will need later

- Pixel 6a (`bluejay`) has **no `init_boot` partition**.
- Its `boot.img` may be **kernel-only**, with the ramdisk in `vendor_boot.img`. That is
  expected, not a broken download — see the Step 4 note.
- `vendor_boot` and `vbmeta` exist on the device but are **read-only in this procedure**.

---

## Step 0 — Recon

```bash
adb devices -l
adb shell 'getprop ro.build.id; getprop ro.boot.slot_suffix; \
           getprop ro.boot.verifiedbootstate; getprop ro.boot.flash.locked; \
           getprop ro.boot.veritymode; getprop ro.build.fingerprint; cat /proc/version'
```

**Gate:** `adb devices -l` shows `device` (not `unauthorized`), and
`ro.boot.flash.locked=0`.

Record these and use them for the rest of the task:

| Needed        | Source                                                  |
| ------------- | ------------------------------------------------------- |
| Build number  | `ro.build.id` (e.g. `CP2A.260705.006`)              |
| Active slot   | `ro.boot.slot_suffix` (`_a` → write to `boot_a`) |
| Kernel string | `cat /proc/version`                                   |

Take the build from `ro.build.id` directly. Do **not** slice it out of
`ro.build.fingerprint`: that string is `brand/device/codename:release/build/incremental:type/tags`,
so counting slashes yields the codename, not the build.

If the bootloader is locked, stop here and report.

---

## Step 1 — Get the factory image for exactly that build

Factory images are named `<device>-<build>-factory-<hash>.zip`, where `<hash>` is the first
8 hex characters of the file's own SHA-256.

```bash
cd "$WORK"
export BUILD=cp2a.260705.006     # lowercase form of ro.build.id from Step 0
export FACTORY="$WORK/bluejay-${BUILD}-factory-XXXXXXXX.zip"   # paste the real hash
curl -L --retry 5 -o "$FACTORY" \
  "https://dl.google.com/dl/android/aosp/$(basename "$FACTORY")"
```

To obtain the real `<hash>` and the published checksum: open
`https://developers.google.com/android/images` in a browser, click the terms
**Acknowledge** button, then read the row for `bluejay` + your build number. The page is
JavaScript-rendered and gated, so a plain HTTP fetch of it returns no download links.

**Gate — verify the download twice:**

```bash
sha256sum "$FACTORY"
```

1. The result must **equal the SHA-256 published on that page**.
2. The result must **start with the same 8 characters as the `<hash>` in the filename**.

If either check fails, do not use the file. Re-download or find the correct URL.

---

## Step 2 — Extract the stock `boot.img`

```bash
unzip "$FACTORY" '*image-*.zip'
unzip "bluejay-${BUILD}/image-bluejay-${BUILD}.zip" boot.img
sha256sum boot.img
```

`unzip` recreates the inner zip at its stored path, `bluejay-<build>/image-bluejay-<build>.zip`,
which is why the second command names that directory explicitly. Getting this path wrong is
the difference between extracting `boot.img` and silently extracting nothing.

---

## Step 3 — Prove the image matches the device *before* patching

A build mismatch is the single most common cause of failure, and it is silent until boot.
Use three checks. Only the third is conclusive.

**(a) Build number — the primary gate.** The `<build>` in the factory image filename must
equal `ro.build.id` from Step 0. If it does not, you have the wrong image: return to Step 1.

**(b) Kernel version token — corroborating.**

```bash
strings boot.img | grep -o '6\.[0-9]*\.[0-9]*-android[0-9]*-[0-9]*-g[0-9a-f]*'
# compare the token with the one in `cat /proc/version` from Step 0
```

Do **not** grep for `^Linux version` here. `boot.img` carries a compressed kernel, so
`strings` returns fragments — the version token sits mid-string and an anchored match finds
nothing. **An empty result from an anchored grep is not evidence of a mismatch**, and
treating it as one is how you end up in a loop re-downloading a correct image.

**(c) Partition hash — conclusive, but needs root.**

Once `fastboot boot` (Step 5) has given you root, compare the on-disk partition with the
image you extracted:

```bash
adb shell "su -c 'sha256sum /dev/block/by-name/boot_a'"
sha256sum boot.img
```

Identical hashes prove the extracted image is byte-for-byte the stock boot image of the
running build. This is the check to trust, and Step 6 makes it mandatory before writing.

---

## Step 4 — Patch `boot.img` with Magisk (no root required)

Magisk's own `boot_patch.sh` runs fine as the unprivileged `shell` user, so you do not need
root to gain root.

```bash
unzip "$WORK/Magisk-v30.7.apk" 'assets/*' 'lib/arm64-v8a/*' -d apk_x

mkdir patchdir
cp apk_x/lib/arm64-v8a/libmagiskboot.so patchdir/magiskboot
cp apk_x/lib/arm64-v8a/libmagiskinit.so patchdir/magiskinit
cp apk_x/lib/arm64-v8a/libmagisk.so     patchdir/magisk
cp apk_x/lib/arm64-v8a/libinit-ld.so    patchdir/init-ld
cp apk_x/lib/arm64-v8a/libbusybox.so    patchdir/busybox
cp apk_x/assets/stub.apk                patchdir/
cp apk_x/assets/boot_patch.sh           patchdir/
cp apk_x/assets/util_functions.sh       patchdir/
chmod 755 patchdir/*

adb push patchdir/. /data/local/tmp/mp/
adb push boot.img  /data/local/tmp/mp/boot.img

adb shell 'cd /data/local/tmp/mp && \
  KEEPVERITY=true KEEPFORCEENCRYPT=true PATCHVBMETAFLAG=false RECOVERYMODE=false \
  ./busybox sh ./boot_patch.sh boot.img'

adb shell 'ls -l /data/local/tmp/mp/new-boot.img'
```

Rules that make this work:

- **The renamed file names are mandatory.** `boot_patch.sh` looks for `magiskboot`,
  `magiskinit`, `magisk`, `init-ld` and `stub.apk` in its own directory.
- **Run it with `./busybox sh`, not the system `sh`.** `boot_patch.sh` sources
  `util_functions.sh`, whose `grep_prop` shells out to `dos2unix`. Only BusyBox provides
  that applet here, so running under `busybox sh` is what makes the script behave the way
  Magisk intends, instead of silently falling back to `getprop`.
- **Keep `KEEPVERITY=true` and `KEEPFORCEENCRYPT=true`.** These preserve AVB/dm-verity and
  force-encryption, and are Magisk's defaults on Pixel. Turning them off is how you get
  "your data may be corrupt".

**Gate:** `new-boot.img` exists and is the same size as `boot.img` (both are the partition
size, e.g. 67108864 bytes).

### Expected, non-fatal output — do not "fix" these

| You will see                                      | Meaning                                                                            | Action                                                                  |
| ------------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `RAMDISK_SZ [0]` when unpacking `boot.img`    | Android 17 ships`boot.img` as kernel-only; the ramdisk is in `vendor_boot.img` | **Still patch `boot.img`.** Do *not* retarget `vendor_boot` |
| `Failed to patch` three times                   | A ramdisk was created from scratch, so there is nothing to patch inside it         | Ignore                                                                  |
| `Cannot connect to daemon` from `./magisk -v` | Device is not rooted yet                                                           | Expected;`./magisk --preinit-device` still works unrooted             |

---

## Step 5 — Validate with `fastboot boot` (writes nothing)

```bash
adb reboot bootloader
fastboot devices
fastboot boot new-boot.img
adb wait-for-device
adb shell "su -c 'id'"
```

**Gate:** output contains `uid=0(root)` and `context=u:r:magisk:s0`.

- If the device does not boot: reboot, and **nothing has been written**. Re-check the build
  match (Step 3) and repeat from Step 4.
- If it boots but `su: inaccessible or not found`: the patched image does not match the
  running build. Go back to Step 1. Do not proceed to Step 6.
- **The first `su` may appear to hang.** Magisk shows a superuser prompt on the phone for
  the `shell` user; it blocks until someone taps **Grant**. Watch the device screen; this is
  not a failure. Later `su` calls return immediately.

---

## Step 6 — Persist to the active slot

Only after Step 5 has passed.

**First, assert the target partition is what you think it is.** Compare the on-disk `boot`
partition against the stock image you patched:

```bash
adb shell "su -c 'sha256sum /dev/block/by-name/boot_a'"
sha256sum boot.img
```

**Gate:** the two hashes are identical. If they differ, the slot does not hold the stock
image for this build. Stop and do not write; re-run Step 0 and Step 3.

**Then write:**

```bash
adb shell "su -c 'dd if=/data/local/tmp/mp/new-boot.img \
                    of=/dev/block/by-name/boot_a bs=4096 && sync'"
```

Use `boot_a` when `ro.boot.slot_suffix` is `_a`; use `boot_b` when it is `_b`. **Active slot
only** — the inactive slot may hold a different build.

This is exactly what Magisk's app calls *Direct Install*, done from the shell. (Equivalent
GUI route: install the APK, open it, tap **Install** → **Direct Install (Recommended)**.
Do one or the other, not both.)

---

## Step 7 — Verify

```bash
# read back and compare with the source image
adb shell "su -c 'sha256sum /dev/block/by-name/boot_a /data/local/tmp/mp/new-boot.img'"

adb reboot
adb wait-for-device
adb shell "su -c 'id'"                    # must survive a cold boot
adb shell "su -c 'magisk -v'"             # expect e.g. 30.7:MAGISK:R
adb shell 'getprop ro.boot.veritymode'    # must still read: enforcing
```

**Definition of done:** after a cold reboot, `su -c id` returns `uid=0(root)`,
`magisk -v` reports the Magisk version, and `ro.boot.veritymode` is still `enforcing`.

---

## If it goes wrong

| Symptom                                       | Cause                                                    | Fix                                                                                    |
| --------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| No`su` after `fastboot boot`              | patched image build ≠ running build                     | redo from Step 1                                                                       |
| No`su` after Step 6                         | same, now written                                        | flash the stock`boot.img` from Step 2 back to the active slot, then redo from Step 1 |
| Bootloop                                      | same                                                     | `fastboot flash boot <path>/boot.img`, reboot                                        |
| `fastboot boot` drops back to fastboot      | image did not boot                                       | re-verify build match; nothing was written                                             |
| `/sdcard` or `/data/media/0` not listable | device is locked, credential-encrypted storage is sealed | unlock the device.**This is not data loss**                                      |

Recovering the device is always the same move: restore the stock `boot.img` for the running
build to the active slot.

```bash
adb reboot bootloader
fastboot flash boot <path>/boot.img
fastboot reboot
```

No `vbmeta`, no `vendor_boot`, no wipe.

---

## Never do

- `fastboot flash vbmeta --disable-verity --disable-verification` (any slot)
- any of the above combined with `fastboot -w`
- `fastboot erase userdata`
- patch and flash `vendor_boot.img` as the root target
- flash a patched image whose build does not match the running system
- flash a patched `boot` into a slot holding a different build
- reuse a patched image from a previous build — an OTA invalidates it

---

## Report at the end

State, in this order: the build and slot you worked against, which factory image you used
and that its checksum verified, that `fastboot boot` produced root before any write, which
partition you wrote, and the result of the cold-boot verification. If you stopped early,
say which gate failed and what you did not do.
