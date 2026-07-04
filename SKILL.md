---
name: python-escpos-cli
description: "Control ESC/POS thermal receipt printers (Epson TM-T20/T88/TM-m30, Star, etc.) from the command line using the python-escpos CLI. Covers install, YAML config file creation for USB/Serial/Network/File/Dummy/LP/CUPS/Win32Raw printers, and every CLI subcommand: text, block_text, qr, barcode, image, cut, cashdraw, charcode, set, hw, control, panel_buttons, raw, demo, version, version_extended. Includes known bugs in v3.1 and workarounds. Use when the user wants to print receipts, barcodes, QR codes, or images to a thermal printer, set up a python-escpos configuration, run printer demos, or script receipt printing from a shell. Triggers on: python-escpos, escpos CLI, thermal printer, receipt printer, ESC/POS, print barcode, print QR code, Epson TM-T, TM-m30, print receipt, cash drawer, cut paper, printer config.yaml, python-escpos cut, python-escpos text, python-escpos barcode."
---

## Overview

`python-escpos` is a Python library and CLI for driving ESC/POS-compatible thermal receipt printers (Epson TM-T20/T88/III/TM-m30II, Star Micronics, and many others). The CLI — invoked as `python-escpos` — mirrors the library's ESC/POS commands as subcommands, so you can print text, barcodes, QR codes, images, cut paper, and kick cash drawers directly from a shell or script **without writing Python code**.

The CLI **requires a YAML configuration file** that defines which printer to use and how to connect to it. Every subcommand loads this config, instantiates the printer, executes the command, and exits. There is no interactive session.

This skill was built from the context7 llms.txt documentation, verified against the actual CLI source (`cli.py`), and **tested on physical hardware** (Epson TM-m30II, USB). Where the documentation differs from tested reality, this skill reflects what actually works and notes known bugs.

## Artistic Expression on Thermal Printers

ESC/POS thermal printers are a **natural canvas for ASCII art and text-based art**. The monochrome, fixed-width, dot-matrix medium is uniquely suited to creative expression — this is not a bug, it's the medium's aesthetic. When printing for anything beyond a plain receipt, **prefer artistic expression over plain utility**.

### Why ASCII art works so well here

- The printer is inherently **1-bit monochrome** — black dots on white paper. ASCII art maps directly to this constraint (inked characters vs whitespace).
- **Fixed-width fonts** (Font A and Font B) mean every character occupies the same grid cell — perfect for aligning ASCII art columns.
- **80mm paper** gives ~48 characters per line in Font A, or ~64 in Font B. This is the classic terminal width ASCII art was designed for.
- The medium is **ephemeral** — receipts are meant to be discarded. This invites experimentation without commitment.

### Recommended techniques

1. **ASCII art headers and dividers** — Use box-drawing characters, decorative borders, and figlet-style banners instead of plain `====` separators:
   ```bash
   python-escpos text --txt "  +-------------------+"
   python-escpos text --txt "  |  WELCOME TO CAFE  |"
   python-escpos text --txt "  |    MILANO ROSA    |"
   python-escpos text --txt "  +-------------------+"
   ```

2. **Text-based borders and frames** — Box-drawing characters (`+`, `-`, `|`, `*`) create visual structure that plain text lacks:
   ```bash
   python-escpos text --txt "+-------------------------------------------+"
   python-escpos text --txt "|  Coffee..............................$3.50  |"
   python-escpos text --txt "|  Bagel...............................$2.00  |"
   python-escpos text --txt "|  TOTAL...............................$5.50  |"
   python-escpos text --txt "+-------------------------------------------+"
   ```

3. **Figlet / banner text** — Generate large ASCII text for headers using `figlet` or manual block text:
   ```bash
   # Generate with figlet, then print
   figlet -f standard "MILAN" | while IFS= read -r line; do python-escpos text --txt "$line"; done
   ```
   Or use the CLI's `set --width 2 --height 2` for built-in large text.

4. **ASCII decorations** — Use characters like `*`, `~`, `.`, `=`, `#`, `@`, and Unicode box characters for visual variety:
   ```bash
   python-escpos text --txt "  *  *  *  *  *  *  *  *  *  *  *"
   python-escpos text --txt "    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~"
   python-escpos text --txt "  ###############################"
   ```

5. **Whitespace as art** — Thermal printers respect leading spaces. Use indentation and spacing for visual rhythm, alignment, and emphasis. A blank line (`python-escpos text --txt " "`) creates vertical breathing room.

