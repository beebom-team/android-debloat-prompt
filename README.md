# Android Debloat Prompt

A copy-paste prompt that turns any AI assistant (Claude, ChatGPT, Gemini, and others) into a careful guide for removing preinstalled bloatware from an Android phone. No root needed.

The AI reads your phone over a USB cable and checks every app against the community-maintained [Universal Android Debloater Next Generation](https://github.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation) list. It shows you what is safe to remove and **waits for your approval** before changing anything. It writes an undo file before each removal, so everything can be put back.

**➡️ The prompt is in [PROMPT.md](PROMPT.md).**

---

## What it does by default

The first three are pre-selected on every phone, and you confirm them once with a single reply ("go"). Bloatware removal is shown as a list for you to pick from:

| | What | How |
|---|---|---|
| 🔒 | **Removes Glance and lock-screen "stories" feeds** on every brand: Glance on Samsung, Xiaomi, Motorola, Nothing and others; Lock Screen Magazine on OPPO, realme and OnePlus; vivo Magazine; Wallpaper Carousel; Huawei/Honor Magazine; Tecno/Infinix feeds | Uninstalls them. If the feed is built into the lock screen itself (some OPPO/realme/OnePlus phones), it blocks the feed instead and shows you the exact switch to turn it off in Settings |
| 🚫 | **Stops ad pop-ups** | Removes ad services (for example Xiaomi MSA, App Vault, AppCloud, Digital Turbine), and blocks "display over other apps" for apps that shouldn't have it |
| 🔕 | **Silences preinstalled app stores** that aren't Google Play (GetApps, Galaxy Store, App Market, vivo App Store and others) | Removes the store where it's safe to. Where the phone needs it (for example Samsung Pay needs Galaxy Store, or a phone without Google Play), it turns off the store's notifications and background activity instead |
| 🧹 | **Removes bloatware**: preinstalled apps you don't need, such as duplicate OEM apps (browser, music, video, themes), Facebook and other partner apps, carrier apps, games and trackers | Lists every one with a plain-English description and a risk level (Recommended, Advanced, Expert), checked against the [UAD-ng](https://github.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation) community list. You pick what goes, for example `go + all Recommended`. Expert apps are only disabled, never uninstalled |

At the end, the AI gives you a **before/after report** with measured numbers:

| Measure | Before | After | Change |
|---|---|---|---|
| Apps installed | e.g. 214 | e.g. 171 | −43 |
| Storage used on /data | … | … | … |
| RAM used by the removed apps | … | 0 | … |

(Example numbers only. Yours come from your phone.)

---

## Is this safe?

It is designed to be as safe as possible, but no tool can promise it works on every phone. Read this before you start.

- **Nothing is deleted from the phone's system.** Apps are removed only for your user profile (`pm uninstall --user 0`). For apps that are part of the system, the installation file stays on the phone, so they can be restored with one command or by a factory reset. The removed apps' own data and cache are deleted; that's what frees storage.
- **Some preinstalled apps are really "downloaded" apps** (for example partner games, Mi Store or WPS Office on Xiaomi). These are deleted completely, and the undo file can't bring them back; you reinstall them from the Play Store. The AI marks them with † in the list so you know before you approve.
- **Apps you probably use are never in a "remove all" choice.** UAD-ng rates apps like WhatsApp, Instagram, Google Pay and Gmail as removable. The AI puts them in a separate "Probably yours" list and removes one only if you name it.
- **You approve every removal.** The AI stops and shows you a table first.
- **An undo file is written first.** Before each removal, the AI adds the restore command to `rollback.bat` (Windows) and `rollback.sh` (Mac/Linux).
- **Important apps are protected.** The AI is told never to touch your dialer, SMS app, keyboard, home screen, Play Store, Google Play services, camera, emergency alerts, security, or system update components.
- **Numbers are measured, not guessed.** The AI must take every number in the report from a command it ran, or say "not measured".
- **Only one restart.** The AI applies everything you approved, then restarts the phone once at the end and asks you to test calls, SMS, camera and other basics. If something broke, it undoes the newest group first.

**Do not use this on a work or school phone** that your organisation manages. The prompt detects this and stops.

---

## What you need

| | |
|---|---|
| A computer | Windows, macOS or Linux |
| A USB cable | One that can transfer data, not just charge |
| An AI assistant | Any. One that can run terminal commands (see Step 4) is the easiest |
| About 30–60 minutes | Most of it is reviewing the list |

---

## Step-by-step

### Step 1: Back up your phone
Back up your photos and important data, for example with Google Photos or by copying them to your computer. You probably won't need it, but a factory reset is the last-resort fix if something goes badly wrong.

Charge the phone above 50%.

### Step 2: Install ADB on your computer
ADB (Android Debug Bridge) is Google's official tool for talking to a phone over USB.

**Windows**, in PowerShell:
```powershell
winget install Google.PlatformTools
```
Then close and reopen PowerShell.

**macOS**, with [Homebrew](https://brew.sh):
```bash
brew install --cask android-platform-tools
```

**Linux** (Debian/Ubuntu):
```bash
sudo apt install adb
```

**Or download it directly** from Google: [SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools). Unzip it and run commands from inside that folder.

Check that it works:
```bash
adb version
```

### Step 3: Turn on USB debugging on the phone
1. Open **Settings → About phone**.
2. Tap **Build number** 7 times until it says "You are now a developer". On some phones it's under **Software information** (Samsung) or **MIUI/HyperOS version** (Xiaomi).
3. Go back to **Settings → System → Developer options** (on some phones, **Settings → Additional settings → Developer options**).
4. Turn on **USB debugging**.
   - **Xiaomi/Redmi/POCO:** also turn on **USB debugging (Security settings)**. This needs a SIM card and a Mi account.
5. Connect the phone to your computer with the USB cable.
6. If the phone asks "Allow USB debugging?", tick **Always allow from this computer** and tap **Allow**.
7. If the phone asks what the USB connection is for, choose **File transfer**.

Check the connection:
```bash
adb devices
```
You should see one line ending in `device`. If it says `unauthorized`, unlock the phone and tap **Allow**.

### Step 4: Give the prompt to your AI

**Option A: an AI that can run commands (easiest).** Examples are [Claude Code](https://claude.com/claude-code) and the Claude desktop app's Code tab, OpenAI Codex CLI, and Gemini CLI.
1. Make a new empty folder, for example `phone-debloat`, and open the AI there. The AI will save its lists and undo files in this folder.
2. Open [PROMPT.md](PROMPT.md), copy everything inside the grey box, and paste it into the AI.
3. The AI runs the commands itself and shows you what it found.

**Option B: a chat AI** (ChatGPT, Claude.ai, Gemini on the web).
1. Paste the prompt into the chat.
2. The AI gives you one command at a time. Run each one in a terminal (PowerShell on Windows, Terminal on Mac) **inside the same folder every time**, and paste the output back to the AI.
3. When the AI shows you the contents of the undo files, save them in that folder with the exact names it gives (`rollback.bat`, `rollback.sh`).

A chat AI usually can't download the UAD-ng list. It will tell you so, and mark its ratings as "AI knowledge, unverified". Be more cautious with those, or use Option A.

### Step 5: Review the list carefully
The AI measures the phone first (the "before" numbers), without restarting it. Then it shows you:

| Group | What it means | Selected? |
|---|---|---|
| **Group A** | Glance, lock-screen feeds, ad services, pop-ups, app-store nags | ✅ Pre-selected |
| **Recommended** | Other bloatware that's safe for almost everyone | You choose |
| **Advanced** | Features some people use | You choose; read each one |
| **Expert** | Can break things; only disabled, never uninstalled | Skip unless you know you don't need it |

Reply with exactly what to apply, for example:
- `go` applies Group A only
- `go, except A3 and A7` applies Group A without those two items
- `go + all Recommended`
- `go + Recommended except 4 and 9`

The AI will not change anything you didn't approve.

### Step 6: Test after the restart
When every approved change is done, the AI restarts the phone once. Unlock it, then check:

- [ ] Unlock works (PIN, fingerprint, face)
- [ ] The lock screen shows a normal wallpaper, with no stories or ads
- [ ] Make and receive a call
- [ ] Send and receive an SMS
- [ ] Mobile data and Wi-Fi
- [ ] Bluetooth
- [ ] Camera (photo and video)
- [ ] Notifications from a messaging app
- [ ] Play Store opens and updates an app
- [ ] Alarms
- [ ] Banking or payment apps open
- [ ] Settings → System update opens

Tell the AI the result. If something broke, it undoes the newest group first (one extra restart each time) and narrows down which app caused it.

### Step 7: Read your report
When everything works, the AI measures the phone again (no extra restart) and gives you `report.md` with:
- **Apps:** how many were installed before and after (preinstalled and downloaded counted separately)
- **Storage freed:** measured on the phone, plus an estimate from the removed apps' sizes
- **RAM freed:** the RAM the removed apps were using, plus the whole-phone before/after change (with a note that part of that change comes from the restart)
- **What happened to Glance and each app store,** and how to turn off anything that could only be blocked

Storage freed is often smaller than people expect. Preinstalled apps live on a read-only part of the phone, so only their updates, data and cache can be freed. The report explains this.

Finally, the AI asks **"Do you want to remove any more apps?"** and shows what's still possible (the tiers you skipped, and the "Probably yours" list). Name what you want removed, and it does another careful round with its own undo lines, one restart and the same checks. Or say no, and you're done.

---

## How to undo

**Undo everything:** connect the phone with USB debugging on, open a terminal in the folder with the undo files, and run:

- **Windows:** `rollback.bat`
- **macOS / Linux:** `sh rollback.sh`

**Undo one app:**
```bash
adb shell cmd package install-existing --user 0 com.example.package
```
On Android 7 or older:
```bash
adb shell pm install-existing --user 0 com.example.package
```
If an app was disabled instead of uninstalled:
```bash
adb shell pm enable --user 0 com.example.package
```
If an app store was muted, or an app's pop-ups were blocked, the exact restore lines (with the original settings) are in the undo files.

**Last resort:** a factory reset brings back every removed app, but it erases your data.

---

## Files the AI creates

| File | What it is |
|---|---|
| `report.md` | The final before/after report |
| `packages_before*.txt` / `packages_after*.txt` | App lists before and after (all, preinstalled, downloaded, disabled) |
| `rollback.bat` / `rollback.sh` | Undo commands for every change |
| `removal_log.csv` | What was changed, when, and whether it worked |
| `meminfo_before.txt` / `meminfo_after.txt` | RAM readings, including how much each app was using |
| `uptime_before.txt` / `uptime_after.txt` | How long the phone had been on at each reading |
| `df_before.txt` / `df_after.txt` | Storage readings |
| `diskstats_before.txt` | Size of each app before removal (some phones, such as HyperOS 3, only record totals here) |
| `battery_before.txt` | Battery level and temperature |

Keep the folder. Your undo files are in it.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `adb` is not recognized | Reopen the terminal after installing, or run commands from inside the platform-tools folder. If you unzipped it to `C:\platform-tools`, an AI that runs commands finds it there and uses the full path |
| `adb devices` shows nothing | Use a different cable (many cables are charge-only), choose **File transfer** mode, toggle USB debugging off and on, then run `adb kill-server` |
| `unauthorized` | Unlock the phone and tap **Allow** on the prompt. No prompt? Turn on **Developer options → Revoke USB debugging authorizations**, then reconnect |
| `Failure [DELETE_FAILED_USER_RESTRICTED]` on Xiaomi | Turn on **USB debugging (Security settings)** in Developer options |
| `Failure [not installed for 0]` | The app is already removed. Nothing to do |
| `Failure [-1000]` on Xiaomi HyperOS | HyperOS protects some of Xiaomi's own apps (for example ShareMe, Scanner, Compass, Weather, Notes, Calculator, Screen Recorder). The app stays as it was. Nothing broke |
| An app store still sends notifications after it was muted | On Android 13 and newer, the phone can undo the notification block right away. Turn it off by hand: **Settings → Apps → (the store) → Notifications** |
| An app came back after a system update | Normal. Run the process again after big updates |
| The phone is stuck on the logo after a restart | If `adb devices` still shows it, run the undo file. Otherwise, factory reset from recovery mode |

---

## FAQ

**Will this make my phone faster?**
It usually helps privacy, storage and battery more than speed, and it removes lock-screen ads and nagging notifications. On a newer phone the difference in speed is often small. The report shows measured before/after numbers so you can see for yourself.

**Glance came back, or my lock screen still shows stories. Why?**
On some phones the feed is built into the lock screen and can't be removed safely, so the AI blocks it and gives you the Settings switch to turn it off. A big system update can also reinstall Glance. Run the prompt again after updates.

**Why wasn't Galaxy Store (or my phone's app store) removed?**
Some stores are needed by the phone. Samsung Pay needs Galaxy Store, and on phones without Google Play the brand's store is the only way to install apps. In those cases the AI turns off the store's notifications and background activity instead, which stops the nagging.

**Does it need root, or does it void my warranty?**
No root is needed, and it doesn't unlock the bootloader. It uses only standard ADB commands.

**Is it reversible?**
Mostly. System apps are removed for your user profile only, and the undo files restore them. Apps marked † in the list (preinstalled "downloaded" apps such as partner games) are deleted completely; you reinstall those from the Play Store if you want them back.

**Which phones does it work on?**
Android 5 and newer from any brand. The prompt detects Samsung One UI, Xiaomi MIUI/HyperOS, OPPO ColorOS, realme UI, OnePlus OxygenOS, vivo Funtouch/OriginOS, Huawei EMUI and Honor MagicOS, and uses brand-specific knowledge where the UAD-ng list has gaps.

---

## Credits

App ratings come from the [Universal Android Debloater Next Generation](https://github.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation) community list. If you prefer a desktop app with buttons instead of an AI, use UAD-ng directly.

Use at your own risk. Removing system apps can have effects nobody predicted, so review every list before you approve it.
