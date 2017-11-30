# Chrome OS VM for Chromium developers

This workflow allows developers with a Chromium checkout to download and launch
a Chrome OS VM on their workstations, update the VM with locally built Chrome,
and run various tests.

[TOC]

## Prerequisites

1.  [depot_tools installed]
2.  [Linux Chromium checkout]
3.  [qemu-kvm downloaded]
4.  [Virtualization enabled]: To check if kvm is already enabled: `sudo kvm-ok`.
    For Goobuntu, interrupt the BIOS bootup with Esc for Options, F10 for
    Computer Setup, in the Security menu, System Security tab,
    Enable Virtualization Technology in the overlay.

## Typography conventions

| Label         | Paths, files, and commands                             |
|---------------|--------------------------------------------------------|
|  (shell)      | on your build machine, outside the sdk/chroot          |
|  (sdk)        | inside the `chrome-sdk` [Simple Chrome] shell          |
|  (chroot)     | inside the `cros_sdk` [chroot]                         |
|  (vm)         | inside the VM ssh session                              |


## Install QEMU
Download QEMU from [go/cros-qemu], and extract the files.
```bash
(shell) $ cd ~ && tar xvf ~/Downloads/qemu.tar.gz
```
You can now specify the qemu path and qemu bios path when running
`cros_vm --start`
```bash
(sdk) .../chrome/src $ cros_vm --start \
--qemu-path ~/qemu/bin/qemu-system-x86_64 \
--qemu-bios-path ~/qemu/share
```

Alternatively, you can use the qemu in the chromeos [chroot], if you have it:
```bash
(sdk) .../chrome/src $ cros_vm --start \
--qemu-path <cros_code_dir>/chroot/usr/bin/qemu-system-x86_64
```

## Download the VM

cd to your Chromium repository, and enter the [Simple Chrome] environment with
`--download-vm`:
```bash
(shell) .../chrome/src $ cros chrome-sdk --board=amd64-generic \
--download-vm --clear-sdk-cache --log-level info
```

### chrome-sdk options

*   `--download-vm` downloads a pre-packaged VM (takes a few minutes).
*   `--clear-sdk-cache` recommended, clears the cache.
*   `--debug` for debug output.
*   `--board=betty` will download an ARC-enabled VM (Googler-only at the moment)
*   `--internal` will set $GN_ARGS to build and deploy an internal Chrome build.
*   `--version` to download a non-LKGM version, eg 10070.0.0.

Some boards do not generate VM images. `amd64-generic` and `betty` (for ARC,
internal only) are recommended.

## Launch a Chrome OS VM

From within the [Simple Chrome] environment:
```bash
(sdk) .../chrome/src $ cros_vm --start
```

### Stop the VM

```bash
(sdk) .../chrome/src $ cros_vm --stop
```
Note that the window size of the VM is a bit small so the OOBE screen is cut
off. There’s also no [mouse cursor].

## Remotely run a sanity test in the VM

```bash
(sdk) .../chrome/src $ cros_vm --cmd -- /usr/local/autotest/bin/vm_sanity.py
```
The command output on the VM will be output to the console after the command
completes. Other commands run within an ssh session below can also run with
`--cmd`, e.g. `run_tests`.

> Note that the CrOS test private RSA key cannot be world readable, so you may
need to do:
```bash
(shell) .../chrome/src $ chmod 600 ./third_party/chromite/ssh_keys/testing_rsa
```

## SSH into the VM

```bash
(sdk) .../chrome/src $  ssh root@localhost -p 9222
```
> Password is `test0000`

To avoid having to type a password and skip the RSA key warning:
```bash
(sdk) .../chrome/src $ ssh -o UserKnownHostsFile=/dev/null -o \
StrictHostKeyChecking=no -i ./third_party/chromite/ssh_keys/testing_rsa \
root@localhost -p 9222
```

## Run a local sanity test in the VM

```
(vm) localhost ~ # /usr/local/autotest/bin/vm_sanity.py
```

## Run telemetry unit tests in the VM

SSH into the VM:
```bash
(vm) localhost ~ # python \
/usr/local/telemetry/src/third_party/catapult/telemetry/bin/run_tests [test]
```
Or from your workstation:
```bash
(shell) .../chrome/src $ ./third_party/catapult/telemetry/bin/run_tests \
--browser=cros-chrome --remote=localhost --remote-ssh-port=9222 [test]
```
Catapult developers can run this from their catapult checkout.

## Update Chrome in the VM

### Build Chrome