6. **Compositions mixing art + data** — Combine ASCII borders, banner text, data (QR codes, barcodes), and images into a single composed piece. The CLI's stateful `set` command lets you mix alignment and sizes within one print job.

### Design principles for thermal print art

- **Less ink, more whitespace.** Thermal paper burns ink (dots) irreversibly. Dense ASCII blocks can cause paper curl or heat bleed. Favor sparse, high-contrast designs.
- **Test with `dummy` printer first.** Use `type: dummy` in the config to preview output without wasting paper.
- **Font A for art, Font B for density.** Font A is wider (better for ASCII art alignment); Font B is narrower (more characters per line, denser look).
- **Avoid character values starting with `-`** — argparse in v3.1 misinterprets them as flags. For separators, use `=`, `.`, `*`, `~`, `#`, `_`, or `+` instead of dashes.
- **Embrace the impermanence.** Receipt art is meant to be seen, smiled at, and discarded. Don't over-engineer; iterate fast and print often.

### Known Bugs in v3.1 (tested)

| Command | Status | Issue | Workaround |
|---|---|---|---|
| `set --text_type` | **BROKEN** | `Escpos.set()` has no `text_type` parameter; `TypeError: unexpected keyword argument 'text_type'`. The CLI defines `--text_type` but the library method doesn't accept it. | Use `--bold`, `--underline`, `--width`, `--height`, `--invert` etc. (the params `Escpos.set()` actually accepts). Note: the CLI only exposes `--align`, `--font`, `--text_type` (broken), `--width`, `--height`, `--density`, `--invert`, `--smooth`, `--flip`. For bold/underline, use the Python API or `raw` to send ESC/POS bytes directly. |
| `fullimage` | **BROKEN** | `KeyError: 'fullimage'` — the CLI defines the subcommand but there's no corresponding `fullimage()` function in the module (the method doesn't exist on `Escpos` class in v3.1). | Use `image --img_source` instead. |
| `software_columns` | **NOT AVAILABLE** | Subcommand doesn't exist in the installed v3.1 CLI (it appears in GitHub master source but was added after the 3.1 release). | Use `block_text` or manually pad text with spaces. |
| `demo --text` / `demo --qr` | **BROKEN** | The `main()` function filters args with `if v is not None`, but `store_true` defaults to `False` (not `None`), so ALL demo flags are passed to `demo()`. It iterates over every key and crashes on invalid UPC-E demo data. | Don't use `demo`. Test individual commands instead: `python-escpos text --txt "Hello"`, `python-escpos qr --content "..."`, etc. |
| `cashdraw --pin` | **BROKEN from CLI** | `argparse` validates `--pin` as a string but the choices are integers (`[2, 5]`), causing `invalid choice: '2'`. | Use the Python API directly: `python3 -c "from escpos.printer import Usb; p = Usb(0x04b8, 0x0e2a); p.cashdraw(2)"` |
| `charcode --code UTF8` | **ERROR** | `UTF8` is not a valid codepage name in the capabilities DB. Only `CP*` names exist (e.g. `CP437`, `CP858`, `CP1252`). | Use valid codepage names: `CP437`, `CP858`, `CP1252`, etc. |

### Commands confirmed working on v3.1

`text`, `block_text`, `qr`, `barcode`, `image`, `cut`, `charcode` (with valid codepages), `set` (alignment/width/height/density/invert/smooth/flip only), `hw`, `control`, `panel_buttons`, `raw`, `version`, `version_extended`

This skill walks through the full workflow: install → identify printer → write config → run commands → troubleshoot.

## When to Use

- User wants to print text, barcodes, QR codes, or images to a thermal/receipt printer from the command line.
- User mentions `python-escpos`, ESC/POS, or a specific printer model (Epson TM-T20, TM-T88, TM-TIII, TM-m30II, Star, etc.).
- User wants to set up a `config.yaml` for a USB, serial, network, or CUPS printer.
- User wants to script receipt printing from a shell script.
- User wants to kick a cash drawer, cut paper, or send raw ESC/POS bytes.

**Don't use for:**
- Printing to standard desktop printers via CUPS/lp on regular paper — this is specifically for ESC/POS thermal protocols.
- Full programmatic control where conditional logic is needed — use the Python library directly instead of the CLI.
- `demo`, `fullimage`, `software_columns`, or `set --text_type` — these are broken in v3.1. See Known Bugs above.

## Step-by-Step Workflow

### Step 1 — Install python-escpos

```bash
pip install python-escpos[all]
```

