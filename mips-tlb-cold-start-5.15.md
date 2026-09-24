# MIPS routers hang on cold start (kernel 5.15.190 to 5.15.208)

**In short:** On a subset of MIPS routers the kernel stops during a **cold
start**, before it prints its first memory line. Warm restarts and firmware
upgrades go through without a problem, and that is what makes this one nasty:
it only shows up when part of your fleet fails to come back after a power cut.

**Cause:** a kernel commit that has been uniquifying TLB entries at boot since
5.15.190 and kills some boards while doing so. It is properly fixed only in
**5.15.209**. The openwrt-23.05 branch has carried the fix since 2026-07-11
(kernel 5.15.211), but firmware pinned to an older 23.05 state is still
affected, and that includes **Gluon v2023.2.6, which ships 5.15.198**. Gluon
picked up the newer base on its v2023.2.x branch on 2026-09-22; no tagged
release contains it yet.

**Applies to** firmware built on an openwrt-23.05 state from before
2026-07-11, such as Gluon v2023.2.6, on r4k-class MIPS devices: ath79, lantiq,
ramips. Last updated 2026-09-24.

---

## How to recognise it

The device does not come back after a power cut, is silent on the network and
emits no WLAN. Power cycling changes nothing, because every attempt is another
cold start.

On the serial console the output stops in the middle of the kernel start:

```
[    0.000000] Linux version 5.15.198 ...
[    0.000000] CPU0 revision is: 00019750 (MIPS 74Kc)
...
Starting kernel ...
```

Nothing follows. The line that would come next is the memory summary:

```
[    0.000000] Memory: 55916K/65536K available (...)
```

That is the reliable discriminator: **`Starting kernel` appears, `Memory:` does
not.** Between the two sits `tlb_init()`, and that is where it stops.

**The decisive test:** restart the device through sysupgrade or a warm reboot.
If it comes up that way and only fails after pulling the plug, this is the bug
and not a dead device. After a warm restart the TLB has already been uniquified
and the faulty routine no longer finds anything to choke on.

## Which devices

This **cannot** be predicted from the CPU core. What matters is what the
bootloader leaves behind in the TLB, and that differs from board to board. Our
measurements, 11 to 21 cold starts each:

| Device | SoC | Core | RAM | boots unpatched |
| --- | --- | --- | ---: | --- |
| TP-Link Archer C25 v1 | QCA956X | 74Kc | 64 MB | **0 of 21** |
| TP-Link TL-WR1043ND v2 | QCA9558 | 74Kc | 64 MB | **0 of 20** |
| TP-Link TL-WDR3600 v1 | AR9344 | 74Kc | 128 MB | 11 of 11 |
| Ubiquiti EdgeRouter X | MT7621 | 1004Kc | 256 MB | 11 of 11 |
| Xiaomi Mi Router 4A Gigabit | MT7621 | 1004Kc | 128 MB | 11 of 11 |

The WDR3600 is the same 74Kc as the two affected devices and boots reliably.
Which also means: **one device type booting fine at your site says nothing
about another one with the same SoC.**

Here the bug took out roughly 57 nodes in March 2026, 50 of them on ath79, the
rest single nodes on lantiq and ramips. No non-MIPS device was affected.

## The cause in the kernel

`35ad7e181541` ("MIPS: mm: tlb-r4k: Uniquify TLB entries on init") arrived in
5.15.190 and calls `r4k_tlb_uniquify()` from `r4k_tlb_configure()`. That
function operates on whatever the bootloader left behind in the TLB.

On affected boards the bootloader hands the TLB over as it was at reset: with
the hidden valid bit set and possibly duplicate entries. Resetting the page
sizes then raises a machine check exception, and the boot ends there.

**`9f048fa48740` in 5.15.197 does not fix it.** Worth stressing, because the
version number suggests the matter is settled. Nothing further landed up to
5.15.208.

## What to do

### The easy way: move to a current base

