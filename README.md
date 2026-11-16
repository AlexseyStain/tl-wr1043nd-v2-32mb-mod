# TP-Link TL-WR1043ND v2 — 32 MB flash mod for OpenWrt

OpenWrt 24.10 on 8 MB of flash is a tight fit: kernel ~2.7 MB + rootfs ~4 MB leaves
almost nothing. This mod swaps the stock W25Q64 (8 MB) for a W25Q256 (32 MB) and
runs OpenWrt with a **27.4 MB writable overlay**, native MAC and original Wi-Fi
calibration preserved.

| | Stock | This mod |
|---|---|---|
| Flash | 8 MB (W25Q64) | 32 MB (W25Q256) |
| Overlay | ~5 MB | 27.4 MB |
| Bootloader | stock u-boot | stock u-boot (from original dump) |
| MAC / Wi-Fi cal | preserved | preserved |
| OpenWrt | 24.10 / 25.12 | **24.10 recommended** (see 25.12 status below) |

> **Disclaimer:** you desolder and reprogram at your own risk. A full programmer
dump of the original flash before you start makes this close to risk-free.

## Hardware

- TP-Link TL-WR1043ND v2 — QCA9558 SoC, 64 MB RAM, SOIC-8 SPI flash.
- Replacement: Winbond **W25Q256** (32 MB). Any package works (SOIC-8 or the
  16-pin `FVFG` variant via adapter). Buy from a reputable seller.
- Programmer: CH341A / MiniPro / Xgpro — anything that speaks SPI and verifies.
- Serial (UART 115200 8N1, 3.3 V) strongly recommended for the first boot.

Carry over from the original flash:
1. **ART** — last 64 KB, radio calibration for the internal QCA9558 Wi-Fi.
2. **MAC** — *not* in ART on these TP-Links. It sits at **0x1FC00** inside the
   u-boot region (`macaddr@1fc00` in the DTS). Keep the stock u-boot block
   byte-identical and the MAC takes care of itself.

## Partition map (32 MB)

One DTS edit in `qca9558_tplink_tl-wr1043nd.dtsi`:

```dts
partition@20000 {
    compatible = "tplink,firmware";
    label = "firmware";
    reg = <0x020000 0x1fd0000>;   // stock: <0x020000 0x7d0000>
};

partition@1ff0000 {
    label = "art";
    reg = <0x1ff0000 0x010000>;   // stock: partition@7f0000
    read-only;
};
```

## Building OpenWrt

Tested on `openwrt-24.10` (clean host build). Device block
(`target/linux/ath79/image/generic-tp-link.mk`):

```makefile
define Device/tplink_tl-wr1043nd-v2
  $(Device/tplink-8mlzma)
  SOC := qca9558
  DEVICE_MODEL := TL-WR1043N/ND
  DEVICE_VARIANT := v2
  DEVICE_PACKAGES := kmod-usb2 kmod-usb-ledtrig-usbport
  TPLINK_HWID := 0x10430002
  TPLINK_FLASHLAYOUT := 32Mlzma
  IMAGE_SIZE := 31744k
  KERNEL_INITRAMFS := kernel-bin | append-dtb   # uncompressed rescue kernel
  SUPPORTED_DEVICES += tl-wr1043nd-v2
endef
```

`32Mlzma` does not exist in `mktplinkfw` — add it via the tool's src/ override
(`tools/firmware-utils/src/src/mktplinkfw.c`, copied over pristine sources at
prepare time; anchor on the terminating table entry):

```c
}, {
    .id             = "32Mlzma",
    .fw_max_len     = 0x1f00000,   // 31744k; partition is 0x1fd0000, slack like stock
    .kernel_la      = 0x80060000,
    .kernel_ep      = 0x80060000,
    .rootfs_ofs     = 0x100000,    // unused: tplink-v1-image passes -a
}, {
    /* terminating entry */
```

