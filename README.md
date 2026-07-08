# c60-kernel-patches

> **Superseded.** The C60 series now lives in the converged
> [poly-kernel-patches](https://github.com/Polycom-Open-Firmware/poly-kernel-patches)
> as `patches/c60/`, consumed by `poly-firmware-build --target=c60`. This
> repo is kept as the C60 bring-up reference.

Mainline Linux 6.6 patches for the Poly Trio C60 video conferencing
phone (i.MX 8M Mini, codename **Kepler proto1**). Enables CPU, eMMC,
networking (RTL8363NB-VB DSA switch on a FEC fixed-PHY conduit), audio,
and console on a vanilla upstream tree.

The series is self-contained: it carries the C60 board device tree and
every out-of-tree driver that DT binds. The shared i.MX 8M Mini
peripheral drivers (RTL8363NB-VB DSA, FEC fixed-PHY conduit, TAS5751M
codec) also serve the sister TC8 panel (`poly-kernel-patches`, `patches/tc8/`), whose
series layout this one mirrors; the C60-specific bits are the
`imx8mm-kepler-proto1` DTS and its Makefile registration, plus
tlv320adc3xxx secondary-codec support for the C60's multi-mic TDM
topology.

## Apply

```bash
git clone --branch v6.6 --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git am ../c60-kernel-patches/patches/*.patch
```

Applies cleanly with `git am` against `v6.6`. The `c60-firmware-build`
pipeline runs this apply step for you and produces the flashable slot-A
image set: `boot.img` (Android boot.img v0, kernel + DTB in `second`),
`dtbo.img`, `vbmeta.img`, and a Debian rootfs image for `system_a`.

## Patches

| # | Patch | Subsystem |
|---|---|---|
| 0001 | `arm64: dts: freescale: add imx8mm-kepler-proto1 board support` | arm64 / DT |
| 0002 | `net: dsa: realtek: add RTL8363NB-VB switch support` | net/dsa |
| 0003 | `net: fec: support fixed-phy DSA conduit on i.MX8MM` | net/fec |
| 0004 | `ASoC: tas571x: add TAS5751M support` | ASoC |
| 0005 | `ASoC: tlv320adc3xxx: implement set_tdm_slot for multi-codec TDM` | ASoC |
| 0006 | `ASoC: tlv320adc3xxx: add -secondary compatible for clock-consumer mode` | ASoC |
| 0007 | `ASoC: dt-bindings: tlv320adc3xxx: document -secondary` | dt-bindings |

Patches 0002-0004 add the out-of-tree i.MX 8M Mini drivers the board DT
binds: RTL8363NB-VB DSA switch support, the FEC fixed-PHY DSA conduit,
and the TAS5751M codec. Patches 0005-0007 add secondary-codec support to
the mainline `tlv320adc3xxx` driver so three TLV320ADC3101 mics can share
one SAI in a TDM topology (one clock master, the rest clock consumers).
More patches will land as device-tree growth uncovers chips that need new
or modified mainline drivers.

## Hardware enabled by 0001

- Console (`ttymxc1` @ 115200, ec_imx6q earlycon)
- eMMC (USDHC3, 8-bit HS400)
- BD71837 PMIC + power key (`/dev/input/event*`)
- FEC + RTL8363NB-VB DSA switch (`lan@end0` user port, "pc" pass-through
  port stubbed) — wired up by `0002` (DSA switch) and `0003` (FEC conduit)
- ARM PSCI, gic-v3, ARM-armv8 timer (mainline-clean)
- TAS5751M audio codec on SAI1 — driver support from `0004`

## Hardware stubbed (`status = "disabled"` in 0001 — for later iteration)

- TC358743 HDMI→MIPI-CSI bridge (`i2c-3` 0x1e)
- ADV7535 HDMI-out (`i2c-3` 0x3d) — stock probe fails on proto1, may
  not be populated
- Raydium RM67191 MIPI-DSI panel — stock probe fails on proto1
- Focaltech FTS touch (`i2c-1` 0x38)
- ADC3101 mic codec on SAI1 — driver support added by 0005-0007; the
  DT nodes stay `disabled` in 0001 pending bring-up
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

The C60 is HAB-open, so it boots a RAM-loaded u-boot rather than any
on-eMMC bootloader. Mainline u-boot is loaded over the i.MX SDP
interface (`uuu -b spl flash.bin`) and comes up as a fastboot gadget. A
slot is booted by reading its Android boot.img v0 from eMMC
(`mmc read boot_a`; kernel plus DTB in the `second` area) and handing it
to `booti`, with AVB skipped. Slot B is left as the untouched stock
Android image. The native NXP `boota`/AVB path is compiled in but is not
on the critical path (its AVB expects an RPMB key that is not
provisioned). See the `c60-firmware-build` project for the image build
and flash flow.

## License

GPL-2.0-or-later (matches the Linux kernel).
