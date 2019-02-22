# Cros Flash

## Overview

Cros Flash is a script to update a Chromium OS device with an image, or to copy
image onto a removable device (e.g. a USB drive). It replaces
`image_to_live.sh`, `cros_image_to_target.py` and `image_to_usb.sh`.

Cros Flash utilizes the [devserver] to download images and/or generate payloads.

When updating a Chromium OS device, Cros Flash relies on an SSH connection to
talk to the device (which is enabled in the test images). The main difference
with previous update tools is that Cros Flash assumes that the device is NOT
capable of initiating an SSH connection to your workstation.  This allows Cros
Flash to be used in a more restricted/secured network environment.

To know more about the design and implementation of Cros Flash, please look at
the [design doc] (may be slightly outdated). The source code for `cros flash` is
[available here].

[TOC]

## Requirements

### Updating a chromium device

1.  Chroot: Cros Flash needs chroot to run [devserver] to generate payloads.
2.  A SSH-able Chromium OS device: Any Chromium OS test image will work.
3.  SSH test keys so that the script can SSH into the test device without a
    password. See Setting up [SSH Access] to your test device.
4.  Full Chromium OS checkout (until crbug.com/403086 is addressed).


If your device is currently running a non-`test` image, you may want to use a
USB stick to image the device first (see [developer guide]).

