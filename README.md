# Brother QL-810W Linux Setup

Complete setup guide for the Brother QL-810W label printer on Linux (Ubuntu/Debian, amd64), including two-color (DK-2251) tape support.

## Prerequisites

- Ubuntu/Debian-based Linux (amd64)
- Brother QL-810W connected via USB
- CUPS installed (`sudo apt install cups`)

## Step 1: Install the Brother LPD Driver (i386)

The official Brother driver is only available as an i386 package. Enable multiarch and install:

```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo dpkg -i ql810wpdrv-3.1.5-0.i386.deb
sudo apt install -f
```

Download the driver from [Brother's support page](https://support.brother.com/g/b/downloadtop.aspx?c=us&lang=en&prod=lpql810weus).

The errors about `/var/spool/lpd/` during install are harmless (that's the old LPD system, not CUPS). The "drivers are deprecated" warning is also safe to ignore.

## Step 2: Disable Editor Lite Mode

**This is critical.** If the Editor Lite button on your printer has a yellow light, the printer will appear as a USB mass storage device instead of a printer.

**Press the Editor Lite button to turn it off.** The yellow light should go out.

You can verify the mode by checking `dmesg` after plugging in USB:

```bash
# BAD - mass storage mode (Editor Lite is ON):
# scsi: Direct-Access Brother QL-810W

# GOOD - printer mode (Editor Lite is OFF):
# usblp: USB Bidirectional printer
```

## Step 3: Fix AppArmor (Ubuntu)

CUPS may be blocked from accessing USB by AppArmor. Check with:

```bash
sudo dmesg | grep -i apparmor | grep cups
```

If you see `DENIED` entries, add the required capability:

```bash
echo "capability net_admin," | sudo tee /etc/apparmor.d/local/usr.sbin.cupsd
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.cupsd
sudo systemctl restart cups
```

## Step 4: Verify USB Detection

```bash
# Check the printer is detected as a printer (not mass storage)
lsusb | grep -i brother
sudo dmesg | tail -10
ls -l /dev/usb/lp*

# Check CUPS USB backend sees it
sudo /usr/lib/cups/backend/usb
```

You should see output like:
```
direct usb://Brother/QL-810W?serial=XXXXXXXXXXXX "Brother QL-810W" ...
```

## Step 5: Register the Printer with CUPS

```bash
sudo lpadmin -p QL810W -E \
  -v "usb://Brother/QL-810W?serial=YOUR_SERIAL_HERE" \
  -P /usr/share/cups/model/Brother/brother_ql810w_printer_en.ppd
```

Replace `YOUR_SERIAL_HERE` with your printer's serial from the USB backend output.

Verify:
```bash
lpstat -p QL810W
```

## Step 6: Print (Standard Single-Color Tapes)

For standard DK tapes (single-color black), you can print through CUPS:

```bash
echo "Hello World" | lp -d QL810W -o media=62X1
```

### Common Media Options

| Option | Tape Type |
|--------|-----------|
| `29x90` | 29mm x 90mm die-cut labels |
| `62x29` | 62mm x 29mm die-cut labels |
| `62x100` | 62mm x 100mm die-cut labels |
| `62X1` | 62mm continuous roll |
| `29X1` | 29mm continuous roll |

Set the media to match the tape loaded in your printer. Full list available in the PPD file.

## Two-Color Printing (DK-2251)

The official Brother Linux driver **does not support two-color (red/black) DK-2251 tape**. The `rastertobrpt1` binary has no two-color handling — it sends single-color raster data, and the printer rejects it with a red flashing power light.

### Solution: brother_ql

Use the `brother_ql` Python library, which bypasses CUPS and talks directly to the printer. It fully supports two-color printing on the QL-810W.

#### Install

```bash
pipx install brother_ql
```

Or with pip in a venv:
```bash
python3 -m venv ~/brother_ql_env
source ~/brother_ql_env/bin/activate
pip install brother_ql
```

#### Discover the Printer

```bash
brother_ql -b pyusb discover
```

Output:
```
usb://0x04f9:0x209c
```

#### Print an Image (Two-Color)

```bash
sudo $(which brother_ql) -b pyusb -m QL-810W -p usb://0x04f9:0x209c \
  print -l 62red --red /path/to/image.png
```

- `-l 62red` tells it you have 62mm two-color tape loaded
- `--red` enables two-color mode
- Red pixels in the source image print in red; everything else prints in black
- `sudo` is required for USB access

#### Print a PDF (Two-Color)

Convert the PDF to an image first, resize to 696px wide (62mm at 300dpi), then print:

```bash
pdftoppm -r 300 -png yourfile.pdf /tmp/label_page
convert /tmp/label_page-1.png -resize 696x /tmp/label_resized.png
sudo $(which brother_ql) -b pyusb -m QL-810W -p usb://0x04f9:0x209c \
  print -l 62red --red /tmp/label_resized.png
```

#### Print an Image (Single-Color, 62mm)

```bash
sudo $(which brother_ql) -b pyusb -m QL-810W -p usb://0x04f9:0x209c \
  print -l 62 /path/to/image.png
```

### brother_ql Label Sizes

| Label ID | Tape |
|----------|------|
| `62` | 62mm continuous (single-color) |
| `62red` | 62mm continuous (two-color, DK-2251) |
| `29` | 29mm continuous |
| `29x90` | 29mm x 90mm die-cut |
| `62x29` | 62mm x 29mm die-cut |
| `62x100` | 62mm x 100mm die-cut |

Full list: `brother_ql info labels`

## Troubleshooting

### Printer shows as USB mass storage device
Editor Lite mode is on. Press the Editor Lite button on the printer to turn it off (yellow light should go out). Unplug and replug USB.

### Red flashing power light
Media mismatch. The paper size setting doesn't match the tape loaded. Make sure `-l` (brother_ql) or `-o media=` (CUPS) matches your tape. If using DK-2251 two-color tape, you **must** use `brother_ql` with `-l 62red --red`.

### "Waiting for printer to become available"
USB connection lost. Unplug and replug the USB cable. Verify with `ls -l /dev/usb/lp*`.

### AppArmor DENIED errors in dmesg
See Step 3 above.

### brother_ql crashes with `ANTIALIAS` error
The image is too large and the installed Pillow version removed `ANTIALIAS`. Pre-resize your image to the correct width before printing:
```bash
convert input.png -resize 696x resized.png  # 696px = 62mm at 300dpi
```

## Dependencies

- `cups` - printing system
- `pipx` - for installing brother_ql
- `brother_ql` - direct USB printing with two-color support
- `imagemagick` - image resizing (`convert`)
- `poppler-utils` - PDF to image conversion (`pdftoppm`)
- Brother LPD driver (`ql810wpdrv-3.1.5-0.i386.deb`) - for CUPS single-color printing

## License

This guide is provided as-is. The Brother driver source code is GPL v2. The `brother_ql` library is GPL v3.
