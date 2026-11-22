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
- On Windows, Python is often missing even when "python" seems to exist (it can be only a Microsoft Store shortcut that prints "Python was not found"). If so, use PowerShell for every script. Windows PowerShell 5.1's ConvertFrom-Json cannot load the UAD-ng list, because some keys differ only by letter case. Load it with .NET instead: Add-Type -AssemblyName System.Web.Extensions, then a System.Web.Script.Serialization.JavaScriptSerializer with MaxJsonLength set to [int]::MaxValue.
- Privacy: "pm list users" shows the phone owner's name, and "adb devices" shows the phone's serial number. Do not repeat either in the chat, in report.md or in any file you share.
- Use only the commands in the ALLOWED COMMANDS section at the end. Never use root, never flash anything, never touch the bootloader, never run pm clear, rm, settings put, adb install, or any pm uninstall without --user 0.
- Never guess. If you are not sure what a package does, call it "unknown" and keep it.
- Never invent a number. Every number in the final report must come from a command output saved in this session. If a number could not be measured, say "not measured" and why.
- If any command fails or shows something unexpected, stop and explain before doing anything else.
- Phases 0 to 3 are read-only. Nothing on the phone changes until I approve a list in Phase 4.
- Restart the phone only ONCE, after all approved changes are applied (Phase 6). Frequent restarts are annoying. The only exceptions are if something breaks and must be undone, or if I approve more removals in Phase 8 (one restart per extra round).

=====================================================================
PHASE 0: PREFLIGHT (read-only)
=====================================================================
1. Run: adb version
   If adb is missing, tell me to install Google's official "SDK Platform-Tools" from https://developer.android.com/tools/releases/platform-tools and stop. Do not suggest downloads from any other site.
   If I say I installed it but "adb" is still not recognized, it is usually unzipped but not on the PATH. Before asking me again, check the usual places (read-only): C:\platform-tools\adb.exe, %LOCALAPPDATA%\Android\Sdk\platform-tools\adb.exe, %LOCALAPPDATA%\Microsoft\WinGet\Links\adb.exe, and platform-tools inside my Downloads and Desktop folders (on macOS/Linux: ~/platform-tools, ~/Downloads/platform-tools, ~/Library/Android/sdk/platform-tools). If you find it, use that full path for every command in this session. In rollback.bat, start with "set ADB=adb", then 'if exist "<full path>" set ADB="<full path>"', and write every command as %ADB% shell ..., so the undo file works even when adb is not on the PATH.
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
   From packages_before_all.txt (it shows each package's file path), mark every package that is in packages_before_downloaded.txt as "downloaded-type". This includes preinstalled apps whose path starts with /data/app, /product/data-app or /cust/app (for example partner games and Xiaomi's Notes, Calendar or Weather on some phones). On a phone with one user, "pm uninstall --user 0" deletes a downloaded-type app completely. "install-existing" cannot bring it back, only the Play Store or a factory reset can. Tested on HyperOS 3 / Android 16: 18 downloaded-type apps were removed, and packages_all went from 434 to 416.

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
   List every app allowed to draw over other apps that is NOT protected, NOT an accessibility service, and NOT a messaging, screen-recorder, floating-calculator, floating-notes, or video-player app I might use on purpose. Suggest restricting the rest, but only user-facing apps (ones with their own app icon, such as weather, music, cleaner, games hub or app store).
   This list is usually long and is mostly system services that need pop-ups to work: the in-call screen, battery manager, security center, permission dialogs, SystemUI, Settings, updaters, Find My Device, account and sync services, casting, and OEM frameworks (for example com.android.incallui, com.miui.powerkeeper, com.miui.securitycore, com.xiaomi.xmsf). Never restrict these. Name them in the review as "kept on purpose" so I can see you checked them.
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
     If "appops get" prints "No operations." the op is at its default mode: record the mode as "default" (the rollback line is then "cmd appops set <package> <op> default").
     On Android 12 or older, POST_NOTIFICATION does not exist: tell me to turn off this app's notifications in Settings > Apps > <store> > Notifications, and list it in the final report.
     On Android 13 and newer, notifications are a runtime permission, and the system may undo POST_NOTIFICATION right away (tested on HyperOS 3 / Android 16: after "set ... ignore", "get" showed "allow" again). The Phase 5 check catches this. Log it as failed, do not try another method, and give me the manual Settings steps in the final report. RUN_ANY_IN_BACKGROUND and RUN_IN_BACKGROUND did work.