If you build Gluon yourself, rebase onto the **v2023.2.x branch from
2026-09-22 onwards** ([freifunk-gluon/gluon#3841](https://github.com/freifunk-gluon/gluon/pull/3841)),
which brings 5.15.211. If you build OpenWrt 23.05 directly, any branch state
from 2026-07-11 onwards has it. Drop any local workaround for this bug when
you do, the kernel code it touches has changed.

### If you have to stay on 5.15.198: backport the fix

Upstream fixed this properly in **5.15.209**, released 2026-06-01, with five
commits:

| stable 5.15 | upstream | Title |
|---|---|---|
| `da0f6cd551dc` | `841ecc979b18` | MIPS: mm: kmalloc tlb_vpn array to avoid stack overflow |
| `2eadfb3b649e` | `01cc50ea5167` | mips: mm: Allocate tlb_vpn array atomically |
| `0e39d8dd8762` | `8374c2cb83b9` | MIPS: Always record SEGBITS in cpu_data.vmbits |
| `88af0913282f` | `74283cfe2163` | MIPS: mm: Suppress TLB uniquification on EHINV hardware |
| `79ad8f65712f` | `540760b77b8f` | MIPS: mm: Rewrite TLB uniquification for the hidden bit feature |

The last one carries `Fixes: 9f048fa48740` and rewrites the uniquification so
that it copes with a TLB in reset state.

The five patches **apply cleanly to 5.15.198**, in this order, without fuzz and
without a reject, and no OpenWrt 23.05 patch touches the same files. They
belong in `target/linux/generic/backport-5.15/`. To fetch them:

```sh
for c in da0f6cd551dc 2eadfb3b649e 0e39d8dd8762 88af0913282f 79ad8f65712f; do
  curl -sO "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/patch/?id=$c"
done
```

Measured on the TL-WR1043ND v2: **20 of 20 cold starts**, no hang. In the same
rig a build without any TLB patch hung 3 of 3, and a build with the small fix
below came up 3 of 3. So the rig detects both states.

### The small way: drop the call

If you just want the things to boot again, take the call out and restore the
behaviour of 5.15.189. Three changed lines, one file:

```diff
--- a/arch/mips/mm/tlb-r4k.c
+++ b/arch/mips/mm/tlb-r4k.c
@@
-static void r4k_tlb_uniquify(void)
+static void __maybe_unused r4k_tlb_uniquify(void)
 {
 	unsigned long tlb_vpns[1 << MIPS_CONF1_TLBS_SIZE];
@@
 	/* From this point on the ARC firmware is dead.	 */
-	r4k_tlb_uniquify();
 	local_flush_tlb_all();
```

Drop it in as
`target/linux/generic/hack-5.15/999-mips-tlb-r4k-no-uniquify.patch`. Across
both affected devices: **0 of 41** cold starts unpatched against **41 of 41**
patched.

**The downside:** it is a local deviation you have to explain and re-check on
every kernel bump. Given the choice, take the 5.15.209 fix.

### What does not help

* **Rebooting.** Every power-cycle attempt is another cold start.
* **Swapping the device.** The identical replacement has the same bootloader.

### Fixed from OpenWrt 24.10 onwards

That runs 6.6, which contains the fix. Measured with 6.6.144: 21 of 21 cold
starts on the Archer C25.

## Measuring it yourself

To check your own device you need three things: a serial console, a remotely
switchable mains socket and a TFTP server. The image is loaded **into RAM**,
the flash stays untouched, and the device is back to its old state once you
power it off.

1. Power off, wait at least five seconds, power on.
2. Catch the bootloader. The prompt differs per device, for instance `ap135>`
   on the TL-WR1043ND v2, and on TP-Link devices it is often only the magic
   word `tpl` that interrupts the countdown, not Ctrl-C.
3. `setenv serverip <tftp server>` and `tftpboot 0x82000000 <image>`, then
   `bootm 0x82000000`.
4. Classify: `Memory:` seen means it booted, only `Starting kernel` means it
   hung.

On old TP-Link images the ordinary sysupgrade file is enough, the uImage header
sits at the front. No initramfs needed, the kernel only has to get far enough
to print the memory line.

**Two traps from practice:** the bootloader switches its active network
interface between cold starts, and it answers no ARP requests. If the transfer
lands on the interface without a server it runs into a timeout that looks like
a device fault. Simply issue `tftpboot` a second time within the same boot, and
put a static ARP entry for the device on the server.

All of this only becomes meaningful with **several runs and a control**: once
with and once without the fix, from the same build tree. Otherwise you may be
measuring nothing but your own test rig.
