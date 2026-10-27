# OnePlus 9 Pro WCD938x SoundWire notes

## Scope and evidence

This checkout already contains the Qualcomm downstream WCD938x split-codec and
SoundWire slave drivers. The changes in this patch fix error handling and
debug-only register access; they do not add a new codec, SoundWire master, or
board wiring. The source checkout reviewed was
[`3495395e7cfdabd36b73f99c3bdec2418000cc86`](https://github.com/alecsandru90/Levion_kernel_OP9/tree/3495395e7cfdabd36b73f99c3bdec2418000cc86).

The board facts below are from this repository's device tree and build
configuration:

* `arch/arm64/boot/dts/vendor/oplus/lemonadev/lemonadep-19815-overlay.dts`
  identifies `lemonadep` and includes `lemonade_common.dtsi`; the T0 overlay
  uses the same shared audio file.
* `arch/arm64/boot/dts/vendor/qcom/lahaina-mtp.dtsi` includes
  `lahaina-audio-overlay.dtsi`. That file defines the WCD938x codec, split RX
  and TX SoundWire slave nodes, and their channel maps. The RX slave is under
  `swr1` and the TX slave under `swr2`; both use
  `compatible = "qcom,wcd938x-slave"`.
  The RX master is `rx_macro@3200000` / `swr1` (master ID 2, six ports,
  HCTL `0x032a90a0`, I/O base `0x3210000`); the TX master is
  `tx_macro@3220000` / `swr2` (master ID 3, three ports, HCTL
  `0x032a90a8`, I/O base `0x3230000`). The WCD RX and TX nodes have the
  DT `reg` values `<0x0D 0x01170224>` and `<0x0D 0x01170223>`,
  respectively. These are recorded from the vendor DT, not guessed device
  addresses.
* The WCD reset node selects `wcd938x_reset_active` / `wcd938x_reset_sleep`.
  `lahaina-pinctrl.dtsi` assigns GPIO 88, high in the active state and low in
  sleep. RX/TX macro clocks are defined in `lahaina-audio-overlay.dtsi` at
  22.5792 MHz (RX) and 19.2 MHz (TX); the SoundWire masters also reference
  LPASS core/audio hardware-vote clocks. The codec's static supplies there
  reference S10B for RX/TX, I/O, and buck rails at 1.8 V and BOB for the
  microphone-bias supply at 3.296 V. `lemonade_common.dtsi` overrides
  microphone-bias 2 to 2.7 V. This patch changes none of these settings.
  The compatible identifies the WCD938x family; the board DT alone does not
  specify a part revision or register map.
* `lemonade_common.dtsi` disables the WSA SoundWire master, WSA883x devices,
  SoundWire DMIC devices, and SoundWire haptics. It enables two `nxp,tfa98xx`
  I2C devices at addresses `0x34` and `0x35`, each with an S10B supply and
  reset GPIOs 15 and 42. The sound-card node selects
  `oplus,speaker-pa = "nxp"`.
  Do not treat the WSA or SoundWire-DMIC definitions in the Lahaina base tree
  as enabled lemonadep hardware.
* The shared sound-card routing lists AMIC1-5 and their mic-bias routes. It
  enables USB-C headset audio through MBHC. This is distinct from USB digital
  audio over the USB data interface, which is outside this WCD/MBHC route and
  is not established by this audio DT. Neither is a reason to route speaker
  playback through a WSA SoundWire speaker.
* `techpack/audio/config/lahainaauto.conf` enables both
  `CONFIG_SND_SOC_WCD938X` and `CONFIG_SND_SOC_WCD938X_SLAVE`; the
  `wcd938x/Kbuild` file builds the slave object into
  `wcd938x_slave_dlkm.ko`.

The matching public OnePlus source repository is
[android_kernel_oneplus_sm8350](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8350).
The public `oneplus/SM8350_R_11.0` ref resolved to commit
[`13ca08fdf0979fdd61d5e8991661874bb2d19150`](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8350/commit/13ca08fdf0979fdd61d5e8991661874bb2d19150),
a December 2021 release synchronization for OxygenOS 11. It is an historical
vendor-source reference, not evidence that the current Android/firmware bundle
uses the same audio configuration. Its pinned
[`lahaina-audio-overlay.dtsi`](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8350/blob/13ca08fdf0979fdd61d5e8991661874bb2d19150/arch/arm64/boot/dts/vendor/qcom/lahaina-audio-overlay.dtsi)
confirms the separate RX/TX SoundWire nodes and the `wcd938x-slave`
compatible. This vendor DT evidence is used to confirm integration shape, not
to infer undocumented registers or override the target's shared Oplus DTS.

The
[upstream WCD938x SoundWire implementation](https://github.com/torvalds/linux/blob/83f32c773255720a661aa45d83e36278135f23f5/sound/soc/codecs/wcd938x-sdw.c)
at Linux commit `83f32c773255720a661aa45d83e36278135f23f5` is a comparison
reference only. It uses mainline SoundWire interfaces, unlike this kernel's
downstream `soc/soundwire.h` API. No upstream register sequence or API was
transplanted into this driver.

Public documentation limits:

* [Qualcomm WCD9385 product information](https://www.qualcomm.com/audio/applications/compute-and-mobile-audio/products/wcd9385)
  describes product capabilities, not a complete register-programming
  specification.
* Qualcomm's public
  [QRB5165 data sheet, 80-PV086-1 Rev. E](https://docs.qualcomm.com/doc/80-PV086-1/80-PV086-1_REV_E_QRB5165_Data_Sheet.pdf)
  points to WCD9380/WCD9385 Device Specification 80-PL335-1. QRB5165 is not
  the OnePlus 9 Pro SoC, and that pointer is not evidence for this phone's
  wiring. This work has not read the restricted 80-PL335-1 document and does
  not infer register values or power sequences from it.
* The [MIPI SoundWire overview](https://www.mipi.org/specifications/soundwire)
  is a protocol overview; the full specification is member-only. The
  [Linux SoundWire driver documentation](https://docs.kernel.org/driver-api/soundwire/index.html)
  is general API reference, not target-board evidence.

## Build and static checks

Use the repository's Lahaina configuration and matching LLVM/aarch64 toolchain:

```sh
make O=out ARCH=arm64 LLVM=1 vendor/lahaina-qgki_defconfig
make -j"$(nproc)" O=out ARCH=arm64 LLVM=1 CC=clang LD=ld.lld \
     CROSS_COMPILE=aarch64-linux-gnu-
```

The driver's module configuration is also exported by
`techpack/audio/config/lahainaauto.conf`. Run `scripts/checkpatch.pl` on driver
changes and build the kernel/module with the exact downstream toolchain. There
is no WCD938x-specific unit-test harness in this tree; the debug register
interface requires SoundWire hardware to validate transfer behavior.

## Device test checklist

Do not flash automatically. Preserve the existing amplifier protection,
calibration, thermal/current limits, DSP profiles, and mixer controls. Start
at low volume and test on the exact phone/ROM combination:

1. Check RX and TX slave enumeration on `swr1` and `swr2`; capture `dmesg`
   around codec probe, SoundWire enumeration, and any ADSP/SSR events.
2. Verify low-volume headphone playback and capture through each intended
   analog microphone route; check left/right routing and headset detection.
3. Verify built-in speaker playback through the enabled NXP amps using the
   matching vendor firmware, calibration, HAL, and mixer configuration.
4. Test USB-C headset detection/audio separately from USB digital audio.
5. Test stream stop/restart and system suspend/resume, then check for stale
   enumeration, timeouts, or failed register transfers.
6. If supported by the ROM, exercise ADSP subsystem restart (SSR) recovery and
   repeat enumeration and stream tests.

Record the kernel/ROM build, codec and SoundWire logs, `dmesg`, and relevant
`logcat` audio/HAL errors. A successful compile or static check is not hardware
validation; no physical-device test is claimed here.
