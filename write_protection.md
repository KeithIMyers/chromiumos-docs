# Write Protection

[TOC]

## Background

The core of the Chrome OS security model is rooted in a firmware image that we
fully control and whose integrity we can guarantee. All existing designs have
accomplished this through a dedicated flash part which is guaranteed to be
read-only once the device is shipped by OEMs to customers. This is colloquially
referred to as the "read-only (RO) firmware".

The RO firmware is the first thing executed at power on/boot and is responsible
for verifying & loading the next piece of code in the system which is usually
the Linux kernel. There might be other components that are loaded/chained
(read-write (RW) firmware, etc...) before loading the Linux kernel, but those
details are immaterial here. The point is that the entire system security
hinges upon the integrity of the RO firmware.

## Hardware Write protect

We guarantee the RO firmware integrity via the Write Protect (WP) signal.  This
is a physical line to the flash (where the RO firmware is stored) that tells
the flash chip to mark some parts as read-only and to reject any modification
requests. So even if Chrome OS was full of bugs and was exploited to gain all
the permissions for direct write access to all pieces of hardware in the
system, any write attempts from code running on the CPU would be stopped by the
flash chip itself. Then when the system reboots, the RO firmware would detect
any modifications or corruption to the hard drive (e.g. kernel/root
filesystem). Thus we can confidently tell customers: if you reboot a
Chromebook, you know it's secure.

This is a somewhat tricky topic since write protection implementations can
differ between chips and the hardware write protection has changed over time.

Historically, the WP signal has been controlled by a physical screw
(colloquially referred to as the "WP screw") inside of the Chromebook.  Our
security goals have been that, in order to disable flash WP, someone needs
extended physical access to the device, and it would take a non-trivial amount
of effort and time to open up the device in order to remove the WP screw (and
thus disable the WP signal making the RO firmware writable from software).

In newer devices, we've moved away from the WP signal being controlled by
a physical screw and to a separate chip controlling the WP signal.  That way we
have stronger control over the WP signal and making sure only device owners are
authorized to disable it.

That separate chip is often referred to as a secure element (SE), and in newer
devices is handled by the CR50 which is fully authored and controlled by
Google.

## Software write protect

In addition to the hardware WP signal, there is a software WP setting that
allows us some more flexibility in managing write protection. The software WP
setting is configured in the flash chip to designate the region in the flash
memory that is subject to write protection. This region is actually what the
flash chip checks when performing access control checks for flash writes. The
flash chip enforces that software WP setting can be switched off by the CPU
only when hardware WP is not asserted. Software WP can however be enabled while
hardware WP is asserted.

This allows us to ship systems with hardware WP on, but leaving software WP off
until a point in time where software decided to engage it. One case where we're
using this is devices that originally get flashed with development signing
keys, but eventually get upgraded to production keys, after which software WP
would get enabled.

## Write-protect for developers

Some people ask why we allow the RO firmware to be made writable in the first
place if it's so critical to the security of Chrome OS.  It's not an
unreasonable question.

Since the start of the project, Chrome OS strongly believes that when someone
buys a device, they own it fully.  The device does not belong to OEMs, to
Google, to Chrome OS, or to anyone else.  If the user wants to run Windows, or
Linux, or some other random software on their device that isn’t Chrome OS, they
should have that freedom.

To that end, we strongly believe that users must be free to fully program their
device in any way they want.  Developer mode and support for developers and
free software is extremely important to us as a project.  So if someone wanted
to write their own firmware (which they can as the firmware on devices is open
source that we release), they should have that ability.  We’ve seen some
projects do just that, as well as users who have leveraged that.

The ability to reprogram firmware is also a core requirement for
Google-internal development needs.  It would be much more complicated during
the early phases of a new hardware project to iron out kinks without the
ability to fix firmware on the chips in the devices themselves.

## Application Processor (AP) Firmware

AP firmware (also known as "SOC firmware", "host firmware", "main firmware" or
even "BIOS") typically resides on a SPI ROM. Protection registers on the SPI
ROM are programmed to protect the read-only region, and these registers
normally cannot be modified while the SPI ROM WP (write protect) pin is
asserted. This pin is asserted through various physical means, but with effort,
users can unprotect devices they own.

## Embedded Controller (EC) Firmware

The Chrome OS Embedded Controller (EC) typically has a WP input pin driven by
the same hardware that generates SOC firmware write protect. While this pin is
asserted, certain debug features (eg. arbitrary I2C access through host
commands) are locked out. Some ECs load code from external storage, and for
these ECs, RO protection works similar to SOC firmware RO protection (WP pin is
asserted to EC SPI ROM). Other ECs use internal flash, and these ECs emulate
SPI ROM protection registers, disabling write access to certain regions while
the WP pin is asserted.

## Reading Write Protection Status

### Hardware write protection Status

To check hardware write protection status, run the following command:
`crossystem | grep wpsw`

```
localhost ~ # crossystem | grep wpsw
wpsw_cur                = 1                              # [RO/int] Firmware write protect hardware switch current position
```

Values of `1` indicate that write protection is enabled.

### Software Write Protection status