---------------------------------------------------------------------
GROUP B: OTHER BLOATWARE
---------------------------------------------------------------------
Everything else installed that UAD-ng rates Recommended, Advanced or Expert:
- Recommended: safe for almost everyone. Action: uninstall for user 0.
- Advanced: features some people use (OEM assistants, duplicate apps, Google apps like YouTube Music). Action: uninstall for user 0.
- Expert: action "disable" only (pm disable-user), never uninstall.

UAD-ng also rates many popular apps "Recommended", for example WhatsApp, Instagram, Facebook, Netflix, Spotify, Google Pay / Wallet, Gmail, Google Photos, Maps, YouTube, Drive and Files by Google. Uninstalling without -k deletes their data (chats, notes, downloads), so a reply like "all Recommended" must never remove them by accident. Put these, messaging, banking and payment apps, the phone maker's everyday tools (calculator, clock, gallery, notes, file manager, sound recorder, screen recorder), and anything in packages_before_downloaded.txt that I may have installed myself, in a separate list called "Probably yours: not suggested". It is not part of any tier, and you remove an app from it only if I name it.

PARTNER PRELOADS go into Recommended, NOT into "Probably yours". These are third-party apps the phone maker or carrier preinstalls for money. They are usually downloaded-type (path in /data/app, /my_preload, /my_stock/del-app, /product/data-app or /cust/app). If UAD-ng does not list one, mark it "AI knowledge, unverified". They include:
- Casual games, for example Candy Crush, Block Blast, Vita Mahjong, Tile Explorer, Water Sort, Bubble Pop, Fruit Shooting, Word Connect, and other preinstalled games such as those from katanlabs.
- Shopping and deals apps, for example Amazon, Meesho and Flipkart.
- Commonly preloaded social, networking and AI apps, for example Snapchat, LinkedIn and ChatGPT.
Mark each one "†" and write in "What I lose": "Partner preload. If you use it, exclude it. Reinstall it from the Play Store." WhatsApp, Instagram, Facebook, Netflix and Spotify stay in "Probably yours" even when they were preloaded.

KEEP FOR SAFETY even when UAD-ng rates them Recommended or Advanced, and name them in the review as "left out for safety":
- SIM, modem and calling parts: package names with a part that starts with "uim" or "sim", or that contain "atfwd", "iwlan" (VoWi-Fi), "xdivert" (call divert), "qcrilmsgtunnel", "confdialer", "vsim", "simactivate", "entitlement" or "kryten". Also com.android.stk (SIM toolkit), com.android.mms and com.android.mms.service.
- Emergency alerts: descriptions mentioning emergency, alerts or public announcements (for example com.qualcomm.qti.xrcb), and personal-safety apps.
- Wi-Fi and Bluetooth helpers: descriptions mentioning Wi-Fi, Bluetooth, aptX, captive portal or hotspot login.
- Display colour, power saving, touch, biometric or payment authentication helpers (for example colorservice, qcolor, powersavemode, ifaa, soter, trustservice, touchservice, credentialmanager).
- Android mainline modules, which Google Play system updates need: anything whose name contains "mainline", and adservices, ondevicepersonalization and federatedcompute.
- Parts of the OEM home screen, lock screen or quick settings (for example miui.systemui.plugin, launcher "res" or "layout" packages, always-on display, theme manager, wallpaper pickers).

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
   Number the items per tier (R1, R2 ... for Recommended, V1 ... for Advanced, E1 ... for Expert) so I can name them in my reply.
   Mark every downloaded-type package (Phase 2) with a "†" and say in "What I lose": "The undo file cannot restore it. Reinstall it from the Play Store."
   On Xiaomi HyperOS, mark Xiaomi's own user-facing apps (for example ShareMe, Scanner, Compass, Weather, Notes, Calculator, Screen Recorder) as "HyperOS may refuse this (Failure [-1000])". Tested on HyperOS 3: all seven were refused.
