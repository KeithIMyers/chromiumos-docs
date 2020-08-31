# How to get a stack trace at runtime for debugging purposes

While debugging it's often useful to print stack traces at runtime. This
document describes how to do so.

[TOC]

## Typography conventions


| Label         | Paths, files, and commands                            |
|---------------|-------------------------------------------------------|
|  (shell)      | outside the chroot and SDK shell on your workstation  |
|  (sdk)        | inside the `chrome-sdk` SDK shell on your workstation |
|  (device)     | in your [VM] or Chrome OS device                      |

## Chromium

### How to build and deploy a Chrome that can print runtime stack traces

#### Making sure `exclude_unwind_tables` is false

You can skip this section if you are not building an official build, i.e.
not passing either `--internal` or `--official` to chrome-sdk, and not
explicitly setting `is_official_build=true` in your gn arguments.

However, if you are building an official build then `exclude_unwind_tables`
will be true by default. To add flags to gn args, you can either [add them via]
`--gn-extra-args`:

```sh
(shell) cros chrome-sdk --board=eve --gn-extra-args="exclude_unwind_tables=false"
```

Or, you can run the following command, which will let you edit
the arguments directly. N.B. your path to your build directory may be
different, and re-running `cros chrome-sdk` will overwrite these.

```sh
(sdk) gn args out_${SDK_BOARD}/Release
```

Then append `exclude_unwind_tables=false`.

[add them via]: https://chromium.googlesource.com/chromiumos/docs/+/master/simple_chrome_workflow.md#cros-chrome_sdk-options

#### Deploying Chrome

After rebuilding (see example command below), make sure to pass the
`--nostrip` or `--strip-flags=-S` flags to `deploy_chrome`. The `--nostrip`
flag will usually result in a much bigger binary, so if you only need stack
traces at runtime, prefer `--strip-flags=-S`.

```sh
# Rebuild example
(sdk) autoninja -C out_${SDK_BOARD}/Release chrome nacl_helper
# Deploy chrome example
(sdk) deploy_chrome --build-dir=out_${SDK_BOARD}/Release --device=DUT --mount --strip-flags=-S
```

Note that the effects of the `--mount` option will not survive a reboot. If you
reboot, re-run deploy_chrome.

As of the time of writing (2020/06), `--mount` is optional as we have enough
space on the rootfs.

### How to use base::StackTrace

Next, include the following header in the file from which you want runtime
stack traces:

```c
#include "base/debug/stack_trace.h"
```

Then, add this code to where you want the stack trace.

```c
LOG(ERROR) << base::debug::StackTrace();
```

This code will output to the standard Chrome log, which is accessible via:

```sh
(device) tail -F /var/log/chrome/chrome
```

See the [chrome os logging] document for further info.

[chrome os logging]: https://chromium.googlesource.com/chromium/src.git/+/master/docs/chrome_os_logging.md

It's also possible to log the stack trace to stderr with the following code:

```c
base::debug::StackTrace().Print();
```

However, this prints to stderr rather than the standard log, so it won't
show up in `/var/log/chrome/chrome`.

Be aware that taking a runtime stack trace is an expensive operation, so if
you put it in a hot codepath it can make Chrome's performance very bad.
In timing dependent code (e.g. race conditions, graphics related stuff),
it can even change the result.

Alternatively, you can save the stack trace (before printing it) and print it
out at a later time:

```c
auto stack = base::debug::StackTrace();
LOG(ERROR) << stack;
```

For reference, getting the stack trace itself takes on the order of
milliseconds. Symbolizing the stack trace takes on the order of hundreds
of milliseconds.

### Printing stack traces from the GPU process

Printing a stack trace at runtime requires access to /proc/self/maps among
other files. The GPU process does not have access to this, so stack traces
will show unsymbolized. To symbolize them, add the following flag to your
`/etc/chrome_dev.conf`:

```sh
--disable-gpu-sandbox
```

To know if you are in the GPU process or not, add a `LOG(ERROR)` to the code
you are inspecting, and get the process ID. For example, it's 8689 in the below
example:

```[8689:8689:0612/144957.826867:ERROR:window_state.cc(866)]```

Then get the chrome invocation for that PID:

```sh
(device) cat /proc/8689/cmdline
```

This should give something like this:

```sh
/opt/google/chrome/chrome --type=gpu-process <...>
```

In particular, `--type=gpu-process` tells you it is the GPU process.

### Printing stack traces from a renderer process

Similar to the GPU process, it requires permissions. You can identify the
process by its invocation having `--type=renderer`. To get runtime stack traces
while in a renderer process by adding the following flag to your
`/etc/chrome_dev.conf`:

```sh
--no-sandbox
```

### Debugging runtime stack traces not being symbolized

Sometimes stack traces are printed without being symbolized ("<unknown>").
If this happens, there are a few things to check:

1\. Check `exclude_unwind_tables` is false.

2\. Check the process you are printing from isn't sandboxed.

Add the sandbox disabling options mentioned in the document to rule this out.
Processes need to be able to access files in `/proc/self/` to inspect their
memory maps.

3\. Check that the binary on the DUT (device-under-test) hasn't been stripped.

Make sure deploy_chrome is run with `--nostrip` or `--string-flags=-S`. You can
compare the size between the binary on your machine and the DUT to check
further.

4\. Check that you are running the correct chrome binary on the DUT.

Re-running `deploy_chrome` with the `--mount` option can rule this out.

5\. Check that the appropriate symbol_level gn argument is set.

Not setting `symbol_level` should be sufficient to avoid this issue.
However, you can rule this being a problem out by explicitly specifying
`symbol_level=1`.

See the below table to see which configurations will let you print stack
traces if you are unsure. However, building with symbol_level=-1 (default) and
exclude_unwind_tables=false should let you print runtime stack traces.

Stripping with `--strip-flags=-S` vs `--nostrip` produces a much smaller
binary if you are building with symbol_level=1 or symbol_level=2. For stack
trace purposes, symbol_level=1 should be enough.

| symbol_level | deploy_chrome arg |  size  | runtime traces? |
|--------------|-------------------|--------|-----------------|
| 2            | nostrip           | 1.7 GB | yes             |
| 2            | strip-flags=-S    | 282 MB | yes             |
| 2            | <none>            | 191 MB | no              |
| 1            | nostrip           | 1.7 GB | yes             |
| 1            | strip-flags=-S    | 282 MB | yes             |
| 1            | <none>            | 191 MB | no              |
| 0            | nostrip           | 284 MB | x86/x64 only    |
| 0            | strip-flags=-S    | 282 MB | x86/x64 only    |
| 0            | <none>            | 191 MB | no              |
(data collected on eve, 2020/06, ~M85)

x86/x64 symbols available with symbol_level=0 ([source]).

[source]: https://source.chromium.org/chromium/chromium/src/+/master:build/config/compiler/compiler.gni;l=222;drc=97cb4f90bf93714f139f5d0b8702256499a42075

### Example stack trace output

```c
#0 0x5788e671fe19 base::debug::CollectStackTrace()
#1 0x5788e66f2eb3 base::debug::StackTrace::StackTrace()
#2 0x5788e62bb264 content::ContentMainRunnerImpl::RunServiceManager()
#3 0x5788e62baca1 content::ContentMainRunnerImpl::Run()
#4 0x5788e62dc7b1 service_manager::Main()
#5 0x5788e62b916e content::ContentMain()
#6 0x5788e4127b65 ChromeMain
#7 0x7c58b7c8dad4 __libc_start_main
#8 0x5788e41279fa _start
```
