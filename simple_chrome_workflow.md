# Building Chrome for Chrome OS (Simple Chrome)

This workflow allows you to quickly build/deploy Chromium to any Chromium OS
device without needing a Chromium OS source checkout or chroot. It's useful for
trying out your changes on a real device while you're doing Chromium
development. If you have an OS checkout and want your local Chromium changes to
be included when building a full OS image, see the [OS development guide].

At its core is the `chrome-sdk` shell which sets up the shell environment and
fetches the necessary SDK components (CrOS toolchain, sysroot, etc.).

[TOC]

## Typography conventions

| Label         | Paths, files, and commands                             |
|---------------|--------------------------------------------------------|
|  (outside)    | on your build machine, outside the chroot            |
|  (inside)     | inside the `chrome-sdk` shell on your build machine (1)|
|  (device)     | on your Chromium OS device                             |
|  (chroot)     | inside the `cros_sdk` crhoot                           |

(1) Note: This is not the same thing as the `cros_sdk` chroot.

## Getting started

First make sure you have a:

1. [Local copy of the Chromium source code and depot_tools]
2. USB flash drive 4 GB or larger (for example, a Sandisk Extreme USB 3.0)
3. USB to Gigabit Ethernet adapter

Googlers: Chromestop has the hardware.

### Get Google API keys

In order to sign in to your Chromebook you must have Google API keys:

* External contributors, see http://www.chromium.org/developers/how-tos/api-keys
  You'll need to put them in your `out_board/Release/args.gn file`, see below.
* Googlers, see http://go/building-chrome to get internal source. If you have
  `src-internal` in your `.gclient` file the official API keys will be set up
  automatically.

## Set up gsutil

Use depot_tools/gsutil.py and run `gsutil.py config` to set the authentication
token. (Googlers: Use your @google.com account.) Otherwise steps below may run
slowly and fail with "Login Required" from gsutil.

When prompted for a project ID, enter `134157665460` (this is the Chrome OS
project ID).

## Fetch the Chrome OS toolchain and SDK for building Chrome

Toolchains are customized for each Chromebook model (or "board"). Look up your
[Chromium OS board name] by navigating to the URL `about:version` on the device.
For example: `Platform 10176.47.0 (Official Build) beta-channel samus`
has board `samus`.

Run this from within your Chromium checkout (not the Chromium OS chroot):

```
(outside) cd /path/to/chrome/src
(outside) export BOARD=samus  # or your board name
(outside) cros chrome-sdk --board=$BOARD --log-level=info
```

`cros chrome-sdk` will fetch the latest Chrome OS SDK for building Chrome, make
an output directory, create a custom args.gn file and put you in a shell with a
command prompt starting with `(sdk $BOARD $VERSION)`.

`cros chrome-sdk` will also automatically install and start the Goma server,
with the active server port stored in the `$SDK_GOMA_PORT` (inside) environment
variable.

Non-Googlers: Only generic boards have publicly available SDK downloads, so you
will need to use a generic board (e.g. amd64-generic) or create your own local
build. Star http://crbug.com/360342 for updates.

### cros chrome-sdk options:

* `--nogn-gen` Do not run 'gn gen' automatically.
* `--gn-extra-args='extra_arg=foo other_extra_arg=bar'` For setting
  extra gn args, e.g. 'dcheck_always_on = true'.
* `--internal` Sets up Simple Chrome to build and deploy the official *Chrome*
  instead of *Chromium*.
* `--log-level=info` Set the log level to 'info' or 'debug' (default is 'warn').

**Note:** See below if you want to use a custom Chrome OS build.

**Note:** When you sync/update your Chromium source, the Chrome OS SDK build
number may change (src/chromeos/CHROMEOS_LKGM). When the SDK version changes
you may have to exit and re-enter the Simple Chrome environment to
successfully build and deploy Chromium.


## Build Chrome

To build Chrome, run:

```
(inside) autoninja -C out_${SDK_BOARD}/Release chrome chrome_sandbox nacl_helper
```

This runs Goma with a large number of concurrent jobs.  To watch the build
progress, find the Goma port (`$ echo $SDK_GOMA_PORT`) and open
http://localhost:<port_number> in a browser.