The `[all]` extra pulls in every platform-specific dependency (pyusb, pyserial, Pillow, qrcode, python-barcode). For a minimal install (only the deps you need):

```bash
pip install python-escpos              # core only
pip install "python-escpos[usb]"       # USB printers (pyusb + libusb)
pip install "python-escpos[serial]"    # Serial printers (pyserial)
pip install "python-escpos[image]"     # Image printing (Pillow)
```

**System dependencies:**
- **USB:** `libusb-1.0` must be installed. On macOS: `brew install libusb`. On Debian/Ubuntu: `sudo apt-get install libusb-1.0-0-dev`.
- **Serial:** no extra system package; pyserial is pure Python.

Verify the install and the CLI is on PATH:

```bash
python-escpos version
```

### Step 2 — (Optional) Enable shell tab-completion

```bash
eval "$(register-python-argcomplete python-escpos)"
```

Add this to your `~/.bashrc` or `~/.zshrc` for persistence. Requires the `argcomplete` package (`pip install argcomplete`). If it's missing, the CLI still works — you just don't get tab completion.

### Step 3 — Identify your printer

You need to know the connection type and parameters before writing the config.

#### USB printers

Find the Vendor ID and Product ID:
- **macOS:** `system_profiler SPUSBDataType | grep -B2 -A8 "0x04b8"` (or search for your printer manufacturer name)
- **Linux:** `lsusb`

```bash
# macOS example output:
#   Product ID: 0x0e2a
#   Vendor ID: 0x04b8  (Seiko Epson Corp.)
#   Manufacturer: EPSON
#   1284 Device ID: MFG:EPSON;CMD:ESC/POS;MDL:TM-m30II;CLS:PRINTER;DES:EPSON TM-m30II

# Linux example:
# Bus 001 Device 005: ID 04b8:0202 Seiko Epson Corp. TM-T88III
#                      ^^^^ ^^^^
#                   vendor_id product_id
```

The IDs are `0x04b8` (vendor) and `0x0e2a` / `0x0202` (product). Most Epson printers use `0x04b8` as the vendor ID.

**Linux udev rule (recommended):** By default, USB devices require root. Create a udev rule so non-root users can access the printer:

```bash
sudo tee /etc/udev/rules.d/99-escpos.rules <<EOF
SUBSYSTEM=="usb", ATTRS{idVendor}=="04b8", ATTRS{idProduct}=="0e2a", MODE="0664", GROUP="dialout"
EOF
sudo udevadm control --reload
sudo udevadm trigger
```

Add your user to the `dialout` group if not already: `sudo usermod -aG dialout $USER` (log out/in to apply).

**macOS:** No udev rules needed. USB access works without root on macOS.

#### Serial printers

Identify the device path:
- Linux: `/dev/ttyUSB0`, `/dev/ttyS0`, etc. Check with `ls /dev/ttyUSB* /dev/ttyS*`.
- macOS: `/dev/tty.usbserial-*`. Check with `ls /dev/tty.usbserial*`.
- Windows: `COM1`, `COM2`, etc.

You also need to know baud rate (commonly 9600 or 19200), data bits (8), parity (N), and stop bits (1).

#### Network printers

Find the printer's IP address (e.g., `192.168.1.100`). The default ESC/POS network port is `9100`.

#### File/LP printers

- **File:** The device node, e.g. `/dev/usb/lp0` (Linux). Check with `ls /dev/usb/lp*`.
- **LP:** Uses the UNIX `lp` command to talk to CUPS. The printer must be configured in CUPS. List with `lpstat -p`.

#### CUPS printers

