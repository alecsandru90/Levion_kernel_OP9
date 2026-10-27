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

The continuation audit and its checks are recorded below under
[PCM capabilities](#pcm-capabilities-continuation-audit).

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

## PCM capabilities (continuation audit)

### Baseline and evidence

This follow-up is PR #2 against `aox-backup-20261014`, after the merge of
[PR #1](https://github.com/alecsandru90/Levion_kernel_OP9/pull/1).
Commit `9c794139527401dae434f3682f70b19f629c1d15` is an ancestor; its
SoundWire debug-access fixes are retained unchanged. The audited baseline is
[`810b123984702c474d6b98a2b56671a5099db867`](https://github.com/alecsandru90/Levion_kernel_OP9/tree/810b123984702c474d6b98a2b56671a5099db867).
The older speaker-24-bit change is not in this baseline: both the tertiary
MI2S default and its Q6 DAI format mask are S16_LE. This continuation does
not change that default or claim to restore a verified speaker mode.

Additional pinned comparison sources:

* [Linux v6.12 WCD938x](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/sound/soc/codecs/wcd938x.c),
  rate/format definitions: integer rates through 192 kHz, fractional playback
  rates through 176.4 kHz and S16_LE/S24_LE. This is implementation comparison,
  not a replacement for downstream SoundWire programming.
* [OnePlus OxygenOS 12.1 C.61 source](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8350/tree/1e7b5673e7cc29d9af9fb0f58597055d07de6c6a/kernel/msm-5.4/techpack/audio):
  [WCD938x rate definitions and DAI declarations](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8350/blob/1e7b5673e7cc29d9af9fb0f58597055d07de6c6a/kernel/msm-5.4/techpack/audio/asoc/codecs/wcd938x/wcd938x.c),
  [Q6 MI2S hw_params/DAIs](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8350/blob/1e7b5673e7cc29d9af9fb0f58597055d07de6c6a/kernel/msm-5.4/techpack/audio/asoc/msm-dai-q6-v2.c), and
  [TFA formats, startup and hw_params](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8350/blob/1e7b5673e7cc29d9af9fb0f58597055d07de6c6a/kernel/msm-5.4/techpack/audio/asoc/codecs/tfa98xx-v6/tfa98xx_v6.c).
  The vendor also has the WCD mask/192 kHz ceiling discrepancy and S16-only
  tertiary MI2S advertisement; these are inherited, not missing vendor imports.
* The earlier pinned OxygenOS 11 source commit `13ca08fdf0979fdd61d5e8991661874bb2d19150`
  was reverified as the December 2021 OnePlus 9/9 Pro source release.
* The Qualcomm WCD9385 page above could not be retrieved in this audit.
  Reported product maxima (384 kHz playback, 192 kHz recording, and differing
  PCM-interface limits) are therefore **not used as enablement evidence**.
  No restricted register specification was obtained.

The NXP driver reads revision register `0x03` in `tfa98xx_i2c_probe()`.
Its low-byte cases identify 0x72/73/74 as TFA9872/73/74, 0x88 as TFA9888,
0x13 as TFA9912, 0x94 as TFA9894, 0x80/81 as TFA9890, 0x92 as TFA9891,
0x12 as TFA9895, and 0x97/96 as TFA9897/96. Neither the generic compatible
nor the `tfa9874_vdd-supply` property establishes the installed revision.
No actual revision readout is available, so neither the
[generic Tfa9xxx SDK](https://mcuxpresso.nxp.com/api_doc/dev/4667/a00009.html)
nor an unrelated TFA9879 datasheet justifies extending this route.

### Route matrix

All rates below are discrete, in kHz, not continuous ranges. Let **I** mean
8, 16, 32, 48, 96, 192 and **F** mean 44.1, 88.2, 176.4.
These are kernel component limits, not a promise of HAL exposure, firmware
support, bit-perfect output, or physical analogue resolution.

| Route/direction | Baseline declarations and programming | Result of this continuation |
| --- | --- | --- |
| WCD headphone/earpiece/aux playback | `wcd938x.c` advertised I+F+352.8+384, but `rate_max=192000`; S16_LE, S24_LE, S24_3LE, S32_LE. | Mask now I+F, matching the unchanged 192 kHz ceiling; formats unchanged. No new playback mode. |
| WCD analog-mic/headset capture | Same broad codec mask/formats as playback, but the linked TX macro advertises only I and S16_LE/S24_LE/S24_3LE. | WCD capture now declares that same integer-rate and format intersection. No fractional capture or S32_LE claim. |
| Bolero RX primary/mix interpolation | `rx-macro.c:sr_val_tbl` maps I+F+352.8+384 to existing register values; RX DAIs allow these rates and all four playback formats. Unknown rates return `-EINVAL`. | Unchanged: macro capability does not remove the WCD DAI's 192 kHz ceiling or establish a high-rate SoundWire mode. |
| Bolero TX decimation | `tx-macro.c` DAIs allow I and three formats through 24 bits. hw_params also contained an unadvertised 384 kHz case and no format validation. | Remove that case and validate formats before register writes: unsupported rates/formats passed through backend fixup return `-EINVAL` rather than programming a mode outside the DAI contract. |
| RX echo capture | `rx_macro_echo`: 8/16/48; S16_LE/S24_LE/S24_3LE. | Unchanged. Not a high-rate microphone or speaker capture route. |
| VA/low-power capture | `va-macro.c`: I and S16_LE/S24_LE/S24_3LE; independent decimator/DMIC-clock validation. | Unchanged. No high-rate/32-bit enablement inferred from RX. |
| NXP speaker playback | TERT MI2S RX defaults to 48/S16_LE/stereo. Q6 TERT DAI advertises S16_LE; TFA advertises S16_LE/S24_LE/S32_LE and 8/11.025/16/22.05/32/44.1/48. | Unchanged effective S16_LE intersection. Every requested rate still needs a matching selected TFA container profile. No 96/192/384 kHz speaker claim. |
| NXP feedback/capture | Separate machine TX configuration defaults to 48/S16_LE/mono. Shared MI2S clocks and vendor DSP protection/calibration are dependencies. | Unchanged; TFA capture advertisement alone is not evidence for a new microphone or independent 24-bit feedback route. |
| Voice/AUXPCM, Bluetooth, USB digital, display audio | Separate backends, fixups, firmware and/or external sink negotiation. | Unchanged; WCD marketing maxima do not apply to these routes. |

WCD now also validates hw_params against its direction-specific DAI rates,
rate bounds and formats. Backend fixups can bypass normal capability-mask
negotiation, so declarations alone are insufficient. This check rejects
forced 352.8/384 kHz playback and invalid capture modes without changing
register programming or silently substituting another mode.

End-to-end integration points checked:

* `techpack/audio/asoc/msm_dailink.h` pairs `rx_dma_rx0/1/2/3/5` with
  `rx_macro_rx1/2/3/4/5` and WCD, and `tx_dma_tx3/4` with TX macro and WCD.
  RX6 instead serves SoundWire haptics, disabled by the board. Not every
  enumerated AFE port needs a board DAI link.
* `lahaina.c` CDC DMA mixer rate lists include 11.025/22.05 and rates through
  384 kHz; format lists include S32_LE. They are shared with other routes and
  variants, not a hardware capability matrix. `cdc_dma_get_sample_rate*()`
  maps all 13 entries in both directions. `msm_be_hw_params_fixup()` forces
  the selected rate/format/channels. Unsupported combinations still need to
  fail negotiation or macro hw_params; do not silently substitute 48 kHz.
  WCD9380 versus WCD9385 playback controls are selected by the probed variant.
* `msm-dai-q6-v2.c:msm_dai_q6_cdc_dma_hw_params()` maps S16 to 16,
  S24_LE/S24_3LE to 24 and S32 to 32 AFE bits, rejecting unknown formats.
  `q6afe.c:afe_port_start()` sends the CDC DMA configuration to the DSP;
  passing a rate to AFE is not evidence that the loaded firmware accepts it.
* `wcd938x.c` uses DT channel maps and `swr_connect_port()`, not mainline
  SoundWire PCM hw_params. `lahaina-port-config.h` supplies bus frame/port
  parameters. The overlay supplies RX 22.5792 MHz and TX 19.2 MHz clock
  references; ADC mode selects separate TX bus clock rates. These are not
  PCM sample rates and must not be changed to match a product-page maximum.
* The active overlay chain and `lahainaauto.conf`/`lahainaautoconf.h` already
  integrate WCD938x/slave, RX/TX/VA macros, SoundWire and Oplus TFA98xx-v6.
  No missing source file required by these paths was demonstrated. No files,
  firmware, register maps or DT properties were imported.

### Bit depth, slots and unresolved evidence

PCM payload bitrate is `sample_rate * channels * valid_bits`, not a compressed
codec bitrate setting. S24_LE has 24 valid bits in a 32-bit memory container;
S24_3LE packs those bits in three bytes. S32_LE advertisement does not prove a
32-bit ADC or DAC. AFE sample width and wire slot width are separate.

For example, `lahaina.c:get_mi2s_bits_per_sample()` uses 32-bit MI2S slots
for either 24-bit format. At 48 kHz/stereo, 24-bit payload is 2.304 Mbit/s,
but two 32-bit slots require 3.072 MHz bit clock. This is **not** evidence
that the installed TFA container configures those slots: its TDM setter is
a no-op and its hw_params selects a profile by rate, not format.
The existing S16 default remains 1.536 Mbit/s with two 16-bit slots.
SoundWire adds its own framing/port overhead; PCM payload is not bus bandwidth.
Generic Q6 TDM accepts only 16/32-bit slots, despite the machine's shared
16/24/32 slot-width control; no TDM expansion is made for this board.

Unresolved before any capability expansion:

* Actual WCD variant, amplifier revision IDs and target-board clock/transport
  qualification at 352.8/384 kHz or new sample widths.
* Public direction/interface-specific register documentation; do not equate
  RX interpolation support with ADC, SoundWire or PCM-interface support.
* Installed `tfa98xx.cnt` profiles, amplifier interface settings, calibration,
  protection and VI-feedback duplex clock compatibility. Startup constraints
  default off (`pcm_no_constraint=1`); the hw_params profile-rate check remains.
  The TFA `rate_to_fssel` lookup also contains 12/24 kHz, but its DAI mask
  does not advertise them. Neither a lookup entry nor a generic family SDK
  proves matching handset profiles/clocking, so these rates are not enabled.
* Matching Android audio policy/HAL mixer choices, DSP firmware and calibration.
  Kernel masks alone neither expose a new Android mode nor prove sound quality.

### Regression and safe runtime cases

There is no dedicated audio constraint unit-test harness in this tree; none
is introduced solely for this small mask/validation correction. Review the
WCD masks against `rate_min/max`, RX `sr_val_tbl` and TX hw_params:

* Playback retains every I+F rate and all four formats at or below 192 kHz.
  WCD capture retains every I rate and S16_LE/S24_LE/S24_3LE.
* Reject WCD capture 44.1/88.2/176.4 kHz and S32_LE; reject 11.025/22.05
  on WCD paths and 352.8/384 on the WCD link. TX hw_params must reject
  fractional rates, 384 kHz and S32_LE even when supplied by backend fixup.
* Check the unchanged 48 kHz/S16 defaults first. On hardware, request each
  intended backend mode through the matching HAL/mixer setup and inspect the
  actual backend parameters, not just the frontend stream's rate.
* Only with verified container/interface settings, test S24_LE/32-slot
  negotiation separately from packed S24_3LE and S32_LE. Do not force them
  on the speakers on the basis of this document.
* At low volume, repeat headphone playback and microphone capture separately
  and concurrently, then speaker playback with protection/feedback enabled.
  Confirm invalid requests fail without a stuck clock, stale stream or DSP
  restart. Repeat stop/start, suspend/resume and supported SSR recovery using
  the earlier device checklist. Do not flash automatically.

### Checks run for this continuation

The available toolchain is Ubuntu clang/LLVM 17.0.6, not the Android clang
`r416183b` requested by `build.config.common`. The following reproducible
commands built the changed drivers and adjacent machine/Q6/RX paths with the
unmodified Lahaina defconfig, including CFI/LTO:

```sh
src=/home/runner/work/Levion_kernel_OP9/Levion_kernel_OP9
out=/tmp/levion-audio-cfi-build
export PATH=/usr/lib/llvm-17/bin:$PATH
export LLVM_NM=llvm-nm-17
set -- O="$out" ARCH=arm64 LLVM=1 CC=clang-17 LD=ld.lld-17 \
    AR=llvm-ar-17 NM=llvm-nm-17 OBJCOPY=llvm-objcopy-17 \
    OBJDUMP=llvm-objdump-17 READELF=llvm-readelf-17 HOSTCC=clang-17 \
    AUDIO_ROOT="$src/techpack/audio"
make -C "$src" "$@" vendor/lahaina-qgki_defconfig
make -C "$src" "$@" -j4 \
    techpack/audio/asoc/codecs/wcd938x/wcd938x_dlkm.o \
    techpack/audio/asoc/codecs/wcd938x/wcd938x_slave_dlkm.o \
    techpack/audio/asoc/codecs/bolero/tx_macro_dlkm.o \
    techpack/audio/asoc/codecs/bolero/rx_macro_dlkm.o \
    techpack/audio/asoc/lahaina.o techpack/audio/asoc/msm-dai-q6-v2.o

git -C "$src" diff --check 810b123984702c474d6b98a2b56671a5099db867 HEAD
git -C "$src" diff 810b123984702c474d6b98a2b56671a5099db867 HEAD |
    perl "$src/scripts/checkpatch.pl" --no-tree -
```

Results and limits:

* Defconfig generation and the above focused build passed. With LTO, the
  component objects are LLVM bitcode and composite module objects are
  archives, not final loadable `.ko` files.
* A separate compatibility build in `/tmp/levion-audio-build` disabled
  `CFI_CLANG` and `LTO_CLANG` as shown below, with the same tool
  arguments. It built the same paths and linked AArch64 ELF relocatable
  WCD codec/slave and RX/TX macro module objects. This is not a production
  configuration recommendation. The final WCD validator compiled in both
  configurations.
* The broader `Image modules` compatibility build below was intentionally
  stopped after approximately ten minutes. It reached audio
  `machine_dlkm.o`, `platform_dlkm.o` and `tfa98xx-v6_dlkm.o` linking without
  a fatal compile error. It emitted unrelated scheduler warnings and did
  **not** finish the kernel Image, final `.ko` linking or modpost validation.
  This is an incomplete build, not a passed full build or a diagnosed audio
  build failure. Session logs were kept outside the repository in
  `/tmp/levion-audio-full-build.log` and `/tmp/levion-audio-cfi.log`.
* Initial build issues were resolved: explicitly selecting LLVM's nm
  (rather than an unavailable unversioned tool), and including
  `<sound/pcm_params.h>` for the new format validation.
* `git diff --check` passed; patch-mode checkpatch reported zero errors and
  zero warnings. Manual review checked both masks against the six TX
  switch cases, all nine playback rates against the RX table, format versus
  container width, and early rejection before TX register writes.
* Secret scanning of all three changed files was clean. Code review found
  no remaining significant issues. CodeQL was requested, but **skipped
  because the C/C++ database was too large**; it did not pass an analysis.
* No matching-toolchain/KMI certification, physical testing, HAL negotiation
  test, or runtime rejection test is claimed. The cases above remain device
  regression requirements, not results from an attached phone.

Compatibility-build commands (reuse `src` and tool arguments from above;
the last `O=` overrides the earlier output directory):

```sh
out=/tmp/levion-audio-build
make -C "$src" "$@" O="$out" vendor/lahaina-qgki_defconfig
"$src/scripts/config" --file "$out/.config" -d CFI_CLANG -d LTO_CLANG
make -C "$src" "$@" O="$out" olddefconfig
make -C "$src" "$@" O="$out" -j4 \
    techpack/audio/asoc/codecs/wcd938x/wcd938x_dlkm.o \
    techpack/audio/asoc/codecs/wcd938x/wcd938x_slave_dlkm.o \
    techpack/audio/asoc/codecs/bolero/tx_macro_dlkm.o \
    techpack/audio/asoc/codecs/bolero/rx_macro_dlkm.o \
    techpack/audio/asoc/lahaina.o techpack/audio/asoc/msm-dai-q6-v2.o
make -C "$src" "$@" O="$out" -j4 Image modules
```