**IMPORTANT**: Do not attempt to build targets other than **chrome, chrome_sandbox,
nacl_helper**, or (optionally) **chromiumos_preflight** in Simple Chrome, they will
likely fail.

Congratulations, you've now built Chromium for Chromium OS!

Once you've built everything the first time, you can build incrementally:

```
(inside) autoninja -C out_${SDK_BOARD}/Release chrome
```

**Tip:** The default extensions will be installed by the test image you use below.

## Set up the Chromium OS device

Before you can deploy your build of Chromium to the device, it needs to have a
"test" OS image loaded on it. A test image has tools like rsync that are not part
of the end-user image.

### Create a bootable USB stick

Use your workstation browser to download a test image from the URL
`https://storage.cloud.google.com/chromeos-image-archive/$BOARD-release/<version>/chromiumos_test_image.tar.xz`
where `$BOARD` and `<version>` come from your SDK prompt. For example (`sdk link
R62-9800.0.0`) is the board `link` using version `R62-9800.0.0`).

Googlers: Prefer the above link for public boards. Images for non-public boards
are available on go/goldeneye.

After you download the compressed tarball containing the test image (it should
have "test" somewhere in the file name), extract the image by running:

```
(inside) tar xf ~/Downloads/<image-you-downloaded>
```

Copy the image to your drive using `cros flash`:

```
(inside) cros flash usb:// chromiumos_test_image.bin
```

If `cros flash` does not work you can do it the old-fashioned way using `dd`.
See below.

### Put your Chrome OS device in dev mode

**Note:** Switching to dev mode wipes all data from the device (for security
reasons).

Most recent devices can use the [generic instructions]. To summarize:

1. With the device on, hit Esc + Refresh (F2 or F3) + power button
2. Wait for the white "recovery screen"
3. Hit Ctrl-D to switch to developer mode (there's no prompt)
4. Press enter to confirm
5. Once it is done, hit Ctrl-D again to boot, then wait

From this point on you'll always see the white screen when you turn on
the device. Press Ctrl-D to boot.

Older devices may have [device-specific instructions].

Googlers: If the device asks you to "enterprise enroll" it, click the X
in the top-right of the dialog to skip it. Trying to use your google.com
credentials will result in an error.

### Enable booting from USB

By default Chromebooks will not boot off a USB stick for security reasons.
You need to enable it.

1. Start the device
2. Press Ctrl-Alt-F2 to get a terminal. (You can use Ctrl-Alt-F1 to switch
back if you need to.)
3. Login as `root` (no password yet, there will be one later)
4. Run `enable_dev_usb_boot`

### Install the test image onto your device

**Note:** Do not log into this test image with a username and password you care
about. The root password is public ("test0000"), so anyone with SSH
access could compromise the device. Create a test Gmail account and use that.

1. Plug the USB stick into the machine and reboot.
2. At the dev-mode warning screen, press Ctrl-U to boot from the USB stick.
3. Switch to terminal by pressing Ctrl-Alt-F2
4. Login as user `chronos`, password `test0000`.
5. Run `/usr/sbin/chromeos-install`
6. Wait for it to copy the image
7. Run `poweroff`

You can now unplug the USB stick.

### Connect device to Ethernet

Use your USB-to-Ethernet adapter to connect the device to a network.

Googlers: You can use a corp Ethernet jack, which will place you on a special
restricted network.

## Deploying Chrome to the device

To deploy the build to a device, you will need direct SSH access to it from your
computer. The scripts below handle everything else.

### Checking the IP address

1. Click the status area in the lower-right corner
2. Click the network icon
3. Click the circled `i` symbol in the lower-right corner
4. A small window pops up that shows the IP address

This also works both before and after login. Another option is to run `ifconfig`
from `crosh` (Ctrl-Alt-t) after guest login.

Try pinging that IP address from your Linux workstation.

### Using deploy_chrome

Just type `deploy_chrome` from with your chrome-sdk shell. It will use rsync to
incrementally deploy to the device.

Specify the build output directory to deploy from using `--build-dir`, and the
IP address of the target device (which must be ssh-able as user 'root') using
`--to`, where `%CHROME_DIR%` is the path to your Chrome checkout:

```
(inside) cd %CHROME_DIR%/src
(inside) deploy_chrome --build-dir=out_${SDK_BOARD}/Release --to=12.34.56.78
```

**Tip:** `deploy_chrome` lives under
`$CHROME_DIR/src/third_party/chromite/bin`. You can run `deploy_chrome` outside
of a `chrome-sdk` shell as well.

**Tip:** Specify the `--target-dir` flag to deploy to a custom location on the
target device. See `deploy_chrome --help` for other useful options.

Congratulations, you're now running your own Chromium build on your device!

## Updating the OS image

Every week you should update the Chrome OS image on the device so that
Chrome and Chrome OS don't get out of sync. Follow the above instructions.
(When http://crbug.com/761102 is fixed you'll be able to do this over the
network with `cros flash`.)

## Debugging

### Log files

Chrome-related logs are written to several locations on the device:

* `/var/log/ui/ui.LATEST` contains messages written to stderr by Chrome
  before its log file has been initialized.
* `/var/log/chrome/chrome` contains messages logged by Chrome before a
  user has logged in.
* `/home/chronos/user/log/chrome` contains messages logged by Chrome
  after a user has logged in.
* `/var/log/messages` contains messages logged by `session_manager`
  (which is responsible for starting Chrome), in addition to kernel
  messages when a Chrome process crashes.

### Command-line flags and environment variables

If you want to tweak the command line of Chrome or its environment, you have to
do this on the device itself.

Edit the `/etc/chrome_dev.conf` (device) file. Instructions on using it are in
the file itself.

### Custom build directories

This step is only necessary if you run `cros chrome-sdk` with `--nogn-gen`.

To create a GN build directory, run the following inside the chrome-sdk shell:

```
(inside) gn gen out_$SDK_BOARD/Release --args="$GN_ARGS"
```

This will generate `out_$SDK_BOARD/Release/args.gn`.

* You must specify `--args`, otherwise your build will not work on the device.
* You only need to run `gn gen` once within the same `cros chrome-sdk` session.
* However, if you exit the session or sync/update chrome the `$GN_ARGS` might
  change and you need to `gn gen` again.

You can edit the args with:

```
(inside) gn args out_$SDK_BOARD/Release
```

You can replace `Release` with `Debug` (or something else) for different
configurations. See **Debug build** section below.

[GN build configuration] discusses various GN build configurations. For more
info on GN, run `gn help` on the command line or read the [quick start guide].

### Debug build

For cros chrome-sdk GN configurations, Release is the default. A debug build of
Chrome will include useful tools like DCHECK and debug logs like DVLOG. For a
Debug configuration, specify
`--args="$GN_ARGS is_debug=true is_component_build=false"` (inside).

Alternately, you can just turn on DCHECKs for a release build. You can do this
with `--args="$GN_ARGS dcheck_always_on=true"`.

### Deploying debug build without stripping symbols

You need to add `--nostrip` to `deploy_chrome` because otherwise it will strip
symbols even from a debug build.  The rootfs will probably not be big enough to
hold all the binaries so you need to put it on the stateful partition and bind
mount over the real directory. Create the directory `/usr/local/chrome` on your
device and run:

```
(inside) deploy_chrome --build-dir=out_$BOARD/Debug \
                       --to=<ip-address> \
                       --target-dir=/usr/local/chrome \
                       --mount-dir=/opt/google/chrome \
                       --nostrip
```

Notes:

* If you just want crash backtraces in the logs you can deploy a release build
  with `--nostrip`. You don't need a debug build.
* The remount from `/usr/local/chrome` to `/opt/google/chrome` is transient, so
  you need to remount after reboot. Simplest way is to run the same command
  after reboot (It will not redeploy binaries less there is a change)
* You may hit `DCHECKs` during startup time, or when you login, which eventually
  may reboot the device. Please check log files in `/var/log/chrome` or
  `/home/chronos/user/log`.
     * You may create `/run/disable_chrome_restart` to prevent restart
       loop and investigate.
     * You can temporarily disable these `DCHECKs` to proceed, but please
       file a bug for such `DCHECK` because it's most likely a bug.
* You may not be able to create `/usr/local/chrome` until
  [rootfs has been removed] on the device, and the device has been
  [remounted as read-write].  The mounting will need to be applied on each
  boot. If the startup needs to be tested a symbolic link will be needed instead
     * ssh to device
          * `mkdir /home/chrome`
          * `rm -R /opt/google/chrome`
          * `ln -s /home/chrome /opt/google/chrome`
     * `deploy_chrome --build-dir=out_${SDK_BOARD}/Release --to=172.11.11.11`

### Remote GDB

Core dumps are disabled by default. See [additional debugging tips] for how to
enable core files.

On the target machine, open up a port for the gdb server to listen on, and
attach the gdb server to the top-level Chrome process.

```
(device) sudo /sbin/iptables -A INPUT -p tcp --dport 1234 -j ACCEPT
(device) sudo gdbserver --attach :1234 $(pgrep chrome -P $(pgrep session_manager))
```

On your host machine (inside the chrome-sdk shell), run gdb and start the Python
interpreter:

```
(inside) cd %CHROME_DIR%/src
(inside) gdb out_${SDK_BOARD}/Release/chrome
Reading symbols from /usr/local/google2/chromium2/src/out_link/Release/chrome...
(gdb) pi
>>>
```

**Note:** These instructions are for targeting an x86_64 device.  For now, to
target an ARM device, you need to run the cross-compiled gdb from within a
chroot.

Then from within the Python interpreter, run these commands:

```
import os
sysroot = os.environ['SYSROOT']
gdb.execute('set sysroot %s' % sysroot)
gdb.execute('set solib-absolute-prefix %s' % sysroot)
gdb.execute('set debug-file-directory %s/usr/lib/debug' % sysroot)
gdb.execute('set solib-search-path out_%s/Release/lib' % os.environ['SDK_BOARD'])  # Or "Debug" for a debug build
gdb.execute('target remote 12.34.56.78:1234')
```

If you wish, after you connect, you can Ctrl-D out of the Python shell.

Extra debugging instructions are located at http://www.chromium.org/chromium-os/how-tos-and-troubleshooting/debugging-tips.

## Additional instructions

### Updating the version of the Chrome OS SDK

When you invoke `cros chrome-sdk`, the script fetches the version of the SDK
that corresponds to your Chrome checkout.  To update the SDK, sync your Chrome
checkout and re-run `cros chrome-sdk`.

**IMPORTANT NOTES:**

* Every time that you update Chrome or the Chrome OS SDK, it is possible that
  Chrome may start depending on new features from a new Chrome OS image. This
  can cause unexpected problems, so it is important to update your image
  regularly. Instructions for updating your Chrome OS image are above in
  [Set Up the Chromium OS device].
* Don't forget to re-configure your custom build directories if you have them
  (see **Custom build directories** below).

### Specifying the version of the Chrome OS SDK to use

You can specify a version of Chrome OS to build against. This is handy for
tracking down when a particular bug was introduced.

```
(outside) cros chrome-sdk --board=$BOARD --version=3680.0.0
```

Once you are finished testing the old version of the chrome-sdk, you can always start a new shell with the latest version again. Here's an example:

```
(outside) cros chrome-sdk --board=$BOARD
```

### Updating Chrome

**Tip:** If you update Chrome inside the chrome-sdk, you may then be using an SDK
that is out of date with the current Chrome.
See **Updating the version of the Chrome OS SDK** section above.

```
(inside) git rebase-update
(inside) gclient sync
```

### Updating Deployed Files

Since the gyp files don't define which targets get installed, that information
is maintained in the [chromite repo] as part of Chromium OS. That repo is also
integrated into the Chromium source tree via the `DEPS` file.

In order to add/remove a file from the installed list:

1. Go into the chromite directory and modify `lib/chrome_util.py`
     1. Look for the `_COPY_PATHS` list
     2. Add your new file with optional=True
2. Commit your change locally using git
3. Upload your change to gerrit<br>
   `git push origin master:refs/for/master`
4. Get it reviewed by someone on the Chromium OS build team
   (e-mail chromium-os-dev@ if you can't find anyone)
5. Merge the change into Chromium OS via gerrit
6. Update the DEPS file in Chromium to use the new chromite sha1
7. Check in the Chromium change like normal
8. Once everything has settled, then go back and remove the `optional=True`
   from the file list<br>
   Unless the file is actually optional, then keep it

### Using a custom Chromium OS build from your Chromium OS chroot (optional)

If you are making changes to Chromium OS and have a Chromium OS build inside a
chroot that you want to build against, run `cros chrome-sdk` with the `--chroot`
option.

```
(outside) cros chrome-sdk --board=$BOARD --chroot=/path/to/chromiumos/chroot
```

### Flashing an image to USB using dd

In the below command, the X in sdX is the path to your usb key, and you can use
dmesg to figure out the right path (it'll show when you plug it in). Make sure
that the USB stick is not mounted before running this. You might have to turn
off automounting in your operating system.

```
(inside) sudo dd if=chromiumos_test_image.bin of=/dev/sdX bs=1G iflag=fullblock oflag=sync
```

Be careful - you don't want to have `dd` write over your root partition!

### Using cros flash with xbuddy to download images

`cros flash` with `xbuddy` will automatically download an image and write it to
USB for you. It's very convenient, but for now it requires a full Chrome OS
checkout and must be run inside the Chrome OS chroot. ([issue 437877])

```
(chroot) cros flash usb:// xbuddy://remote/$BOARD/<version>/test
```

Replace `$BOARD` and `<version>` with the right values. Both can be seen in your
SDK prompt (e.g. `(sdk lumpy R27-3789.0.0)` is the lumpy board using version
R27-3789.0.0).

See the [Cros Flash page] for more details.

### Setting a custom prompt (optional)

By default, cros chrome-sdk prepends something like '`(sdk link R52-8315.0.0)`'
to the prompt (with the version of the prebuilt system being used).

If you prefer to colorize the prompt, you can set `PS1` in
`~/.chromite/chrome_sdk.bashrc`, e.g. to prepend a yellow '`(sdk link 8315.0.0)`'
to the prompt:

```
PS1='\[\033[01;33m\](sdk ${SDK_BOARD} ${SDK_VERSION})\[\033[00m\] \w \[\033[01;36m\]$(__git_ps1 "(%s)")\[\033[00m\] \$ '
```
NOTE: Currently the release version (e.g. 52) is not available as an environment variable.

### GYP

The legacy `GYP` build system is no longer supported.

[OS development guide]: https://www.chromium.org/chromium-os/developer-guide#TOC-Making-changes-to-the-Chromium-web-
[Local copy of the Chromium source code and depot_tools]: https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md
[Chromium OS board name]: http://www.chromium.org/chromium-os/developer-information-for-chrome-os-devices
[GN build configuration]: http://www.chromium.org/developers/gn-build-configuration
[quick start guide]: https://chromium.googlesource.com/chromium/src/+/master/tools/gn/docs/quick_start.md
[device-specific instructions]: https://www.chromium.org/chromium-os/developer-information-for-chrome-os-devices
[generic instructions]: https://www.chromium.org/a/chromium.org/dev/chromium-os/developer-information-for-chrome-os-devices/generic
[rootfs has been removed]: https://www.chromium.org/chromium-os/poking-around-your-chrome-os-device#TOC-Making-changes-to-the-filesystem
[remounted as read-write]: http://www.chromium.org/chromium-os/how-tos-and-troubleshooting/debugging-tips#TOC-Setting-up-the-device
[additional debugging tips]: http://www.chromium.org/chromium-os/how-tos-and-troubleshooting/debugging-tips#TOC-Enabling-core-dumps
[Set Up the Chromium OS device]: http://www.chromium.org/chromium-os/how-tos-and-troubleshooting/building-chromium-browser#TOC-Set-up-the-Chromium-OS-device
[chromite repo]: https://chromium.googlesource.com/chromiumos/chromite/
[issue 437877]: https://bugs.chromium.org/p/chromium/issues/detail?id=403086
[Cros Flash page]: http://www.chromium.org/chromium-os/build/cros-flash
