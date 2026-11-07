# HP LaserJet 1018 / 1020 driver for macOS on Apple silicon

A native **arm64** driver for the **HP LaserJet 1018** and **HP LaserJet 1020** USB laser printers on Apple silicon Macs (M1, M2, M3, M4 and later). It needs no Rosetta, no Ghostscript and no Homebrew, and you do not have to turn off System Integrity Protection.

> **Tested on:** MacBook Pro 14" (M2 Pro), macOS 26.5, HP LaserJet 1018 connected through a USB‑C docking station.
> The LaserJet 1020 uses the same code path and has been tested in simulation, but not on real hardware yet. Reports are welcome; see [Reporting problems](#reporting-problems).

[Български](README.bg.md)

---

## Contents

- [Why this exists](#why-this-exists)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Installation](#installation)
- [Testing](#testing)
- [Printing from apps](#printing-from-apps)
- [Driver options](#driver-options)
- [Everyday use](#everyday-use)
- [Troubleshooting](#troubleshooting)
- [Command reference](#command-reference)
- [Uninstalling](#uninstalling)
- [What gets installed](#what-gets-installed)
- [Building from source only](#building-from-source-only)
- [Reporting problems](#reporting-problems)
- [Credits and licenses](#credits-and-licenses)

---

## Why this exists

HP never released a driver for the LaserJet 1018/1020 that runs natively on Apple silicon. The old Intel‑only drivers depend on Rosetta 2, and newer macOS releases no longer install Rosetta automatically, so those drivers stop working.

These printers are also unusual in two ways:

1. **They are "host‑based".** They do not understand PostScript or PCL. They only accept a compressed bitmap format called **ZjStream** (JBIG‑compressed).
2. **They have no permanent firmware.** The firmware lives in the printer's RAM and **is lost every time the printer is switched off**. The computer has to upload it again after every power‑on. Until that happens, the printer accepts data but prints nothing.

This project handles both: it converts print jobs to ZjStream, and it uploads the firmware automatically whenever the printer is switched on.

---

## How it works

```
Printing:
  App → PDF → cgpdftoraster (built into macOS) → rastertozjs → foo2zjs → macOS USB backend → printer
                                                  (this project)  (ZjStream encoder)

Firmware:
  printer switched on → hp1020-fwd (background service) → detects model → uploads sihp1018.dl / sihp1020.dl
```

| Component | What it does |
|---|---|
| `rastertozjs` | A CUPS filter written in C, compiled for arm64. It takes the raster image macOS renders, rebuilds the full page, turns gray tones into black dots (clustered‑dot screen, error diffusion or a plain threshold) and passes the pages to `foo2zjs`. |
| `foo2zjs` | The well‑established open‑source ZjStream encoder by Rick Richardson (GPL). It is included as source and compiled on your Mac. |
| PPD file | Tells macOS which paper sizes, margins, resolutions and options the printer has. |
| `hp1020-fwd` | A small background service (LaunchDaemon). It watches the USB bus for the printer (1018 = `03f0:4117`, 1020 = `03f0:2b17`) and uploads the correct firmware after every power‑on. While the printer is off it pauses the print queue, so jobs wait instead of getting lost. If an old job is blocking the USB port, it holds the job, uploads the firmware and then lets the job print again. |
| `hp1020-tool` | A helper command for status checks, tests, a manual firmware upload and logs. |

Everything is compiled from source on your own Mac. The only thing downloaded during installation is the printer firmware, which is HP's property and not included in this repository. The installer gets it from the public foo2zjs mirror and checks its SHA‑256 checksum.

---

## Requirements

- A Mac with Apple silicon (M1 or later). Intel Macs will probably work too, but have not been tested.
- macOS 26 or later. Earlier versions may work as well.
- **Xcode Command Line Tools** (a free download from Apple; see step 1 below)
- An internet connection during installation, to download the firmware from GitHub
- An HP LaserJet **1018** or **1020** and a USB cable. Most MacBooks only have USB‑C ports, so you need either:
  - a **USB‑C to USB‑B cable** (recommended, the most reliable option), or
  - a USB‑C hub or docking station with a USB‑A port plus a normal USB‑A to USB‑B printer cable. This works, but see [Docks and hubs](#docks-and-hubs).
- An administrator account on the Mac

---

## Installation

The whole process takes about 5 minutes. Everything happens in the **Terminal** app (press ⌘ Space, type `Terminal` and press Return).

### 1. Install the Command Line Tools

Paste this into Terminal and press Return:

```sh
xcode-select --install
```

A window appears: click **Install** and wait until it finishes. If you see `command line tools are already installed`, you already have them and can go on.

### 2. Connect and switch on the printer

Connect the printer to the Mac and switch it on.

**If macOS adds the printer by itself** (you may see it appear in System Settings → Printers & Scanners as "HP LaserJet 1018" or "HP LaserJet 1020"), remove it. Those automatic entries have no working driver and they **block the USB port**, which keeps the firmware from being uploaded. You can remove them in System Settings → **Printers & Scanners** → select the printer → **–** (minus), or later from Terminal (see [Duplicate printers](#duplicate-printers-appear)).

### 3. Download this project

Pick one of these options:

- **Option A: ZIP download.** Click the green **Code** button on this page → **Download ZIP**. Safari usually unpacks it into your Downloads folder automatically. Then run:

  ```sh
  cd ~/Downloads/hp1020-macos-main
  ```

  (The folder name may differ slightly, so check your Downloads folder.)

- **Option B: git.**

  ```sh
  cd ~/Downloads
  git clone https://github.com/<your-account>/hp1020-macos.git
  cd hp1020-macos
  ```

### 4. Run the installer

```sh
sudo sh ./install.sh
```

Enter your Mac password when asked. Nothing appears on screen while you type; that is normal.

The installer:

1. compiles the driver for your Mac;
2. downloads and checks the firmware for both models;
3. installs the files (see [What gets installed](#what-gets-installed));
4. starts the firmware service;
5. creates a print queue for your printer, with A4 as the default paper size;
6. waits for the firmware to be uploaded and prints a status report.

During the firmware upload the printer **clicks once and its lights blink**. That is normal.

**Using Letter paper (US/Canada)?** Run the installer this way instead:

```sh
sudo sh ./install.sh --paper Letter
```

Other installer options:

| Option | Meaning |
|---|---|
| `--paper NAME` | Default paper size: `A4`, `Letter`, `Legal`, `A5`, `A6`, `B5`, `Executive`, `EnvDL`, … |
| `--queue NAME` | Name of the print queue (default `HP_LaserJet_1020`; for example `--queue HP1018`). Letters, digits, `_` and `-` only. |
| `--firmware FILE` | Use your own `sihp1018.dl`/`.img` or `sihp1020.dl`/`.img` instead of downloading one (for offline installs). The model is taken from the file name. |
| `--no-queue` | Install the driver but do not create a print queue. |

You can safely run the installer again at any time, for example to update the driver. Your settings are kept.

### 5. Allow the background service

macOS may show a notification saying **"Background Items Added"**. That is the firmware service, and it must be allowed to run.

If the installer reported `Bootstrap failed: 5: Input/output error`, or if `hp1020-tool status` says the daemon is not loaded:

1. Open **System Settings → General → Login Items & Extensions**.
2. Under **Allow in the Background**, switch **on** the entry for `hp1020-fwd`. It may be listed as `sh` or under an unknown developer.
3. Start the service:

   ```sh
   sudo launchctl bootstrap system /Library/LaunchDaemons/local.hp1020.fwloader.plist
   ```

> The message `lpadmin: Printer drivers are deprecated and will stop working in a future version of CUPS` is only a general notice from macOS's printing system and can be ignored.

---

## Testing

Run these three checks in order. Each one tests a larger part of the chain, so if something fails you know exactly where.

### Test 1: status

```sh
sudo hp1020-tool status
```

A healthy result looks like this:

```
Files:
  ok      /Library/Printers/HP1020/Filter/rastertozjs [arm64]
  ok      /Library/Printers/HP1020/Filter/foo2zjs [arm64]
  ok      /Library/Printers/HP1020/Firmware/sihp1018.dl
  ok      /Library/Printers/HP1020/Firmware/sihp1020.dl
  ...
Firmware loader daemon (local.hp1020.fwloader):
  state = running
USB:
  printer present on USB (session ...)
  model: HP LaserJet 1018 (firmware file sihp1018.dl)
  URI: usb://Hewlett-Packard/HP%20LaserJet%201018?serial=XXXXXXX
Print queue HP_LaserJet_1020:
  printer HP_LaserJet_1020 is idle.  enabled since ...
  no pending jobs
```

> **Note:** on macOS, `status` may report `firmware: NOT loaded` even when the firmware *is* loaded. macOS shortens the identity string the printer reports and drops the firmware version field from it. The real proof is the firmware service log:
>
> ```sh
> tail -n 15 /Library/Logs/HP1020/hp1020-fwd.log
> ```
>
> Look for `firmware loaded (printer restarted on USB)` or `firmware sent; no confirmation … Assuming it is loaded`, followed by `queue … resumed`.

### Test 2: dry run (no paper used)

```sh
hp1020-tool test-dry
```

This runs the **whole driver** on the built‑in test page exactly as a real print job would, but writes the result to a file instead of sending it to the printer. It then decodes that printer data back into images and opens the first page in Preview. What you see is exactly what the printer would receive.

Expected output:

```
OK: ~100000 bytes of ZjStream, 2 page(s), paper a4, 600 dpi
  sheet 4960x7016 px, raster at +96+96, paper A4 (code 9)
```

The files are saved in `~/hp1020-test/`. You can also test your own file and options:

```sh
hp1020-tool test-dry ~/Documents/some-file.pdf -o PageSize=A5 -o HPHalftone=ErrorDiffusion
```

### Test 3: real test page

```sh
hp1020-tool test-print
```

This prints a two‑page test sheet and follows the job until it finishes. On paper, check that:

- **the frame** around the printable area (about 4 mm from the paper edges) is visible on all four sides, together with the corner marks;
- **the gray scale 0–100 %** shows 11 distinct steps, from white to solid black;
- **the gradient** is smooth, without bands;
- **the lines**, even 0.1 pt, are visible;
- **small text** at 5 pt is readable, including Cyrillic;
- **page 2** prints too, which shows that multi‑page jobs work.

---

## Printing from apps

Open any document, choose **File → Print** (⌘P), select the HP LaserJet printer and click **Print**.

The driver's own settings are in the print dialog. Use the pop‑up menu in the middle of the dialog, or click **More Options / Printer Options**, and look for **Printer Features**, **Print Quality** and **Paper Handling**. The exact names depend on the app and the macOS version.

---

## Driver options

| Option | Values | Default |
|---|---|---|
| Media Size | A4, Letter, Legal, Executive, A5, A6, B5 (JIS), B5 (ISO), envelopes #10/DL/C5/Monarch, postcards, 16K | A4 |
| Resolution | 600 dpi; FastRes 1200 (1200×600 dpi) | 600 dpi |
| Media Source | Main Tray; Priority Slot (manual feed) | Main Tray |
| Media Type | Plain, Light, Heavy/Cardstock, Rough, Labels, Envelope, Transparency, Letterhead, Recycled | Plain |
| Print Density | 1 (lightest) … 5 (darkest) | 3 |
| EconoMode | Off / On (saves toner, lighter print) | Off |
| Halftoning | Clustered dot (best for laser) / Error diffusion (more detail in photos) / Threshold (text only, no gray) | Clustered dot |

To change the defaults permanently, use the following command (replace `HP_LaserJet_1020` with your queue name if you chose another one):

```sh
sudo lpadmin -p HP_LaserJet_1020 -o HPDensity=4 -o HPEconoMode=True -o PageSize=Letter
```

---

## Everyday use

- **Switch the printer on and just print.** The firmware is uploaded automatically within a few seconds of switching it on; you will hear one click.
- **Printing while the printer is off is fine.** The job waits in the queue and prints once the printer is on and the firmware is loaded. During that time the queue shows as *Paused*, which is intentional.
- **After restarting the Mac** nothing needs to be done; the service starts by itself.
- **If you switch the printer off in the middle of a job**, the service holds that job, reloads the firmware and prints it again from the start.

---

## Troubleshooting

### Quick checks first

```sh
sudo hp1020-tool status
tail -n 30 /Library/Logs/HP1020/hp1020-fwd.log
lpstat -v          # all print queues and the devices they use
lpstat -o          # all waiting jobs
```

### Nothing prints, the job stays at "Sending data to printer"

The printer has no firmware (or only part of it, from an interrupted upload), and the job is stuck on it. Clear everything and start fresh:

```sh
sudo cancel -a -x                 # delete all jobs
lpstat -o                         # should print nothing
```

Then **switch the printer off, wait 10 seconds and switch it back on**. Watch the log:

```sh
tail -f /Library/Logs/HP1020/hp1020-fwd.log      # press Ctrl+C to stop watching
```

After the firmware line (and the click from the printer), print again.

If `lpstat -o` still shows jobs after `cancel`, restart the printing system and check again:

```sh
sudo launchctl kickstart -k system/org.cups.cupsd
```

### Duplicate printers appear

macOS sometimes adds its own queue for the printer every time it is plugged in, for example `HP_LaserJet_1018`, `HP_LaserJet_1018_2` and so on. **These block the USB port** and should be removed. To list all queues:

```sh
lpstat -v
```

Remove every queue for the printer **except the one created by this driver** (`HP_LaserJet_1020`, or the name you passed with `--queue`):

```sh
sudo lpadmin -x HP_LaserJet_1018
sudo lpadmin -x HP_LaserJet_1018_2
```

### The log says "queue(s) printing to this printer right now … hold the USB port"

Another queue or an old job is using the printer. The service tries to hold such jobs automatically. If it cannot, remove the duplicate queues (see above) and clear the jobs (`sudo cancel -a -x`).

### The log says "USB upload did not finish in 45 s"

The USB port was busy. Usually the cause is a duplicate queue or a stuck job (see the two sections above), less often a sleeping hub or dock. Fix the cause, then switch the printer off and on again, or run:

```sh
sudo hp1020-tool loadfw
```

### "printer NOT found on USB"

- Check that the printer is switched on (green light) and that the cable is plugged in at both ends.
- Check what macOS sees:

  ```sh
  ioreg -p IOUSB -l -w0 | grep -i -A12 "LaserJet" | grep -E "LaserJet|idProduct|idVendor"
  ```

  You should see `idVendor = 1008` and `idProduct = 16663` (1018) or `11031` (1020).
- If you are going through a hub or dock, try another port on it, or connect directly with a USB‑C to USB‑B cable.

### Docks and hubs

The printer works through USB‑C docks and hubs; that is how it was tested. Some docks are slow to wake the port, which shows up as `Waiting for printer to become available` or `The printer is offline` in the log before printing starts. If printing is unreliable, a direct **USB‑C to USB‑B cable** is the quickest way to rule the dock out.

### The background service is not running

`hp1020-tool status` shows `NOT installed` or `installed but not loaded`, or the installer printed `Bootstrap failed`. Follow [step 5 of the installation](#5-allow-the-background-service).

To restart the service:

```sh
sudo launchctl kickstart -k system/local.hp1020.fwloader
```

### "Filter failed"

Turn on detailed logging, print again, read the log, then turn logging off again:

```sh
sudo hp1020-tool debug on
hp1020-tool test-print
sudo hp1020-tool log
sudo hp1020-tool debug off
```

Lines containing `rastertozjs` show the exact reason.

### The wrong model is detected

The firmware is normally chosen automatically. To force one, edit `/Library/Printers/HP1020/hp1020.conf`:

```sh
sudo nano /Library/Printers/HP1020/hp1020.conf
```

Set, for example, `FIRMWARE="/Library/Printers/HP1020/Firmware/sihp1018.dl"`, save with Ctrl+O, Return and Ctrl+X, then restart the service:

```sh
sudo launchctl kickstart -k system/local.hp1020.fwloader
```

### Gray areas look too dark or too light

Change **Print Density** (2 for lighter, 4 for darker). For photos, try **Halftoning → Error diffusion**.

### Settings file

`/Library/Printers/HP1020/hp1020.conf`:

| Setting | Meaning |
|---|---|
| `QUEUE` | Which print queue the service manages |
| `PAUSE_WHEN_ABSENT` | `yes`: pause the queue while the printer is off (recommended); `no`: never pause it |
| `POLL`, `SETTLE` | How often (in seconds) to check USB, and how long to wait after the printer appears |
| `FIRMWARE` | Empty means the firmware is chosen automatically by model; or give a fixed path |

After editing the file, run `sudo launchctl kickstart -k system/local.hp1020.fwloader`.

---

## Command reference

```sh
hp1020-tool status                     # driver, service, USB and queue state (use sudo for full details)
hp1020-tool test-dry [file] [-o ...]   # run the driver without printing and show a preview
hp1020-tool test-print [file] [-o ...] # print the test page (or a file) and follow the job
sudo hp1020-tool loadfw [--force]      # upload the firmware now
sudo hp1020-tool add-queue [--uri URI] # (re)create the print queue
sudo hp1020-tool remove-queue          # delete the print queue
hp1020-tool log                        # service log and recent printing‑system errors
sudo hp1020-tool debug on|off          # detailed logging in the printing system

lpstat -p                              # state of all queues
lpstat -o                              # waiting jobs
cancel -a QUEUE                        # cancel all jobs in a queue
sudo cupsenable QUEUE                  # resume a paused queue
```

---

## Uninstalling

From the project folder:

```sh
sudo sh ./uninstall.sh                 # removes the driver, the service and the print queue
sudo sh ./uninstall.sh --keep-queue    # removes everything except the print queue
```

---

## What gets installed

```
/Library/Printers/HP1020/Filter/rastertozjs, foo2zjs      print filter and ZjStream encoder (arm64)
/Library/Printers/HP1020/Firmware/sihp1018.dl, sihp1020.dl  printer firmware (picked by model)
/Library/Printers/HP1020/bin/                             hp1020-fwd, hp1020-tool, zjsdecode, helpers
/Library/Printers/HP1020/share/                           test page, README, licenses
/Library/Printers/HP1020/hp1020.conf                      settings
/Library/Printers/PPDs/Contents/Resources/HP-LaserJet-1020-arm64.ppd.gz
/Library/LaunchDaemons/local.hp1020.fwloader.plist        background service
/Library/Logs/HP1020/hp1020-fwd.log                       service log
/usr/local/bin/hp1020-tool                                link to the helper command
```

Nothing else on the system is modified, and `uninstall.sh` removes all of the above.

---

## Building from source only

```sh
sh ./build.sh                          # compile into ./build and download the firmware
sh ./build.sh --firmware FILE          # use a local firmware file
sh ./build.sh --clean                  # remove ./build
```

Repository layout:

```
src/rastertozjs.c        the CUPS filter (this project)
src/foo2zjs/             foo2zjs encoder, JBIG‑KIT, zjsdecode, arm2hpdl (upstream, GPL)
scripts/                 firmware service, helper tool, shared functions, default settings
ppd/                     the generated PPD
launchd/                 LaunchDaemon definition
tools/gen_ppd.py         generates the PPD (paper sizes and margins)
tools/make_testpage.py   generates test/testpage.pdf
install.sh, uninstall.sh, build.sh
```

The build also compiles and runs on Linux with CUPS, which is useful for development: `rastertozjs` and `foo2zjs` can be tested there with `cupsfilter`.

### Technical notes

- macOS's USB backend rebuilds the printer's IEEE‑1284 device ID from only `MFG`, `MDL` and `SERN`, so the `FWVER` field that Linux tools use to detect loaded firmware is not visible. Instead, the service treats a firmware upload as confirmed when the printer re‑enumerates on USB right after it. A new USB session within 120 s of an upload is treated as the same power‑on and does not trigger another upload.
- A job that started while the printer had no firmware never finishes and keeps the USB port open. Before uploading, the service holds such jobs through CUPS (`lp -i JOB -H hold`) and releases them afterwards (`-H resume`).
- The firmware is downloaded from a pinned commit of the [foo2zjs mirror](https://github.com/koenkooi/foo2zjs), converted with `arm2hpdl`, and both the image and the result are checked against SHA‑256 checksums in `build.sh`.

---

## Reporting problems

Please open an issue with:

- your Mac model and macOS version (`sw_vers`);
- the printer model;
- how the printer is connected (direct cable, hub or dock);
- the output of:

  ```sh
  sudo hp1020-tool status
  tail -n 40 /Library/Logs/HP1020/hp1020-fwd.log
  /usr/libexec/cups/backend/usb
  lpstat -v
  ```

The printer's serial number appears in some of this output. You can replace it with `XXXX` before posting.

---

## Credits and licenses

- **foo2zjs**, **zjsdecode** and **arm2hpdl**: © Rick Richardson, GPL‑2.0‑or‑later. <http://foo2zjs.rkkda.com/> (mirror: <https://github.com/koenkooi/foo2zjs>)
- **JBIG‑KIT**: © Markus Kuhn, GPL.
- `rastertozjs`, the scripts and the PPD: GPL‑2.0‑or‑later, see [LICENSE](LICENSE).
- The printer **firmware** (`sihp1018`, `sihp1020`) is © HP. It is **not included** in this repository; the installer downloads it from the foo2zjs mirror. See `src/foo2zjs/COPYING` for the upstream notices.

HP and LaserJet are trademarks of HP Inc. This project is not affiliated with or endorsed by HP.