To check software write protection status, run the following commands. Make
sure that the write protect range length is not 0.
*   EC: `flashrom -p ec --wp-status`
*   Host: `flashrom -p host --wp-status`

If PD (Power Delivery) firmware is present, this additional command can be run:
*   EC: `flashrom -p ec:type=pd --wp-status`

## Disabling write protect

### Disabling hardware write protect {#disable-hw-wp}

For systems that do not have cr50:
*   Power down the device and open the case
    *   Locate and remove the write protect screw on the motherboard.
*   Restart the device
*   `crossystem wpsw_cur` should now output `0`

For systems that have cr50:
*   Use [Servo] to connect to the cr50 console
    *   Enter the chroot with `cros_sdk --no-ns-pid`
    *   Ensure servod is running with `sudo servod &` if necessary.
    *   Run `dut-control cr50_uart_pty` to discover the pty file for the console.
    *   Connect to that console with minicom, cu, or screen.
*   Perform the ["CCD open"] process to enable functionality.
    *    Run `wp disable` on the [Cr50 console].

If you don't want to go through the CCD open process or don't have a [suzyQ], you
can open the case and remove the battery to disable hardware write protect.
*   Disassemble the device, locate the battery connector, remove the battery connector from the
    PCB to disconnect the battery.
*   Reassemble the device, insert the original OEM charger (necessary since the
    battery is no longer providing power to the system), then boot to developer
    mode.

### Disabling software write protect {#disable-sw-wp}

*** note
*NOTE*: You cannot disable software write protect if hardware write protect is
enabled.
***

*   Run `flashrom -p host --wp-disable`.
*   Run `flashrom -p ec --wp-disable`. If `RO_AT_ROOT is not clear` message
    is displayed, check if the write protect screw has been removed. Do an
    [EC Reset](#reset-ec).
*   For devices with PD,
    *   Samus: `flashrom -p ec:type=pd,block=0x800 -w pd.bin`.
    *   Glados: `flashrom -p ec:type=pd --wp-disable` followed by
    `ectool --name=cros_pd reboot_ec RO at-shutdown` and `reboot`.

## Enabling write protect

### Enabling hardware write protect {#enable-hw-wp}

Hardware write protect can be controlled by 3 different mechanisms:
*   A write-protect screw, when present,
*   Cr50 firmware, when present, and
*   A pin on the servo header, when present

#### Write Protect Screw

For systems with a write-protect screw:
*   Power down the device and open the case
*   Insert the write protect screw on the motherboard.
*   Restart the device
*   `crossystem wpsw_cur` should now output `1`

#### cr50

For systems that use cr50, you can control it on the device itself:
*   Run `gsctool -a --wp` to check the current system (look for "Flash WP").
*   If CCD factory mode was enabled, run `gsctool -a -F` to enable WP protect.
    *    This command will fail if factory mode is not enabled.
*   See also the internal
    [crosops docs](https://support.google.com/crosops/answer/9790616).

Or you can control it with a suzyQ:
*   Use [Servo] to connect to the cr50 console
    *   Enter the chroot with `cros_sdk --no-ns-pid`
    *   Ensure servod is running with `sudo servod &` if necessary.
    *   Run `dut-control cr50_uart_pty` to discover the pty file for the console.
    *   Connect to that console with minicom, cu, or screen.
*   Perform the ["CCD open"] process to enable functionality.
    *    Run `wp enable` or `wp follow_batt_pres` on the [Cr50 console].

#### servo header

For systems with a servo header:
*   Enter the chroot with `cros_sdk --no-ns-pid`
*   Start servod inside the chroot: `sudo servod &`
*   Force write-protect on via the servo header: `dut-control fw_wp_state:force_on`.

### Enabling software write protect {#enable-sw-wp}

For AP firmware,
*   Check WP Range.
    Run `mosys eeprom map | grep WP_RO`.
    Example output:
    `host_firmware | WP_RO | 0x00000000 | 0x00200000 | static`
*   Alternately run,
    `flashrom -p host -r /tmp/bios.bin`
    `fmap_decode /tmp/bios.bin`
*   Run `flashrom -p host --wp-enable --wp-range START_RANGE END_RANGE`.
    Example:
    `flashrom -p host --wp-enable --wp-range 0x00000000 0x00200000`

For EC firmware,
*   Check WP range
    *   `flashrom -p ec -r /tmp/ec.bin`
    *   `fmap_decode /tmp/ec.bin | grep EC_RO`
*   Run `flashrom -p ec --wp-enable --wp-range START_RANGE END_RANGE`.

["CCD open"]: https://chromium.googlesource.com/chromiumos/platform/ec/+/refs/heads/master/docs/case_closed_debugging_cr50.md#Open-CCD
[Cr50 console]: https://chromium.googlesource.com/chromiumos/platform/ec/+/refs/heads/master/docs/case_closed_debugging_cr50.md#Consoles
[Servo]: https://chromium.googlesource.com/chromiumos/third_party/hdctools/+/refs/heads/master/README.md
[suzyQ]: https://chromium.googlesource.com/chromiumos/third_party/hdctools/+/master/docs/ccd.md#suzyq-suzyqable
