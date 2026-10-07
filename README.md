# USBShield 🛡️

**Stop trusting USB ports blindly.**

USBShield is a Windows security tool that watches every USB port on your machine and reacts the moment something malicious gets plugged in. It was built because USB attacks are real, embarrassingly easy to execute, and most antivirus software doesn't catch them until it's already too late.

Plug in a Rubber Ducky and USBShield catches it in under a second. Copy 100 MB of files to a flash drive suspiciously fast and it fires a critical alert. Drop a packed executable on a USB and it flags the entropy before you can even open File Explorer.

---

## What it protects against

USBShield covers 15 different USB attack vectors — here's what that actually means in practice:

| Attack | What happens | How USBShield catches it |
|--------|-------------|--------------------------|
| Rubber Ducky / BadUSB | Device pretends to be a keyboard and types commands at 200+ keys/sec | Burst detection: 15 typing keys in 100ms → instant CRITICAL alert |
| HID injection | Automated keystroke injection via any USB HID device | Low-level keyboard hook + timing analysis (variance < 8ms = machine) |
| Known attack devices | Hak5 Rubber Ducky, Bash Bunny, Digispark, LAN Turtle, etc. | VID:PID database lookup on every device connection |
| Malicious scripts | PowerShell with IEX, Mimikatz, vssadmin, encoded payloads | 19 regex heuristics on every .ps1 and .bat file |
| Ransomware on USB | Packed/encrypted binaries with UPX or custom packers | Shannon entropy analysis — anything above 7.2 bits/byte triggers an alert |
| AutoRun attacks | autorun.inf that executes malware on drive insertion | Deleted instantly on drive mount, registry hardened too |
| Trojan files | .exe disguised as .pdf, .doc, or .jpg | PE header detection regardless of file extension |
| Credential theft | .kdbx, .pem, .sql, .ppk, ntds.dit being copied to USB | File extension monitoring on every write to removable drives |
| Data exfiltration | Bulk copying sensitive files to a flash drive | Fires if > 100 MB or > 50 files copied in 60 seconds |
| Rogue network adapters | USB RNDIS/CDC device sniffing your traffic | Adapter name and MAC pattern matching |
| Unsigned drivers | USB device that installs a kernel-level rootkit | WMI driver signature polling |

---

## What it looks like

When a threat is detected, USBShield covers your entire screen with a red alert — hard to miss, impossible to accidentally dismiss. The Dashboard shows a live count of devices seen, threats detected, and devices blocked. Every event is logged with full details.

The Settings panel lets you toggle each protection module individually, and turning any of them off immediately **restores the original system state** — the registry goes back to normal, USB storage re-enables, all of it.

---

## Getting started

**Requirements:** Windows 10/11, Python 3.8+, Administrator privileges

```bash
git clone https://github.com/Vansh-1006/usbshield.git
cd usbshield
pip install -r requirements.txt
```

Then either double-click `run_as_admin.bat` or:

```bash
python main.py
```

USBShield will ask for elevation if it needs it.

**First time setup:** Go to Settings and click "Apply all registry hardening" — this disables AutoRun system-wide and sets a few other sensible defaults. You can revert all of it from the same panel.

---

## Testing it without a USB drive

```bash
python test_all_attacks.py
```

This starts the full GUI and fires all 15 attack types as real events, one every 9 seconds. It also creates actual malicious-signature test files (high-entropy binary, masked PE, real PowerShell patterns) and runs the scanner on them so you can watch detections appear in the Dashboard live.

---

## Testing with a USB drive

1. Generate the binary payloads first (run once on Windows):
   ```bash
   python usb_payloads/generate_binary_payloads.py
   ```

2. Copy the entire `usb_payloads/` folder to your USB drive

3. Start USBShield: `python main.py`

4. Insert the USB — StorageGuard scans automatically and alerts trigger for:
   - `autorun.inf` (deleted immediately)
   - `payload.ps1` (12 detected PowerShell patterns)
   - `ransomware_sim.exe` (entropy ~7.99 bits)
   - `invoice.pdf` (PE header inside a .pdf)
   - Every file in `credentials/` (.kdbx, .pem, .sql, .ppk, .wallet)

---

## Architecture

```
usb_shield/
├── main.py                        Entry point + app orchestration
├── test_all_attacks.py            Interactive test runner (all 15 attacks)
├── usb_payloads/                  Real test files for USB testing
├── config/
│   ├── settings.json              All settings — persisted immediately on change
│   └── whitelist.json             Trusted devices
├── core/
│   ├── settings_manager.py        Persistent settings with rollback support
│   ├── threat_logger.py           Central event log with severity levels
│   ├── whitelist_manager.py       VID:PID device whitelist
│   ├── device_monitor.py          WMI USB device detection
│   ├── hid_detector.py            Keyboard hook + injection analysis
│   ├── storage_guard.py           Drive monitoring, file scanning, exfil detection
│   ├── file_scanner.py            Entropy, PE headers, script heuristics
│   ├── system_hardener.py         Registry hardening + rollback
│   └── driver_network_monitor.py  Driver signatures + rogue adapters
└── gui/
    ├── main_window.py             Dashboard, Devices, Logs, Whitelist, Settings
    ├── alert_screen.py            Full-screen red alert with action buttons
    └── theme.py                   Dark theme + alert color palettes
```

---

## A few things worth knowing

**It needs Administrator.** The keyboard hook, device disable (PowerShell `Disable-PnpDevice`), and registry hardening all require elevation. Launch via `run_as_admin.bat` or right-click → Run as administrator.

**The HID detector watches typing keys only.** Earlier versions triggered on Delete/Backspace/Ctrl shortcuts. The current version filters to printable character VK codes only, so normal use won't set it off. The threshold is 15 typing characters in 100ms — that's 150 keys/sec, well above any human typist.

**Whitelisting a device is permanent.** When you click "Add to Whitelist" on a device alert, that VID:PID is saved to `config/whitelist.json` and never alerted again. Use it for keyboards and mice you trust. You can remove entries from the Whitelist panel at any time.

**Turning off a feature actually reverts it.** Toggle off "Apply registry hardening" in Settings and USBShield writes the original registry values back. Toggle off USB storage blocking and the USBSTOR driver re-enables. This isn't just pausing detection — it's a real rollback.

**File scan alerts don't spam you.** Each USB drive gets at most one full-screen alert per scan (a summary), then a 60-second cooldown before another can fire for the same drive. Individual file detections go to the Threat Logs panel but don't interrupt you with repeated popups.

---

## Dependencies

```
pywin32     WMI access and COM interop (Windows-specific)
wmi         USB device event monitoring
watchdog    Real-time filesystem monitoring on USB drives
psutil      Network adapter enumeration
colorama    Console output (minor)
```

Install all at once: `pip install -r requirements.txt`

---
