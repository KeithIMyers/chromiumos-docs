# GN in platform2

New packages should use
[GN](https://gn.googlesource.com/gn/+/master/docs/reference.md), not GYP, to
build binaries.

See the official [step-by-step introduction](
https://gn.googlesource.com/gn/+/master/docs/quick_start.md#Step_by_step) for
the GN basics. This article discusses platform2 specific stuff.

## How to build your package with GN

Example: [arc/adbd/BUILD.gn](
https://chromium.googlesource.com/chromiumos/platform2/+/master/arc/adbd/BUILD.gn)

- Put `BUILD.gn` in your package directory, which is determined by
  `PLATFORM_SUBDIR` in your ebuild. Existence of `BUILD.gn` indicates to the
  platform2 build system that GN should be used to build this package.
- Have a target named `"all"` in the `BUILD.gn`. It's the root target built
  by the platform2 build system. Typically it's a `group` target depends on all
  the targets to be built.
- `common-mk/` contains common templates or common settings, which your build
  files can utilize.
    - `pkg_config.gni` defines the `pkg_config` rule to generate configs for
      package dependencies.
    - `BUILDCONFIG.gn` defines the default configs for each target type (e.g.
      executable). You can remove default configs in individual target with
      `configs -=` if needed (example: [hammard/BUILD.gn](https://chromium-review.googlesource.com/c/chromiumos/platform2/+/1149942/8/hammerd/BUILD.gn)).
      (TODO(oka): Replace the URL after the CL is submitted)

## How to write ebuilds

Example: [arc-adbd-9999.ebuild](
https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/chromeos-base/arc-adbd/arc-adbd-9999.ebuild)

Note we should add `.gn` in `CROS_WORKON_SUBTREE` so that the platform2 build
system can access the file.

## How to check USE flags in GN

Example: [hammerd/BUILD.gn](
https://chromium-review.googlesource.com/c/chromiumos/platform2/+/1149942/8/hammerd/BUILD.gn)
(TODO(oka): update the link after the CL is submitted)

In GN files, USE flags can be referred as `use.foo`.

Only whitelisted USE flags can be used. If you need to use new USE flags,
update the following files:
- `_IUSE` constant in `platform2/common-mk/platform2.py`
- IUSE variable in your ebuild file

## How to write unit tests

`use.test` flag is set to true on unit testing. Enclose test only targets with
`if (use.test) {}` to reduce compile time on production.
The test targets are typically executables which depend on `//common-mk:test` to
use gtest and gmock.


How to run unit tests is same as before: In chroot, run
`cros_workon --board=$BOARD start $package_name` if you haven't. Then run

```
cros_workon_make --board=$BOARD --test $package_name
```
.

Prepend VERBOSE=1 to see the commands run by GN (plus other logs).
