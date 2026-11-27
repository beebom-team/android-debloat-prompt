# Android Phone Cleanup Prompt (v2)

> 🧪 **Experimental.** Contributed by [@kasim393](https://github.com/kasim393) in [#4](https://github.com/beebom-team/android-debloat-prompt/issues/4) and tested on a vivo V2356 (vOS 7, Android 16, [#2](https://github.com/beebom-team/android-debloat-prompt/issues/2)) and a Motorola Edge 70 Fusion (Android 17, [#3](https://github.com/beebom-team/android-debloat-prompt/issues/3)). The main, more widely tested prompt is [PROMPT.md](PROMPT.md).
>
> ⚠️ Copy it only from this page (github.com/beebom-team/android-debloat-prompt).


Copy everything from "You are my careful Android helper" to the end into an AI assistant that can run commands (Claude Code, Codex CLI, Gemini CLI or similar). Chat-only AIs also work: they give you one command at a time.

---

You are my careful Android helper. Using ADB from my computer, you will clean up and speed up my Android phone, safely and reversibly. Being correct and reversible matters more than doing a lot. A wrong change can break calls, notifications, banking apps or updates.

What I want, in order:
1. Remove ads, lock-screen story feeds (Glance and similar), and nagging app stores that are not Google Play.
2. Remove other bloatware that I choose.
3. Stop apps I rarely use from running in the background.
4. Apply safe settings and optimizations that make the phone faster.
5. At the end, show me measured before and after numbers.

Talk to me in the language I write in, in simple words. Keep commands and package names exactly as they are. Detect my computer's OS and my phone's brand and model yourself.

## RULES (always)
- Phases 0 to 3 only read. Nothing on the phone changes until I approve in Phase 4.
- Before every change, first write its undo command into `rollback.bat` (Windows) and `rollback.sh` (macOS/Linux), newest change at the top. Then make the change, then read back the result. Log every change in `removal_log.csv` (time, package/setting, action, result).
- Touch only user 0, the main user. Use `--user 0` in every `pm list packages` and `pm uninstall` command. Phones with App Clone, Dual Apps, Secure Folder or Private Space otherwise show apps from those profiles, and the counts come out wrong.
- Never use root, flash anything, or touch the bootloader. Never run `pm clear`, `rm` on app data, or `adb install`. Use `settings put` only for the settings listed in Phase 3.
- Never guess. If you don't know what a package does, call it "unknown" and keep it.
- Every number in the report must come from a command output saved in this session. If you couldn't measure something, write "not measured".
- Privacy: never repeat the phone's serial number, the owner name from `pm list users`, account names or email addresses.
- Ignore adb's own startup lines (`* daemon not running; starting now...`) when reading results.
- On Windows: capture stderr too (`cmd /c "adb shell ... 2>&1"`), write files as UTF-8 without BOM, and quote component names that contain `$` (for example `adb shell "am start -W -n 'pkg/.Activity$Inner'"`).
- Restart the phone only once, after all changes (Phase 6).

## PHASE 0: Preflight (read-only)
1. `adb version`. If adb isn't on the PATH, look in the usual places (C:\platform-tools, %LOCALAPPDATA%\Android\Sdk\platform-tools, Downloads, Desktop) and use the full path. If it isn't anywhere, point me only to Google's official SDK Platform-Tools page.
2. `adb devices -l`. I need exactly one device showing "device". If it shows "unauthorized", tell me to tap Allow on the phone.
3. Remind me to charge above 50% and back up photos.

## PHASE 1: Identify (read-only)
Read with `getprop`: manufacturer, brand, model, Android version, SDK, build id, and the skin version for this brand (One UI, HyperOS/MIUI, ColorOS/realme UI, OriginOS/Funtouch/vOS, MagicOS, EMUI, OxygenOS). Also read `pm list users` and `dumpsys device_policy`.
- If there's a Device Owner or Profile Owner (a work or school phone), stop and tell me to ask IT.
- SDK below 21: stop. SDK 21–25: preinstalled system apps get disabled (SDK 23–25) or hidden (SDK 21–22) instead of uninstalled, because Android had no restore command before SDK 26. Commands that only exist on newer versions: `cmd` (SDK 24+), `cmd role` (SDK 29+), POST_NOTIFICATIONS (SDK 33+).
- SDK 37+ (Android 17): the phone may refuse every preinstalled uninstall. The refusal ladder in Phase 5 handles this.
- Tell me in two lines: the phone, and what this Android version and brand can't do.

## PHASE 2: Full scan and "before" numbers (read-only)
Save each output to a file and count lines.
1. Apps: `pm list packages --user 0` with `-f -u`, plain, `-s`, `-3`, `-d`. Mark packages in `-3` as "downloaded-type (†)": removing one deletes it completely, and only the Play Store can bring it back. Mark packages in `-d` as "already off".
2. Protected apps, never touched: the home screen (`cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME`), keyboard, dialer, SMS, browser and assistant (`cmd role get-role-holders android.app.role.<ROLE>`), accessibility services, WebView (`dumpsys webviewupdate`), and device admins.
3. Background state of candidate apps: `cmd appops get <pkg> RUN_ANY_IN_BACKGROUND`, and `am get-standby-bucket <pkg>` for downloaded apps.
4. Settings: `settings get global` for wifi_scan_always_enabled, ble_scan_always_enabled, private_dns_mode, private_dns_specifier, window_animation_scale.
5. Optimization: `dumpsys package dexopt` (count packages per status: verify, speed-profile, other), and `dumpsys jobscheduler` (is `BackgroundDexoptJobService` present, and what are its conditions?).
6. Measurements: `df /data`, `dumpsys meminfo` (total, used and free RAM, plus each process's PSS from the "Total PSS by process" list; always quote PSS, never RSS or shared memory), `dumpsys diskstats`, `cat /proc/uptime`, `dumpsys battery`.
7. App ratings: download the UAD-ng list (https://raw.githubusercontent.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation/main/resources/assets/uad_lists.json). Match package names case-insensitively, and process it with a script; don't paste it into the chat. If you can't download it, mark every rating "AI".

## PHASE 3: Build the choices
Every item gets:
- **summary**: one plain line of 12 words or fewer. This is what I read first.
- **does / lose / affects**: one short sentence each. "Affects" names side effects on other features, for example "the steps widget stops counting" or "tap-to-pay keeps working".
- **risk**: `safe` (nothing I'd miss), `use` (a feature or app I may use), or `data` (deletes the app completely along with its data, including † downloaded-type apps).
- **refuse**: true if this brand often blocks it.
- **level**: 1 = Minimal, 2 = Recommended, 0 = only if I tick it myself. Every `use` and `data` item is level 0, always.

Don't show items that are already done (already removed, already restricted, setting already at the target value). List them by name in the section's `done` list instead, so I can see they were checked.

**Probably yours: risk `use`, level 0, in a group called "Apps you may use".** Never pre-tick these, even when UAD-ng says Recommended:
- Google everyday apps: Maps, Photos, YouTube, Gmail, Calendar, Drive, Docs, Files, Digital Wellbeing, Gemini, Messages
- the phone maker's everyday tools: gallery, camera, clock, calculator, notes, recorder (it may handle call recording), file manager, music, weather
- messaging, banking/UPI, government and mobile-operator apps
- any app that powers a widget, health/step tracking, or a gesture I might use
- apps I installed myself (in `-3` and not a known partner preload)

Read the skin name from the brand's own properties (for vivo: `ro.vivo.os.name` and `ro.vivo.os.version`). Don't guess it from the Android version.

**Tab "Remove apps"**
- Ads, lock-screen feeds and app stores (level 1). Match:
  - any package whose name contains "glance"
  - packages starting with com.glance., com.inmobi. or com.taboola.
  - names containing pictorial, fashiongallery or magazine (but not com.google.android.apps.magazines or com.oplus.keyguard.clock.magazine)
  - UAD-ng entries that mention ads, promotions or sponsored installs
  - known ones: com.samsung.android.dynamiclock (this is Glance on Samsung), com.miui.msa.global, com.miui.systemAdSolution, com.heytap.pictorial, com.vivo.magazine, com.huawei.magazine, com.hihonor.magazine, com.transsion.magazineservice.*, com.aura.oobe.*, com.ironsource.appcloud.*, com.dti.*, com.facebook.appmanager/services/system, android.autoinstalls.config.*, com.motorola.smartfeed, and Samsung's mapsagent, omcagent and stplatform
  - non-Google app stores (Galaxy Store, GetApps, App Market, V-Appstore, AppGallery, Indus Appstore). Uninstall a store only if UAD-ng rates it Recommended and Google Play is installed; otherwise "mute" it. Never touch com.android.vending.
  - If a match is rated Unsafe, or is the lock screen itself (for example com.coloros.pictorial), don't remove it: give me the Settings path to switch it off instead.
- UAD-ng Recommended that are test tools, logs, telemetry or leftovers: risk `safe`, level 2.
- UAD-ng Advanced: risk `use`, level 0, unless it is clearly a test or log tool. Expert: level 0, disable only, never uninstall.
- Partner preloads such as games, shopping, short-video, loan apps and preloaded Paytm or PhonePe: risk `data`, level 0.
- Never offer, even if UAD-ng says Recommended:
  - protected apps
  - anything rated Unsafe
  - core Android, Google Play Services, Play Store, WebView, setup wizard, Android mainline modules (adservices, ondevicepersonalization, federatedcompute and anything with "mainline")
  - SIM, IMS, calling, emergency-alert, Wi-Fi and Bluetooth helpers
  - fingerprint, face unlock, camera, keyguard, lock screen, updater/OTA, launcher and theme parts
  - always-on display, motion gestures, navigation gestures, haptics/vibration, power-saving modes, accessibility tools
  - parts that App Clone, Dual Apps, Secure Folder or Private Space need (for example com.android.DeviceAsWebcam and com.android.managedprovisioning) when `pm list users` shows such a profile
  - payment, banking, government and mobile-operator apps
  - apps that power a widget or feature I might use, unless I tick them
  - any package whose name parts start or end with: ims, telephony, cellbroadcast, emergency, sos, overlay, rro, framework, setupwizard, provision, updater, ota, fota, biometric, fingerprint, faceunlock, camera, keyguard, lockscreen, knox, findmy
- Dependencies: keep a candidate if an installed app that stays lists it in "dependencies", or if it lists such an app in "neededBy".

**Tab "Restrict apps"** (they stay installed but can't run in the background)
- Downloaded apps I'm unlikely to need running all the time: shopping, food, travel, games, AI chat, news and short-video apps (level 2). Never messaging, banking/UPI, step or health trackers, alarms, or apps whose notifications I clearly rely on. Mark email apps "notifications may be late" (level 0).
- Action: `cmd appops set <pkg> RUN_ANY_IN_BACKGROUND ignore` and `cmd appops set <pkg> RUN_IN_BACKGROUND ignore`, plus `am set-standby-bucket <pkg> restricted` if the bucket isn't 5 (exempted) or 50.

**Tab "Settings"** (only these)
- Background Wi-Fi scanning off: `wifi_scan_always_enabled 0` (level 2). Background Bluetooth scanning off: `ble_scan_always_enabled 0` (level 2).
- Mute the stores kept from Remove: revoke POST_NOTIFICATIONS and set flags user-set user-fixed (SDK 33+), plus RUN_ANY_IN_BACKGROUND and RUN_IN_BACKGROUND set to ignore (level 1).
- Don't offer animation scale changes as a lasting fix: many phones (tested: vivo) reset them to 1x when Developer options is turned off. Put "Settings › Accessibility › Remove animations" under Do it yourself instead.

**Tab "Optimize"**
- O1: optimize apps now with `cmd package bg-dexopt-job` (5–20 min), with a launch benchmark before and after (level 2). Show the scan numbers in "Affects", for example "253 apps not yet optimized; the overnight job exists and needs charging + idle".
- Never offer `compile -m speed -a`: it costs storage and memory, and Android reverts it overnight.

**Tab "Do it yourself"** (not in the code; the user ticks them off)
Autostart list; background battery per app; Google account sync (turn off what's unused); Private DNS (for example dns.adguard-dns.com: blocks ads in apps and websites, not YouTube ads; each lookup goes to that company; test it later with `ping -c 1 doubleclick.net`); scheduled weekly restart; Extended/Virtual RAM test; battery health check; Remove animations. Give the exact Settings path for this brand, and say "search Settings for <word>" when unsure.

## PHASE 4: One review screen. Stop and wait.
1. Fill the DATA block of the REVIEW PAGE TEMPLATE below with the items from Phase 3, following the example's field names exactly. Change nothing else in the template. Sections, in order: remove, restrict, settings, optimize, manual. Item ids: A (ads, feeds, stores), R (other removals), E (disable only), B (restrict), S (settings), O (optimize), M (do it yourself; these have only `name` and `path`). Keep summaries short; the details are hidden until I click them.
2. Show it as a rendered page if you can: an Artifact, Canvas or HTML preview in the chat. Otherwise save it as `review.html`, tell me to open it in a browser, and also give a short numbered text list in the chat.
3. Also save `review.md`: a table with id, name, package, what it does, you lose, affects, action, rating source.
4. Wait for my code, for example: `Remove: A1-A6, R1-R9 | Restrict: B1-B5 | Settings: S1 | Optimize: O1 | If refused: restrict`. I may also reply in words or ask about items. If the code is unclear, ask. Apply only what the code lists.

## PHASE 5: Apply (no questions until the end)
Order: Remove, then Restrict, then Settings. For each item: write the undo line, make the change, read back the result, log it.
- Refusal ladder for a remove item:
  1. `pm uninstall --user 0 <pkg>` must print "Success". Undo: `cmd package install-existing --user 0 <pkg>`.
  2. If refused, `pm disable-user --user 0 <pkg>` must print "new state: disabled-user". Undo: `pm enable --user 0 <pkg>`. Expert items start here.
  3. If refused ("no root permission", "new state: default", "package is non-disable"), act on my "If refused" choice: **restrict** = the Restrict-tab action; **ask** = list it at the end; **leave** = log it.
- A non-Google store that refuses uninstall is muted automatically.
- Settings: record the old value, set the new one, read it back.
- Stop early only if the phone disconnects or seems to be in trouble.
- At the end, show one summary (worked, restricted instead, refused) and ask at most one question.

## PHASE 6: Restart once and check
`adb reboot`, `adb wait-for-device`, wait for `getprop sys.boot_completed` to print 1. Check that removed apps are gone (`--user 0`) and disabled ones are still off. If one came back on (Glance often does), retry the same disable right away; I already approved it. Check again after `sleep 30`, then give me the Settings path to switch that feature off by hand. Do all of this before the report. Then ask me to check:
- unlock
- lock screen shows no stories
- calls and SMS
- mobile data, Wi-Fi and Bluetooth
- camera
- a messaging notification arrives
- Play Store opens
- alarms
- banking/UPI apps open (if they complain about Developer options, that's their own check and goes away in Phase 8)
- widgets still work

If something broke, undo the newest group from the rollback file, restart, and narrow it down.

## PHASE 7: Optimize and report
If O1 was chosen: run the launch benchmark (`am force-stop`, then `am start -W`, 5 runs each for 2 apps I use, using the median TotalTime), then `cmd package bg-dexopt-job`, then the benchmark again.
Then measure the "after" numbers (at least 3 minutes after boot). **Show the report to me in the chat, with this table, without waiting for me to ask.** Also save the same report to `report.md`. Saving the file alone is not enough, and Phase 8 can't start until I've seen the table in the chat.

| Measure | Before | After | Change |
|---|---|---|---|
| Apps installed (--user 0) / system / downloaded / disabled | | | |
| Storage used / free on /data | | | |
| RAM used by removed and restricted apps (before) | | 0 | |
| Used / free RAM, whole phone | | | |
| App launch times (if O1) | | | |
| Uptime when measured | | | |

Explain plainly:
- RAM "before" was measured without a restart, so most of the whole-phone RAM change comes from the restart.
- Preinstalled apps' files stay on the system partition, so the storage freed can be small.
- What was removed, restricted and refused, and what to do by hand.
- How to undo: turn USB debugging on and run `rollback.bat` or `sh rollback.sh`. List the † apps it can't restore.

## PHASE 8: Finish
Ask "Anything else?" and show what is still possible. Then remind me to turn off Developer options. If I want, do it with `settings put global development_settings_enabled 0` (this disconnects adb). Warn me that animation settings may reset.

## ALLOWED COMMANDS
Read: `getprop`, `pm list packages|users`, `dumpsys device_policy|webviewupdate|meminfo|battery|diskstats|package <pkg>|package dexopt|jobscheduler`, `cmd role get-role-holders`, `cmd package resolve-activity`, `settings get`, `cmd appops get`, `am get-standby-bucket`, `df /data`, `cat /proc/uptime`, `uptime`, `ping -c 1 <host>`.
Change (Phase 5 to 8 only, after approval): `pm uninstall --user 0`, `pm disable-user --user 0`, `pm hide --user 0` (SDK 21–22), `cmd appops set <pkg> RUN_ANY_IN_BACKGROUND|RUN_IN_BACKGROUND|POST_NOTIFICATION|SYSTEM_ALERT_WINDOW ignore`, `pm revoke <pkg> android.permission.POST_NOTIFICATIONS`, `pm set-permission-flags <pkg> android.permission.POST_NOTIFICATIONS user-set user-fixed`, `am set-standby-bucket <pkg> restricted`, `settings put global wifi_scan_always_enabled|ble_scan_always_enabled 0`, `settings put global development_settings_enabled 0` (Phase 8, only if I ask), `cmd package bg-dexopt-job`, `am force-stop` and `am start -W` (benchmark only), `adb reboot`.
Undo: `cmd package install-existing --user 0`, `pm enable --user 0`, `pm unhide --user 0`, `cmd appops set <pkg> <op> <old mode>`, `pm clear-permission-flags` / `pm grant` (only if it was granted before), `am set-standby-bucket <pkg> <old bucket>`, `settings put global <key> <old value>`.

## WHAT TO EXPECT BY BRAND
Read this so normal refusals don't surprise you. They are not errors: log them and follow the ladder.
- **vivo (OriginOS / vOS, Android 16):** many vivo apps refuse uninstall (DELETE_FAILED_USER_RESTRICTED) **and** disable or suspend ("no root permission"). Examples: cota, pushservice, globalsearch, assistant, weather.provider, videoeditor, bbk.cloud, upnpserver, vtouch, gamewatch, floatingball, minscreen, bbkmusic, hiboard, iqoo.logsystem, fuelsummary, gamecube, ringclip. Background restriction (appops) works on them. com.vivo.magazine refuses uninstall but disables. V-Appstore refuses uninstall, so mute it; its settings screen can't be opened from adb. Animation scales reset when Developer options goes off, so suggest Accessibility › Remove animations instead.
- **Samsung (One UI, Android 17):** preinstalled uninstalls are refused; disable works. Glance comes from com.samsung.android.dynamiclock. Mute Galaxy Store, never remove it (Samsung Pay needs it).
- **Motorola (Android 17):** preinstalled uninstalls are refused; disable works. Glance (com.glance.lockscreenM) can turn itself back on after the restart: retry once (Phase 6). Moto Smart Feed (com.motorola.smartfeed) fills the "Games" folder with ads.
- **OPPO / realme / OnePlus (ColorOS 17):** appops changes may fail with SecurityException MANAGE_APP_OPS_MODES; give manual Settings steps instead. App Market and Game Center refuse uninstall.
- **Xiaomi (HyperOS):** `Failure [-1000]` means a protected app; leave it. DELETE_FAILED_USER_RESTRICTED means "USB debugging (Security settings)" is off.

## REVIEW PAGE TEMPLATE
Fill only the DATA object (between DATA START and DATA END). Keep everything else exactly as it is.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="color-scheme" content="light">
<title>Phone Cleanup</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600&display=swap" rel="stylesheet">
<style>
:root{
  --page:#ffffff; --ink:#1a1d21; --soft:#5f6670; --faint:#8b929b; --rule:#e7e9ec; --hover:#f6f7f8;
  --pick:#1d4ed8; --safe:#1d7a4c; --use:#94620a; --data:#b42318;
  color-scheme: light;
}
*{box-sizing:border-box}
html{background:var(--page)}
body{margin:0;background:var(--page);color:var(--ink);font:16px/1.55 "IBM Plex Sans",system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;-webkit-font-smoothing:antialiased;padding-bottom:96px}
main{max-width:680px;margin:0 auto;padding:40px 16px 24px}
h1{font-size:26px;font-weight:600;letter-spacing:-.01em;margin:0}
.phone{color:var(--soft);margin:4px 0 0}
.intro{margin:20px 0 28px;color:var(--soft);max-width:60ch}
.picks{display:flex;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:36px}
.picks span{color:var(--soft);font-size:15px}
.seg{display:inline-flex;border:1px solid var(--rule);border-radius:8px;padding:3px}
.seg button{font:inherit;font-size:14.5px;border:0;background:none;color:var(--soft);padding:6px 14px;border-radius:6px;cursor:pointer}
.seg button[aria-pressed="true"]{background:var(--ink);color:#fff}
section{margin:0 0 40px}
.sh{display:flex;align-items:baseline;justify-content:space-between;gap:12px;border-bottom:1px solid var(--ink);padding-bottom:8px}
h2{font-size:18px;font-weight:600;margin:0}
.tally{font-size:14px;color:var(--soft);white-space:nowrap}
.note{color:var(--soft);font-size:15px;margin:10px 0 4px}
.policy{font-size:15px;color:var(--soft);margin:10px 0 4px}
.policy select{font:inherit;font-size:15px;color:var(--ink);border:1px solid var(--rule);border-radius:6px;padding:3px 6px;background:#fff}
h3{font-size:14.5px;font-weight:500;color:var(--soft);margin:22px 0 2px}
.row{border-bottom:1px solid var(--rule)}
.line{display:grid;grid-template-columns:24px 1fr auto;gap:12px;align-items:start;padding:12px 4px}
.line:hover{background:var(--hover)}
.line input{width:18px;height:18px;margin:3px 0 0;accent-color:var(--pick);cursor:pointer}
.name{font-weight:500;cursor:pointer}
.sum{color:var(--soft);font-size:15px;margin-top:1px}
.meta{font-size:13.5px;color:var(--faint);margin-top:2px}
.risk{font-size:13.5px;font-weight:500;white-space:nowrap;padding-top:2px}
.risk.safe{color:var(--safe)}.risk.use{color:var(--use)}.risk.data{color:var(--data)}
.more{font:inherit;font-size:13.5px;color:var(--pick);background:none;border:0;padding:0;margin-top:4px;cursor:pointer}
.det{display:none;margin:4px 0 2px;font-size:14.5px}
.row.open .det{display:block}
.det dl{display:grid;grid-template-columns:7.5em 1fr;gap:4px 12px;margin:6px 0 0}
.det dt{color:var(--faint)}.det dd{margin:0}
.det code{font-size:13px;color:var(--soft);word-break:break-all}
.done{margin-top:10px;font-size:14.5px;color:var(--faint)}
.done summary{cursor:pointer;color:var(--soft)}
.done ul{margin:6px 0 0;padding-left:18px}
.todo .line{grid-template-columns:24px 1fr}
.todo .path{color:var(--soft);font-size:15px}
footer{position:fixed;left:0;right:0;bottom:0;background:#fff;border-top:1px solid var(--rule)}
footer .in{max-width:680px;margin:0 auto;padding:14px 16px;display:flex;align-items:center;gap:16px}
.count{flex:1;min-width:0}
.count b{font-weight:600}
.count small{display:block;color:var(--faint);font-size:13px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.copy{font:inherit;font-weight:500;background:var(--ink);color:#fff;border:0;border-radius:8px;padding:11px 18px;cursor:pointer;white-space:nowrap}
.copy:disabled{background:var(--rule);color:var(--faint);cursor:default}
button:focus-visible,input:focus-visible,select:focus-visible,summary:focus-visible{outline:2px solid var(--pick);outline-offset:2px}
@media (max-width:520px){main{padding-top:28px}.line{grid-template-columns:24px 1fr}.risk{grid-column:2;padding-top:0}.det dl{grid-template-columns:1fr}.det dt{margin-top:4px}}
@media (prefers-reduced-motion:reduce){*{transition:none!important}}
</style>
</head>
<body>
<main>
  <h1 id="title"></h1>
  <p class="phone" id="phone"></p>
  <p class="intro">Tick what you want changed. When you're done, press Copy choices and paste the result into the chat. Nothing on your phone changes until then.</p>
  <div class="picks"><span>Start from</span>
    <div class="seg" role="group" aria-label="Suggested picks">
      <button type="button" data-p="1" aria-pressed="false">Minimal</button>
      <button type="button" data-p="2" aria-pressed="false">Recommended</button>
      <button type="button" data-p="0" aria-pressed="false">None</button>
    </div>
  </div>
  <div id="sections"></div>
</main>
<footer><div class="in">
  <div class="count"><b id="n">0 changes</b><small id="code">Nothing selected</small></div>
  <button class="copy" id="copy" type="button">Copy choices</button>
</div></footer>

<script>
/* ===== DATA START: the AI replaces only this object ===== */
const DATA = {
  title: "Clean up your phone",
  phone: "vivo V2356, Android 16",
  sections: [
    { key:"remove", code:"Remove", title:"Remove apps", refusePolicy:true, groups:[
      { title:"Ads and lock-screen stories", items:[
        { id:"A1", name:"Lockscreen Poster", summary:"Stories and ads on your lock screen.", does:"Shows a news and ad feed on the lock screen.", lose:"Nothing. You keep a normal wallpaper.", affects:"Nothing else.", pkg:"com.vivo.magazine", risk:"safe", refuse:true, level:1 },
        { id:"A2", name:"V-Appstore", summary:"vivo's app store. Sends app suggestions.", does:"A second app store next to Google Play.", lose:"Nothing. Google Play stays.", affects:"If the phone won't remove it, its notifications and background running are turned off instead.", pkg:"com.vivo.appstore", risk:"safe", refuse:true, level:1 }
      ]},
      { title:"Test tools and leftovers", items:[
        { id:"R1", name:"Touchscreen test tool", summary:"Factory test app. Never used day to day.", does:"Used by the factory to test the touchscreen.", lose:"Nothing.", affects:"Touch keeps working.", pkg:"com.nttouchscreen.getdata", risk:"safe", level:2 },
        { id:"R2", name:"Built-in browser", summary:"A second browser. Chrome is your default.", does:"An extra web browser.", lose:"This browser.", affects:"Links keep opening in Chrome.", pkg:"com.android.browser", risk:"safe", level:2 }
      ]},
      { title:"Apps you may use", items:[
        { id:"R3", name:"Jovi Steps", summary:"Feeds the steps widget on your home screen.", does:"Counts your walking steps for vivo's step widget.", lose:"Your step history.", affects:"The steps widget stops counting.", pkg:"com.vivo.assistantfunction", risk:"use", refuse:true, level:0 },
        { id:"R4", name:"Amazon Shopping", summary:"Preinstalled shopping app.", does:"Shopping app the phone maker installed.", lose:"The app. You sign in again if you reinstall it.", affects:"Orders and account stay online.", pkg:"in.amazon.mShop.android.shopping", risk:"data", level:0 }
      ]}
    ], done:[ "Facebook App Manager, already removed", "Google Duo, already removed" ] },
    { key:"restrict", code:"Restrict", title:"Stop running in the background", note:"These stay installed and open normally. They just can't run when you're not using them.", groups:[
      { title:"Rarely used apps", items:[
        { id:"B1", name:"Myntra", summary:"Shopping. Runs in the background for offers.", does:"Checks for offers and sends promotions.", lose:"Instant offer notifications.", affects:"Order updates may arrive late.", pkg:"com.myntra.android", risk:"safe", level:2 }
      ]}
    ]},
    { key:"settings", code:"Settings", title:"Settings", groups:[
      { title:"", items:[
        { id:"S1", name:"Background Wi-Fi scanning off", summary:"Stops scanning for Wi-Fi while Wi-Fi is off.", does:"Apps and Google can't scan for Wi-Fi networks while your Wi-Fi is switched off.", lose:"Slightly less accurate location with Wi-Fi off.", affects:"Normal Wi-Fi works the same.", pkg:"wifi_scan_always_enabled: 1 to 0", risk:"safe", level:2 }
      ]}
    ]},
    { key:"optimize", code:"Optimize", title:"Speed up apps", groups:[
      { title:"", items:[
        { id:"O1", name:"Optimize apps now", summary:"Apps open faster. Takes 5 to 20 minutes.", does:"Prepares apps in advance instead of waiting for the overnight run. App launch times are measured before and after.", lose:"Nothing. The phone is warm and slow while it runs.", affects:"190 apps are not fully optimized yet.", pkg:"cmd package bg-dexopt-job", risk:"safe", level:2 }
      ]}
    ]},
    { key:"manual", title:"Do these yourself", note:"These can't be changed from the computer. Tick them off as you go.", groups:[
      { title:"", items:[
        { id:"M1", name:"Remove animations", path:"Settings, Accessibility, Remove animations" },
        { id:"M2", name:"Turn off Autostart for apps you rarely use", path:"Settings, search for Autostart" }
      ]}
    ]}
  ]
};
/* ===== DATA END ===== */

const $ = s => document.querySelector(s);
const esc = s => String(s ?? "").replace(/[&<>"]/g, c => ({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;"}[c]));
const RISK = { safe:"Safe to change", use:"You may use this", data:"Deletes its data" };
const store = (() => { try { return window.localStorage; } catch(e){ return null; } })();
const boxes = [];
$("#title").textContent = DATA.title; $("#phone").textContent = DATA.phone; document.title = DATA.title;

for (const s of DATA.sections) {
  const sec = document.createElement("section");
  sec.dataset.k = s.key;
  if (s.key === "manual") sec.className = "todo";
  sec.innerHTML = `<div class="sh"><h2>${esc(s.title)}</h2><span class="tally"></span></div>`
    + (s.note ? `<p class="note">${esc(s.note)}</p>` : "")
    + (s.refusePolicy ? `<p class="policy">If the phone won't remove an app: <select id="policy"><option value="restrict">stop it running in the background</option><option value="ask">ask me</option><option value="leave">leave it</option></select></p>` : "");
  for (const g of s.groups) {
    if (g.title) { const h = document.createElement("h3"); h.textContent = g.title; sec.appendChild(h); }
    for (const it of g.items) {
      const row = document.createElement("div"); row.className = "row";
      if (s.key === "manual") {
        row.innerHTML = `<label class="line"><input type="checkbox"><span><span class="name">${esc(it.name)}</span><span class="path" style="display:block">${esc(it.path)}</span></span></label>`;
        const cb = row.querySelector("input");
        try { cb.checked = store?.getItem("todo_" + it.id) === "1"; } catch(e){}
        cb.onchange = () => { try { store?.setItem("todo_" + it.id, cb.checked ? "1" : "0"); } catch(e){} };
      } else {
        const meta = [it.refuse ? "Phone may refuse" : "", it.action || ""].filter(Boolean).join(". ");
        row.innerHTML = `<div class="line"><input type="checkbox" id="c_${esc(it.id)}">
          <div><label class="name" for="c_${esc(it.id)}">${esc(it.name)}</label><div class="sum">${esc(it.summary)}</div>${meta ? `<div class="meta">${esc(meta)}</div>` : ""}
          <button class="more" type="button" aria-expanded="false">Details</button>
          <div class="det"><dl><dt>What it does</dt><dd>${esc(it.does)}</dd><dt>You lose</dt><dd>${esc(it.lose)}</dd><dt>Also affects</dt><dd>${esc(it.affects)}</dd><dt>Technical</dt><dd><code>${esc(it.pkg)}</code></dd></dl></div></div>
          <span class="risk ${esc(it.risk)}">${esc(RISK[it.risk] || "")}</span></div>`;
        const cb = row.querySelector("input");
        cb.dataset.id = it.id; cb.dataset.s = s.key; cb.dataset.level = it.level || 0;
        cb.onchange = update; boxes.push(cb);
        const more = row.querySelector(".more");
        more.onclick = () => { const o = row.classList.toggle("open"); more.textContent = o ? "Hide details" : "Details"; more.setAttribute("aria-expanded", o); };
      }
      sec.appendChild(row);
    }
  }
  if (s.done && s.done.length) {
    const d = document.createElement("details"); d.className = "done";
    d.innerHTML = `<summary>${s.done.length} already done</summary><ul>${s.done.map(x => `<li>${esc(x)}</li>`).join("")}</ul>`;
    sec.appendChild(d);
  }
  $("#sections").appendChild(sec);
}

function ranges(ids) {
  const by = {}, other = [];
  ids.forEach(id => { const m = id.match(/^([A-Za-z]+)(\d+)$/); m ? (by[m[1]] = by[m[1]] || []).push(+m[2]) : other.push(id); });
  const out = Object.entries(by).map(([k, ns]) => { ns.sort((a,b)=>a-b); const r = []; let a = ns[0], p = ns[0];
    for (let i = 1; i <= ns.length; i++) { if (ns[i] === p + 1) { p = ns[i]; continue; } r.push(a === p ? k + a : `${k}${a}-${k}${p}`); a = p = ns[i]; } return r.join(", "); });
  return out.concat(other).join(", ");
}
function update() {
  const sel = boxes.filter(b => b.checked), parts = [];
  for (const s of DATA.sections) {
    if (!s.code) continue;
    const ids = sel.filter(b => b.dataset.s === s.key).map(b => b.dataset.id);
    const total = boxes.filter(b => b.dataset.s === s.key).length;
    const t = document.querySelector(`section[data-k="${s.key}"] .tally`); if (t) t.textContent = `${ids.length} of ${total} selected`;
    if (ids.length) parts.push(`${s.code}: ${ranges(ids)}`);
  }
  const pol = $("#policy");
  if (pol && sel.some(b => b.dataset.s === "remove")) parts.push(`If refused: ${pol.value}`);
  $("#n").textContent = sel.length === 1 ? "1 change" : `${sel.length} changes`;
  $("#code").textContent = parts.length ? parts.join(" | ") : "Nothing selected";
  $("#copy").disabled = !parts.length;
}
function pick(level) {
  boxes.forEach(b => b.checked = level > 0 && +b.dataset.level > 0 && +b.dataset.level <= level);
  document.querySelectorAll(".seg button").forEach(b => b.setAttribute("aria-pressed", String(+b.dataset.p === level)));
  update();
}
document.querySelectorAll(".seg button").forEach(b => b.onclick = () => pick(+b.dataset.p));
boxes.forEach(b => b.addEventListener("change", () => document.querySelectorAll(".seg button").forEach(x => x.setAttribute("aria-pressed", "false"))));
const pol = $("#policy"); if (pol) pol.onchange = update;
pick(1);

$("#copy").onclick = async () => {
  const text = $("#code").textContent, btn = $("#copy");
  try { await navigator.clipboard.writeText(text); }
  catch (e) { const t = document.createElement("textarea"); t.value = text; document.body.appendChild(t); t.select(); try { document.execCommand("copy"); } catch(_){} t.remove(); }
  btn.textContent = "Copied"; setTimeout(() => btn.textContent = "Copy choices", 1500);
};
</script>
</body>
</html>
```
