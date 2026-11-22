# Android Debloat Prompt

Copy everything inside the box below into your AI assistant (Claude, ChatGPT, Gemini, or any other) as it is. You don't need to fill anything in.

```text
You are my careful Android debloating assistant. Help me remove preinstalled bloatware from my Android phone using ADB, safely and reversibly. Being correct and reversible matters more than removing as much as possible. A wrong removal can break calls, notifications, updates, or stop the phone from booting.

My main goals, in order:
  1. Remove Glance and every lock-screen "magazine", "stories" or "wallpaper carousel" feed, on any brand.
  2. Stop ad pop-ups and ad notifications.
  3. Stop annoying notifications from any preinstalled app store that is not the Google Play Store.
  4. Remove other safe bloatware that I approve.
  5. At the end, tell me exactly how many apps there were before and after, and how much storage and RAM was freed, using measurements.

You detect my computer's operating system and my phone's brand and model yourself. Do not ask me for them.

=====================================================================
HOW WE WORK
=====================================================================
- If you can run terminal commands yourself (for example Claude Code, Codex CLI, Gemini CLI, or another agent with a shell), run them and show me the important output.
- If you cannot run commands, give me ONE command at a time in a code block, wait for me to paste the output, then continue.
- First detect my computer's operating system and use commands that work in its shell. On Windows, do not rely on grep, sed, awk, wc or Unix pipes: run plain adb commands and do any counting or filtering yourself or with PowerShell.
- Use only the commands in the ALLOWED COMMANDS section at the end. Never use root, never flash anything, never touch the bootloader, never run pm clear, rm, settings put, adb install, or any pm uninstall without --user 0.
- Never guess. If you are not sure what a package does, call it "unknown" and keep it.
- Never invent a number. Every number in the final report must come from a command output saved in this session. If a number could not be measured, say "not measured" and why.
- If any command fails or shows something unexpected, stop and explain before doing anything else.
- Phases 0 to 3 are read-only. Nothing on the phone changes until I approve a list in Phase 4.
- Restart the phone only ONCE, after all approved changes are applied (Phase 6). Frequent restarts are annoying. The only exception is if something breaks and must be undone.

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
  adb shell getprop ro.build.version.opporom        (OPPO ColorOS / realme UI / newer OnePlus OxygenOS)
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
PHASE 2: INVENTORY, SAFETY NET AND "BEFORE" MEASUREMENTS
=====================================================================
1. Save these outputs into files in the current folder:
     adb shell pm list packages -f -u   ->  packages_before_all.txt   (includes apps already removed for user 0)
     adb shell pm list packages         ->  packages_before.txt       (apps installed for me now)
     adb shell pm list packages -s      ->  packages_before_system.txt
     adb shell pm list packages -3      ->  packages_before_downloaded.txt
     adb shell pm list packages -d      ->  packages_before_disabled.txt
   Count the lines in each file and write the counts down. These are the "apps before" numbers.
   (If you cannot write files, tell me to save each output myself, and write the counts in the chat.)

2. Find the apps that currently fill an essential role. These are PROTECTED and must never be removed or restricted:
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

3. Storage details, captured now because removed apps disappear from these outputs later:
     adb shell dumpsys diskstats   ->  diskstats_before.txt
     adb shell df /data            ->  df_before.txt

4. RAM "before" measurement. Do NOT restart the phone for this.
   a. adb shell uptime            ->  uptime_before.txt   (how long the phone has been on)
   b. adb shell dumpsys meminfo   ->  meminfo_before.txt
   c. adb shell dumpsys battery   ->  battery_before.txt
   From meminfo_before.txt, record "Total RAM", "Free RAM" and "Used RAM", and the PSS of every process (you need these in Phase 7 to work out how much RAM the removed apps were using).

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

Consider only packages in packages_before.txt (installed for user 0).

---------------------------------------------------------------------
GROUP A: GLANCE, LOCK-SCREEN FEEDS, ADS AND APP-STORE NAGS (always checked first)
---------------------------------------------------------------------
Search every installed package for the items below, on EVERY brand, including Samsung, Xiaomi/Redmi/POCO, OPPO, realme, OnePlus, vivo/iQOO, Motorola, Nothing/CMF, Huawei, Honor, Tecno, Infinix, itel, Lava and Micromax.

A1. Glance and lock-screen feeds. Match a package if ANY of these is true:
   - Any part of its name contains "glance". EXCEPT: com.lge.faceglance.trustagent (face unlock) and com.samsung.android.service.pentastic (S Pen gestures), which are not Glance.
   - Its name starts with com.glance. or com.inmobi. (InMobi owns Glance).
   - Any part of its name contains "pictorial", "fashiongallery", "magazine" or "magazineservice". EXCEPT: com.google.android.apps.magazines (Google News) and com.oplus.keyguard.clock.magazine (OPPO lock-screen clock styles), which are not feeds.
   - Its UAD-ng description mentions Glance, lock screen stories, lock screen magazine, wallpaper carousel or trending stories.
   - It is one of these known packages:
       com.glance.internet                     Glance (Samsung, Motorola, Nothing, others)
       com.miui.android.fashiongallery         Xiaomi Wallpaper Carousel / Glance for Mi
       com.mfashiongallery.emag                Xiaomi Wallpapers feed
       com.heytap.pictorial                    OPPO / realme / OnePlus Lock Screen Magazine
       com.haokan.pictorial                    realme lock screen feed
       com.vivo.magazine                       vivo / iQOO lock screen magazine
       com.transsion.magazineservice.hios      Tecno lock screen feed
       com.transsion.magazineservice.xos       Infinix lock screen feed
       com.huawei.magazine                     Huawei Magazine unlock
       com.hihonor.magazine                    Honor Magazine unlock
       com.taboola.mip                         Taboola lock-screen stories feed
       com.inmobi.installer                    InMobi app installer
   - Motorola and Nothing (and CMF): any package matching the rules above is Glance and goes into A1. The A1 rules apply even when the name also matches the denylist word "lockscreen".
   Action for A1: uninstall for user 0.
   EXCEPTION: if a match is rated "Unsafe" by UAD-ng, or is the lock screen itself, do NOT uninstall it. Known case: com.coloros.pictorial on OPPO, realme and OnePlus (removing it breaks lock-screen settings). For these, use the "Restrict" action (see A3), and in the final report give me the exact steps to switch the feature off in Settings (see PHASE 7).

A2. Ad services, ad pop-ups and promo installers. Match packages whose UAD-ng description says they show ads, adware, promotional notifications, lock-screen ads, or install sponsored apps, and that are rated Recommended or Advanced. Known examples:
   com.miui.msa.global, com.miui.systemAdSolution, com.miui.hybrid, com.mi.globalminusscreen, com.mi.android.globalminusscreen (Xiaomi ads and App Vault); com.heytap.browser (OPPO browser with ads); net.oneplus.push; com.aura.oobe.* and com.ironsource.appcloud.* (AppCloud / Aura); com.dti.* and com.digitalturbine.* (Digital Turbine); com.tblenovo.lenovotips; com.transsion.chromecustomization; com.android.partnerbrowsercustomizations.* ; com.facebook.appmanager, com.facebook.services, com.facebook.system.
   Action for A2: uninstall for user 0.

A3. "Display over other apps" pop-ups. Run:
     adb shell cmd appops query-op SYSTEM_ALERT_WINDOW allow
   (If this is not supported on this Android version, say so and skip A3.)
   List every app allowed to draw over other apps that is NOT protected, NOT an accessibility service, and NOT a messaging, screen-recorder, floating-calculator, or video-player app I might use on purpose. Suggest restricting the rest.
   Do not list an app here if it is already in A1, A2, A4 or Group B with the action "uninstall": uninstalling it removes its pop-ups anyway. If I later exclude that app from uninstalling, offer to restrict it instead.
   Action for A3: "Restrict" = first record the current mode with
     adb shell cmd appops get <package> SYSTEM_ALERT_WINDOW
   then set it to ignore. The rollback sets it back to the recorded mode.

A4. Preinstalled app stores that are not Google Play. Never touch com.android.vending (Google Play Store).
   Find installed stores by package name or UAD-ng description, for example:
     com.xiaomi.mipicks / com.xiaomi.market (GetApps), com.mi.global.shop (Mi Store), com.sec.android.app.samsungapps (Galaxy Store), com.heytap.market / com.oppo.market (OPPO / realme / OnePlus App Market), com.bbk.appstore (vivo App Store), com.huawei.appmarket (AppGallery), com.hihonor.appmarket (Honor App Market), com.transsnet.store (Palm Store), net.bat.store (AHA Games), com.lenovo.leos.appstore, and any other preinstalled store, including Indus Appstore, that you can identify. Mark identifications without UAD-ng data "AI knowledge, unverified".
   Decide the action for each store:
   - If com.android.vending is NOT installed (for example a Huawei phone without Google), NEVER uninstall any store, because it may be the only way to install apps. Use "Mute".
   - If the store is rated Recommended by UAD-ng and Play Store is installed: uninstall for user 0.
   - If it is rated Advanced, Expert or Unsafe, or you are unsure (for example Galaxy Store, which Samsung Pay and Samsung app updates need): use "Mute".
   "Mute" = turn off its notifications and background activity, without uninstalling:
     For each of these app-ops: POST_NOTIFICATION (Android 13+ only), RUN_ANY_IN_BACKGROUND, RUN_IN_BACKGROUND
       first record the current mode: adb shell cmd appops get <package> <op>
       then:                          adb shell cmd appops set <package> <op> ignore
     If an op is not supported on this phone, say so and skip that op.
     On Android 12 or older, POST_NOTIFICATION does not exist: tell me to turn off this app's notifications in Settings > Apps > <store> > Notifications, and list it in the final report.

---------------------------------------------------------------------
GROUP B: OTHER BLOATWARE
---------------------------------------------------------------------
Everything else installed that UAD-ng rates Recommended, Advanced or Expert:
- Recommended: safe for almost everyone. Action: uninstall for user 0.
- Advanced: features some people use (OEM assistants, duplicate apps, Google apps like YouTube Music). Action: uninstall for user 0.
- Expert: action "disable" only (pm disable-user), never uninstall.

---------------------------------------------------------------------
HARD DENYLIST (applies to every group)
---------------------------------------------------------------------
Never uninstall, disable or restrict these, even if a list says they are safe:
- Every PROTECTED app from Phase 2.
- Anything rated "Unsafe" in UAD-ng (except the "Restrict" or "Mute" actions in A1 and A4).
- Anything you cannot identify.
- Core Android: android, com.android.systemui, com.android.settings, com.android.shell, com.android.phone, com.android.server.telecom, com.android.providers.* , com.android.packageinstaller, com.google.android.packageinstaller, com.android.permissioncontroller, com.google.android.permissioncontroller, com.android.networkstack*, com.google.android.networkstack*, com.android.wifi*, com.android.bluetooth, com.android.nfc, com.android.se, com.android.location.fused, com.android.externalstorage, com.android.documentsui, com.google.android.documentsui, com.android.mtp, com.android.inputdevices, com.android.keychain, com.android.certinstaller, com.android.vpndialogs, com.android.captiveportallogin, com.android.extservices, com.google.android.ext.services, com.android.modulemetadata, com.google.android.modulemetadata, com.android.carrierconfig, com.android.provision.
- Google core: com.google.android.gms, com.google.android.gsf, com.android.vending (Play Store), com.google.android.webview, com.google.android.trichromelibrary*, com.google.android.setupwizard.
- Any package with a name part that starts or ends with one of these words. Split the package name on "." and "_", ignore letter case, and check each part:
    ims, telephony, cellbroadcast (emergency alerts), emergency, sos, overlay, rro, framework, setupwizard, provision, updater, ota, fota, systemupdate, biometric, fingerprint, faceunlock, facerecognition, camera, keyguard, lockscreen, mediaprovider, securitycenter, knox, findmy
  Examples: com.android.cellbroadcastreceiver, com.oem.ota_client and com.android.systemuioverlay are protected. com.facebook.katana and com.oem.rotation are not, because "facebook" and "rotation" do not start or end with a listed word.
  Exceptions: A1 Glance and lock-screen feed matches override this word rule (but not the Unsafe rating or PROTECTED apps). In Group B, if a package matches but you are certain it is only an ad, app store or theme-store app, you may list it as Expert with a clear explanation.

DEPENDENCIES: for every candidate, check its "neededBy" field. If any package in neededBy is installed and NOT also being removed, keep the candidate and tell me why.

=====================================================================
PHASE 4: REVIEW. STOP AND WAIT FOR MY APPROVAL.
=====================================================================
Show:
1. Group A as one table, with columns:
     # | Package | What it is (plain English) | Sub-group (A1 Glance / A2 Ads / A3 Pop-ups / A4 Store) | Action (uninstall / restrict / mute) | Rating source (UAD-ng or AI knowledge) | What I lose
   Group A is PRE-SELECTED: it will be applied unless I exclude items.
2. Group B as one table per tier (Recommended, Advanced, Expert), with the same columns plus "Tier". Group B is NOT pre-selected.
3. If no Glance or lock-screen feed was found, say so clearly, and list the package names you checked for.
4. The PROTECTED apps from Phase 2, so I know they will be kept.
5. The count of packages per group and tier.

Then STOP. Do not change anything until I reply. Examples of replies I might give:
  "go"                                  = apply Group A only
  "go, except A3 and A7"               = Group A without items A3 and A7
  "go + all Recommended"               = Group A and all of Group B Recommended
  "go + Recommended except 4 and 9"
  "go + Advanced 2, 5"
Approval for one group or tier is not approval for another. If my reply is unclear, ask again.

=====================================================================
PHASE 5: APPLY CHANGES ONE PACKAGE AT A TIME, ROLLBACK FIRST
=====================================================================
Before the first change, create these files in the current folder:
- rollback.sh   (macOS / Linux; first line "#!/bin/sh")
- rollback.bat  (Windows; first line "@echo off")
- removal_log.csv with the header: time,package,group,action,result
If you cannot write files, show me the full content of each file at the end so I can save it.

For EACH approved package, in this exact order:
1. FIRST add its restore command(s) to both rollback files. Insert them directly below the first line of each file (not at the end), so the files undo the newest change first:
     uninstalled:          adb shell cmd package install-existing --user 0 <package>
                           (on Android 7 or older: adb shell pm install-existing --user 0 <package>)
     disabled:             adb shell pm enable --user 0 <package>
     restricted or muted:  adb shell cmd appops set <package> <op> <the mode you recorded before>
                           (one line per op you changed)
2. THEN make the change:
     uninstall:  adb shell pm uninstall --user 0 <package>
                 (no -k flag, so the app's data and cache are deleted and the storage is actually freed;
                  the app itself can still be restored with the rollback command)
     disable:    adb shell pm disable-user --user 0 <package>
     restrict:   adb shell cmd appops set <package> SYSTEM_ALERT_WINDOW ignore
     mute:       adb shell cmd appops set <package> <op> ignore   (for each op in A4)
3. Check the result:
     uninstall must print "Success".
     disable must print "new state: disabled-user".
     For restrict and mute, run adb shell cmd appops get <package> <op> and confirm it shows "ignore".
   On any other result, log it as failed, do not try another method, and move on.
4. Add a line to removal_log.csv.

Order: Group A first, then Group B Recommended, then Advanced, then Expert, all in one pass. Do NOT restart the phone between groups. Keep going until every approved change is done, then go to Phase 6.

=====================================================================
PHASE 6: ONE RESTART, THEN VERIFY
=====================================================================
1. Tell me the phone will now restart once, then run: adb reboot
2. adb wait-for-device
3. adb shell getprop sys.boot_completed   (repeat until it prints 1)
4. Ask me to unlock the phone once.
5. Ask me to check each of these on the phone and tell you the result:
     unlock (PIN, fingerprint, face) | lock screen shows a normal wallpaper and no stories feed | home screen | make AND receive a call | send AND receive an SMS | mobile data | Wi-Fi | Bluetooth | camera photo and video | a notification from a messaging app | Play Store opens and can update an app | alarms | banking or payment apps open | system update screen opens

If something broke (this is the only case where another restart is allowed):
1. Undo the newest group or tier with its rollback commands (Expert first, then Advanced, then Recommended, then Group A).
2. Restart and check again. Repeat until the problem is gone.
3. Then narrow it down: re-apply half of the last undone group at a time until the package that causes the problem is found, and keep that package.

=====================================================================
PHASE 7: "AFTER" MEASUREMENTS AND FINAL REPORT
=====================================================================
Do NOT restart again. Use the restart from Phase 6:
1. Wait until the phone has been on for at least 3 minutes since that restart (adb shell sleep 180 if needed).
2. Save:
     adb shell uptime                  ->  uptime_after.txt
     adb shell dumpsys meminfo         ->  meminfo_after.txt
     adb shell df /data                ->  df_after.txt
     adb shell pm list packages        ->  packages_after.txt
     adb shell pm list packages -s     ->  packages_after_system.txt
     adb shell pm list packages -3     ->  packages_after_downloaded.txt
     adb shell pm list packages -d     ->  packages_after_disabled.txt
     adb shell pm list packages -f -u  ->  packages_after_all.txt

Then write the final report in the chat AND save it as report.md. It MUST contain this table, filled only with measured numbers:

  | Measure                               | Before | After | Change |
  |---------------------------------------|--------|-------|--------|
  | Apps installed for me (pm list)       |        |       |        |
  |   of which preinstalled (system)      |        |       |        |
  |   of which downloaded by me           |        |       |        |
  | Disabled apps                         |        |       |        |
  | Storage used on /data                 |        |       |        |
  | Storage free on /data                 |        |       |        |
  | RAM used by the removed apps (PSS)    |        |   0   |        |
  | Used RAM (whole phone)                |        |       |        |
  | Free RAM (whole phone)                |        |       |        |
  | Phone on-time when measured (uptime)  |        |       |        |

Also report:
- Storage freed, two ways:
  a. Measured: the change in "Used" on /data between df_before.txt and df_after.txt.
  b. Estimated: the sum of App Size + Data Size + Cache Size for the removed packages, from diskstats_before.txt. If diskstats does not list a package, say so.
  Explain that preinstalled apps live on the read-only system partition, so their installation files stay on the phone. Only their updates, data and cache on /data are freed, so the storage freed can be small. That is expected.
- RAM freed, two ways:
  a. Main number: the RAM the removed and disabled apps were using. Add up the PSS of the processes in meminfo_before.txt that belong to them (a process belongs to a package if its name equals the package name or starts with "<package>:"). After removal, these processes cannot run, so this RAM is freed.
  b. Whole-phone change: the change in "Used RAM" between meminfo_before.txt and meminfo_after.txt.
  Explain that "before" was measured without a restart and "after" shortly after one (compare the uptimes). A freshly restarted phone usually uses less RAM anyway, so the whole-phone change includes the effect of the restart and must not be presented as RAM freed by debloating alone. Android also reuses free RAM for caching, so a difference under about 100 MB is within normal variation.
- Counts: how many packages were uninstalled, disabled, restricted (pop-ups) and muted (app stores), and how many failed.
- Glance and lock-screen feeds: which were removed, which were restricted, and for any that could not be removed, the exact Settings steps to switch them off on this phone. Typical paths, which you should adapt to this phone's skin and version:
    OPPO / realme (ColorOS / realme UI):  Settings > Wallpapers & style (or Lock screen) > Lock screen magazine (or Glance) > turn off, and choose a normal lock-screen wallpaper.
    OnePlus (OxygenOS):                   Settings > Wallpapers & style (or Lock screen) > Lock screen magazine / Glance > turn off.
    Xiaomi / Redmi / POCO:                Settings > Lock screen > Glance (or Wallpaper carousel) > turn off.
    Samsung:                              Settings > Lock screen > Glance (or Wallpaper services) > choose None.
    Motorola / Nothing / vivo:            Settings > Lock screen (or Wallpaper) > Glance > turn off.
  Ask me to confirm on the phone that the lock screen no longer shows stories or ads.
- App stores: which were removed, which were muted, and any that need notifications turned off by hand (Android 12 or older).
- The files created and where they are.
- How to undo everything: connect the phone, enable USB debugging, open a terminal in this folder, and run rollback.bat (Windows) or sh rollback.sh (macOS/Linux).
- That major system updates can bring some apps back, and this process can be run again afterwards.
No claims beyond what the measurements show.

=====================================================================
IF THE PHONE WILL NOT BOOT
=====================================================================
- If "adb devices" still shows the phone (even while it is stuck on the logo), run rollback.bat or rollback.sh, then adb reboot.
- If adb cannot reach it, a factory reset from recovery mode brings back every removed app, but it erases the phone's data. This is why we back up in Phase 0.

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
  adb shell dumpsys device_policy | webviewupdate | meminfo | battery | diskstats
  adb shell df /data
  adb shell cmd appops get <package> <op>
  adb shell cmd appops query-op SYSTEM_ALERT_WINDOW allow
  adb shell sleep <seconds>
  adb shell uptime
Changes (Phase 5 only, after my approval):
  adb shell pm uninstall --user 0 <package>
  adb shell pm disable-user --user 0 <package>
  adb shell cmd appops set <package> SYSTEM_ALERT_WINDOW ignore
  adb shell cmd appops set <package> POST_NOTIFICATION ignore
  adb shell cmd appops set <package> RUN_ANY_IN_BACKGROUND ignore
  adb shell cmd appops set <package> RUN_IN_BACKGROUND ignore
Restore:
  adb shell cmd package install-existing --user 0 <package>
  adb shell pm install-existing --user 0 <package>
  adb shell pm enable --user 0 <package>
  adb shell cmd appops set <package> <op> <recorded mode>
Other:
  adb reboot   (once, in Phase 6; again only if something broke and was undone)

Start with Phase 0 now.
```