`make menuconfig` → ath79/Generic → TL-WR1043N/ND v2 → build. Verify with
`staging_dir/host/bin/mktplinkfw -i bin/targets/.../*-sysupgrade.bin`:
V1 header, HW ID `0x10430002`, kernel at 0x200, squashfs right behind.

## Why stock u-boot (and not breed)

breed r1337 (`breed-qca9558-wr1043ndv2.bin`, 2021 — newest official build for this
board; the project is abandoned) proved unusable on this board/chip combination:

- The autoloader answers `Error: Unrecognised firmware header!` for **every**
  TP-Link image format — V1, both V2 header flavours, V3 — valid MD5s,
  byte-correct placement.
- `boot linux <addr>` is a raw jump: LZMA kernels never start; only
  **uncompressed** kernels already in RAM boot.
- `mem copy` silently performs no copy; `flash read` returns buffers whose CRC
  mismatches the source; the web flasher wrote one image shifted by 1 byte.
- What does work in breed: `wget 0x80060000 http://<host>/vmlinux.bin`
  (**address first!**) then `boot linux 0x80060000` — full OpenWrt initramfs from
  RAM; also the escape hatch if the flash side dies.

Stock TP-Link u-boot (Sep 2015, ap135) parses V1 headers and decompresses LZMA
kernels natively — the path OpenWrt factory images were designed for. Taken
straight from the original flash dump, MAC block intact, it just works.

## Assembling the 32 MB image

```text
[0x000000] u-boot   128 KB   stock, from the original dump (MAC @ 0x1FC00 inside)
[0x020000] firmware          OpenWrt *-sysupgrade.bin
[0x1FF0000] art    64 KB     from the original dump
[everything else] 0xFF
```

```bash
python3 - <<'PYEOF'
dump  = open('original-flash-dump-8mb.bin','rb').read()
sysup = open('sysupgrade-openwrt-24.10.bin','rb').read()
img = bytearray(b'\xff' * 0x2000000)
img[0x00000:0x20000]     = dump[0x00000:0x20000]
img[0x020000:0x020000+len(sysup)] = sysup
img[0x1FF0000:0x2000000] = dump[0x7F0000:0x800000]
open('fullflash-32mb.bin','wb').write(img)
PYEOF
```

Programmer: detect → **erase → program → verify (readback hash)** → solder.
First boot: u-boot banner, `Autobooting in 1 seconds`, kernel, 27.4 MB overlay.

## OpenWrt 25.12 status

25.12 builds and runs **except Wi-Fi**: the internal QCA9558 WMAC fails probe with
`ath: phy0: Unable to initialize hardware; initialization status: -5` (reset never
completes). Tracked as openwrt/openwrt#23805 (Meraki MR18, same SoC). The 25.12.6
backport *mac80211: ath9k: call reset on init* fixed MR18 per forum reports but
**not** this board — hence 24.10 is the recommended image. All mod components
(DTS, layouts, fullflash) are identical between branches.

Build-host note for 25.12: on gcc-15/glibc-2.42 hosts, tools/elfutils (0.192)
fails inside gnulib; take elfutils 0.196's Makefile from origin/master plus small
gnulib shims. 24.10 builds clean.

## Known quirks

- Soft `reboot` hangs in u-boot (SGMII calibration completes only from a cold
  start). Power-cycle instead.
- Stock u-boot prints `flash size 8MB, sector count = 128` for the W25Q256 —
  no 32 MB entry in its JEDEC table. Cosmetic (it only reads the kernel region).
  Never flash through u-boot beyond 8 MB; use `sysupgrade` from inside the OS.

## Files

See SHA256SUMS. `*-fullflash.bin` = whole 32 MB chip (programmer).
`*-sysupgrade*.bin` = in-OS upgrades. `original-flash-dump-8mb.bin` = untouched
stock dump (contains your stock u-boot + ART + MAC region — restore = back to
factory).

## Recovery

Worst case: programmer + original dump restore the router to stock in minutes.

---

*spider_web* - donations: bc1qjl7hq0s5vv7ypvj3n8w3g76q87lepa63tkxum7
