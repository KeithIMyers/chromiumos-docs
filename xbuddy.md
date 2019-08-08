# XBuddy for Devserver

[TOC]

## Overview

XBuddy for devserver allows you to obtain Chromium OS images off Google Storage
and reference locally built images. When used with devserver (or scripts that
use it), it can also be used to update your Chromium OS machine to any image.

## Prerequisites

### Google Storage Credentials

You need them to get anything off of Google Storage!
Run:

```bash
gsutil config
```

Ensure that the resulting .boto file is discoverable from the environment from
which you are running the devserver from, whether inside or outside the chroot,
so copy it over if necessary.

## Example Usage - Imaging a Device

On your developer machine, start a devserver. (Please see [Using the Devserver]
for more details about the devserver.)

```bash
cd ~/chromiumos
cros_sdk  # if not already in the chroot
start_devserver
```

### Update a device to the latest released build

On your test machine, with `BOARD` as your device board:

```bash
update_engine_client --update --omaha_url="http://${HOST}:${PORT}/update/xbuddy/remote/${BOARD}/latest/dev"
```

### Update a device to the latest locally built image

On your test machine, get to a root shell (either Ctrl-Alt-F2 gets a vt2
terminal to login as root, or else to stay in the GUI, Ctrl-Alt-t for crosh
followed by `shell` and `sudo su`), and use `${HOST}:${PORT}` is where your
devserver is running from.

```bash
update_engine_client --update --omaha_url="http://${HOST}:${PORT}/update"
```

### Use an arbitrary XBuddy path.

On your test machine, with `XBUDDY_PATH` as some path to an image:

```bash
update_engine_client --update --omaha_url="http://${HOST}:${PORT}/update/xbuddy/${XBUDDY_PATH}"
```

(See [XBuddy Paths](#xbuddy-paths) for all path options.)

## Example Usage - Staging Artifacts onto DevServer

### Get a particular artifact from the devserver

On your developer machine, call the xbuddy RPC with an xbuddy path to download
the latest local build on the devserver at `${HOST}:${PORT}` to the
`chromiumos_test_image.bin`

```bash
wget http://${HOST}:${PORT}/xbuddy/local/x86-generic/latest/test -O chromiumos_test_image.bin
```

### Stage an artifact on the devserver and get the path to it
Similarly, download the path to the update payload on that devserver.

```bash
wget http://${HOST}:${PORT}/xbuddy/remote/eve/latest-official/full_payload?return_dir=true -O path.txt
```

See [XBuddy Interface and Usage](#xbuddy-interface-and-usage) for more details.

## XBuddy Paths

* **local/remote** - Optional (defaults to local). If a path is explicitly
  remote, XBuddy will interpret the rest of the path to query into google
  storage for artifacts. If local, XBuddy will attempt to find the artifact
  locally.
* **board name** - Required. Mostly self-explanatory e.g. eve.
* **version alias** - Optional (defaults to latest). A version string or an
  alias that maps to a version string.
    + *latest (default)* - If 'local', latest locally built image for
      that board. If 'remote', the newest stable image on Google Storage.
    + *latest-{canary|dev|beta|stable}* - Latest build produced for the given
      channel.
    + *latest-official* - Latest available release image built for that board in
      Google Storage.
    + *latest-official-{suffix}* - Latest available image with that board suffix
      in Google Storage.
	+ *latest-{branch-prefix}* - Latest build with given branch prefix
      i.e. RX-Y.X or RX-Y or just RX -- seems broken, see line below
    + *{branch-prefix}* - Latest build with given branch prefix i.e. RX-Y.X or
      RX-Y or just RX.
* **image type** - Optional (defaults to test). Any of the artifacts that
  devserver recognizes.
    + *test (default)* - A dev image that has been modified to make testing even
      easier.
    + *dev* - Developer image. Like base but with additional dev packages.
    + *base* - A pristine Chrome OS image.
    + *recovery* - A base image that can be flashed to a USB stick and used to
      recover a Chrome OS install.
    + *signed* - A recovery image that has been signed by Google and can be
      installed through autoupdate.

### XBuddy Path Overrides
There are abbreviations for some common paths.  For example, an empty path is
taken to refer to the latest local build for a board.

The defaults are found in `src/platform/dev/xbuddy_config.ini`, and can be
overridden with `shadow_xbuddy_config.ini`

## XBuddy Interface and Usage

### Devserver RPC: xbuddy

If there is a devserver running, stage artifacts onto it using xbuddy paths by
calling "xbuddy" The path following xbuddy/ in the call url is interpreted as an
xbuddy path, in the form "{local|remote}/build_id/artifact"

The up to date documentation can be found under the help section of devserver's
index, `http://${HOST}:${PORT}/doc/xbuddy`

```bash
http://${HOST}:${PORT}/xbuddy/{remote/local}/board/version/artifact

http://${HOST}:${PORT}/xbuddy/remote/amd64-generic/R77-4000.0.0/test # To stage an x86-generic test image

http://${HOST}:${PORT}/xbuddy/remote/amd64-generic/R77-4000.0.0/test?return_dir=true # To get the directory name that the artifact will be found in
```

### Devserver RPC: xbuddy_list

If there is a devserver running, check which images are currently cached on it
by calling `xbuddy_list`

```bash
http://${HOST}:${PORT}/xbuddy_list
```

### Updating from a DUT: update_engine_client

If there is a devserver running, and you have access to the test device, update
the device by calling `update_engine_client` with an `omaha_url` that contains
an xbuddy path.  The `omaha_url` is a call to devserver's update
RPC. Documentation found under the help section of devserver's
index. `http://${HOST}:${PORT}/doc/update`

```bash
update_engine_client --update --omaha_url=${HOST}:${PORT}/update/

update_engine_client --update --omaha_url=${HOST}:${PORT}/update/xbuddy/{remote/local}/board/version/{image_type}

update_engine_client --update --omaha_url=${HOST}:${PORT}/update/board/version
```

### Scripts (cros flash)

To use xbuddy path with cros flash:

```bash
cros flash ${DUT_IP} xbuddy://${XBUDDY_PATH}
```

See [cros flash] page for more details.

[cros flash]: https://chromium.googlesource.com/chromiumos/docs/+/master/cros_flash.md
[Using The Devserver]: https://chromium.googlesource.com/chromiumos/chromite/+/master/docs/devserver.md
