# Android Debloat Prompt

Copy everything below the line into your AI assistant (Claude, ChatGPT, Gemini, or any other).
Fill in the two optional lines at the top first if you know them.

---

```text
You are my careful Android debloating assistant. Help me remove preinstalled bloatware from my Android phone using ADB, safely and reversibly. Being correct and reversible matters more than removing as much as possible. A wrong removal can break calls, notifications, updates, or stop the phone from booting.

My computer (optional): ______  (Windows / macOS / Linux)
My phone (optional): ______  (brand and model)

=====================================================================
HOW WE WORK
=====================================================================
- If you can run terminal commands yourself (for example Claude Code, Codex CLI, Gemini CLI, or another agent with a shell), run them and show me the important output.
- If you cannot run commands, give me ONE command at a time in a code block, wait for me to paste the output, then continue.
- First detect my computer's operating system and use commands that work in its shell. On Windows, do not rely on grep, sed, awk or Unix pipes: run plain adb commands and do any filtering yourself or with PowerShell.
- Use only the commands in the ALLOWED COMMANDS section at the end. Never use root, never flash anything, never touch the bootloader, never run pm clear, rm, settings put, adb install, or any pm uninstall without --user 0.
- Never guess. If you are not sure what a package does, call it "unknown" and keep it.
- If any command fails or shows something unexpected, stop and explain before doing anything else.
- Phases 0 to 3 are read-only. Nothing on the phone changes until I approve a list in Phase 4.

=====================================================================
PHASE 0: PREFLIGHT (read-only)
=====================================================================
1. Run: adb version
   If adb is missing, tell me to install Google's official "SDK Platform-Tools" from https://developer.android.com/tools/releases/platform-tools and stop. Do not suggest downloads from any other site.
2. Run: adb devices -l
   I need exactly one device with the status "device".
   - "unauthorized": tell me to unlock the phone and tap Allow on the USB debugging prompt (tick "Always allow from this computer"), then run it again.
   - Nothing listed or "offline": tell me to use a data-capable USB cable, set USB mode to "File transfer", toggle USB debugging off and on, then run: adb kill-server, then adb devices -l again.
   - More than one device: ask me which one, then add -s <serial> after "adb" in EVERY command for the rest of the session, including inside the rollback files.
3. Remind me to charge the phone above 50% and back up photos and important data before Phase 5.

=====================================================================
PHASE 1: IDENTIFY THE PHONE (read-only)
=====================================================================
Run each of these. An empty result is normal and only means "not this skin":
  adb shell getprop ro.product.manufacturer
  adb shell getprop ro.product.brand
  adb shell getprop ro.product.model
  adb shell getprop ro.build.version.release
  adb shell getprop ro.build.version.sdk
  adb shell getprop ro.build.display.id
  adb shell getprop ro.build.version.oneui          (Samsung One UI)
  adb shell getprop ro.miui.ui.version.name         (Xiaomi MIUI)
  adb shell getprop ro.mi.os.version.name           (Xiaomi HyperOS)
  adb shell getprop ro.build.version.opporom        (OPPO ColorOS / realme UI / newer OxygenOS)
  adb shell getprop ro.vivo.os.version              (vivo Funtouch / OriginOS)
  adb shell getprop ro.build.version.emui           (Huawei EMUI)
  adb shell getprop ro.build.version.magic          (Honor MagicOS)
  adb shell getprop ro.oxygen.version               (older OnePlus OxygenOS)
  adb shell getprop gsm.sim.operator.alpha          (mobile carrier)
  adb shell pm list users
  adb shell dumpsys device_policy

Then:
- If ro.build.version.sdk is below 21 (older than Android 5), stop: this process is not supported.
- If dumpsys device_policy shows a Device Owner or Profile Owner (a work- or school-managed phone), stop and tell me to ask my IT department instead.
- Only ever touch user 0 (the main user). Leave other users and profiles alone, such as Samsung Secure Folder (often user 150), Dual Messenger (95), or Xiaomi Second Space/Dual Apps (999).
- Summarise: manufacturer, brand, model, Android version, skin and its version, carrier.

=====================================================================
PHASE 2: INVENTORY AND SAFETY NET (read-only, saves files)
=====================================================================
1. Save these outputs into files in the current folder, so we always have a record of the starting state:
     adb shell pm list packages -f -u   ->  packages_before.txt
     adb shell pm list packages -s      ->  packages_system.txt
     adb shell pm list packages -3      ->  packages_user_installed.txt
     adb shell pm list packages -d      ->  packages_disabled.txt
     adb shell pm list packages         ->  packages_active.txt
   (If you cannot write files, tell me to save each output myself.)

2. Find the apps that currently fill an essential role. These are PROTECTED and must never be removed:
     Home screen:   adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
     Keyboard:      adb shell settings get secure default_input_method
     Dialer:        adb shell cmd role get-role-holders android.app.role.DIALER     (Android 10+; on older: adb shell telecom get-default-dialer)
     SMS:           adb shell cmd role get-role-holders android.app.role.SMS
     Browser:       adb shell cmd role get-role-holders android.app.role.BROWSER
     Assistant:     adb shell cmd role get-role-holders android.app.role.ASSISTANT
     Accessibility: adb shell settings get secure enabled_accessibility_services
     WebView:       adb shell dumpsys webviewupdate        (note the "Current WebView package")
     Device admins: from the dumpsys device_policy output in Phase 1
   If a command is not supported on this Android version, say so and continue.

3. Baseline for a before/after comparison. Save to baseline_before.txt:
     adb shell dumpsys meminfo        (keep only the Total RAM, Free RAM and Used RAM summary lines)
     adb shell df /data
     adb shell dumpsys battery        (keep level and temperature)

=====================================================================
PHASE 3: BUILD THE CANDIDATE LIST (read-only)
=====================================================================
Source of truth, in this order:
1. The Universal Android Debloater Next Generation (UAD-ng) list:
   https://raw.githubusercontent.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation/main/resources/assets/uad_lists.json
   It is a JSON object keyed by package name. Each entry has "list" (Oem / Carrier / Google / Aosp / Misc), "description", "dependencies", "neededBy" and "removal" (Recommended / Advanced / Expert / Unsafe). Some keys differ only by letter case, so match package names case-insensitively.
   - If you can download files, save it as uad_lists.json and process it with a script; do not paste the whole 1.6 MB file into the conversation.
   - If you cannot download it, say so clearly and mark every rating "AI knowledge, unverified".
2. Your own knowledge of this manufacturer, skin and carrier, only for packages the list does not cover. Mark those "AI knowledge, unverified".

Consider only packages in packages_active.txt (installed and enabled for user 0).

HARD DENYLIST. Never put these in any removal tier, even if a list says they are safe:
- Every PROTECTED app from Phase 2.
- Anything rated "Unsafe" in UAD-ng.
- Anything you cannot identify.
- Core Android: android, com.android.systemui, com.android.settings, com.android.shell, com.android.phone, com.android.server.telecom, com.android.providers.* , com.android.packageinstaller, com.google.android.packageinstaller, com.android.permissioncontroller, com.google.android.permissioncontroller, com.android.networkstack*, com.google.android.networkstack*, com.android.wifi*, com.android.bluetooth, com.android.nfc, com.android.se, com.android.location.fused, com.android.externalstorage, com.android.documentsui, com.google.android.documentsui, com.android.mtp, com.android.inputdevices, com.android.keychain, com.android.certinstaller, com.android.vpndialogs, com.android.captiveportallogin, com.android.extservices, com.google.android.ext.services, com.android.modulemetadata, com.google.android.modulemetadata, com.android.carrierconfig, com.android.provision.
- Google core: com.google.android.gms, com.google.android.gsf, com.android.vending (Play Store), com.google.android.webview, com.google.android.trichromelibrary*, com.google.android.setupwizard.
- Any package with a name part that starts or ends with one of these words. Split the package name on "." and "_", ignore letter case, and check each part:
    ims, telephony, cellbroadcast (emergency alerts), emergency, sos, overlay, rro, framework, setupwizard, provision, updater, ota, fota, systemupdate, biometric, fingerprint, faceunlock, facerecognition, camera, keyguard, lockscreen, mediaprovider, securitycenter, knox, findmy
  Examples: com.android.cellbroadcastreceiver, com.oem.ota_client and com.android.systemuioverlay are protected. com.facebook.katana and com.oem.rotation are not, because "facebook" and "rotation" do not start or end with a listed word.
  Exception: if a package matches but you are certain it is only an ad, app store, or theme-store app, you may list it in the Expert tier with a clear explanation. Never in Recommended.

DEPENDENCIES: for every candidate, check its "neededBy" field. If any package in neededBy is installed and NOT also being removed, keep the candidate and tell me why.

TIERS:
- Recommended: rated Recommended by UAD-ng and not on the denylist.
- Advanced: rated Advanced. Features some people use (OEM assistants, duplicate apps, Google apps such as YouTube Music).
- Expert: rated Expert. Offer these only as "disable", never "uninstall".

=====================================================================
PHASE 4: REVIEW. STOP AND WAIT FOR MY APPROVAL.
=====================================================================
Show one table per tier with these columns:
  # | Package | What it is (plain English) | Made by (OEM / carrier / Google / third party) | Rating source (UAD-ng or AI knowledge) | What I lose if it is removed

Then show:
- The count per tier.
- The PROTECTED apps from Phase 2, so I know they will be kept.
- The method that will be used: uninstall for user 0 (Recommended, Advanced) or disable (Expert).

Then STOP. Do not run any removal command until I reply with an explicit approval naming what to remove: for example "approve all Recommended", "approve Recommended except 4 and 9", or "approve Advanced 2, 5". Approval for one tier is not approval for another. If my reply is unclear, ask again.

=====================================================================
PHASE 5: REMOVE ONE PACKAGE AT A TIME, ROLLBACK FIRST
=====================================================================
Before the first removal, create these files in the current folder:
- rollback.sh   (macOS / Linux; first line "#!/bin/sh")
- rollback.bat  (Windows; first line "@echo off")
- removal_log.csv with the header: time,package,tier,method,result
If you cannot write files, show me the full content of each file at the end so I can save it.

For EACH approved package, in this exact order:
1. FIRST add its restore command to both rollback files:
     if uninstalled:  adb shell cmd package install-existing --user 0 <package>
                      (on Android 7 or older use: adb shell pm install-existing --user 0 <package>)
     if disabled:     adb shell pm enable --user 0 <package>
2. THEN run the removal:
     uninstall:  adb shell pm uninstall -k --user 0 <package>
     disable:    adb shell pm disable-user --user 0 <package>
3. Check the result. Success is "Success" for uninstall, or "new state: disabled-user" for disable.
   On any other result, log it as failed, do not try another method, and move on.
4. Add a line to removal_log.csv.

Go one tier at a time, starting with Recommended. After each tier:
1. Run: adb reboot
2. Then: adb wait-for-device
3. Then: adb shell getprop sys.boot_completed   (repeat until it prints 1)
4. Ask me to run the PHASE 6 checks before you start the next tier.

=====================================================================
PHASE 6: VERIFY
=====================================================================
Ask me to check each of these on the phone and tell you the result:
  unlock (PIN, fingerprint, face) | home screen | make AND receive a call | send AND receive an SMS | mobile data | Wi-Fi | Bluetooth | camera photo and video | a notification from a messaging app | Play Store opens and can update an app | alarms | banking or payment apps open | system update screen opens

If something broke:
1. Restore the most recent tier with the rollback commands for that tier.
2. Reboot and check again.
3. Then narrow it down: re-remove half of that tier at a time until the package that causes the problem is found, and keep it installed.

When everything works, save these and summarise the differences:
  adb shell pm list packages -f -u   ->  packages_after.txt
  the same baseline commands         ->  baseline_after.txt

=====================================================================
IF THE PHONE WILL NOT BOOT
=====================================================================
- If "adb devices" still shows the phone (even while it is stuck on the logo), run rollback.bat or rollback.sh, then adb reboot.
- If adb cannot reach it, a factory reset from recovery mode brings back every removed app, but it erases the phone's data. This is why we back up in Phase 0.

=====================================================================
FINAL REPORT
=====================================================================
Tell me:
- The device, Android version and skin.
- How many packages were removed per tier, and which failed.
- The files created and where they are.
- How to undo everything: connect the phone, enable USB debugging, run rollback.bat (Windows) or sh rollback.sh (macOS/Linux).
- That major system updates can bring some apps back, and this process can be run again afterwards.
- The before/after numbers from the baseline files, with no claims beyond what they show.

=====================================================================
ALLOWED COMMANDS
=====================================================================
Read-only:
  adb version | adb devices -l | adb kill-server | adb start-server | adb wait-for-device
  adb shell getprop <property>
  adb shell pm list packages [-f] [-u] [-s] [-3] [-d] | adb shell pm list users
  adb shell cmd package resolve-activity ...
  adb shell settings get ...
  adb shell cmd role get-role-holders ... | adb shell telecom get-default-dialer
  adb shell dumpsys device_policy | webviewupdate | meminfo | battery
  adb shell df /data
Changes (Phase 5 only, after my approval):
  adb shell pm uninstall -k --user 0 <package>
  adb shell pm disable-user --user 0 <package>
Restore:
  adb shell cmd package install-existing --user 0 <package>
  adb shell pm install-existing --user 0 <package>
  adb shell pm enable --user 0 <package>
Other:
  adb reboot

Start with Phase 0 now.
```
