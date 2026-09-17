# python-escpos-cli Skill

An AI skill for controlling ESC/POS thermal receipt printers from the command line using the [python-escpos](https://github.com/python-escpos/python-escpos) CLI.

## What This Skill Does

This skill teaches an AI agent (or serves as a human reference) how to:

- **Install** python-escpos and its dependencies (pyusb, libusb, Pillow, etc.)
- **Identify** a connected thermal printer (USB, serial, network, file, LP, CUPS, Win32Raw)
- **Configure** a `config.yaml` file for any of the 8 supported printer types
- **Print** text, barcodes, QR codes, images, and cut paper, all from the shell
- **Troubleshoot** common issues and known bugs in v3.1

It was built from the [context7 llms.txt](https://context7.com/python-escpos/python-escpos/llms.txt) documentation, verified against the actual CLI source code (`cli.py`), and **tested on physical hardware** (Epson TM-m30II, USB).

## Supported Printers

| Type | Config `type` | Use Case |
|---|---|---|
| USB | `usb` | Direct USB connection (most common) |
| Serial | `serial` | RS-232 serial port |
| Network | `network` | TCP/IP (port 9100) |
| File | `file` | Raw device node (`/dev/usb/lp0`) |
| Dummy | `dummy` | Testing without hardware |
| LP | `lp` | CUPS via `lp` command |
| CUPS | `cupsprinter` | CUPS server |
| Win32Raw | `win32raw` | Windows raw printing |

Tested on: **Epson TM-m30II** (USB `0x04b8:0x0e2a`), python-escpos 3.1, macOS 14.6.

## CLI Commands Covered

| Command | Status | Description |
|---|---|---|
| `text` | ✅ Working | Print plain text |
| `block_text` | ✅ Working | Print word-wrapped text |
| `qr` | ✅ Working | Print a QR code |
| `barcode` | ✅ Working | Print barcodes (EAN13, CODE128, UPC, etc.) |
| `image` | ✅ Working | Print bitmap images (bitImageRaster, bitImageColumn, graphics) |
| `cut` | ✅ Working | Cut paper (FULL or PART) |
| `set` | ✅ Partial | Set text alignment/size/density (align/width/height work; `--text_type` is broken) |
| `hw` | ✅ Working | Hardware operations (INIT, SELECT, RESET) |
| `control` | ✅ Working | Control sequences (LF, FF, CR, HT, VT) |
| `panel_buttons` | ✅ Working | Enable/disable feed button |
| `raw` | ✅ Working | Send raw ESC/POS bytes |
| `charcode` | ✅ Working | Set character code page |
| `fullimage` | ❌ Broken | KeyError in v3.1: use `image` instead |
| `software_columns` | ❌ N/A | Not in v3.1 release |
| `demo` | ❌ Broken | Argparse filtering bug crashes all modes |
| `cashdraw` | ❌ Broken from CLI | Argparse type mismatch: use Python API |
| `version` | ✅ Working | Print version |
| `version_extended` | ✅ Working | Diagnostics + driver usability |

## Quick Start

### 1. Install

```bash
pip install python-escpos[all]

# macOS USB support
brew install libusb
pip install pyusb

# Linux USB support
sudo apt-get install libusb-1.0-0-dev
pip install pyusb
```

### 2. Identify your printer

```bash
# macOS
system_profiler SPUSBDataType | grep -B2 -A8 "0x04b8"

# Linux
lsusb
```

Look for the Vendor ID and Product ID (e.g. `0x04b8:0x0e2a` for Epson TM-m30II).

### 3. Create config

```bash
mkdir -p ~/Library/Application\ Support/python-escpos  # macOS
# or: mkdir -p ~/.config/python-escpos                  # Linux
```

Create `config.yaml`:

```yaml
printer:
  type: usb
  idVendor: 0x04b8
  idProduct: 0x0e2a
  profile: default
```

### 4. Print

```bash
python-escpos text --txt "Hello World"
python-escpos qr --content "https://example.com" --size 6
python-escpos barcode --code 4006381333931 --bc EAN13 --height 64 --width 2
python-escpos cut --mode FULL
```

## Known Bugs in v3.1

| Bug | Workaround |
|---|---|
| `set --text_type` → `TypeError` | Use `raw --msg "\x1bE\x01"` for bold, or Python API `p.set(bold=True)` |
| `fullimage` → `KeyError` | Use `image --img_source` instead |
| `demo` → `BarcodeCodeError` | Don't use `demo`; test individual commands |
| `cashdraw --pin` → invalid choice | Use `python3 -c "from escpos.printer import Usb; p=Usb(0x04b8,0x0e2a); p.cashdraw(2)"` |
| `charcode --code UTF8` → `KeyError` | Only `CP*` names valid: `CP437`, `CP858`, `CP1252`, etc. |
| `text --txt '---'` → argparse error | Any value starting with `-` breaks argparse. Use `=`, `.`, or `_` separators |

## How to Install This Skill

### For AI agent harnesses that use `~/.agents/skills/`

```bash
# Clone the repo
gh repo clone faramirezs/python-escpos-cli-skill ~/.agents/skills/python-escpos-cli
```

The skill is auto-discovered on the next agent session. The trigger keywords are in the `description` frontmatter field.

### For Hermes-agent / other skill systems

Copy `SKILL.md` into your skills directory following your harness's conventions. See the [hermes-agent-skill-authoring](https://github.com/hernes-agent) docs for in-repo skill placement.

## File Structure

```
.
├── SKILL.md      # The skill (frontmatter + full documentation)
└── README.md     # This file
```

## Sources

- **context7:** https://context7.com/python-escpos/python-escpos/llms.txt
- **CLI source (verified):** `src/escpos/cli.py` from python-escpos v3.1
- **Config source (verified):** `src/escpos/config.py`
- **Capabilities DB:** https://github.com/escpos/escpos-printer-db
- **Official docs:** https://python-escpos.readthedocs.io/
- **Hardware tested:** Epson TM-m30II (USB `0x04b8:0x0e2a`)

## License

MIT: same as python-escpos itself.