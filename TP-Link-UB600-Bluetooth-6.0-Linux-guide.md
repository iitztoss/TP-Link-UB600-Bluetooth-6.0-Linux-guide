# TP-Link UB600 Bluetooth 6.0 on Linux — Setup Guide

A fully reproducible procedure to get the TP-Link UB600 (Realtek RTL8761BU)
Bluetooth 6.0 USB adapter working on Linux with the real BT 6.0 firmware.
Validated on Ubuntu 24.04 HWE, kernel `7.0.0-31-generic`.

---

## Hardware facts

- Bluetooth adapter: TP-Link UB600, USB `37ad:0600` ("TP-Link Bluetooth USB Adapter")
- Chip: Realtek RTL8761BU (HCI manufacturer code 93 / 0x5d)
- `linux-firmware`'s stock `rtl8761bu_fw.bin` only enables BT 5.1 (`hci_ver=0x0a`)
- The BT 6.0 blob ships only inside TP-Link's Windows driver and is NOT redistributable
  (licensing) — extract it from your own driver download
- Kernel fixes: `btusb` entry for `37ad:0600` is upstream since v7.2-rc1 (commit `bc597f0`).
  On <7.2 the patched DKMS module below is required.

## Files / artifacts involved

| Path | Purpose |
|------|---------|
| `/usr/src/btusb-ub600-1.0/` | DKMS source: patched `btusb.c` (+ `btintel.h`, `btbcm.h`, `btrtl.h`, `btmtk.h`, `Makefile`, `dkms.conf`) |
| DKMS module `btusb-ub600/1.0` | Installed to `/lib/modules/<ver>/updates/dkms/` so it wins over the stock module and auto-rebuilds on kernel updates |
| `/lib/firmware/rtl_bt/rtl8761bu_fw.bin.zst` | BT 6.0 firmware blob (zstd, from Windows driver) |
| `/lib/firmware/rtl_bt/rtl8761bu_fw.bin.zst.bak-5.1` | Original stock 5.1 firmware backup |
| `/usr/local/share/tp-link-ub600/rtl8761b_mp_chip_bt40_fw_asic_rom_patch_new` | Permanent stash of the raw (uncompressed) BT6.0 blob; never deleted — used by the restore script |
| `restore-ub600-bt6.sh` (in your working directory) | Restore script to re-apply BT6.0 firmware after a `linux-firmware` update |

## Common issues

1. **rfkill soft-block** — symptom: adapter detected but `Failed to set power on` /
   `Failed to set mode: Failed (0x03)`. Fix: `sudo rfkill unblock bluetooth`.
2. **Stock boot before module load** — if Bluetooth is dead after a boot/kernel update,
   check `modinfo btusb | grep filename`; if not under `updates/dkms/`, run
   `sudo dkms autoinstall` and `sudo modprobe -r btusb && sudo modprobe btusb`.
3. **`linux-firmware` apt updates** overwrite `rtl8761bu_fw.bin.zst`, silently reverting
   to BT 5.1. After any `apt`/`linux-firmware` upgrade, redo the firmware swap (Step 6)
   and reload `btusb` — or, better, run the restore script (Step 9).
4. Secure Boot: DKMS must re-sign on build. The existing MOK auto-signs during
   the DKMS build; otherwise sign the module / enroll a MOK.

---

# Reproducible procedure

Target: `/usr/src/btusb-ub600-1.0/` + firmware swap.

## Step 1 — Diagnose

```bash
lsusb | grep -i tp-link          # ID 37ad:0600 TP-Link Bluetooth USB Adapter
rfkill list                      # if soft blocked: sudo rfkill unblock bluetooth
sudo dmesg | grep -iE "RTL:|rtl8761"
```

Bug signature: `hci0` exists and powers on, but **no `RTL:` lines** in `dmesg`
(firmware never uploaded) and scanning finds nothing.

## Step 2 — Fetch kernel sources (pre-7.2 kernels only)

```bash
sudo apt install -y dkms build-essential curl unzip zstd linux-headers-$(uname -r)
mkdir -p /usr/src/btusb-ub600-1.0 && cd /usr/src/btusb-ub600-1.0
VER="v$(uname -r | cut -d'.' -f1,2)"        # e.g. v7.0
BASE='https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth'
for f in btusb.c btintel.h btbcm.h btrtl.h btmtk.h; do
  curl -fsSL -o "$f" "$BASE/$f?h=$VER"
done
```