3. If no Glance or lock-screen feed was found, say so clearly, and list the package names you checked for.
4. The PROTECTED apps from Phase 2, so I know they will be kept.
5. The "Probably yours: not suggested" list and the "left out for safety" list from Phase 3, as short lists of names.
6. The count of packages per group and tier.

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
                  a preinstalled system app can still be restored with the rollback command, but a
                  downloaded-type app (marked † in Phase 4) is deleted completely and must be reinstalled from the Play Store)
     disable:    adb shell pm disable-user --user 0 <package>
     restrict:   adb shell cmd appops set <package> SYSTEM_ALERT_WINDOW ignore
     mute:       adb shell cmd appops set <package> <op> ignore   (for each op in A4)
3. Check the result:
     uninstall must print "Success".
     disable must print "new state: disabled-user".
     For restrict and mute, run adb shell cmd appops get <package> <op> and confirm it shows "ignore".
   On any other result, log it as failed, do not try another method, and move on.
   Known results that are not dangerous, so log them and keep going:
     - "Failure [-1000]" on Xiaomi HyperOS: the system protects that app from uninstalling. It stays installed and unchanged.
     - "Failure [not installed for 0]": the app was already removed.
     - POST_NOTIFICATION reads back as "allow" on Android 13+, or SYSTEM_ALERT_WINDOW reads back as "default" instead of "ignore": the system overrode the change. Log it as failed and, at the end, give me the manual Settings steps (or offer to uninstall the app instead, if it is in Group B).
   Any other failure is unexpected: stop and explain before the next package.
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
  b. Estimated: the sum of App Size + Data Size + Cache Size for the removed packages, from diskstats_before.txt. If diskstats does not list a package, say so. On some phones (tested: HyperOS 3 / Android 16), diskstats has no "Package Names", "App Sizes", "App Data Sizes" or "Cache Sizes" lines at all, only partition totals. Then write "not measured: diskstats on this phone has no per-app sizes".
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
- App stores: which were removed, which were muted, and any that need notifications turned off by hand (Android 12 or older, or where POST_NOTIFICATION did not stick).
- Failed items: each one, why it failed, and what I can do by hand instead.
- The files created and where they are.
- How to undo everything: connect the phone, enable USB debugging, open a terminal in this folder, and run rollback.bat (Windows) or sh rollback.sh (macOS/Linux). List the downloaded-type apps (†) that the undo file cannot bring back, and say that they can be reinstalled from the Play Store.
- That major system updates can bring some apps back, and this process can be run again afterwards.
No claims beyond what the measurements show.

=====================================================================
PHASE 8: ASK IF I WANT TO REMOVE MORE APPS
=====================================================================
After the report, always end by asking me:
  "Do you want to remove any more apps?"
Next to the question, show a short numbered list of what is still possible: the Group B items I did not approve (by tier), the "Probably yours: not suggested" list, and any Group B apps that failed but could be disabled instead. Do not ask for anything or change anything else.
- If I say no, we are done.
- If I name apps or tiers, treat that as a new Phase 4 approval for only those items. Apply them with Phase 5 (rollback lines first, newest first), then do one restart and the Phase 6 checks, then update the report with new "after" numbers (Phase 7). Then ask this question again.
- If I name an app that is not on any list, check it against UAD-ng and the HARD DENYLIST first, and tell me what it is and what I would lose before doing anything.

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
  adb reboot   (once, in Phase 6; again only if something broke and was undone, or once per extra round in Phase 8)

Start with Phase 0 now.
```
