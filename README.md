# Waking and unlocking with the fingerprint reader — ThinkPad X1 Tablet 3rd gen

Two closely related questions, with two different answers. Both are worth writing down,
and the negative one is probably the more useful.

| | |
| --- | --- |
| **Unlock a locked, blank screen by touching the sensor** | ✅ **works, entirely in userspace** |
| **Wake the machine from sleep by touching the sensor** | ❌ **not possible from userspace** |

This repository is a findings write-up. The code that implements the working half lives in
**[x1-tablet-gen3-fingerprint](https://github.com/sandbranch/x1-tablet-gen3-fingerprint)** (`synaspi/wake.py`).

---

## ✅ What works: touch-to-unlock on a blank, locked session

The fingerprint daemon polls the sensor. When it sees a finger while the session is
locked, it matches against the enrolled template and, on a match, unlocks the session
directly — no lock screen interaction, no client, no `Claim` from anything else.

Verified 2026-08-19, immediately after a resume from real S3:

    synaspi.wake: touch-to-wake: press while locked, presence 27.7 score +0.6654 (threshold 0.400)
    synaspi.wake: touch-to-wake: unlocked session 2

### How to be sure it is really this path

This is easy to fool yourself about, because a normal PAM fingerprint login looks similar
in the logs. The distinguishing tell is that the unlock happens **with no preceding
`Claim`** — nothing else had the device; it came from the daemon's own poll loop.

If you see a `Claim` from a lock-screen process just before the match, you are looking at
an ordinary fprintd unlock, not touch-to-wake.

### Combined with folio wake, the result is quite pleasant

With S3 and lid wake in place (see
**[x1-tablet-gen3-wake-on-open](https://github.com/sandbranch/x1-tablet-gen3-wake-on-open)**), the whole sequence
needs no deliberate interaction at all:

1. Open the folio → machine wakes from S3
2. Your thumb is already on the sensor from gripping the tablet
3. The session is unlocked by the time the screen is up

That is not a hypothetical — it is how it was first observed, accidentally.

### Security shape (read this before copying the idea)

It does **not** unlock on any touch. It unlocks on a touch that *matches the enrolled
template*, at the configured threshold. Somebody else opening the folio gets a lock
screen.

But be honest about what it changes: an incidental grip by the enrolled user can unlock
the machine without them intending to. That is a usability/security trade, and whether it
is acceptable depends on your threat model. If you want a deliberate press instead,
gate the unlock on something stronger than mere presence.

---

## ❌ What does not work: waking the machine itself

**A finger on the sensor cannot wake this machine from sleep, and no amount of userspace
code will change that.** Two independent reasons, both checkable in a few seconds:

### 1. The ACPI device has no `_PRW`

`_PRW` is the ACPI "power resources for wake" object. A device without one is not
declared as a wake source, so the platform will not arm it.

    $ sudo python3 -c "d=open('/sys/firmware/acpi/tables/DSDT','rb').read(); \
        i=d.find(b'FPNT'); print('FPNT count:', d.count(b'FPNT'), \
        '_PRW in DSDT:', d.count(b'_PRW'), '_PRW near FPNT:', b'_PRW' in d[i:i+700])"
    FPNT count: 1  _PRW in DSDT: 53  _PRW near FPNT: False

The DSDT uses `_PRW` **53 times** for other devices — so this is not a firmware that
omits them generally. The fingerprint device specifically does not have one.
Correspondingly, `FPNT` never appears in `/proc/acpi/wakeup`.

### 2. The sensor's interrupt line is unowned

The sensor's interrupt is GPI0 pin **41** on `gpiochip0` (`INT344B:00`), right next to pin
42, which is the enable line the driver holds. Nothing claims pin 41:

    $ sudo cat /sys/kernel/debug/gpio | grep -E 'gpio-4[0-3]'
     gpio-42  (                    |synaspi             ) out hi

Pin 41 is absent from the list entirely — no driver has requested it, so nothing is
configured to fire on it, let alone as a wake source.

### What the firmware DOES declare — and why that is not enough

It is worth being precise, because "no wake support" is too crude a summary. The full
`_CRS` for `\_SB.PCI0.SPI1.FPNT` (`_HID` = `SYNA8002`) is:

    SpiSerialBusV2 (... "\\_SB.PCI0.SPI1" ...)
    GpioIo (Exclusive, PullDefault, IoRestrictionOutputOnly, "\\_SB.PCI0.GPI0") { 0x002A }
    GpioIo (Exclusive, PullDefault, IoRestrictionInputOnly,  "\\_SB.PCI0.GPI0") { 0x0029 }
    Interrupt (ResourceConsumer, Edge, ActiveHigh, Exclusive) { 0x00000029 }

    Method (_INI) { SHPO (SFPI, 0x01)  SHPO (SFPS, 0x01) }   // pads -> host software
    Method (_PS0) { \_SB.SGOV (0x02010012, 0x01) }           // firmware drives enable
    Method (_PS3) { \_SB.SGOV (0x02010012, 0x00) }

So there **is** a declared, edge-triggered interrupt on pin 0x29 (41), and a driver
binding to `SYNA8002` would get it populated automatically — no GPIO plumbing required.
Pin 41 is host-software-owned and is readable from userspace today:

    $ sudo gpioget --chip gpiochip0 41
    "41"=inactive

**But it never asserts.** Watching it with `gpiomon --edges=both` across deliberate
finger presses produces **zero events**; the line stays inactive throughout.

The sensor evidently has to be *told* to drive that line — an "arm finger detect" or
low-power "wait for finger" command over SPI, which is a protocol feature that has not
been found. **That, not the kernel, is the actual blocker.** An interrupt handler with
nothing to handle buys nothing.

Also note `_PS0`/`_PS3`: the firmware can drive the enable pin itself. A driver that used
ACPI power states would not need to hold the enable GPIO from userspace at all.

### What it would actually take

In order, and note that the small kernel patch is **not** the hard part:

1. **Protocol RE: find the command that arms finger-detect.** Until the sensor drives pin
   41, everything downstream is moot. This is the real work and it is the same class of
   effort as the rest of the driver.
2. **A thin kernel shim (~150 lines)** that binds `SYNA8002`, takes the already-declared
   IRQ and calls `enable_irq_wake()`. Genuinely small — *once step 1 exists*.
3. **For S3 specifically, a `_PRW`.** This could be added with an additive SSDT, exactly
   the technique used in [x1-tablet-gen3-s3-sleep](https://github.com/sandbranch/x1-tablet-gen3-s3-sleep). But
   `_PRW` must name a GPE that the silicon actually routes this pad to. Lenovo omitted
   `_PRW` here while using it **53 times** elsewhere in the same DSDT, which is a strong
   hint the pad was never wired for wake. Software cannot invent a routing that hardware
   does not have.

Step 2 alone would likely give wake from **s2idle**, since that is just deep idle and any
armed IRQ can break it. It would not give wake from S3.

**A full in-kernel driver would be the wrong shape anyway.** This sensor requires a TLS
handshake before it will do anything, and TLS does not belong in the kernel — which is
exactly why libfprint-class drivers live in userspace. The right architecture is the
userspace driver you already have, plus a thin IRQ shim.

From userspace you can hold the rail and poll the sensor, which is what the working half
above does. You cannot arm an interrupt as a system wake source.

### Do not trust wake findings measured under s2idle

Our original analysis of all of this was carried out before S3 existed on this machine,
under `s2idle`. s2idle and S3 arm materially different things, so when S3 landed, every
one of those conclusions had to be re-checked. The two above survived that re-check —
they are properties of the firmware and the GPIO map, not of the sleep state. Other
findings did not survive: lid wake, for instance, went from "not applicable" to "works
out of the box" purely because the sleep state changed.

If you have wake notes that predate your S3 work, re-measure before relying on them.

---

## Prerequisites

* The fingerprint driver itself — **[x1-tablet-gen3-fingerprint](https://github.com/sandbranch/x1-tablet-gen3-fingerprint)**.
  There is no in-tree Linux driver for this sensor (Synaptics SYNA8002 over SPI).
* For the folio-wake combination, S3 — **[x1-tablet-gen3-s3-sleep](https://github.com/sandbranch/x1-tablet-gen3-s3-sleep)**.

## License

MIT