## Step 3 — Patch `btusb.c`

In the Realtek block after `/ 8922AE Bluetooth devices /`, add:

```c
	/* TP-Link UB600 (RTL8761BU under TP-Link VID) */
	{ USB_DEVICE(0x37ad, 0x0600), .driver_info = BTUSB_REALTEK |
						     BTUSB_WIDEBAND_SPEECH },
```

## Step 4 — DKMS glue

`Makefile`:
```make
obj-m := btusb.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

`dkms.conf`:
```
PACKAGE_NAME="btusb-ub600"
PACKAGE_VERSION="1.0"
CLEAN="make clean"
MAKE="make"
BUILT_MODULE_NAME[0]="btusb"
BUILT_MODULE_LOCATION[0]="."
DEST_MODULE_LOCATION[0]="/updates/dkms"
REMAKE_INITRD="no"
AUTOINSTALL="yes"
```

## Step 5 — Build, install, load

```bash
sudo dkms add /usr/src/btusb-ub600-1.0
sudo dkms build -m btusb-ub600 -v 1.0
sudo dkms install -m btusb-ub600 -v 1.0
modinfo btusb | grep filename     # must show .../updates/dkms/btusb.ko.zst
sudo modprobe -r btusb; sleep 2; sudo modprobe btusb; sleep 3
sudo dmesg | grep -i RTL
```

Expected: `RTL: loading rtl_bt/rtl8761bu_fw.bin` ... `fw version 0xdfc6d922`.
Secure Boot error on load => sign module / enroll MOK.

## Step 6 — Upgrade firmware to real BT 6.0

```bash
cd /tmp
curl -fsSL -o UB600_Win.zip \
  "https://static.tp-link.com/upload/driver/2025/202512/20251203/UB600_V1_2.11.3032.3001_Win7_Win81_Win10_Win11.zip"
unzip -q UB600_Win.zip -d UB600_Win
BLOB="UB600_Win/BT_Driver/Win10X64_New/rtl8761b_mp_chip_bt40_fw_asic_rom_patch_new"
ls -l "$BLOB"                    # MUST be exactly 29215 bytes
sudo mkdir -p /usr/local/share/tp-link-ub600
sudo cp "$BLOB" /usr/local/share/tp-link-ub600/   # persistent stash for the restore script
sudo cp /lib/firmware/rtl_bt/rtl8761bu_fw.bin.zst /lib/firmware/rtl_bt/rtl8761bu_fw.bin.zst.bak-5.1
zstd -19 -q -c "$BLOB" | sudo tee /lib/firmware/rtl_bt/rtl8761bu_fw.bin.zst >/dev/null
sudo modprobe -r btusb; sleep 2; sudo modprobe btusb; sleep 3
sudo dmesg | grep "RTL: fw version"     # expect: fw version 0x2afb4be5
```

## Step 7 — Verify

```bash
sudo rfkill unblock bluetooth
hciconfig hci0 up
hciconfig hci0 version        # HCI Version (0xe) = Bluetooth 6.0
bluetoothctl scan on
```

Success = `HCI Version (0xe)`, `lmp_subver=4be5`, `fw version 0x2afb4be5`,
and `bluetoothctl devices` lists nearby devices.

## Step 8 — Clean up temporary files

Remove the downloaded driver package and extraction; keep the persistent stuff:

```bash
rm -f /tmp/UB600_Win.zip
rm -rf /tmp/UB600_Win
```

**Do NOT delete** (needed to keep the fix working):
- `/usr/src/btusb-ub600-1.0/` — DKMS source; `sudo dkms remove` first if you ever want to undo the module patch
- `/usr/local/share/tp-link-ub600/` — permanent BT6.0 blob stash used by the restore script
- `/lib/firmware/rtl_bt/rtl8761bu_fw.bin.zst` and its `.bak-5.1` backup

## Step 9 — Restore script (recovery after `linux-firmware` updates)

An `apt` upgrade of `linux-firmware` silently overwrites `rtl8761bu_fw.bin.zst` back to
BT 5.1. Save the following script as `restore-ub600-bt6.sh` **on the Desktop**
(`~/Desktop/restore-ub600-bt6.sh`), then make it executable and **owned by your
current user** (the exact username returned by `whoami`):

```bash
cd ~/Desktop
whoami                                 # OWNER=$(whoami)
chmod +x restore-ub600-bt6.sh
chown "$(whoami):$(whoami)" restore-ub600-bt6.sh
```

`restore-ub600-bt6.sh`:

```bash
#!/usr/bin/env bash
# Restore TP-Link UB600 Bluetooth 6.0 firmware on Linux.
# Run:  sudo ~/Desktop/restore-ub600-bt6.sh
set -euo pipefail

