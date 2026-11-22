# Android Debloat Prompt

A copy-paste prompt that turns any AI assistant (Claude, ChatGPT, Gemini, and others) into a careful guide for removing preinstalled bloatware from an Android phone. No root needed.

The AI reads your phone over a USB cable and checks every app against the community-maintained [Universal Android Debloater Next Generation](https://github.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation) list. It shows you what is safe to remove and **waits for your approval** before changing anything. It writes an undo file before each removal, so everything can be put back.

**➡️ The prompt is in [PROMPT.md](PROMPT.md).**

---

## Is this safe?

It is designed to be as safe as possible, but no tool can promise it works on every phone. Read this before you start.

- **Nothing is deleted from the phone.** Apps are removed only for your user profile (`pm uninstall --user 0`). The app file stays on the phone, so it can be restored with one command or by a factory reset.
- **You approve every removal.** The AI stops and shows you a table first.
- **An undo file is written first.** Before each removal, the AI adds the restore command to `rollback.bat` (Windows) and `rollback.sh` (Mac/Linux).
- **Important apps are protected.** The AI is told never to touch your dialer, SMS app, keyboard, home screen, Play Store, Google Play services, camera, emergency alerts, security, or system update components.
- **It goes in small steps.** The AI removes one group at a time, then restarts the phone and asks you to test calls, SMS, camera and other basics.

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
The AI shows you a table for each risk level:

| Tier | What it means | Suggestion |
|---|---|---|
| **Recommended** | Safe for almost everyone | Approve these first |
| **Advanced** | Features some people use | Read each one |
| **Expert** | Can break things; only disabled, never uninstalled | Skip unless you know you don't need it |

Reply with exactly what to remove, for example:
- `approve all Recommended`
- `approve Recommended except 4 and 9`
- `approve Advanced 2, 5`

The AI will not remove anything you didn't name.

### Step 6: Test after each group
After each group, the AI restarts the phone. Then check:

- [ ] Unlock works (PIN, fingerprint, face)
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

Tell the AI the result. If something broke, it restores that group and narrows down which app caused it.

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

**Last resort:** a factory reset brings back every removed app, but it erases your data.

---

## Files the AI creates

| File | What it is |
|---|---|
| `packages_before.txt` | Every app on the phone before any change |
| `packages_after.txt` | Every app after debloating |
| `rollback.bat` / `rollback.sh` | Undo commands for everything that was removed |
| `removal_log.csv` | What was removed, when, and whether it worked |
| `baseline_before.txt` / `baseline_after.txt` | RAM, storage and battery readings for comparison |

Keep the folder. Your undo files are in it.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `adb` is not recognized | Reopen the terminal after installing, or run commands from inside the platform-tools folder |
| `adb devices` shows nothing | Use a different cable (many cables are charge-only), choose **File transfer** mode, toggle USB debugging off and on, then run `adb kill-server` |
| `unauthorized` | Unlock the phone and tap **Allow** on the prompt. No prompt? Turn on **Developer options → Revoke USB debugging authorizations**, then reconnect |
| `Failure [DELETE_FAILED_USER_RESTRICTED]` on Xiaomi | Turn on **USB debugging (Security settings)** in Developer options |
| `Failure [not installed for 0]` | The app is already removed. Nothing to do |
| An app came back after a system update | Normal. Run the process again after big updates |
| The phone is stuck on the logo after a restart | If `adb devices` still shows it, run the undo file. Otherwise, factory reset from recovery mode |

---

## FAQ

**Will this make my phone faster?**
It usually helps privacy, storage and battery more than speed. On a newer phone the difference in speed is often small. The prompt saves before/after RAM, storage and battery readings so you can see for yourself.

**Does it need root, or does it void my warranty?**
No root is needed, and it doesn't unlock the bootloader. It uses only standard ADB commands.

**Is it reversible?**
Yes. Apps are removed for your user profile only, and the undo files restore them.

**Which phones does it work on?**
Android 5 and newer from any brand. The prompt detects Samsung One UI, Xiaomi MIUI/HyperOS, OPPO ColorOS, realme UI, OnePlus OxygenOS, vivo Funtouch/OriginOS, Huawei EMUI and Honor MagicOS, and uses brand-specific knowledge where the UAD-ng list has gaps.

---

## Credits

App ratings come from the [Universal Android Debloater Next Generation](https://github.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation) community list. If you prefer a desktop app with buttons instead of an AI, use UAD-ng directly.

Use at your own risk. Removing system apps can have effects nobody predicted, so review every list before you approve it.
