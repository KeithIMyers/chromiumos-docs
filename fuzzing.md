# How to do fuzz testing on Chrome OS

The target audience for this document are developers in Chrome OS who work on
packages. It explains how to write a fuzz test target for a package, which
will then be automatically picked up and run regularly by ClusterFuzz. It
assumes some basic familiarity with the Chrome OS development environment.

[TOC]

## What is fuzz testing?

Fuzz testing (or "fuzzing") is the process of testing an application by feeding
invalid, malformed, or malicious inputs to a target program in an attempt to
crash it. It is particularly useful for testing APIs because it can play the
part of an attacker sending malicious input to the API.

The input data is created by randomly mutating other inputs. Sometimes the
initial inputs are provided by the user in what is known as a *seed corpus*,
otherwise the fuzzer will generate inputs from scratch. Coverage-guided fuzz
testing is an extension of fuzz testing that explicitly attempts to increase
code coverage when it generates new tests, adding the new tests to its test
corpus when it finds tests that increase coverage, i.e. when a new test executes
a new code path in the target program.

LLVM has a built-in fuzzing engine, [libFuzzer], that is provided as part of
LLVM's sanitizer instrumentation. Google has a cross-platform fuzzing
infrastructure, [ClusterFuzz], that works with libFuzzer and aids developers
by providing an end-to-end pipeline that automatically picks up new (and
existing) fuzz targets, runs fuzz testing on these targets, reports bugs
(assigning them to appropriate owners), and even verifies fixes. Chromium on
Linux and MacOS has been using [libFuzzer and ClusterFuzz] for a while, and
they have proved very useful, finding thousands of [security bugs] and
[non-security bugs].

A fuzz test target is nothing more than a piece of code that can take a chunk
of bytes and do something with them by calling the code to be tested. For the
remainder of this document we will use the word *fuzzer* to refer to a fuzz test
target that gets linked against libFuzzer to produce an executable that
exercises the code under test. This is what ClusterFuzz will run.

Chrome OS has tooling to support writing fuzzers for Chrome OS packages. This
document will walk you through the steps necessary to get your fuzzers up and
running on ClusterFuzz.

## Quickstart guide

