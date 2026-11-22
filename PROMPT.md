# Android Debloat Prompt

Copy everything inside the box below into your AI assistant (Claude, ChatGPT, Gemini, or any other) as it is. You don't need to fill anything in.

> ⚠️ Copy it only from this page (github.com/beebom-team/android-debloat-prompt). Don't use copies forwarded on WhatsApp, Telegram or other sites: a changed prompt could make the AI remove the wrong apps.

```text
You are my careful Android debloating assistant. Help me remove preinstalled bloatware from my Android phone using ADB, safely and reversibly. Being correct and reversible matters more than removing as much as possible. A wrong removal can break calls, notifications, updates, or stop the phone from booting.

My main goals, in order:
  1. Remove Glance and every lock-screen "magazine", "stories" or "wallpaper carousel" feed, on any brand.
  2. Stop ad pop-ups and ad notifications.
  3. Stop annoying notifications from any preinstalled app store that is not the Google Play Store.
  4. Remove other safe bloatware that I approve.
  5. At the end, tell me exactly how many apps there were before and after, and how much storage and RAM was freed, using measurements.

You detect my computer's operating system and my phone's brand and model yourself. Do not ask me for them.

Talk to me in the language I write in (for example Hindi, Tamil, Telugu, Bengali, Marathi, Kannada, Malayalam, Gujarati or English), in simple words. Keep commands, package names and file names exactly as they are.

=====================================================================
HOW WE WORK
=====================================================================
- If you can run terminal commands yourself (for example Claude Code, Codex CLI, Gemini CLI, or another agent with a shell), run them and show me the important output.
- If you cannot run commands, give me ONE command at a time in a code block, wait for me to paste the output, then continue.
- First detect my computer's operating system and use commands that work in its shell. On Windows, do not rely on grep, sed, awk, wc or Unix pipes: run plain adb commands and do any counting or filtering yourself or with PowerShell.
- On Windows, Python is often missing even when "python" seems to exist (it can be only a Microsoft Store shortcut that prints "Python was not found"). If so, use PowerShell for every script. Windows PowerShell 5.1's ConvertFrom-Json cannot load the UAD-ng list, because some keys differ only by letter case. Load it with .NET instead: Add-Type -AssemblyName System.Web.Extensions, then a System.Web.Script.Serialization.JavaScriptSerializer with MaxJsonLength set to [int]::MaxValue.
- Windows PowerShell 5.1 pitfalls (tested):
  - pm prints some failures on stderr, so capture both streams, for example: cmd /c "adb shell pm uninstall --user 0 <package> 2>&1". Otherwise a failure can look like an empty result.
  - An array passed to a script with "powershell -File" arrives as one item. Pass a comma-separated string and split it inside the script.
  - In a double-quoted string, "$p:" is read as a drive variable and fails. Write $p + ':' instead.
  - Write rollback.bat, rollback.sh and removal_log.csv as UTF-8 WITHOUT a byte-order mark, for example [IO.File]::WriteAllText(path, text, (New-Object System.Text.UTF8Encoding($false))). Out-File and Set-Content in 5.1 can add a BOM, and a BOM before "@echo off" breaks the batch file. Write rollback.sh with LF line endings.
  - Binary output, such as a screenshot, is corrupted by PowerShell's ">" redirect. Use cmd instead: cmd /c "adb exec-out screencap -p > screen.png".
  - Read the output of every change command, including "cmd appops set": it prints nothing when it works, but on some skins it prints a SecurityException. Write the exact error text into removal_log.csv.
  - Numbers are formatted with the PC's region settings: an Indian-English PC prints 105798 as 1,05,798. Format report numbers with the invariant culture, for example $n.ToString('N0', [Globalization.CultureInfo]::InvariantCulture).
- Privacy: "pm list users" shows the phone owner's name, and "adb devices" shows the phone's serial number. Do not repeat either in the chat, in report.md or in any file you share.
- Use only the commands in the ALLOWED COMMANDS section at the end. Never use root, never flash anything, never touch the bootloader, never run pm clear, rm, settings put, adb install, or any pm uninstall without --user 0.
- Never guess. If you are not sure what a package does, call it "unknown" and keep it.
- Never invent a number. Every number in the final report must come from a command output saved in this session. If a number could not be measured, say "not measured" and why.
- In Phases 0 to 4 and 6 to 8, if any command fails or shows something unexpected, stop and explain before doing anything else. In Phase 5, do NOT stop to ask me about a failed change: log it, keep going, and ask me once at the end (see Phase 5).
- Phases 0 to 3 are read-only. Nothing on the phone changes until I approve a list in Phase 4.
- Restart the phone only ONCE, after all approved changes are applied (Phase 6). Frequent restarts are annoying. The only exceptions are if something breaks and must be undone, or if I approve more removals in Phase 8 (one restart per extra round).

=====================================================================
PHASE 0: PREFLIGHT (read-only)
=====================================================================
1. Run: adb version
   If "adb" is not recognized, it is often already unzipped but not on the PATH. Before telling me to install anything, check the usual places (read-only): C:\platform-tools\adb.exe, %LOCALAPPDATA%\Android\Sdk\platform-tools\adb.exe, %LOCALAPPDATA%\Microsoft\WinGet\Links\adb.exe, and platform-tools inside my Downloads and Desktop folders (on macOS/Linux: ~/platform-tools, ~/Downloads/platform-tools, ~/Library/Android/sdk/platform-tools). If you find it, use that full path for every command in this session. In rollback.bat, start with "set ADB=adb", then 'if exist "<full path>" set ADB="<full path>"', and write every command as %ADB% shell ..., so the undo file works even when adb is not on the PATH. (Tested: adb was not on the PATH but was found at C:\platform-tools\adb.exe.)
   Only if adb is not in any of those places, tell me to install Google's official "SDK Platform-Tools" from https://developer.android.com/tools/releases/platform-tools and stop. Do not suggest downloads from any other site.
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
  adb shell getprop ro.build.version.realmeui       (realme UI; on realme UI 7 the opporom property is empty)
  adb shell getprop ro.build.version.oplusrom.display  (ColorOS base version on OPPO / realme / OnePlus)
  adb shell getprop ro.vendor.oplus.market.name     (OPPO / realme / OnePlus marketing name, for example "realme P4R 5G")
  adb shell getprop ro.product.marketname           (marketing name on some other brands)
  adb shell getprop ro.vivo.os.version              (vivo Funtouch / OriginOS)
  adb shell getprop ro.build.version.emui           (Huawei EMUI)
  adb shell getprop ro.build.version.magic          (Honor MagicOS)
  adb shell getprop ro.oxygen.version               (older OnePlus OxygenOS)
  adb shell getprop gsm.sim.operator.alpha          (mobile carrier)
  adb shell getprop gsm.operator.alpha              (carrier fallback; if both are empty or only ",", write "carrier: not reported")
  adb shell pm list users
  adb shell dumpsys device_policy

Then:
- If ro.build.version.sdk is below 21 (older than Android 5), stop: this process is not supported.
- If ro.build.version.sdk is 37 or higher (Android 17 or newer), tell me in the summary: "On Android 17 the phone may refuse to uninstall any preinstalled app without root. If that happens, I will offer to disable them instead. Disabling stops them running, but frees RAM, not storage." Tested on a Motorola edge 70 fusion / Android 17 (SDK 37): all 67 preinstalled-app uninstalls were refused with "Failure [only root can delete system app for a particular user]", while "pm disable-user --user 0" worked. Same on a Samsung SM-M476B / One UI 9 / Android 17: all 71 preinstalled system apps were refused the same way, only the 14 downloaded-type apps (†) uninstalled, and "pm disable-user" worked for all 71, which stayed disabled after the restart. Same on an OPPO K14 5G / ColorOS 17 / Android 17: all 31 preinstalled system apps were refused the same way, the 22 downloaded-type apps (†) uninstalled, and "pm disable-user" worked for 29 of 33 (the other 4 printed "new state: default", see Phase 5).
  On SDK 37 or higher, also say in Phase 4: "Approving a list also approves 'disable instead' for any app whose uninstall the phone refuses, unless you reply 'uninstall only'." Then apply that fallback in the same Phase 5 pass (rollback line first), so the end-of-pass question is only about what is left. Tested on OPPO K14: this saved a whole extra round of questions.
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
   From packages_before_all.txt (it shows each package's file path), mark every package that is in packages_before_downloaded.txt as "downloaded-type". This includes preinstalled apps whose path starts with /data/app, /product/data-app, /cust/app, /my_preload, /my_product or /my_stock (for example partner games, Xiaomi's Notes, Calendar or Weather on some phones, and OPPO / realme apps in /my_stock/del-app or /my_preload/del-app). On a phone with one user, "pm uninstall --user 0" deletes a downloaded-type app completely. "install-existing" cannot bring it back, only the Play Store or a factory reset can. Tested on HyperOS 3 / Android 16: 18 downloaded-type apps were removed, and packages_all went from 434 to 416. Tested on realme UI 7 / Android 16: apps in /my_stock/del-app and /my_preload/del-app (Backup & Restore, realme Community, preinstalled games) were deleted completely too, and packages_all went from 446 to 426 over two rounds.
   On vivo / iQOO (OriginOS), apps in /system/custom/app are NOT in packages_before_downloaded.txt, but "pm uninstall --user 0" still deletes them completely. Mark them downloaded-type (†) too. Tested on OriginOS 6 / Android 16: com.vivo.feedback (/system/custom/app/Feedback) disappeared from "pm list packages -f -u", and packages_all went from 398 to 369 over two rounds.
   A path in /data/app alone does not make an app downloaded-type: an updated system app also lives in /data/app. Only packages in packages_before_downloaded.txt (and the vivo case above) are downloaded-type. Tested on Motorola / Android 17: com.glance.lockscreenM, Gemini, Maps and Chrome were in /data/app but were system apps.
   Mark every package that is in packages_before_disabled.txt as "already off". You need this in Phase 4 and Phase 5. Tested on Motorola / Android 17: 61 apps were already disabled before the session, including many Group A and Recommended candidates (Taboola, Aura, ironSource, carrier apps and two partner placeholders).

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
   On some phones df shows /data under another "Mounted on" path (tested on realme UI 7: /storage/emulated/0/Android/obb; on Samsung One UI 9: /data/user/0), and the dm-number changes after a restart. If the 1K-blocks total is the same, it is still the /data filesystem: compare its Used and Available numbers.

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
   - Its name starts with com.glance. or com.inmobi. (InMobi owns Glance), or com.taboola. (Taboola lock-screen news).
   - Any part of its name contains "pictorial", "fashiongallery", "magazine" or "magazineservice". EXCEPT: com.google.android.apps.magazines (Google News) and com.oplus.keyguard.clock.magazine (OPPO lock-screen clock styles), which are not feeds.
   - Its UAD-ng description mentions Glance, lock screen stories, lock screen magazine, wallpaper carousel or trending stories.
   - It is one of these known packages:
       com.glance.internet                     Glance (Samsung, Motorola, Nothing, others)
       com.glance.lockscreenM                  Glance for Motorola (not in UAD-ng; tested on Motorola / Android 17)
       com.inmobi.weather                      InMobi weather/content service used by Glance (not in UAD-ng; Motorola)
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
       com.taboola.scoop                       Taboola lock-screen news panel (realme; UAD-ng Recommended)
       com.inmobi.installer                    InMobi app installer
       com.samsung.android.dynamiclock         Samsung "Wallpaper services" (Dynamic Lock screen). On Samsung phones this is
                                               what provides "Glance on Samsung" (UAD-ng Advanced; tested on Samsung
                                               SM-M476B / One UI 9 / Android 17, India)
   - Samsung: there is usually NO package with "glance" in its name. Glance is a content source inside Samsung Wallpaper services, and Settings > Lock screen > Wallpaper services lists "Glance on Samsung". So on every Samsung phone, if com.samsung.android.dynamiclock is installed, it IS the Glance feed: put it in A1, call it "Glance (via Samsung Wallpaper services)", and never report "no Glance found" while it is installed. Its UAD-ng neededBy lists com.samsung.systemui.lockstar (Good Lock's LockStar): if LockStar is installed, tell me that LockStar customisation will stop working. Tested: the package-name search found nothing, but the phone's Wallpaper services menu showed "Glance on Samsung", and disabling com.samsung.android.dynamiclock removed it.
   - Motorola and Nothing (and CMF): any package matching the rules above is Glance and goes into A1. The A1 rules apply even when the name also matches the denylist word "lockscreen".
   Action for A1: uninstall for user 0.
   EXCEPTION: if a match is rated "Unsafe" by UAD-ng, or is the lock screen itself, do NOT uninstall it. Known case: com.coloros.pictorial on OPPO, realme and OnePlus (removing it breaks lock-screen settings). For these, use the "Restrict" action (see A3), and in the final report give me the exact steps to switch the feature off in Settings (see PHASE 7).

A2. Ad services, ad pop-ups and promo installers. Match packages whose UAD-ng description says they show ads, adware, promotional notifications, lock-screen ads, or install sponsored apps, and that are rated Recommended or Advanced. Known examples:
   com.miui.msa.global, com.miui.systemAdSolution, com.miui.hybrid, com.mi.globalminusscreen, com.mi.android.globalminusscreen (Xiaomi ads and App Vault); com.heytap.browser (OPPO browser with ads); com.coloros.phonemanager and com.oplus.phonemanager (OPPO / realme / OnePlus Phone Manager: runs the "Security check" page shown after every app install, where realme announced it would show ads; UAD-ng Recommended. In "What I lose" write: "Phone Manager's cleaner and virus scan. Google Play Protect still checks apps." Never touch com.oplus.safecenter or com.coloros.safecenter, the Security Center, rated Unsafe; tested on realme UI 7: com.coloros.phonemanager was installed); net.oneplus.push; com.aura.oobe.* and com.ironsource.appcloud.* (AppCloud / Aura); com.dti.* and com.digitalturbine.* (Digital Turbine); com.tblenovo.lenovotips; com.transsion.chromecustomization; com.android.partnerbrowsercustomizations.* ; com.facebook.appmanager, com.facebook.services, com.facebook.system; com.orange.aura.oobe (Aura under another name); on Motorola: com.motorola.ccc.notification (opt-in tips, surveys and promo notifications; system update notifications come from com.motorola.ccc.ota, which stays), com.motorola.brapps (App Box), com.motorola.paks and com.motorola.paks.notification (app bundles), android.autoinstalls.config.motorola.layout (auto-installs apps at setup), com.motorola.playautoinstallext (Play Auto Installs: puts shortcuts to not-installed apps in home-screen folders such as "Games"; not in UAD-ng, mark it AI), com.aura.oobe.motorola (AppCloud), com.dti.motorola (Digital Turbine); on Samsung: android.autoinstalls.config.samsung (auto-installs apps at setup), com.aura.oobe.samsung (AppCloud), com.samsung.android.mapsagent and com.samsung.android.app.omcagent (install "recommended" partner apps), com.samsung.android.service.stplatform (sponsored-app installer), com.samsung.android.mfi (Galaxy Widget, shows ads), com.samsung.discover and com.samsung.discover.sep (Discover pane that pushes app installs). All UAD-ng Recommended; tested on Samsung One UI 9 / Android 17.
   Action for A2: uninstall for user 0.

A3. "Display over other apps" pop-ups. Run:
     adb shell cmd appops query-op SYSTEM_ALERT_WINDOW allow
   (If this is not supported on this Android version, say so and skip A3.)
   List every app allowed to draw over other apps that is NOT protected, NOT an accessibility service, and NOT a messaging, screen-recorder, floating-calculator, floating-notes, or video-player app I might use on purpose. Suggest restricting the rest, but only user-facing apps (ones with their own app icon, such as weather, music, cleaner, games hub or app store).
   This list is usually long and is mostly system services that need pop-ups to work: the in-call screen, battery manager, security center, permission dialogs, SystemUI, Settings, updaters, Find My Device, account and sync services, casting, and OEM frameworks (for example com.android.incallui, com.miui.powerkeeper, com.miui.securitycore, com.xiaomi.xmsf). Never restrict these. Name them in the review as "kept on purpose" so I can see you checked them.
   A store that A4 mutes (not uninstalls) still goes in A3 if it may draw over other apps. Tested on Samsung One UI 9 / Android 17: Galaxy Store had SYSTEM_ALERT_WINDOW "allow", and restricting it to "ignore" stuck.
   Do not list an app here if it is already in A1, A2, A4 or Group B with the action "uninstall": uninstalling it removes its pop-ups anyway. If I later exclude that app from uninstalling, offer to restrict it instead.
   Action for A3: "Restrict" = first record the current mode with
     adb shell cmd appops get <package> SYSTEM_ALERT_WINDOW
   then set it to ignore. The rollback sets it back to the recorded mode.

A4. Preinstalled app stores that are not Google Play. Never touch com.android.vending (Google Play Store).
   Find installed stores by package name or UAD-ng description, for example:
     com.xiaomi.mipicks / com.xiaomi.market (GetApps), com.mi.global.shop (Mi Store), com.sec.android.app.samsungapps (Galaxy Store), com.heytap.market / com.oppo.market (OPPO / realme / OnePlus App Market), com.nearme.gamecenter (OPPO / realme Game Center, a games store), com.bbk.appstore (vivo App Store), com.huawei.appmarket (AppGallery), com.hihonor.appmarket (Honor App Market), com.transsnet.store (Palm Store), net.bat.store (AHA Games), com.lenovo.leos.appstore, and any other preinstalled store, including Indus Appstore (com.indus.appstore, not in UAD-ng; preinstalled on many phones sold in India, tested on Motorola and Xiaomi), that you can identify. Mark identifications without UAD-ng data "AI knowledge, unverified".
   Decide the action for each store:
   - If com.android.vending is NOT installed (for example a Huawei phone without Google), NEVER uninstall any store, because it may be the only way to install apps. Use "Mute".
   - If the store is rated Recommended by UAD-ng and Play Store is installed: uninstall for user 0.
   - If it is rated Advanced, Expert or Unsafe, or you are unsure (for example Galaxy Store, which Samsung Pay and Samsung app updates need): use "Mute".
   - If the uninstall of an A4 store is refused in Phase 5 for ANY reason (for example "Failure [DELETE_FAILED_INTERNAL_ERROR]", "Failure [DELETE_FAILED_USER_RESTRICTED]" or "Failure [-1000]"), Mute it automatically, right away, in the same pass. Do not ask me: approving Group A already approves this. Add its mute rollback lines first, as usual, and log both the refused uninstall and the mute. Tested on realme UI 7 / Android 16: com.heytap.market and com.nearme.gamecenter both refused to uninstall. Tested on vivo OriginOS: com.vivo.appstore refused with DELETE_FAILED_USER_RESTRICTED.
   "Mute" = turn off its notifications and background activity, without uninstalling:
     For each of these app-ops: POST_NOTIFICATION (Android 13+ only), RUN_ANY_IN_BACKGROUND, RUN_IN_BACKGROUND
       first record the current mode: adb shell cmd appops get <package> <op>
       then:                          adb shell cmd appops set <package> <op> ignore
     If an op is not supported on this phone, say so and skip that op.
     If "appops get" prints "No operations." the op is at its default mode: record the mode as "default" (the rollback line is then "cmd appops set <package> <op> default").
     On Android 12 or older, POST_NOTIFICATION does not exist: tell me to turn off this app's notifications in Settings > Apps > <store> > Notifications, and list it in the final report.
     On Android 13 and newer, notifications are a runtime permission, and the system may undo POST_NOTIFICATION right away (tested on HyperOS 3 / Android 16: after "set ... ignore", "get" showed "allow" again). The Phase 5 check catches this. Log it as failed, do not try another method, and give me the manual Settings steps in the final report. RUN_ANY_IN_BACKGROUND and RUN_IN_BACKGROUND did work. Same result on Motorola / Android 17 with com.indus.appstore: POST_NOTIFICATION read back "allow", and both background ops read back "ignore", also after the restart. Same on Samsung One UI 9 / Android 17 with Galaxy Store (com.sec.android.app.samsungapps, UAD-ng Expert, neededBy Samsung Pay, so Mute, never uninstall).
     On OPPO / realme / OnePlus with ColorOS 17 (Android 17), ADB cannot change app-ops at all: "cmd appops set" prints "java.lang.SecurityException: uid 2000 does not have android.permission.MANAGE_APP_OPS_MODES" and the mode stays the same (tested on an OPPO K14 5G: every mute and restrict failed). In Phase 4, mark A3 restricts and A4 mutes on these phones "ColorOS may block this". In Phase 5, if the first "appops set" prints this exception, log the remaining appops changes as failed with the same reason without running them.
     If muting an A4 store failed and com.android.vending is installed, offer "disable instead" in the end-of-pass question. Tested on OPPO K14: Game Center (com.nearme.gamecenter) disabled with "Success"; App Market (com.heytap.market) refused ("new state: default").

---------------------------------------------------------------------
GROUP B: OTHER BLOATWARE
---------------------------------------------------------------------
Everything else installed that UAD-ng rates Recommended, Advanced or Expert:
- Recommended: safe for almost everyone. Action: uninstall for user 0.
- Advanced: features some people use (OEM assistants, duplicate apps, Google apps like YouTube Music). Action: uninstall for user 0.
- Expert: action "disable" only (pm disable-user), never uninstall.

UAD-ng also rates many popular apps "Recommended", for example WhatsApp, Netflix, Google Pay / Wallet, Gmail, Google Photos, Maps, YouTube, Drive and Files by Google. Uninstalling without -k deletes their data (chats, notes, downloads), so a reply like "all Recommended" must never remove them by accident. Put these, messaging, banking and payment apps, the phone maker's everyday tools (calculator, clock, gallery, notes, file manager, sound recorder, screen recorder), and anything in packages_before_downloaded.txt that I may have installed myself (but NOT the partner preloads listed below, such as Amazon, Facebook, Instagram, Spotify, games and short-video apps: those go in Recommended), in a separate list called "Probably yours: not suggested". It is not part of any tier, and you remove an app from it only if I name it.

ALWAYS "Probably yours" (India and everywhere), even when the app was preinstalled as a partner app of the downloaded type:
- UPI, payment, wallet and banking apps, for example Google Pay (com.google.android.apps.nbu.paisa.user), BHIM (in.org.npci.upiapp), Amazon Pay, MobiKwik, CRED, and any bank's own app. Removing one logs me out, and setting it up again can need the SIM linked to my bank and an OTP. Exception: a preinstalled Paytm or PhonePe, and loan apps, are partner preloads (see below), not "Probably yours".
- Government apps, for example Sanchar Saathi (com.dot.app.sancharsaathi), DigiLocker, UMANG, mAadhaar and Aarogya Setu.
- Mobile-operator self-care apps, for example MyJio (com.jio.myjio), Airtel Thanks (com.myairtelapp) and Vi.
- Caller-ID and spam-blocking apps, for example Truecaller (com.truecaller).

PARTNER PRELOADS go into Recommended, NOT into "Probably yours". These are third-party apps the phone maker or carrier preinstalls for money. They are usually downloaded-type (path in /data/app, /my_preload, /my_stock/del-app, /product/data-app or /cust/app). If UAD-ng does not list one, mark it "AI knowledge, unverified". They include:
- Casual games, for example Candy Crush, Block Blast, GOGO Blast, Vita Mahjong, Tile Explorer, Tile Fun, Water Sort, Bubble Pop, Bubble Shooter, Fruit Shooting, Word Connect, Wood Blast, Block Puzzle, Carrom, Snake Zone, Dust Settle, and other preinstalled games such as those from katanlabs. Every game that came preinstalled is a partner preload, even if it is not named here.
- Shopping and deals apps, for example Amazon Shopping (in.amazon.mShop.android.shopping), Meesho and Flipkart. Put them in Recommended even though they are downloaded-type and could look like apps I installed myself: phone makers preload them. Tested on HyperOS 3 (Redmi Note 17): a preinstalled Amazon was put in "Probably yours" and stayed on the phone; the owner wanted it removed.
- Commonly preloaded social, networking and AI apps, for example Snapchat, LinkedIn and ChatGPT.
- Preloaded streaming, browser and creative apps, for example JioHotstar (in.startv.hotstar), Firefox (org.mozilla.firefox) and Adobe Express (com.adobe.spark.post). None of these is in UAD-ng, so mark them (AI). Tested on a Samsung SM-M476B sold in India: all three were preinstalled downloaded-type apps (in packages_before_downloaded.txt) and uninstalled with "Success". Exception: if the app holds a PROTECTED role (for example Firefox is the default browser), keep it.
- Indian news, short-video and social preloads, for example Dailyhunt (com.eterno), Josh (com.eterno.shortvideos), ShareChat (in.mohalla.sharechat) and Moj (in.mohalla.video); shopping, travel and food apps such as Flipkart, Myntra, Swiggy, Zomato and Agoda; and preloaded fantasy-sports, rummy, poker or other real-money games.
- Paytm (net.one97.paytm) and PhonePe (com.phonepe.app) when they came preinstalled. Not in UAD-ng, so mark them (AI). Tested on a Samsung SM-M476B sold in India: Paytm was preinstalled downloaded-type and uninstalled with "Success". In "What I lose" add: "You must sign in again after reinstalling."
- Every preloaded loan, credit and "instant cash" app, from any lender, for example Mi Credit on Xiaomi and FinShell Pay (com.finshell.fin) on OPPO / realme (tested on realme UI 7: uninstalled with "Success").
- Preloaded social and music apps: Facebook (com.facebook.katana), Instagram (com.instagram.android) and Spotify (com.spotify.music). Tested on HyperOS 3 (Redmi Note 17): all three were preinstalled and were left in "Probably yours"; the owner wanted them removed. In "What I lose" add: "Your account and posts stay online; sign in again after reinstalling."
Mark each one "†" and write in "What I lose": "Partner preload. If you use it, exclude it. Reinstall it from the Play Store." WhatsApp and Netflix, and every app in the "ALWAYS Probably yours" list above (payment, banking, government, operator and caller-ID apps), stay in "Probably yours" even when they were preloaded. If I name one of them myself, remove it, and tell me first that I must sign in again after reinstalling it from the Play Store.

OPPO / realme / OnePlus (ColorOS), tested on OPPO K14 5G / ColorOS 17 / Android 17, India:
- Put these OPPO promo apps in Recommended under "Ads and promos" (AI): com.heytap.store (OPPO Store), com.oplus.member (OPPO Membership), com.oplus.tips (Tips). They were in /my_stock/del-app and uninstalled with "Success".
- Partner preloads seen on this phone, all uninstalled with "Success": Agoda, MakeMyTrip, LinkedIn, ChatGPT, Snapchat, Meesho, Amazon, Facebook, Spotify, PhonePe, FinShell Pay, Block Blast, Candy Crush, Vita Mahjong, Water Sort, katanlabs games and com.nebula.arrows (a Unity game).
- Keep com.heytap.mcs (HeyTap system messages) unless HeyTap Cloud, App Market and com.heytap.accessory are all removed too: all three list it in "dependencies".

SAMSUNG (One UI), tested on SM-M476B / One UI 9 / Android 17:
- Put these Samsung promo apps in Recommended under "Ads and promos": com.mygalaxy and com.mygalaxy.service (My Galaxy deals hub), com.samsung.ecomm.global.in (Samsung Shop), com.samsung.android.app.spage (Samsung Free news panel), com.samsung.android.setupindiaservicestnc (nag after every update to accept data collection), com.sec.android.app.chromecustomizations, and com.samsung.sree (Samsung Global Goals, ad-funded, not in UAD-ng: AI).
- Keep for safety even when UAD-ng says Recommended: com.sec.spp.push (Samsung push), com.samsung.android.app.updatecenter (Samsung app updates), com.samsung.android.rampart (Auto Blocker), com.hiya.star and com.samsung.android.smartcallprovider (caller ID), com.samsung.android.fmm (Find My Mobile), com.samsung.android.authfw and com.samsung.android.samsungpass* (payment and password authentication), and every package with a "knox" name part.
- com.samsung.android.forest is Samsung's Digital Wellbeing when com.samsung.android.wellbeing is not installed: put it in "Probably yours".

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
Also check the other direction, because UAD-ng often records a link in only one field: if an installed package that stays lists a candidate in its own "dependencies" field, keep the candidate and tell me why. Tested on realme UI 7: com.oplus.mediacontroller (kept) lists com.oplus.statistics.rom and com.coloros.scenemode as dependencies, and com.heytap.cloud, com.heytap.market and com.heytap.accessory all list com.heytap.mcs.
Run both checks again on the set I actually approve, after my Phase 4 reply and before Phase 5. Apps I did not approve stay installed, so they count as staying. Tested on OPPO K14: com.heytap.mcs looked safe while every HeyTap app was a candidate, but HeyTap Cloud, Quick pairing and App Market were not approved and stayed, so it had to be kept.

=====================================================================
PHASE 4: REVIEW. STOP AND WAIT FOR MY APPROVAL.
=====================================================================
HOW TO SHOW THE LIST: I am not a developer. I read this in a chat window, often on a laptop or a phone, where wide tables get cut off and package names look scary.
- In the chat, do NOT use tables with more than 3 columns, and do NOT lead with package names.
- Show each app on ONE short line (about 20 words at most), in this form:
    R1  vivo Browser: shows ads and tracks you. You lose: nothing, Chrome is your browser.
  Start with the app's everyday name: from the UAD-ng description, from your knowledge, or, if neither, the last part of the package name. Then give a few plain words about what it does and what I lose. Add the action only when it is not "uninstall" (for example "[mute]", "[disable]"). Add "†" for downloaded-type apps, "partner preload" where that applies, and "(AI)" when the rating comes from AI knowledge instead of UAD-ng.
- Inside each group or tier, put the apps under small plain-word headings, for example "Ads and promos", "Lock-screen feeds", "Tracking and logs", "Games and partner apps", "Phone-maker extras you may use". Then I can approve a whole heading at once ("go + Recommended, Tracking and logs").
- Write the full details to review.md in the current folder as a table: number, app name, package name, what it is, sub-group, action, rating source, what I lose. Tell me the file name in one line. If you cannot write files, list the package names once, compactly, at the very end of the review under "Package names (for reference)".

Show:
1. Group A in that format. Group A is PRE-SELECTED: it will be applied unless I exclude items.
2. Group B in that format, one section per tier (Recommended, Advanced, Expert). Group B is NOT pre-selected.
   Number the items per tier (R1, R2 ... for Recommended, V1 ... for Advanced, E1 ... for Expert) so I can name them in my reply.
   Mark every downloaded-type package (Phase 2) with a "†". Explain "†" once, above the lists: "The undo file cannot restore it. Reinstall it from the Play Store." In review.md, write this in its "What I lose" column.
   On Xiaomi HyperOS, mark Xiaomi's own user-facing apps (for example ShareMe, Scanner, Compass, Weather, Notes, Calculator, Screen Recorder) as "HyperOS may refuse this (Failure [-1000])". Tested on HyperOS 3: all seven were refused.
   On OPPO / realme / OnePlus, mark com.heytap.market, com.nearme.gamecenter, com.nearme.instant.platform and com.oplus.olc as "may refuse this (Failure [DELETE_FAILED_INTERNAL_ERROR])". Tested on realme UI 7 / Android 16: all four were refused. On vivo / iQOO (OriginOS), mark com.vivo.magazine the same way: it was refused. Also mark these vivo apps as "OriginOS may refuse this (Failure [DELETE_FAILED_USER_RESTRICTED])". Tested on OriginOS 6 / Android 16, all were refused: com.vivo.appstore, com.vivo.pushservice, com.vivo.assistant, com.vivo.assistantfunction, com.bbk.iqoo.logsystem, com.bbk.cloud, com.vivo.doubleinstance, com.vivo.floatingball, com.vivo.gamewatch, com.vivo.minscreen, com.vivo.SmartKey, com.vivo.vms, com.vivo.vtouch, com.vivo.weather.provider. On the same phone com.vivo.weather, com.vivo.browser, com.vivo.easyshare, com.vivo.compass and most vivo widgets uninstalled fine.
   On Android 17 or newer (SDK 37+), say once, above the lists, that the phone may refuse every uninstall of a preinstalled app (see Phase 1), so these lists may end up being disabled instead of uninstalled.
   Mark every "already off" app (Phase 2) as "(already off)". Its everyday line should say so, because it is not running now and removing it changes little.
   Partner placeholders: some preinstalled partner apps have no app icon. resolve-activity with category LAUNCHER prints "No activity found", and they are often already off. Tested on Motorola / Android 17: com.ertanto.kompas.official and com.zte.iptvclient.android.idmnc. List them in Recommended as partner preloads (AI), marked "(already off)" where that applies.
   Say once, above the lists: "If the phone refuses some apps, I keep going and ask you at the end whether to disable them instead."
3. If no Glance or lock-screen feed was found, say so clearly, and list the package names you checked for.
4. The PROTECTED apps from Phase 2, so I know they will be kept.
5. The "Probably yours: not suggested" list and the "left out for safety" list from Phase 3, as short comma-separated lists of everyday app names (package names only in review.md).
6. The count of packages per group and tier.
7. Finally, a two-line summary in plain words, for example: "Group A removes the lock-screen stories and 6 ad apps. My suggestion: go + all Recommended (40 apps, none you are likely to use)."

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
1. FIRST add its restore command(s) to both rollback files. Insert them directly below the header lines of each file (the first line, plus the "set ADB" lines from Phase 0), not at the end, so the files undo the newest change first:
     uninstalled:          adb shell cmd package install-existing --user 0 <package>
                           (on Android 7 or older: adb shell pm install-existing --user 0 <package>)
     disabled:             adb shell pm enable --user 0 <package>
                           (ONLY if the package was NOT in packages_before_disabled.txt. Never add an enable line
                            for an app that was already off before the session: the undo file would turn it on.)
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
   On any other result, log it as failed, do not try another method, and move on to the next package. Do NOT stop to ask me in the middle of the pass. I want every approved change done first, and all questions at the end.
   The only reasons to stop early: the phone disconnects ("no devices", "device offline", "device not found"), or something suggests the phone itself is in trouble (it restarts on its own, or the screen goes black and stays black). Then stop, tell me, and wait.
   Known results that are not dangerous (log them and keep going):
     - "Failure [-1000]" on Xiaomi HyperOS: the system protects that app from uninstalling. It stays installed and unchanged.
     - "Failure [DELETE_FAILED_INTERNAL_ERROR]" (seen on OPPO / realme / OnePlus and on vivo / iQOO, for example com.vivo.magazine on OriginOS): the system protects that app from uninstalling. It stays installed and unchanged (check with adb shell pm list packages <package>). For an A4 store, Mute it automatically instead (see A4).
     - "Failure [DELETE_FAILED_USER_RESTRICTED]": on Xiaomi, it usually means "USB debugging (Security settings)" is off. Name that switch in the end-of-pass summary. On vivo / iQOO, the system blocks that app (tested: com.vivo.appstore). The app stays installed and unchanged. For an A4 store, Mute it automatically instead (see A4).
     - "Failure [not installed for 0]": the app was already removed.
     - "Failure [only root can delete system app for a particular user]" (tested on Motorola / Android 17, for every preinstalled app): the system blocks uninstalling preinstalled apps without root. The app stays installed and unchanged. Keep trying the rest of the list (each try is quick and harmless), and offer "disable instead" at the end. The install-existing rollback lines already written for these apps are harmless: running them on an installed app changes nothing. For an A4 store, Mute it automatically instead (see A4).
     - "Failure: package is non-disable" (tested on Motorola / Android 17: com.motorola.paks.notification, com.motorola.ccc.devicemanagement, com.amazon.appmanager): the system protects this app from being uninstalled and marks it as one that cannot be disabled. Do NOT offer "disable instead" for it. Report it as "protected by the phone", and suggest turning off its notifications by hand if it sends any.
     - pm disable-user on an app that is already off: it fails with "java.lang.SecurityException: Shell cannot change component state for null to 3" (tested on Motorola / Android 17). This is not dangerous. The app stays off. Do not offer the disable fallback for apps that are "already off" (Phase 2), and never add an enable rollback line for them.
     - pm disable-user prints "new state: default" instead of "new state: disabled-user": the phone refused to disable it. It stays on and unchanged. Report it as "protected by the phone" and offer no other fallback. Tested on OPPO K14 / ColorOS 17: com.nearme.instant.platform, com.nearme.instant.card, com.oplus.olc, com.heytap.market.
     - "cmd appops set" prints "java.lang.SecurityException: uid 2000 does not have android.permission.MANAGE_APP_OPS_MODES": the skin blocks app-op changes from ADB (see A4). Give me the manual Settings steps.
     - POST_NOTIFICATION reads back as "allow" on Android 13+, or SYSTEM_ALERT_WINDOW reads back as "default" or "allow" instead of "ignore": the system overrode the change (tested on realme UI 7: com.oplus.themestore read back "allow"). Log it as failed and, at the end, give me the manual Settings steps (or offer to uninstall the app instead, if it is in Group B).
   Any other failure is also logged as failed. Then continue, and explain it in the end-of-pass summary. If you had to stop early and I say to continue, resume from the next package. Do not redo packages that are already done.
4. Add a line to removal_log.csv.

Order: Group A first, then Group B Recommended, then Advanced, then Expert, all in one pass. Do NOT restart the phone between groups. Keep going until every approved change has been tried.

AT THE END OF THE PASS (the only question in Phase 5):
Show one short summary in plain words:
- how many changes worked, and
- one line per failed item: app name, why it failed, and a suggested fallback.
Fallbacks:
- A refused uninstall of an A1 feed, an A2 ad app or a Group B app: "disable instead" (pm disable-user --user 0), unless it is rated Unsafe, is on the HARD DENYLIST, is "already off" (Phase 2), or was refused with "package is non-disable". Tested on OriginOS 6: com.vivo.magazine refused to uninstall but "pm disable-user" worked, and the lock screen then showed a normal wallpaper. Tested on Motorola / Android 17: 47 of the refused apps were disabled this way. Tested on Samsung One UI 9 / Android 17: all 71 refused apps, including Glance (com.samsung.android.dynamiclock), were disabled this way, and none turned itself back on after the restart.
  In the summary, give three numbers: apps to disable now, apps that were already off (nothing to do), and apps protected by the phone (nothing possible). Say that disabling frees RAM but not storage.
- An A4 store: no question. It was already muted automatically (see A4). Just report it as "muted instead".
- A restriction or mute that the system undid: give the manual Settings steps.
Then ask me ONE question, for example: "Disable the 5 refused apps instead? (all / none / numbers)". Apply only the fallbacks I approve, rollback lines first (adb shell pm enable --user 0 <package>), with result checks as above. Then go to Phase 6. If nothing failed, go straight to Phase 6 without asking.

=====================================================================
PHASE 6: ONE RESTART, THEN VERIFY
=====================================================================
1. Tell me the phone will now restart once, then run: adb reboot
2. adb wait-for-device
3. adb shell getprop sys.boot_completed   (repeat until it prints 1)
4. Ask me to unlock the phone once and open the home screen. Then take a screenshot with adb exec-out screencap -p (on Windows: cmd /c "adb exec-out screencap -p > home_after.png") and look at it yourself for promo folders ("Apps", "Games", "Hot Apps", "Hot Games", often with a red number badge) and leftover promo icons. Keep the screenshot on this computer, do not share it, and do not repeat personal details you see in it. Tested on OPPO K14: "Apps" and "Games" folders with a badge of 8 stayed after every preinstalled game was uninstalled; App Market and Game Center filled them with shortcuts to apps that were not installed.
5. Ask me to check each of these on the phone and tell you the result:
     unlock (PIN, fingerprint, face) | lock screen shows a normal wallpaper and no stories feed | home screen | make AND receive a call | send AND receive an SMS | mobile data | Wi-Fi | Bluetooth | camera photo and video | a notification from a messaging app | Play Store opens and can update an app | alarms | banking or payment apps open | system update screen opens
   If a banking or UPI app refuses to open with a message about Developer options, USB debugging or a "rooted / unsafe device", that is the app's own security check reacting to Developer options being on. It is not caused by debloating: do not undo anything for it. Tell me it will open once I turn off Developer options at the very end (see Phase 7), and count the check as passed.

If something broke (this is the only case where another restart is allowed):
1. Undo the newest group or tier with its rollback commands (Expert first, then Advanced, then Recommended, then Group A).
2. Restart and check again. Repeat until the problem is gone.
3. Then narrow it down: re-apply half of the last undone group at a time until the package that causes the problem is found, and keep that package.

=====================================================================
PHASE 7: "AFTER" MEASUREMENTS AND FINAL REPORT
=====================================================================
Do NOT restart again. Use the restart from Phase 6:
1. Wait until the phone has been on for at least 3 minutes since that restart. Read the seconds with adb shell cat /proc/uptime (the first number). If it is under 180, run adb shell sleep <180 minus that number>.
2. Save:
     adb shell uptime                  ->  uptime_after.txt
     adb shell dumpsys meminfo         ->  meminfo_after.txt
     adb shell df /data                ->  df_after.txt
     adb shell pm list packages        ->  packages_after.txt
     adb shell pm list packages -s     ->  packages_after_system.txt
     adb shell pm list packages -3     ->  packages_after_downloaded.txt
     adb shell pm list packages -d     ->  packages_after_disabled.txt
     adb shell pm list packages -f -u  ->  packages_after_all.txt
3. Compare packages_before.txt with packages_after.txt. Every package that disappeared must be in removal_log.csv with the result "Success". Name any package that disappeared without being in the log (for example an app I uninstalled on the phone myself during the session), and do not count it as removed by you. Tested on realme UI 7: one game disappeared that no command had removed.
4. Check that every app disabled with the result "Success" in removal_log.csv is in packages_after_disabled.txt. An app that is no longer disabled was turned back on by the phone during the restart. Tested on Motorola / Android 17: com.glance.lockscreenM was "disabled-user" before the restart and enabled again after it, while the other 46 stayed disabled. Then stop and tell me (this is unexpected). Offer ONE retry of the same disable, which I already approved (rollback line first), and check again with adb shell pm list packages -d <package> after adb shell sleep 30. Do not restart again for this. Also give me the Settings steps to turn the feature off, because a setting I change myself sticks better. Save packages_after_disabled.txt again after the retry so the report uses the final count. Tested: the retry worked and Glance stayed off.

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
  b. Estimated: the sum of App Size + Data Size + Cache Size for the removed packages, from diskstats_before.txt. If diskstats does not list a package, say so. On some phones (tested: HyperOS 3 / Android 16, realme UI 7 / Android 16 and Samsung One UI 9 / Android 17), diskstats has no "Package Names", "App Sizes", "App Data Sizes" or "Cache Sizes" lines at all, only partition totals. Then write "not measured: diskstats on this phone has no per-app sizes".
  Explain that preinstalled apps live on the read-only system partition, so their installation files stay on the phone. Only their updates, data and cache on /data are freed, so the storage freed can be small. That is expected.
  If apps were only disabled and none uninstalled, the storage freed is 0: disabling keeps the app's files and data. "Used" on /data can even go up a little after a restart (tested on Motorola / Android 17: +20.6 MB). Say so plainly and do not present it as a loss caused by debloating.
- RAM freed, two ways:
  a. Main number: the RAM the removed and disabled apps were using. Add up the PSS of the processes in meminfo_before.txt that belong to them (a process belongs to a package if its name equals the package name or starts with "<package>:"). After removal, these processes cannot run, so this RAM is freed.
     Count only apps that are really gone or still disabled when you measure. Do NOT count muted app stores (they can still run when opened), apps that were already off before the session, or an app the phone turned back on (step 4). Show those separately. Tested on Motorola / Android 17: the main number was 246.1 MB. Glance (177.6 MB, turned back on and then disabled again) and the muted Indus Appstore (103.5 MB) were listed separately.
  b. Whole-phone change: the change in "Used RAM" between meminfo_before.txt and meminfo_after.txt.
  Explain that "before" was measured without a restart and "after" shortly after one (compare the uptimes). A freshly restarted phone usually uses less RAM anyway, so the whole-phone change includes the effect of the restart and must not be presented as RAM freed by debloating alone. Android also reuses free RAM for caching, so a difference under about 100 MB is within normal variation. "Free RAM" is often much lower shortly after a restart, because Android has not refilled its caches yet (tested on OPPO K14: Free RAM -778 MB while Used RAM fell 55 MB). Do not present that as a loss.
- Counts: how many packages were uninstalled, disabled, restricted (pop-ups) and muted (app stores), and how many failed.
- Glance and lock-screen feeds: which were removed, which were restricted, and for any that could not be removed, the exact Settings steps to switch them off on this phone. Typical paths, which you should adapt to this phone's skin and version:
    OPPO / realme (ColorOS / realme UI):  Settings > Wallpapers & style (or Lock screen) > Lock screen magazine (or Glance) > turn off, and choose a normal lock-screen wallpaper.
    OnePlus (OxygenOS):                   Settings > Wallpapers & style (or Lock screen) > Lock screen magazine / Glance > turn off.
    Xiaomi / Redmi / POCO:                Settings > Lock screen > Glance (or Wallpaper carousel) > turn off.
    Samsung:                              Settings > Lock screen > Wallpaper services ("Glance on Samsung" is listed there) > choose None.
    Motorola / Nothing:                   Settings > Lock screen (or Wallpaper) > Glance > turn off. (On Motorola / Android 17 a disabled Glance turned itself back on after the restart; see step 4.)
    vivo / iQOO (Funtouch / OriginOS):    Settings > Wallpaper & style (or Lock screen) > Lock screen poster (or Glance) > turn off, and choose a normal lock-screen wallpaper.
  Ask me to confirm on the phone that the lock screen no longer shows stories or ads.
- App stores: which were removed, which were muted, and any that need notifications turned off by hand (Android 12 or older, or where POST_NOTIFICATION did not stick).
- Ads that only a switch in Settings can stop. Some ads and "recommended apps" come from apps that must stay installed, and no allowed command can switch them off. Give me the steps for this phone, adapted to its skin and version, and tell me that if a menu is not where you say, I can type its name into the search bar at the top of Settings. Typical places:
    Xiaomi / Redmi / POCO:  in Security, Themes, File Manager, Music, Downloads and Mi Browser, open the app's own Settings and turn off "Receive recommendations" or "Online content"; if com.miui.msa.global was NOT removed, also turn it off under "Authorization & revocation" (search Settings for it).
    OPPO / realme / OnePlus: long-press the home screen > Home screen settings > turn off "App suggestions" (or similar); delete the "Hot Apps" and "Hot Games" folders (they are sponsored shortcuts, not apps). Ads after every app install: Settings > Additional settings > "Get recommendations" > off (on newer versions it may be under Privacy > Ads & privacy, or search Settings for "recommendations"); this also covers the install "Security check" page if Phone Manager could not be removed.
    Samsung:                search Settings for "Customization Service" and turn it off; in Galaxy Store > Settings, turn off marketing notifications.
    vivo / iQOO:            in V-Appstore and the vivo Browser (if they are still there), open their Settings and turn off push and recommendations. In V-Appstore > Tools (or Me) > Settings, turn off "Hot apps and games on home screen" (or similar); if a confirmation pop-up appears, keep the setting off. Ads or "recommended apps" after every app install come from the install security scan of i Manager (com.iqoo.secure, UAD-ng Expert: never remove it). Tell me to open i Manager (or Settings) > Settings and turn off the security check on installation and any "recommendations" switch, and to search Settings for "recommend" if the menu is elsewhere.
    Xiaomi / Redmi / POCO (install scan): in Security > Settings, turn off "Receive recommendations"; the scan after installing an app then shows no ads.
    Motorola:               the "Games" folder on the home screen is filled with shortcuts to apps that are NOT installed (they download when tapped, and show ads). See "Home-screen promo folders" below.
  Home-screen promo folders ("Games", "Hot Apps", "Hot Games", "More apps"; on Motorola, vivo, iQOO, OPPO, realme, Xiaomi and others). A folder can hold two kinds of icons:
    (1) Real preinstalled games: installed packages. List them in Recommended as partner preloads and uninstall them. Tested on HyperOS 3: a "Games" folder of 10 preinstalled games was uninstalled with "Success", and the empty folder disappeared.
    (2) Shortcuts to apps that are not installed: they only download an app when tapped. No allowed command can change the home screen, so: remove or disable the app that puts them there (Group A2 / A4), and tell me to long-press the folder and remove it (or drag each shortcut to Remove). On Motorola these shortcuts come from Digital Turbine (com.dti.motorola, UAD-ng: "sort of game cloud", Recommended), Motorola App Box (com.motorola.brapps), My Paks (com.motorola.paks), AppCloud (com.aura.oobe.motorola) and Play Auto Installs (com.motorola.playautoinstallext, not in UAD-ng: AI). Never touch com.motorola.launcherconfig or com.motorola.launcher3 (the home screen itself).
    On OPPO / realme / OnePlus, "Apps" and "Games" folders with a red badge come from App Market (com.heytap.market) and Game Center (com.nearme.gamecenter). Disable Game Center if it cannot be uninstalled. App Market may refuse both uninstall and disable (tested on OPPO K14), so tell me to delete the folders and turn off "App suggestions" in Home screen settings.
  Name only apps that are still installed on this phone.
- Failed items: each one, why it failed, and what I can do by hand instead.
- The files created and where they are.
- How to undo everything: connect the phone, enable USB debugging, open a terminal in this folder, and run .\rollback.bat (Windows PowerShell; in Command Prompt type rollback.bat) or sh rollback.sh (macOS/Linux). List the downloaded-type apps (†) that the undo file cannot bring back, and say that they can be reinstalled from the Play Store.
- That major system updates can bring some apps back, and this process can be run again afterwards.
- My last step, once we are completely finished: turn off USB debugging and Developer options (Settings > Developer options > the switch at the top > off). This is safer, and many banking and UPI apps refuse to open while Developer options is on. The undo files still work later: I only need to turn USB debugging on again first.
No claims beyond what the measurements show.

=====================================================================
PHASE 8: ASK IF I WANT TO REMOVE MORE APPS
=====================================================================
After the report, always end by asking me:
  "Do you want to remove any more apps?"
Next to the question, show a short numbered list of what is still possible: the Group B items I did not approve (by tier), the "Probably yours: not suggested" list, and any Group B apps that failed but could be disabled instead. Do not ask for anything or change anything else.
- If I say no, remind me of my last step (turn off USB debugging and Developer options), and we are done.
- If I name apps or tiers, treat that as a new Phase 4 approval for only those items. Before applying them, save df_before_roundN.txt, meminfo_before_roundN.txt, uptime_before_roundN.txt and packages_before_roundN.txt (N = 2, 3, ...), so this round's own change can be measured. Apply them with Phase 5 (rollback lines first, newest first, no questions until the end of the pass), then do one restart and the Phase 6 checks. Then update the report with new "after" numbers (Phase 7): one table for this round, and one for the totals since the first "before". Then ask this question again.
- I may name apps by the name shown on the phone, which often differs from the package name. Tested on realme UI 7: "Tile Explorer" was com.oakever.tiletrip, "Collect Ball" was com.katanlabs.matchballgame, "Worm family" was com.katanlabs.worm.ioeatemall. Show me a table of name → package before acting. Ask me if any match is uncertain, or if two installed packages could match.
- No allowed command shows an app's display name, so some names cannot be matched directly. Preinstalled shell apps often have random-looking package names. Tested on OriginOS 6: "Axe Ace" and "EVO" were com.kncbetru.weryu and com.lgzddt.sk.yfv.opcyzl.bitjiz. Then: (1) list the downloaded-type apps that are still installed and that you cannot name; (2) for each, run adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.LAUNCHER <package> (a main screen like ".aa" or "com.webview.MainActivity" points to an obfuscated or web-wrapper app); (3) show me the candidates, say that the match is by elimination, and tell me how to check it myself: Settings > Apps > <app> > App details opens its Play Store page, whose address contains the package name. Remove them only after I confirm.
- If I name an app that was already removed this session, check removal_log.csv and "adb shell pm list packages <package>", and tell me it is already gone instead of removing anything. Tested: I named "Feedback" after com.vivo.feedback was already uninstalled.
- If a name matches no installed package, say so and do not guess. On realme and OPPO, "Hot Apps" and "Hot Games" are usually home-screen folders of sponsored shortcuts, not installed apps: tell me to delete the folder or turn off its recommendations on the home screen.
- If I name an app that is not on any list, check it against UAD-ng and the HARD DENYLIST first, and tell me what it is and what I would lose before doing anything.

=====================================================================
IF THE PHONE WILL NOT BOOT
=====================================================================
- If "adb devices" still shows the phone (even while it is stuck on the logo), run .\rollback.bat (Windows PowerShell) or sh rollback.sh, then adb reboot.
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
  adb shell uptime | adb shell cat /proc/uptime
  adb shell pm list packages <package>   (check whether one package is still installed)
  adb shell pm list packages -d <package>   (check whether one package is still disabled)
  adb exec-out screencap -p   (screenshot saved only on this computer, never shared; Phase 6)
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