There are plans to eliminate both requirements so that Cros Flash can be
integrated into the [SimpleChrome] workflow. You can track the progress
[here](http://crbug.com/309848).

### Download images/payloads from Google Storage

1.  Chroot: Cros Flash relies on xbuddy/[devserver] to download the files.
    Devserver currently runs only in chroot (This is an artificial requirement
    and may be lifted in the future)
2.  Credentials to download from Google Storage. External developers may not have
    such credentials.

## Example Usage

```bash
cros flash <device> <image>
```

*   `<device>`: Required. `ssh://IP[:port]` of your ChromiumOS device,
    `usb://path/to/removable/device`, or `file://path/to/a/file`
*   `<image>`: Optional.  Can be a path to an image, an xbuddy path, a payload
    directory, or simply `latest` for latest locally-built image. Defaults to
    `latest`.

For example, to update the device with the latest locally-built image, you can
use the shortcut `latest`:
```bash
cros flash ssh://192.168.1.7 latest
```

or simply type:
```bash
cros flash ssh://192.168.1.7
```

To use a locally-built (test) image:
```bash
cros flash ssh://192.168.1.7 xbuddy://local/x86-mario/R33-5100.0.0
```

To use a local image:
```bash
cros flash ssh://192.168.1.7 path/to/image
```

To download the latest canary image:
```bash
cros flash ssh://192.168.1.7 xbuddy://remote/x86-mario/latest-canary
```

You can replace `ssh://192.168.1.7` in the above examples with
`usb://device/path` or `usb://` to copy the image onto a removable device.
However, you have to specify the board to use.
```bash
cros flash ssh://192.168.1.7 board/latest

cros flash usb:///dev/sdc path/to/image

cros flash usb:// path/to/image
```

Finally, if you just want to look at the image you are installing (using
`mount_gpt_image.sh`) or save the file for later, you can use `file://` to save
it to a file of your choice e.g.
```bash
cros flash file://path/to/save/image.bin path/to/image
```

## Device

### Chromium OS Device

Device can be `ssh://hostname[:port]` or `hostname[:port]`. Port number is the
SSH port to use to connect to your device; it is optional (defaults to `22`).

### Removable Device

Device can be `usb://path/to/removable/device` or `usb://`. If a device path is
specified, Cros Flash will check if the device is indeed removable. If no path
is given, user will be prompted to choose from a list of removable devices. Note
that auto-mounting of USB devices should be turned off as it may corrupt the
disk image while it's being written.

## xBuddy (CrOS Buddy) paths

xbuddy path is recognized by devserver and can be very powerful and flexible.
You can specify a remote image for devserver to download and serve, or use any
locally-built image. To use xbuddy path, use the `xbuddy://` prefix in your
path.

The following examples applies to both `ssh://` and `usb://` devices.

```
xbuddy://{local|remote}/board/version/image_type

local/remote: optional (defaults to local)
board:        required (e.g. x86-mario, peppy)
version:      required. See the xbuddy docs for more information.
image_type:   optional (e.g. test, base, dev, recovery, signed; defaults to test)*
```

> If you use Cros Flash to install a non-test image, Cros Flash will not be able
> to confirm that the device has rebooted successfully after update. Also, you
> will not be able to use Cros Flash to reimage again because Cros Flash relies
> on being able to ssh into the device. Use a USB stick to image your device
> (`image_to_usb.sh`) in this case.

For example, to use the latest test image from the beta channel for the board:
```bash
cros flash 192.168.1.7 xbuddy://remote/board/latest-beta
```

For more details about xbuddy including the explanation for `version` and
`image_type`, please see the [xbuddy doc].

Note: you may not be able to download the remote images as an external
developer.

### GS path as an xbuddy path

By default, if you pass a remote xbuddy path, xbuddy will first check to see if
it's a valid Google Storage path before attempting to parse the board/version,
etc. This gives users the flexibility to download an arbitrary image as long as
it resides in our Google Storage bucket (`gs://chromeos-image-archive`).

For example,
```bash
cros flash 192.168.1.7 xbuddy://remote/x86-mario-paladin/R32-4830.0.0-rc1
```

If you are prompted to confirm due to board mismatch, simply type 'yes' to
proceed for now. Users will soon be given an option to skip the prompt.

### Using images produced by trybots

By leveraging the "GS path as xbuddy path" feature, you can also update your
device with an image generated by a trybot. You will need the
`${bot-id}/${version}` used for archiving in this case. The information is
available on your tryjob waterfall:

1.  Open the tryjob page (e.g.,
    https://uberchromegw.corp.google.com/i/chromiumos.tryserver/builders/x86-mario-paladin/builds/99)
2.  At Report stage, you will see links such as `Artifacts[x86-mario]:
    x86-mario-paladin/R36-5737.0.0-rc3`
3.  Copy the `${bot-id}/${version}` at from the link above
    (x86-mario-paladin/R36-5737.0.0-rc3).

Alternatively, you can copy the "Artifact" link and find the same information
(https://storage.cloud.google.com/chromeos-image-archive/trybot-peach_pi-release/R34-5500.91.0-b37/index.html).
You can look at the stdio log of the Archive stage.

For example,
```bash
cros flash 192.168.1.7 xbuddy://remote/trybot-peach_pi-release/R34-5500.91.0-b37
```

Again, if you are prompted to confirm due to board mismatch, simply type 'yes'
to proceed for now.  Users will soon be given an option to skip the prompt.

### Pre-generated payloads

Cros Flash can use your pre-generated update payloads directly if you pass a
payload directory as `<image>`:
```bash
cros flash 192.168.1.7 path_to_payload_directory
```

Cros flash looks for `update.gz` and/or `stateful.tgz` in the payload directory
and use them to update the device.

## Known problems and fixes

Many of the problems you have encountered may have been fixed. Make sure you
keep your chroot up-to-date before filing a bug.

### Update Your Chroot

Run `repo sync` to update your tree and then update your chroot (must be run in
chroot) by:

```bash
# From within your chroot
~/trunk/src/scripts/update_chroot
```

### Where is Cros Flash?

Cros Flash is in the chromite repo. The script is
`chromite/cli/cros/cros_flash.py`. If you cannot find it, update your tree with
repo sync. The online repository is [available here].

### Import error: cherrypy

If you see a cherrypy import error, that is because your device is running an
older image (\<R33-4986.0.0) that does not have cherrypy package installed by
default. Cros flash launches a devserver instance on the device to serve the
update payload for rootfs update. You can fix this with one of the following
three options.

1.  Update the device to a newer image with a USB stick (see [developer guide]).
2.  Use `--no-rootfs-update` to update the stateful parition first (with the
    risk that the rootfs/stateful version mismatch may cause some problems).
3.  Use [cros deploy] to deploy cherrypy package to your device.

### Devserver error: cannot specify both update and return\_dir

Cros flash depends on a relatively newer version of `devserver.py`. If you have
not updated your chroot in a long time, you may see an error message like this:

```bash
DEVSERVER HTTPError status: 500 message: Cannot specify both update and
return_dir
```

To fix this, [update your chroot].

### Cannot download daisy_spring (or other boards where there are underscores in the board names) images

There is inconsistency in how we name the Google Storage buckets. Xbuddy has
been fixed to do generate the correct URL for the image. Please [update your
chroot].

### Failures related to update engine

Make sure that update-engine service on device is running and
inhibit-update-engine service is stopped:

```bash
# On device
stop inhibit-update-engine
start update-engine
```

[available here]: https://chromium.googlesource.com/chromiumos/chromite/+/refs/heads/master/cli/cros/cros_flash.py
[devserver]: http://www.chromium.org/chromium-os/how-tos-and-troubleshooting/using-the-dev-server
[design doc]: https://docs.google.com/a/google.com/document/d/1UdDTCC5V9R6VFMsiIzclMCSNwr-Iz42o6t_NfhEEGc4/
[SSH Access]: https://www.chromium.org/chromium-os/testing/autotest-developer-faq/ssh-test-keys-setup
[SimpleChrome]: http://www.chromium.org/chromium-os/how-tos-and-troubleshooting/building-chromium-browser
[developer guide]: http://www.chromium.org/chromium-os/developer-guide
[xbuddy doc]: http://www.chromium.org/chromium-os/how-tos-and-troubleshooting/using-the-dev-server/xbuddy-for-devserver
[cros deploy]: https://sites.google.com/a/chromium.org/dev/chromium-os/build/cros-deploy
[update your chroot]: #Update-Your-Chroot