For testing local Chrome changes in a Chrome OS environment, use the [Simple
Chrome] flow to build Chrome (after entering the Simple Chrome environment as
described above):
```bash
(sdk) .../chrome/src $ gn gen out_$SDK_BOARD/Release --args="$GN_ARGS"
(sdk) .../chrome/src $ ninja -C out_$SDK_BOARD/Release/ -j 1000 -l 20 \
chrome chrome_sandbox nacl_helper
```

### Launch the VM

```bash
(sdk) .../chrome/src $ cros_vm --start
```

### Deploy your Chrome to the VM

```bash
(sdk) .../chrome/src $ deploy_chrome \
--build-dir=out_$SDK_BOARD/Release/ --to=localhost --port=9222
```

## Run an autotest in the VM

From inside your [chroot]:
```bash
(chroot) ~/trunk/src/scripts $ test_that localhost:9222 login_Cryptohome
```

## Run an ARC++ test in the VM

Download the betty VM:
```bash
(sdk) .../chrome/src $ cros chrome-sdk --board=betty --download-vm
```
vm_sanity will detect and run an ARC++ test:
```bash
(vm) localhost ~ # /usr/local/autotest/bin/vm_sanity.py
```
Run a cheets autotest from within your [chroot]:
```bash
(chroot) ~/trunk/src/scripts $ ./build_packages --board=betty
(chroot) ~/trunk/src/scripts $ test_that localhost:9222 \
cheets_ContainerMount
```

## Launch a VM built by a waterfall bot

Find a waterfall bot of interest, such as
[amd64-generic-tot-chromium-pfq-informational], which is a FYI bot that builds
TOT Chrome with TOT Chrome OS, or [amd64-generic-chromium-pfq], which is
an internal PFQ builder. Pick a build, click on artifacts, and download
`chromiumos_qemu_image.tar.xz` to `~/Downloads/`

Unzip:
```bash
(shell) $ tar xvf ~/Downloads/chromiumos_qemu_image.tar.xz
```
Launch a VM from within the [Simple Chrome] environment:
```bash
(sdk) .../chrome/src $ cros_vm --start \
--image-path  ~/Downloads/chromiumos_qemu_image.bin

```

## Launch a locally built VM

Follow instructions to [build Chromium OS] and a VM image. In the [chroot]:
```bash
(chroot) ~/trunk/src/scripts $ export BOARD=betty
(chroot) ~/trunk/src/scripts $ ./setup_board --board=$BOARD
(chroot) ~/trunk/src/scripts $ ./build_packages --board=$BOARD
(chroot) ~/trunk/src/scripts $ ./build_image \
--noenable_rootfs_verification test --board=$BOARD
(chroot) ~/trunk/src/scripts $ ./image_to_vm.sh --test_image
```
From the [Simple Chrome] environment:
```bash
(sdk) .../chrome/src $ cros_vm  --start \
--image-path \
<cros_code_dir>/src/build/images/$SDK_BOARD/latest/chromiumos_qemu_image.bin
```

## Start a VM and run a basic set of tests

This is intended for use by a builder:
```bash
(sdk) .../chrome/src $ cros_run_vm_test
```
This doc is at [go/cros-vm]. Please send feedback to [achuith@chromium.org].

[depot_tools installed]: https://www.chromium.org/developers/how-tos/install-depot-tools
[qemu-kvm downloaded]:
https://chromium.googlesource.com/chromiumos/docs/+/master/cros_vm.md#install-qemu
[go/cros-qemu]:
https://storage.cloud.google.com/achuith-cloud.google.com.a.appspot.com/qemu.tar.gz
[Linux Chromium checkout]: https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md
[Virtualization enabled]: https://g3doc.corp.google.com/tools/android/x20/crow/g3doc/enable_kvm.md?cl=head
[Simple Chrome]: https://chromium.googlesource.com/chromiumos/docs/+/master/simple_chrome_workflow.md
[mouse cursor]: https://bugs.chromium.org/p/chromium/issues/detail?id=604922
[chroot]: https://www.chromium.org/chromium-os/developer-guide
[amd64-generic-tot-chromium-pfq-informational]: https://build.chromium.org/p/chromiumos.chromium/builders/amd64-generic-tot-chromium-pfq-informational
[amd64-generic-chromium-pfq]: https://uberchromegw.corp.google.com/i/chromeos/builders/amd64-generic-chromium-pfq
[build Chromium OS]: https://www.chromium.org/chromium-os/developer-guide
[go/cros-vm]: https://chromium.googlesource.com/chromiumos/docs/+/master/cros_vm.md
[achuith@chromium.org]: mailto:achuith@chromium.org
