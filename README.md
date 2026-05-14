# c60-kernel-patches

Mainline Linux 6.6 patches for the Poly Trio C60 video conferencing
phone (i.MX 8M Mini, codename **Kepler proto1**). Enables CPU, eMMC,
networking (via the same RTL8363NB-VB DSA switch as TC8), audio, and
console on a vanilla upstream tree.

Sister project to [`tc8-kernel-patches`](../tc8-kernel-patches/). Both
boards share the same i.MX 8M Mini SoC and a lot of peripheral routing,
so the TC8 patches contribute most of the cross-board drivers
(RTL8363NB-VB DSA, FEC fixed-PHY conduit, TAS5751M codec). This repo
adds only the C60-specific bits: the `imx8mm-kepler-proto1` DTS and
its Makefile registration.

## Apply

Apply TC8 patches first (for the shared drivers), then C60 on top:

```bash
git clone --branch v6.6 --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git am ../tc8-kernel-patches/patches/*.patch
git am ../c60-kernel-patches/patches/*.patch
```

Both apply cleanly with `git am` against `v6.6`. The full firmware build
pipeline at [`c60-firmware-build`](../c60-firmware-build/) does this
for you and produces flashable `boot-c60.img` + `dtbo-c60.img` +
`vbmeta-c60.img` + `rootfs.img`.

## Patches

| # | Patch | Subsystem |
|---|---|---|
| 0001 | `arm64: dts: freescale: add imx8mm-kepler-proto1 board support` | arm64 / DT |

More patches will land as device-tree growth uncovers chips that need
new or modified mainline drivers.

## Hardware enabled by 0001

- Console (`ttymxc1` @ 115200, ec_imx6q earlycon)
- eMMC (USDHC3, 8-bit HS400)
- BD71837 PMIC + power key (`/dev/input/event*`)
- FEC + RTL8363NB-VB DSA switch (`lan@end0` user port, "pc" pass-through
  port stubbed) — wired up via TC8's `0003`+`0004` patches
- ARM PSCI, gic-v3, ARM-armv8 timer (mainline-clean)
- TAS5751M audio codec on SAI1 — bindings via TC8's `0005`

## Hardware stubbed (`status = "disabled"` in 0001 — for later iteration)

- TC358743 HDMI→MIPI-CSI bridge (`i2c-3` 0x1e)
- ADV7535 HDMI-out (`i2c-3` 0x3d) — stock probe fails on proto1, may
  not be populated
- Raydium RM67191 MIPI-DSI panel — stock probe fails on proto1
- Focaltech FTS touch (`i2c-1` 0x38)
- ADC3101 mic codec on SAI1
- LP5569 LEDs on i2c-1/2/3 (proto1 returns -ENODEV — likely de-populated)
- kepler_cap touch-button ICs on i2c-1/2/3 (proto1 init fails)
- IR receiver, digipyro
- brcmfmac WiFi/BT on PCIe — chip ID not yet pinned down on mainline

## Compatible string

The root node compatible is:
```
compatible = "poly,kepler-proto1", "fsl,imx8mm";
```

`poly,kepler-proto1` is a new entry — pre-existing Polycom vendor prefix
not used in upstream, may need a binding doc entry before posting any of
this upstream.

## Boot path

This board runs with stock Polycom u-boot 2018.03 (unchanged — secure
boot fuses are blown, u-boot is immutable). The TC8 `slotbboot` raw-mmc
trick does NOT work on C60: stock u-boot autoboots with no input poll
window. Boot uses Android `boota` + AVB orange (testkey) on slot A;
see [`c60-firmware-build/BUILDING.md`](../c60-firmware-build/) for
the flash recipe. Slot B holds stock Polycom Android as recovery.

## License

GPL-2.0-or-later (matches the Linux kernel).
