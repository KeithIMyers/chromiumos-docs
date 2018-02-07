# Rust on Chrome OS

This provides information on creating [Rust] projects for installation within Chrome OS and Chrome
OS SDK. All commands and paths given are from within the SDK's chroot.

[TOC]

## Install

Rust and Cargo is already installed in all current SDKs. The particular flavor of Rust that is
installed is targetable to `x86_64`, `armv7a`, and `aarch64`, but requires the installation of
cross-compiling toolchains on chroots that haven't already setup boards for each target
architecture. To quickly install the extra toolchains, run the following:

```shell
sudo $(which cros_setup_toolchains) --targets=boards --include-boards=kevin,lumpy
```

Place the following contents into `~/.cargo/config` to enable cross-compiling:

```toml
[target.armv7a-cros-linux-gnueabi]
linker = "armv7a-cros-linux-gnueabi-gcc"

[target.aarch64-cros-linux-gnu]
linker = "aarch64-cros-linux-gnu-gcc"

[target.x86_64-cros-linux-gnu]
linker = "x86_64-cros-linux-gnu-gcc"
```

The toolchain is configured to use the system allocator (malloc) in resulting binaries instead of
the usual jemalloc. This is to avoid putting a copy of the jemalloc routines in every binary, which
is a significant source of Rust binary bloat.

## Usage

Rust projects can be written and built in the usual fashion using Cargo. Projects intended for usage
on Chrome OS should be located in `~/trunk/src/platform/<project_name>`. If the project is intended
for installation, create an ebuild for it at

`~/trunk/src/third_party/chromiumos-overlay/chromeos-base/<project_name>/<project_name>-9999.ebuild`

with the following template:


```bash
# Copyright <copyright_year> The Chromium OS Authors. All rights reserved.
# Distributed under the terms of the GNU General Public License v2

EAPI=5
CROS_WORKON_PROJECT="chromiumos/platform/<project_name>"
CROS_WORKON_LOCALNAME="../platform/<project_name>"
CROS_WORKON_INCREMENTAL_BUILD=1

CRATES="
example_crate-1.0.0
important_crate-0.2.23
<crate_name>-<crate_version>
"

inherit cargo cros-workon

DESCRIPTION="Incredible Rust Project"

LICENSE="<project_license>"
SLOT="0"
KEYWORDS="~*"
IUSE=""

SRC_URI="$(cargo_crate_uris ${CRATES})"
```

Thanks to the [cargo.eclass], the ebuild for Rust projects should be quite minimal. As long as the
project builds with `cargo build` and is installed with `cargo install` the defaults inherited from
the eclass should work fine.

There is some non-trivial amount of work related to the `CRATES` variable. All Rust crate
dependencies must be listed in `CRATES`, even transitive and build time dependencies. This is so
that a proper `Manifest` file can be generated and so the `ebuild` system can download all necessary
crates from the Chrome OS source mirror before building the project. The line starting with
`SRC_URI` utilizes the inherited cargo eclass to generate all the source URLs needed by this
project.

## Depending on Crates

Because the sources for all ebuilds in Chrome OS must be available at [localmirror] (link only
accessible with Google account), you will have to upload all crate dependencies for the project to
localmirror.

The following will download a crate, upload it to localmirror, and make it accessible for download:

> **WARNING**: localmirror is shared by all Chrome OS developers. If you break it, everybody will have a bad day.

```shell
curl -L 'https://crates.io/api/v1/crates/<crate_name>/<crate_version>/download' >/tmp/crates/<crate_name>-<crate_version>.crate
gsutil cp /path/to/crates/<crate_name>-<crate_version>.crate gs://chromeos-localmirror/distfiles/
gsutil acl ch -u AllUsers:R 'gs://chromeos-localmirror/distfiles/<crate_name>-<crate_version>.crate'
```

## Tips

### Release Profile

Because space is always a premium on Chrome OS devices, put the following in any executable Rust
project's `Cargo.toml`.

```toml
[profile.release]
lto = true
panic = 'abort'
```

This will cause release builds (made with `cargo build --release`) to be built with [link time
optimizations] and for the panic handler to be a simple abort. The optimizations help to minimize
dead code brought in by dependencies. The removal of the default panic handler will save a bit of
space at the price of less useful output on panic. The largest size reduction by far from this is
that the resulting binary can be stripped. Experience shows that this can shave off about half of
the binary size.

Note that building with the release profile will be significantly slower and less friendly to
debugging panics at runtime.

### Cross-compiling

The toolchain that is installed by default is targetable to the following triples:

| Target Triple               | Description                                                                      |
|-----------------------------|----------------------------------------------------------------------------------|
| `x86_64-pc-linux-gnu`       | **(default)** Used exclusively for packages installed in the chroot              |
| `armv7a-cros-linux-gnueabi` | Used by 32-bit usermode ARM devices                                              |
| `aarch64-cros-linux-gnu`    | Used by 64-bit usermode ARM devices (none of these exist as of August 4th, 2017) |
| `x86_64-cros-linux-gnu`     | Used by x86_64 devices                                                           |

When building Rust projects for development, a non-default target can be selected as follows:

```shell
cargo build --target=<target_triple>
```

If a specific board is being targeted, that board's sysroot can be used for compiling and linking
purposes by setting the `SYSROOT` environment variable as follows:

```shell
export SYSROOT="/build/<board>"
```
If C files are getting compiled with a build script that uses the `cc` or `gcc` crates, you may
also need to set the `TARGET_CC` environment variable to point at the appropriate C compiler.

```shell
export TARGET_CC="<target_triple>-clang"
```

[Rust]: https://www.rust-lang.org
[Cargo]: https://crates.io/
[cargo.eclass]: https://chromium.googlesource.com/chromiumos/overlays/portage-stable/+/master/eclass/cargo.eclass
[localmirror]: go/localmirror
[link time optimizations]: https://en.wikipedia.org/wiki/Interprocedural_optimization