The printer must be configured in the CUPS administration interface (http://localhost:631). List with `lpstat -p`.

### Step 4 — Create the configuration file

The CLI reads a YAML file named `config.yaml`. By default, it looks in the platform's user config directory:

| Platform | Default path |
|---|---|
| Linux | `~/.config/python-escpos/config.yaml` |
| macOS | `~/Library/Application Support/python-escpos/config.yaml` |
| Windows | `C:\Users\<user>\AppData\Local\python-escpos\config.yaml` |

You can place it there, or pass a custom path with `-c`/`--config` on any command.

The file must have a top-level `printer` key with a `type` field and connection-specific parameters. The `type` is case-insensitive and maps to a printer class.

#### Config file structure

```yaml
printer:
  type: <printer_type>     # Required. One of: usb, serial, network, file, dummy, cupsprinter, lp, win32raw
  profile: default         # Optional. Printer capabilities profile (see below)
  # ... type-specific parameters below
```

#### Profile selection

The `profile` parameter tells python-escpos which features the printer supports (barcode types, cut modes, code pages, paper width). Available profiles include: `TM-T20II`, `TM-T88II`, `TM-T88III`, `TM-T88IV`, `TM-T88V`, `TM-L90`, `TM-P80`, `TM-U220`, `default`.

**If your exact printer model isn't in the capabilities DB** (e.g. TM-m30II), use `profile: default`. This works for all basic operations. The only limitation: center-alignment of barcodes/images won't work (warning: "The media.width.pixel field of the printer profile is not set"), but left/right alignment and all print operations work fine.

To list all available profiles:
```bash
python3 -c "from escpos.capabilities import CAPABILITIES; print(sorted(CAPABILITIES['profiles'].keys()))"
```

#### USB config example

```yaml
printer:
  type: usb
  idVendor: 0x04b8
  idProduct: 0x0e2a          # Replace with your printer's product ID
  # Optional advanced USB params (defaults shown):
  # in_ep: 0x82
  # out_ep: 0x01
  # timeout: 0
  profile: default
```

#### Serial config example

```yaml
printer:
  type: serial
  devfile: /dev/ttyUSB0
  baudrate: 9600
  bytesize: 8
  parity: N
  stopbits: 1
  timeout: 1.0
  # Flow control (optional):
  # dsrdtr: true
  # xonxoff: false
  profile: default
```

#### Network config example

```yaml
printer:
  type: network
  host: 192.168.1.100
  # port: 9100       # Optional, default 9100
  # timeout: 60      # Optional, default 60
  profile: default
```

#### File config example

```yaml
printer:
  type: file
  devfile: /dev/usb/lp0
  profile: default
```

#### Dummy config (testing without a printer)

```yaml
printer:
  type: dummy
```

The Dummy printer captures all ESC/POS output in memory without sending to hardware. Useful for testing command sequences.

#### LP config (CUPS via lp command)

```yaml
printer:
  type: lp
  # host: localhost    # Optional, for remote CUPS server
  # The printer name is auto-detected; or specify:
  # printer_name: MyReceiptPrinter
```

#### CUPS config

```yaml
printer:
  type: cupsprinter
  # printer_name: MyReceiptPrinter   # Optional, auto-detected if omitted
```

#### Win32Raw config (Windows only)

```yaml
printer:
  type: win32raw
  printer_name: "My Receipt Printer"
```

### Step 5 — Verify installation and driver usability

Before printing, confirm the USB driver (or your relevant driver) is usable:

```bash
python-escpos version_extended
```

This prints python-escpos version, Python version, platform, and whether each driver is usable. If your driver shows `False`, install the missing dependency:

| Driver | Required package | System dep |
|---|---|---|
| USB | `pip install pyusb` | `brew install libusb` (macOS) or `apt install libusb-1.0-0-dev` (Linux) |
| Serial | `pip install pyserial` | none |
| File | (always usable) | none |
| Network | (always usable) | none |
| LP | (always usable on UNIX) | CUPS configured |
| CUPS | `pip install pycups` | CUPS installed |

### Step 6 — Run CLI commands

With the config in place, every command follows this pattern:

```bash
python-escpos [-c <config_path>] <subcommand> [options]
```

If you placed `config.yaml` in the default location, you can omit `-c`. If you pass a directory path (not a file), the CLI appends `config.yaml` to it.

**Quick start (print text and cut):**

```bash
python-escpos text --txt "Hello World"
python-escpos cut
```

Always run `cut` (or `cut --mode FULL`) as the last command to eject the paper.

## CLI Command Reference

Every subcommand below is invoked as `python-escpos <subcommand> [options]`. The global `-c`/`--config` option can precede any subcommand to specify an alternate config path.

**Available subcommands in v3.1:** `qr`, `barcode`, `text`, `block_text`, `cut`, `cashdraw`, `image`, `fullimage` *(broken)*, `charcode`, `set`, `hw`, `control`, `panel_buttons`, `raw`, `demo` *(broken)*, `version`, `version_extended`

### text — Print plain text

```bash
python-escpos text --txt "Hello World"
python-escpos text --txt "Line 1\nLine 2"
```

| Option | Required | Description |
|---|---|---|
| `--txt` | Yes | Plain text to print. Use `\n` for newlines. |

The CLI automatically appends a newline after `text`.

### block_text — Print wrapped text

Wraps long text to fit the printer's paper width.

```bash
python-escpos block_text --txt "This is a very long line of text that will be automatically wrapped to fit the printer width." --columns 32
```

| Option | Required | Description |
|---|---|---|
| `--txt` | Yes | Text to print (will be wrapped). |
| `--columns` | No | Number of columns (characters per line). |
~~~~~~~~

### qr — Print a QR code

```bash
python-escpos qr --content "https://example.com"
python-escpos qr --content "WiFi config data" --size 6
```

| Option | Required | Description |
|---|---|---|
| `--content` | Yes | Text/URL to encode in the QR code. |
| `--size` | No | QR code module size (1–16). Default: 3. |

**Note:** If using `profile: default`, you may see a warning "The media.width.pixel field of the printer profile is not set. The center flag will have no effect." — this is harmless; the QR code still prints, just not center-aligned.

### barcode — Print a barcode

```bash
# EAN-13 (hardware function type A)
python-escpos barcode --code 4006381333931 --bc EAN13

# CODE128 (requires function type B)
python-escpos barcode --code "{B012ABCDabcd" --bc CODE128 --function_type B

# With label position and size
python-escpos barcode --code 123456789012 --bc EAN13 --height 64 --width 2 --pos BELOW --font A
```

| Option | Required | Description |
|---|---|---|
| `--code` | Yes | Barcode data. For CODE128 with function type B, prefix with `{B` for full ASCII. |
| `--bc` | Yes | Barcode format. See supported formats below. |
| `--height` | No | Barcode height in pixels (integer). |
| `--width` | No | Barcode width (integer). |
| `--pos` | No | Label position: `BELOW`, `ABOVE`, `BOTH`, `OFF`. |
| `--font` | No | Label font: `A` or `B`. |
| `--align_ct` | No | Center-align the barcode. Accepts `y`/`n`/`true`/`false`/`1`/`0`. |
| `--function_type` | No | ESC/POS function type: `A` or `B`. B enables more barcode types. |
| `--force_software` | No | Force software rendering as image: `graphics`, `bitImageColumn`, or `bitImageRaster`. |

**Supported barcode formats:**

Function type **A**: UPC-A, UPC-E, EAN13, EAN8, CODE39, ITF, NW7

Function type **B** (additional): CODE93, CODE128A, CODE128B, CODE128C, GS1-128, GS1 DataBar Omnidirectional, GS1 DataBar Truncated, GS1 DataBar Limited, GS1 DataBar Expanded

### image — Print a bitmap image

Prints an image file using ESC/POS bit-image commands. Best for logos and icons.

```bash
python-escpos image --img_source logo.png
python-escpos image --img_source logo.png --impl bitImageRaster --high_density_horizontal true --high_density_vertical true
```

| Option | Required | Description |
|---|---|---|
| `--img_source` | Yes | Path to image file (PNG, GIF, BMP, JPEG, etc.). |
| `--impl` | No | Implementation: `bitImageRaster`, `bitImageColumn`, or `graphics`. |
| `--high_density_horizontal` | No | Horizontal density (bool). |
| `--high_density_vertical` | No | Vertical density (bool). |

**Note:** Image width should match the printer's pixel width: 58mm printers = 384px, 80mm printers = 576px. If the image is wider, it may be truncated or cause errors.

### fullimage — Print a large image *(BROKEN in v3.1)*

**This command is broken in python-escpos 3.1.** The CLI defines the subcommand but `fullimage()` doesn't exist as a function, causing `KeyError: 'fullimage'`. Use `image` instead.

### cut — Cut the paper

```bash
python-escpos cut
python-escpos cut --mode FULL
```

| Option | Required | Description |
|---|---|---|
| `--mode` | No | Cut type: `FULL` or `PART`. Default: `PART`. |

Most printers feed a few lines of paper before cutting automatically.

### cashdraw — Kick the cash drawer *(BROKEN from CLI in v3.1)*

**The CLI `cashdraw` command is broken** — `argparse` validates `--pin` as a string but the choices list is integers `[2, 5]`, causing a type mismatch error: `invalid choice: '2'`.

**Workaround — use the Python API directly:**

```bash
python3 -c "from escpos.printer import Usb; p = Usb(0x04b8, 0x0e2a, profile='default'); p.cashdraw(2)"
```

Replace `0x04b8`, `0x0e2a` with your printer's vendor/product IDs. Use pin `2` or `5`.

### charcode — Set character code table

```bash
python-escpos charcode --code CP437
python-escpos charcode --code CP1252
```

| Option | Required | Description |
|---|---|---|
| `--code` | Yes | Character code page name. Must be a valid codepage from the capabilities DB. |

**Valid codepage names** are `CP*` format only (e.g. `CP437`, `CP858`, `CP1252`, `CP1001`, `CP3041`, etc.). `UTF8` is **not** valid and will raise `KeyError`. To list all valid codepages:

```bash
python3 -c "from escpos.capabilities import CAPABILITIES; print(sorted(CAPABILITIES['encodings'].keys()))"
```

### set — Set text properties

Configures alignment, font, style, and size for subsequent print commands.

```bash
python-escpos set --align center --width 2 --height 2
python-escpos text --txt "Big centered text"
python-escpos set --align left --width 1 --height 1
python-escpos text --txt "normal text"
```

| Option | Required | Description |
|---|---|---|
| `--align` | No | Horizontal alignment: `left`, `center`, `right`. |
| `--font` | No | Font: `A` (default) or `B` (smaller). |
| `--text_type` | No | **BROKEN in v3.1** — raises `TypeError`. Do not use. |
| `--width` | No | Width multiplier (1–8). |
| `--height` | No | Height multiplier (1–8). |
| `--density` | No | Print density (integer). |
| `--invert` | No | White-on-black printing (bool). |
| `--smooth` | No | Text smoothing, effective on 4×4+ text (bool). |
| `--flip` | No | Flip text upside down (bool). |

**Bold/underline workaround:** Since `--text_type` is broken, to print bold or underlined text, either:
1. Use the Python API: `python3 -c "from escpos.printer import Usb; p = Usb(0x04b8, 0x0e2a, profile='default'); p.set(bold=True); p.text('Bold text\n'); p.cut()"`
2. Send raw ESC/POS bytes: `python-escpos raw --msg "\x1bE\x01"` (ESC E 1 = bold on), then print text, then `python-escpos raw --msg "\x1bE\x00"` (bold off).

**Note:** `set` is stateful — properties persist until changed. Always reset to width=1, height=1 after styled output, or send `hw --hw INIT` to fully reset the printer.

### hw — Hardware operations

```bash
python-escpos hw --hw INIT      # Initialize printer (reset to defaults)
python-escpos hw --hw SELECT    # Select printer (make it active)
python-escpos hw --hw RESET     # Hardware reset
```

| Option | Required | Description |
|---|---|---|
| `--hw` | Yes | Operation: `INIT`, `SELECT`, or `RESET`. |

### control — Control sequences

```bash
python-escpos control --ctl LF       # Line feed
python-escpos control --ctl HT --pos 2   # Horizontal tab to position 2
```

| Option | Required | Description |
|---|---|---|
| `--ctl` | Yes | Control sequence: `LF` (line feed), `FF` (form feed), `CR` (carriage return), `HT` (horizontal tab), `VT` (vertical tab). |
| `--pos` | No | Horizontal tab position (1–4). Only relevant for `HT`. |

### panel_buttons — Enable/disable panel buttons

```bash
python-escpos panel_buttons --enable false    # Disable the feed button
python-escpos panel_buttons --enable true     # Re-enable
```

| Option | Required | Description |
|---|---|---|
| `--enable` | Yes | Whether the feed button is enabled (bool). |

### raw — Send raw data

Sends raw bytes or text directly to the printer, bypassing ESC/POS command formatting. For advanced use with custom ESC/POS byte sequences.

```bash
python-escpos raw --msg "\x1b\x40"           # ESC @ = initialize printer
python-escpos raw --msg "Custom raw text"
python-escpos raw --msg "\x1bE\x01"          # ESC E 1 = bold on
python-escpos raw --msg "\x1bE\x00"          # ESC E 0 = bold off
python-escpos raw --msg "\x1b-\x01"          # ESC - 1 = underline on
python-escpos raw --msg "\x1b-\x00"          # ESC - 0 = underline off
```

| Option | Required | Description |
|---|---|---|
| `--msg` | Yes | Raw data string to send. Escape sequences like `\x1b` are interpreted. |

**Common ESC/POS raw sequences:**

| Bytes | Effect |
|---|---|
| `\x1b\x40` | Initialize printer (reset) |
| `\x1bE\x01` | Bold ON |
| `\x1bE\x00` | Bold OFF |
| `\x1b-\x01` | Underline ON (1-dot) |
| `\x1b-\x02` | Underline ON (2-dot) |
| `\x1b-\x00` | Underline OFF |
| `\x1ba\x00` | Left align |
| `\x1ba\x01` | Center align |
| `\x1ba\x02` | Right align |

### demo — Print demonstration output *(BROKEN in v3.1)*

**The `demo` command is broken.** Due to a bug in the CLI's argument filtering (`store_true` defaults to `False` not `None`), all demo flags are passed to the demo function, which then iterates over all demo types including barcodes. It crashes on invalid UPC-E demo data (`BarcodeCodeError`).

**Instead of demo, test individual commands:**

```bash
python-escpos text --txt "Hello, World!"
python-escpos qr --content "https://example.com"
python-escpos barcode --code 4006381333931 --bc EAN13 --height 64 --width 2
python-escpos image --img_source logo.png --impl bitImageRaster
python-escpos cut --mode FULL
```

### version — Print version

```bash
python-escpos version
```

### version_extended — Print extended diagnostic info

```bash
python-escpos version_extended
```

Prints python-escpos version, Python version, platform, and whether each printer driver is usable. **Use this for bug reports** and for diagnosing missing dependencies.

## Common Patterns

### Print a complete receipt

```bash
# Header (big, centered)
python-escpos set --align center --width 2 --height 2
python-escpos text --txt "MY STORE"
python-escpos text --txt "Receipt #001"
python-escpos set --align center --width 1 --height 1

python-escpos text --txt "123 Main Street"
python-escpos text --txt "Tel: 555-0100"
python-escpos text --txt "------------------------"

# Line items (manual padding since software_columns is unavailable)
python-escpos set --align left
python-escpos text --txt "Coffee..................$3.50"
python-escpos text --txt "Bagel...................$2.00"
python-escpos text --txt "Tax.....................$0.30"
python-escpos text --txt "------------------------"
python-escpos text --txt "TOTAL..................$5.80"

# QR code for digital receipt
python-escpos set --align center
python-escpos qr --content "https://example.com/receipt/001" --size 5

# Footer
python-escpos text --txt "Thank you for your visit!"
python-escpos cut --mode FULL
```

### Print and reset text style each time

Because `set` is stateful, if you forget to reset, subsequent prints inherit the styling. Safe pattern:

```bash
python-escpos set --align center --width 2 --height 2
python-escpos text --txt "HEADER"
python-escpos set --align left --width 1 --height 1
python-escpos text --txt "normal text"
```

Or reset the whole printer:

```bash
python-escpos hw --hw INIT
```

### Print bold text (workaround for broken --text_type)

```bash
# Bold ON
python-escpos raw --msg "\x1bE\x01"
python-escpos text --txt "This is bold"
# Bold OFF
python-escpos raw --msg "\x1bE\x00"
python-escpos text --txt "This is normal"
python-escpos cut
```

### Kick cash drawer (workaround for broken CLI command)

```bash
python3 -c "from escpos.printer import Usb; p = Usb(0x04b8, 0x0e2a, profile='default'); p.cashdraw(2)"
```

### Use a non-default config file

```bash
python-escpos -c /path/to/my/config.yaml text --txt "Hello"
python-escpos -c /path/to/config_dir/ text --txt "Hello"   # looks for config_dir/config.yaml
```

### Use capabilities profiles

The `profile` parameter in the config tells python-escpos which features the printer supports (barcode types, cut modes, code pages, paper width). Common profiles: `TM-T20II`, `TM-T88III`, `TM-T88IV`, `TM-T88V`, `default`.

To override the capabilities file entirely (advanced):

```bash
export ESCPOS_CAPABILITIES_FILE=/usr/local/share/python-escpos/capabilities.json
python-escpos text --txt "uses custom capabilities"

unset ESCPOS_CAPABILITIES_FILE   # revert to packaged profile
```

### Generate a test logo image with Python/PIL

```python
from PIL import Image, ImageDraw, ImageFont

img = Image.new('1', (576, 300), 255)  # 1-bit, 576px wide (80mm printer)
draw = ImageDraw.Draw(img)
draw.rectangle([188, 40, 388, 160], fill=0)       # printer body
draw.rectangle([218, 60, 358, 90], fill=255)       # paper slot
draw.rectangle([238, 90, 338, 200], outline=0, width=2)  # paper
draw.line([(248, 110), (328, 110)], fill=0, width=2)     # text lines on paper
draw.line([(248, 130), (328, 130)], fill=0, width=2)
font = ImageFont.truetype('/System/Library/Fonts/Menlo.ttc', 28)
text = 'MY LOGO'
bbox = draw.textbbox((0, 0), text, font=font)
draw.text(((576 - (bbox[2]-bbox[0])) / 2, 220), text, fill=0, font=font)
img.save('logo.png')
```

Then print: `python-escpos image --img_source logo.png --impl bitImageRaster`

## Common Pitfalls

1. **`set --text_type` crashes.** This is a bug in v3.1 — the CLI defines `--text_type` but `Escpos.set()` doesn't accept it. Use `raw` to send ESC/POS bold/underline bytes directly (see Common Patterns), or use the Python API with `p.set(bold=True)`.

2. **`demo` crashes on all subcommands.** The demo function iterates over all flags (due to a `store_true`/`None` filtering bug) and hits invalid UPC-E demo data. Don't use `demo`; test commands individually.

3. **`fullimage` raises KeyError.** The function doesn't exist in v3.1. Use `image` instead.

4. **`software_columns` doesn't exist.** Added after the 3.1 release. Manually pad text with dots/spaces for column layouts.

5. **`cashdraw --pin` rejects valid values.** Argparse type mismatch (string vs int choices). Use the Python API workaround.

6. **`charcode --code UTF8` crashes.** Only `CP*` codepage names are valid. List valid names with: `python3 -c "from escpos.capabilities import CAPABILITIES; print(sorted(CAPABILITIES['encodings'].keys()))"`

7. **No config file found.** The CLI exits with `No printers loaded from config`. The config must be at the default platform path OR passed via `-c`. Verify the path exists and the YAML has a `printer` section with a `type` key.

8. **USB driver shows `False` in version_extended.** Install `pyusb` (`pip install pyusb`) and the system `libusb` (`brew install libusb` on macOS, `apt install libusb-1.0-0-dev` on Linux).

9. **USB permission denied (Linux).** Create a udev rule (see Step 3) or run with `sudo`. Add your user to the `dialout` group.

10. **Barcode type not printing.** Some barcode types (CODE128, CODE93, GS1 variants) require `--function_type B`. If function type A doesn't support it, the printer may error or print garbage.

11. **Forgetting to cut.** Paper won't eject unless you run `cut`. The last command in any sequence should be `python-escpos cut`.

12. **Text style persists.** `set` is stateful. If you set double-width and don't reset, all subsequent text inherits it. Reset with `set --width 1 --height 1` or `hw --hw INIT`.

13. **Wrong baud rate (serial).** If the printer prints garbage characters, the baud rate doesn't match the printer's DIP switch setting. Common: 9600, 19200, 38400.

14. **Image too wide.** If an image causes errors, it exceeds the printer's native width. 58mm = 384px, 80mm = 576px. Resize before printing.

15. **CODE128 data needs prefix.** For function type B CODE128, prefix alphanumeric data with `{B` (e.g., `{BHELLO`).

16. **`--enable` / bool flags need explicit values.** Boolean options like `--align_ct`, `--invert`, `--high_density_vertical` require an explicit value (`true`, `false`, `y`, `n`, `1`, `0`). They are NOT simple flags.

17. **Network printer unreachable.** Ensure port 9100 is open. Test with `nc -zv 192.168.1.100 9100`.

18. **"media.width.pixel" warning with `default` profile.** Harmless — barcodes and QR codes still print, just not center-aligned. Use a specific profile (e.g. `TM-T88III`) if you need center alignment and your printer matches.

## Verification Checklist

- [ ] `python-escpos version` prints the version number (install confirmed)
- [ ] `python-escpos version_extended` shows your driver as `True` (usable)
- [ ] Config file exists at default path or is passed via `-c`
- [ ] Config YAML has `printer:` → `type:` and required connection params
- [ ] `python-escpos text --txt "Hello"` prints text successfully
- [ ] `python-escpos cut` ejects paper
- [ ] `python-escpos qr --content "test"` prints a QR code
- [ ] `python-escpos barcode --code 4006381333931 --bc EAN13` prints a barcode
- [ ] `python-escpos image --img_source logo.png` prints an image
- [ ] USB: udev rule set, user in `dialout` group (Linux only)
- [ ] Text style reset after formatting changes (`set --width 1 --height 1` or `hw --hw INIT`)

## Reference Links

- **PyPI:** https://pypi.org/project/python-escpos/
- **GitHub:** https://github.com/python-escpos/python-escpos
- **Documentation:** https://python-escpos.readthedocs.io/
- **Capabilities DB:** https://github.com/escpos/escpos-printer-db
- **context7 source:** https://context7.com/python-escpos/python-escpos/llms.txt