BLOB_RAW="/usr/local/share/tp-link-ub600/rtl8761b_mp_chip_bt40_fw_asic_rom_patch_new"
FW_DIR="/lib/firmware/rtl_bt"
FW_FILE="rtl8761bu_fw.bin.zst"
BAK_FILE="rtl8761bu_fw.bin.zst.bak-5.1"
EXPECTED_SIZE="29215"

if [ "$(id -u)" -ne 0 ]; then
    echo "This script must run as root. Re-run with:  sudo $0"
    exit 1
fi
echo "==> TP-Link UB600 firmware restore (BT 6.0)"

# 1. Ensure the BT6.0 blob is available (source of truth).
if [ ! -f "$BLOB_RAW" ]; then
    CAND="/tmp/UB600_Win/BT_Driver/Win10X64_New/rtl8761b_mp_chip_bt40_fw_asic_rom_patch_new"
    if [ -f "$CAND" ]; then
        mkdir -p "$(dirname "$BLOB_RAW")"
        cp "$CAND" "$BLOB_RAW"
        echo "   recovered blob from $CAND"
    else
        echo "ERROR: BT6.0 blob not found at $BLOB_RAW."
        echo "       Download the TP-Link UB600 Windows driver (see Step 6),"
        echo "       extract it to /tmp/UB600_Win, and re-run."
        exit 1
    fi
fi
SIZE=$(stat -c %s "$BLOB_RAW")
echo "   blob: $BLOB_RAW ($SIZE bytes)"
if [ "$SIZE" != "$EXPECTED_SIZE" ]; then
    echo "ERROR: expected $EXPECTED_SIZE bytes, got $SIZE. Aborting."
    exit 1
fi

# 2. One-time backup of the currently installed firmware.
if [ -f "$FW_DIR/$FW_FILE" ] && [ ! -f "$FW_DIR/$BAK_FILE" ]; then
    cp "$FW_DIR/$FW_FILE" "$FW_DIR/$BAK_FILE"
    echo "   saved backup: $FW_DIR/$BAK_FILE"
fi

# 3. (Re)install the BT6.0 blob (zstd-compressed).
if [ -f "$FW_DIR/$FW_FILE" ] && cmp -s <(zstd -q -d -c "$FW_DIR/$FW_FILE") "$BLOB_RAW"; then
    echo "   firmware already up to date: $FW_DIR/$FW_FILE"
else
    zstd -19 -q -c "$BLOB_RAW" > "$FW_DIR/$FW_FILE"
    echo "   installed BT6.0 firmware -> $FW_DIR/$FW_FILE"
fi

# 4. Ensure the DKMS-patched btusb module is active.
if ! modinfo btusb 2>/dev/null | grep -q "updates/dkms"; then
    echo "   btusb is not the DKMS build; running dkms autoinstall..."
    dkms autoinstall >/dev/null 2>&1 || echo "   warning: dkms autoinstall failed"
fi
modprobe -r btusb 2>/dev/null || true
sleep 2
modprobe btusb
sleep 3

# 5. Power on and verify.
rfkill unblock bluetooth 2>/dev/null || true
sleep 1
hciconfig hci0 up 2>/dev/null || true
echo "==> Done. Expect 'HCI Version: (0xe)':"
hciconfig hci0 version | grep -E "HCI Version|LMP Version|Manufacturer"
```

Run it whenever Bluetooth went back to 5.1 (after a `linux-firmware` upgrade) or just
preemptively after any `apt upgrade`:

```bash
sudo ~/Desktop/restore-ub600-bt6.sh
```

The script is idempotent (safe to re-run), and if the permanent stash was deleted it
recovers the blob from `/tmp/UB600_Win` as a fallback.

## When to skip the module patch

On kernel >= 7.2 (upstream fix present): skip Steps 1–5, only do the Step 6 firmware
swap. Re-check after upgrades: `hciconfig hci0 version` should still show `(0xe)`.