This section of the document describes the basic steps needed to write a fuzzer
in Chrome OS and have ClusterFuzz pick it up. For more detailed instructions,
see the [Detailed instructions](#Detailed-instructions) section.

If you are working on a platform package (one that is in the `platform` or
`platform2` source directory and whose ebuild inherits from the `platform`
eclass), see the
[Build a fuzzer for a platform package](#Build-a-fuzzer-for-a-platform-package)
section.

### Steps to create a new fuzzer in Chrome OS

*   Write a new test program in your package, whose name ends in `_fuzzer` and
    which defines a function `LLVMFuzzerTestOneInput` with the following
    signature:

    ```c
    extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
        <your test code goes here>
        return 0;
    }
    ```

    Ensure `LLVMFuzzerTestOneInput` calls the function you want to fuzz.

*   Update the build system for your package to build your `*_fuzzer` binary.
*   Update your package's ebuild file:
    1.  Add `asan` and `fuzzer` to the `IUSE` flags list.
    2.  Build you new fuzzer (conditioned on `use fuzzer`), with the
        appropriate flags:
        1.  [Inherit flag-o-matic toolchain-funcs]
        2.  [Set up flags: call asan-setup-env & fuzzer-setup-env]
        3.  [USE flags: fuzzer]
    3.  Install your binary in `/usr/libexec/fuzzers/`
    4.  Build the libraries your fuzzer depends on with the appropriate
        `-fsanitize` flags (optional).
*   Build and test your new fuzzer locally. Commit your changes.
*   Add the package dependency to the `chromium-os-fuzzers` ebuild, and commit
    the change.

That's it! The continuously running [fuzzer builder] on the Chrome OS waterfall
will automatically detect your new fuzzer, build it and upload it to
ClusterFuzz, which will start running it. For more details on what you can do
with ClusterFuzz, see the [Using ClusterFuzz](#Using-ClusterFuzz) section.

## Detailed instructions

This section goes over the steps mentioned in the
[Quickstart guide](#Quickstart-guide) in more detail. If you're working on a
platform package, read
[Build a fuzzer for a platform package](#Build-a-fuzzer-for-a-platform-package).
Otherwise, read
[Build a fuzzer for any other package](#Build-a-fuzzer-for-any-other-package).

### Build a fuzzer

It might be counterintuitive to start by building a fuzzer before one is even
written, but the process of actually writing the fuzzer will be easier with
a fast compile-test cycle, for which we need to be able to compile the fuzzer.

Start with a dummy fuzzer:

```cpp
// Copyright 2018 The Chromium OS Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

struct Environment {
  Environment() {
    // Set-up code.
  }
};

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
  static Environment env;
  // Fuzzing code. Empty for now.
  return 0;
}
```

#### Build a fuzzer for a platform package

These steps assume that you have at least a dummy fuzzer to compile. Check out
the previous section for an example.

1.  Update the build system for your package to build your `*_fuzzer` binary.
    *   For packages that are built with GYP files, update your GYP file to
        build the fuzzer binary:

        ```
        # Fuzzer target.
        ['USE_fuzzer == 1', {
          'targets': [
            {
              'target_name': 'my_fuzzer',
              'type': 'executable',
              'includes': [
                '../common-mk/common_fuzzer.gypi',
              ],
              'dependencies': [
                # This could be an intermediate static library target in your
                # package.
              ],
              'sources': [
                'my_fuzzer.cc',
              ],
            },
          ],
        }],
        ```

        See the [midis GYP file] for a complete example.

    *   If your package is not built with a GYP file, then you will need to
        update your Makefile (or whatever build system you use), in such a way
        that your fuzzer binary gets built when a special flag or argument is
        passed to the normal build command.

        If testing this by hand (without the ebuild file changes in step 2
        below), you will need to manually pass the compiler flags
        `-fsanitize=address -fsanitize=fuzzer-no-link` and the linker flags
        `-fsanitize=address -fsanitize=fuzzer` to your build. You will not
        need to pass these flags manually once you have updated the ebuild
        file.

2.  Update your package's ebuild file:
    1.  Update the ebuild file to build the new binary when the fuzzer
        USE flag is being used. For platform packages built with GYP files,
        you should skip this step and go directly to the next step for
        installing the fuzzer binary.

        1.  Find the `src_compile()` function in your ebuild file. If
            there isn't one, add one:

            ```bash
            src_compile() {
            }
            ```

        2.  Add the call to actually build your fuzzer.

            Find the line in your `src_compile` function that actually builds
            your package (the command will probably look like `emake` or `make`
            or `cmake`). This is the command that is meant by
            *original build command* below. Copy the original build command
            and add whatever flags or arguments you need in order to make it
            build just your fuzzer binary (see step 1 above).
            Replace the original build command in the `src_compile` function
            with a conditional statement similar to the one below, so that when
            `USE="fuzzer"` is used to build the package, it will build your
            fuzzer binary, otherwise it will build the package normally.

            ```bash
            if use fuzzer ; then
                 <modified build command>
            else
                 <original build command>
            fi
            ```

    2.  Install your binary in `/usr/libexec/fuzzers/`

        In your ebuild file, find the `src_install()` function. Add a
        statement to install your fuzzer:

        ```bash
        platform_fuzzer_install "${S}"/OWNERS "${OUT}"/<your_fuzzer>
        ```

3.  Build and test your new fuzzer locally.

    To build your new fuzzer, once you have updated the ebuild file, it should
    be sufficient to build it with `USE="asan fuzzer"`.

    ```bash
    # Run build_packages to build the package and its dependencies.
    $ USE="asan fuzzer" ./build_packages --board=${BOARD} --skip_chroot_upgrade <your-package>
    # If you make more changes to your fuzzer or build, you can rebuild the package with:
    $ USE="asan fuzzer" emerge-${BOARD} <your-package>
    ```

    These flags work with `cros_workon_make` as well for a faster compile cycle:

    ```bash
    $ USE="asan fuzzer" cros_workon_make --board=$BOARD <your-package>
    ```

    You should verify that your fuzzer was built and that it was installed in
    `/usr/libexec/fuzzers/` (make sure the owners file was installed there as
    well). To run your fuzzer locally, you first run this script *outside your
    chroot* to set up your environment properly:

    ```bash
    $ path-to-chroot/chromite/bin/cros_fuzz_test_env --board=${BOARD}
    ```

    Then run your fuzzer, *outside of the chroot as well*:

    ```bash
    $ sudo chroot /path-to-chroot/chroot/build/${BOARD}
    $ ASAN_OPTIONS="log_path=stderr" /usr/libexec/fuzzers/<your_fuzzer>
    ```

    You should also verify that your package still builds correctly without
    `USE="fuzzer"`.

    Once you are happy with your new fuzzer, commit your changes.

4.  Add the package dependency to the `chromium-os-fuzzers` ebuild. Inside your
    chroot:

    ```bash
    $ cd ~/trunk/src/third_party/chromiumos-overlay/virtual/chromium-os-fuzzers
    ```

    Edit `chromium-os-fuzzers-1.ebuild`. In that file, find the `RDEPEND` list
    and add your package/fuzzer (you can look at the other packages there, to
    see how it's done). Don't forget to uprev the ebuild symlink. Commit the
    changes and upload them for review.

5.  Optional: Verify that the `amd64-generic-fuzzer` builder is happy with your
    changes.

    Submit a tryjob *outside of the chroot*:
    ```bash
    $ cros tryjob -g 'CL1 CL2' amd64-generic-fuzzer-tryjob
    ```
    Replace *CL1* and *CL2* with the actual CL numbers.

    You should verify that your package is picked up by the builder by looking
    at the `BuildPackages` stage logs.

    The builder builds the full system with AddressSanitizer and libFuzzer
    instrumentation. If you do not want a particular library pulled in by your
    changes to be instrumented, you can add a call to `filter_sanitizers` in
    the library's ebuild file.

#### Build a fuzzer for any other package

1.  Update the build system for your package to build your `*_fuzzer` binary.

    The exact instructions here are going to vary widely, depending on your
    package and the build system in your package (Make, Ninja, SCons, etc.).
    In general, you will need to be able to invoke your normal build
    command, passing a special flag or argument or environment variable, so
    that it will build your fuzzer binary and only your fuzzer binary. This
    will involve updating GYP files or Makefiles or whatever other build files
    your package uses.

    If testing this by hand (without the ebuild file changes in step 3 below),
    you will need to manually pass the compiler flags
    `-fsanitize=address -fsanitize=fuzzer-no-link` and the linker flags
    `-fsanitize=address -fsanitize=fuzzer` to your build. You will not need
    to pass these flags manually once you have updated the ebuild file.

3.  Update your package's ebuild file:
    1.  Add `asan` and `fuzzer` to the `IUSE` flags list.

        In all probability your package ebuild already contains an IUSE
        definition. Look for a line starting `IUSE="..."`, and add `asan` and
        `fuzzer` to the list. If your file does not already contain such a
        line, add one near the top:

        ```bash
        IUSE="asan fuzzer"
        ```

        See the [puffin ebuild] for a good example.

    2.  Update the ebuild file to build the new binary when the fuzzer
        USE flag is being used:
        1.  Find the `inherit` line in  your ebuild (near the top of the
            file). Make sure that `flag-o-matic` and `toolchain-funcs` are in
            the inherit list. If your file does not have a line that starts with
            `inherit `, add one near the top (after the `EAPI` line and before
            the `KEYWORDS` line):

            ```bash
            inherit flag-o-matic toolchain-funcs
            ```

        2.  Find the `src_compile()` function in your ebuild file. If
            there isn't one, add one:

            ```bash
            src_compile() {
            }
            ```

        3.  Add calls `asan-setup-env` and `fuzzer-setup-env`, near the top of
            `src_compile`, to set the appropriate compiler/linker flags:

            ```bash
            src_compile() {
                asan-setup-env
                fuzzer-setup-env
                ...
            }
            ```

        4.  Find the line in your `src_compile` function that actually builds
            your package (the command will probably look like `emake` or
            `make` or `cmake`). This is the command that is meant by *original
            build command* below. Copy the original build command and add
            whatever flags or arguments you need in order to make it build
            just your fuzzer binary (see step 1 above). Replace the original
            build command in the `src_compile` function with a conditional
            statement similar to the one below, so that when `USE="fuzzer"` is
            used to build the package, it will build your fuzzer binary,
            otherwise it will build the package normally.

            ```bash
            if use fuzzer ; then
                <modified build command>
            else
                <original build command>
            fi
            ```

    3.  Install your binary in `/usr/libexec/fuzzers/`

        In your ebuild file, find the `src_install()` function. Add a
        conditional statement to install your fuzzer:

        ```base
        if use fuzzer; then
            local f="${OUT}/<your_fuzzer>"
            insinto /usr/libexec/fuzzers
            exeinto /usr/libexec/fuzzers
            doexe "${f}"
            newins "${S}/OWNERS" "${f##*/}.owners"
        fi
        ```

        (The owners part above is so that ClusterFuzz knows to whom to assign
        the bugs generated by this fuzzer.)

4.  Build and test your new fuzzer locally.

    To build your new fuzzer, once you have updated the ebuild file, it should
    be sufficient to build it with `USE="asan fuzzer"`:

    ```bash
    # Run build_packages to build the package and its dependencies.
    $ USE="asan fuzzer" ./build_packages --board=${BOARD} --skip_chroot_upgrade <your-package>
    # If you make more changes to your fuzzer or build, you can rebuild the package by:
    $ USE="asan fuzzer" emerge-${BOARD} <your-package>
    ```

    You should verify that your fuzzer was built and that it was installed in
    `/usr/libexec/fuzzers/` (make sure the owners file was installed there as
    well). To run your fuzzer locally, you first run this script *outside your
    chroot* to set up your environment properly:

    ```bash
    $ /path-to-chroot/chromite/bin/cros_fuzz_test_env --board=${BOARD}
    ```

    Then run your fuzzer, *outside of the chroot as well*:

    ```bash
    $ sudo chroot /path-to-chroot/chroot/build/${BOARD}
    $ ASAN_OPTIONS="log_path=stderr" /usr/libexec/fuzzers/<your_fuzzer>
    ```

    You should also verify that your package still builds correctly without
    `USE="fuzzer"`.

    Once you are happy with your new fuzzer, commit your changes.

5.  Add the package dependency to the `chromium-os-fuzzers` ebuild. Inside your
    chroot:

    ```bash
    $ cd ~/trunk/src/third_party/chromiumos-overlay/virtual/chromium-os-fuzzers
    ```

    Edit `chromium-os-fuzzers-1.ebuild`. In that file, find the `RDEPEND` list
    and add your package/fuzzer (you can look at the other packages there, to
    see how it's done). Don't forget to uprev the ebuild symlink. Commit the
    changes and upload for review.

6.  Optional: Verify that the `amd64-generic-fuzzer` builder is happy with your
    changes.

    Submit a tryjob *outside of the chroot* as:
    ```bash
    $ cros tryjob -g 'CL1 CL2' amd64-generic-fuzzer-tryjob
    ```
    Replace *CL1* and *CL2* with the actual CL numbers.

    You should verify that your package is picked up by the builder by looking
    at the `BuildPackages` stage logs.

    The builder builds the full system with AddressSanitizer and libFuzzer
    instrumentation. If you do not want a particular library pulled in by your
    changes to be instrumented, you can add a call to `filter_sanitizers` in
    the library's ebuild file.

### Write a fuzzer

Now that you can build and execute your fuzzer, you can start working on getting
it to actually test things. If the function you want to test takes a chunk of
bytes and a length, then you're done: that's what the fuzzing scaffolding will
provide in the `LLVMFuzzerTestOneInput` function. Just pass that to your
function:

```cpp
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
  // Call your function here.
  return 0;
}
```

Some things to keep in mind:

*   The fuzz target will be executed many times with different inputs in the
    same process.
*   It must tolerate any kind of input (empty, huge, malformed, etc). You can
    write your fuzz target to simply return 0 on certain types of malformed
    input. See the [GURL fuzzer] for such an example.
*   It must not `exit()` on any input.
*   It may use threads but ideally all threads should be joined at the end of
    the function.
*   It must be as deterministic as possible. Non-determinism (e.g. random
    decisions not based on the input bytes) will make fuzzing inefficient.
*   It must be fast. Avoid >= cubic complexity, logging, high memory
    consumption.
*   Ideally, it should not modify any global state (although that's not strict).

### Getting help with modifying ebuild files

Some ebuild files are more complex or confusing than others. There are
several links in the [References](#References) section of this document that
can help you with understanding/editing your ebuild file. If you are still
having difficulties editing your ebuild file and need more help, please file a
bug in the [Chromium issue tracker], assign it to the
*Tools>ChromeOS-Toolchain* component, and send an email to
[chromeos-fuzzing@google.com]. We will try to help you figure this out.

### Using ClusterFuzz

As already mentioned, ClusterFuzz will pick up any fuzzer written using the
above steps, run the fuzzer, and file bugs for any crashes found. ClusterFuzz
runs fuzzers as soon as the [fuzzer builder] completes a build and uploads it
to the Google Cloud Storage bucket
(`gs://chromeos-fuzzing-artifacts/libfuzzer-asan/amd64-generic-fuzzer/`).

ClusterFuzz has many features such as statistics reporting that you may find
useful. Below are links to some of the more important ones:

*   [Fuzzer statistics] - Statistics from fuzzer runs, updated daily. Ignore the
    columns `edge_cov`, `func_cov`, and `cov_report` as these are not supported
    for Chrome OS. Graphs of stats can viewed by changing the "Group by" drop
    down to "Time" and specifying the fuzzer you are interested in, rather than
    "libFuzzer".
*   [Crash statistics] - Statistics on recent crashes.
*   [Fuzzer logs] - Logs output by your fuzzer each time ClusterFuzz runs
    it. This is usually a good place to debug issues with your fuzzer.
*   [Fuzzer corpus] - Testcases produced by the fuzzer that libFuzzer has deemed
    "interesting" (meaning it causes unique program behavior).

## See also

### References

[Setting up Fuzzing for Chrome OS](https://docs.google.com/document/d/1tvY4YV6q5RPVGK8hivkSKLPbNufebZ6BBozT0LqbGRA/edit?usp=sharing)

"[Continuous in-process fuzzing for Chrome OS targets](https://docs.google.com/document/d/1sd1IejWzcbQgF7soVVKlqTMk3HZfvPGR7QYP5T6qBYU/edit#heading=h.5irk4csrpu0y)"

[libFuzzer - a library for coverage-guided fuzz testing.](https://llvm.org/docs/LibFuzzer.html)

["Fault injection through unexpected input data (aka Fuzz Testing)"](https://companydoc.corp.google.com/company/teams/security-privacy/docs/secure-coding/archive/deprecated-ISETeamFuzzing.md?cl=head)

[Getting Started with libFuzzer in Chromium](https://chromium.googlesource.com/chromium/src/+/master/testing/libfuzzer/getting_started.md)

[Gentoo Cheat Sheet](https://wiki.gentoo.org/wiki/Gentoo_Cheat_Sheet)

[Gentoo Ebuild Variables Guide](https://devmanual.gentoo.org/ebuild-writing/variables/)

[Gentoo Ebuild Install Functions Reference](https://devmanual.gentoo.org/function-reference/install-functions/)

[Gentoo Ebuild Writing Developers Manual](https://devmanual.gentoo.org/ebuild-writing/index.html)

### Useful go links

[go/ClusterFuzz](https://sites.google.com/corp/google.com/clusterfuzz/home)

[go/libfuzzer-Chrome](https://chromium.googlesource.com/chromium/src/+/master/testing/libfuzzer/README.md)

[go/fuzzing](https://g3doc.corp.google.com/security/fuzzing/g3doc/index.md?cl=head)

[go/whyfuzz](https://docs.google.com/document/d/1jNDjMBrXyCalNDQsHGmXP-Xc2M1_mmE4eptdOBE0tfA/edit#heading=h.6icp69vxzbq)

[libFuzzer]: https://llvm.org/docs/LibFuzzer.html

[ClusterFuzz]: https://sites.google.com/corp/google.com/clusterfuzz/home

[libFuzzer and ClusterFuzz]: https://chromium.googlesource.com/chromium/src/+/master/testing/libfuzzer/README.md

[security bugs]: https://bugs.chromium.org/p/chromium/issues/list?can=1&q=reporter:clusterfuzz@chromium.org%20-status:duplicate%20-status:wontfix%20type=bug-security

[non-security bugs]: https://bugs.chromium.org/p/chromium/issues/list?can=1&q=reporter%3Aclusterfuzz%40chromium.org+-status%3Aduplicate+-status%3Awontfix+-type%3Dbug-security&sort=modified

[Inherit flag-o-matic toolchain-funcs]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/chromeos-base/libchrome/libchrome-395517.ebuild?q=toolchain-funcs+package:%5Echromeos_public$&dr=C&l=15

[Set up flags: call asan-setup-env & fuzzer-setup-env]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/chromeos-base/libchrome/libchrome-395517.ebuild?q=asan-setup-env+package:%5Echromeos_public$&dr=C

[USE flags: fuzzer]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/chromeos-base/libchrome/libchrome-395517.ebuild?q=IUSE+fuzzer+package:%5Echromeos_public$&dr=C

[Using ClusterFuzz]: #using-clusterfuzz

[bsdiff fuzzer]: https://android.googlesource.com/platform/external/bsdiff/+/master/bspatch_fuzzer.cc

[puffin_fuzzer]: https://android.googlesource.com/platform/external/puffin/+/master/src/fuzzer.cc

[GURL fuzzer]: https://chromium.googlesource.com/chromium/src/+/master/url/gurl_fuzzer.cc

[midis GYP file]: https://chromium.googlesource.com/chromiumos/platform2/+/master/midis/midis.gyp#139

[puffin ebuild]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/dev-util/puffin/puffin-9999.ebuild

[Chromium issue tracker]: https://crbug.com

[fuzzer builder]: https://build.chromium.org/p/chromiumos/builders/amd64-generic-fuzzer

[chromeos-fuzzing@google.com]: mailto:chromeos-fuzzing@google.com

[Fuzzer statistics]: https://clusterfuzz.com/v2/fuzzer-stats/by-fuzzer/fuzzer/libFuzzer/job/libfuzzer_asan_chromeos

[Crash statistics]: https://clusterfuzz.com/v2/crash-stats?block=day&days=7&end=423713&fuzzer=libFuzzer&group=platform&job=libfuzzer_asan_chromeos&number=count&sort=total_count

[Fuzzer logs]: https://console.cloud.google.com/storage/browser/chromeos-libfuzzer-logs/libfuzzer

[Fuzzer corpus]: https://console.cloud.google.com/storage/browser/chromeos-libfuzzer-logs
