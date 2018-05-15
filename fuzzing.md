# How to do fuzz testing on Chrome OS

The target audience for this document are developers in Chrome OS who work on
packages. It explains how to write a fuzz test target for a package, which
will then be automatically picked up and run regularly by ClusterFuzz. It
assumes some basic familiarity with the Chrome OS development environment.

[TOC]

## What is fuzz testing and why would I want to do it?

Fuzz testing (or "fuzzing") is the process of testing an application by
feeding invalid, malformed or malicious test inputs to a target program in an
attempt to crash it. It is particularly appropriate and useful for testing
APIs. The input data is created by randomly mutating other inputs. Sometimes
the initial inputs are provided by a user in what is known as a "seed corpus",
otherwise the fuzzer will generate inputs from scratch. Coverage guided fuzz
testing is an extension of fuzz testing that explicitly attempts to increase
code coverage when it generates new tests, adding the new tests to its test
corpus when it finds tests that increase the coverage, i.e. when a new test
executes a new code path in the target program.

LLVM has a built-in fuzzing engine, [libFuzzer], that is provided as part of
LLVM's sanitizer instrumentation. Google has a cross-platform fuzzing
infrastructure, [ClusterFuzz], that works with libFuzzer and aids developers
by providing an end-to-end pipeline that automatically picks up new (and
existing) fuzz targets, runs the fuzzers on the tests, reports bugs (assigning
them to appropriate owners) and even verifies fixes. Chromium on Linux and
MacOS has been using [this system] for a while, and it has proved very useful,
finding thousands of [security] and [non-security] bugs.

We have now set up Chrome OS to allow developers to easily write fuzz targets
for their packages. This document will walk you through the simple steps
necessary to get your fuzz targets up and running on ClusterFuzz. Given the
ease of writing fuzz tests, and the great benefits in terms of improved
reliability and security, why would you not want to write fuzz tests for your
package?

## Quickstart Guide

This section of the document gives the basic steps needed to write a fuzz test
in Chrome OS and have ClusterFuzz pick it up and start testing. However, this
section does not explain the steps in any detail. That is covered below in the
"[More Detailed Instructions](#More-Detailed-Instructions)" part of this
document.

Note: If you are working on a platform package (one that is in the platform or
platform2 source directory and whose ebuild inherits from the platform
eclass), some of the steps below have been streamlined for you (you might want
to read
"[Adding a Fuzz Target to a Platform Package](#Adding-a-Fuzz-Target-to-a-Platform-Package)"
in the Detailed Instructions section).

**Steps to create a new fuzz target (fuzz test binary) in Chrome OS:**

*   Write a new test program in your package, whose name ends in "_fuzzer" and
which defines a function `LLVMFuzzerTestOneInput` with the following
signature:

    ```c
    extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
        <your test code goes here>
        return 0;
    }
    ```

    Be sure `LLVMFuzzerTestOneInput` calls the function you want to fuzz.

*   Update the build system for your package to build your *_fuzzer binary and
    fix build flags.
*   Update your package ebuild file:
    1.  Add `asan` and `fuzzer` to `IUSE` flags list.
    2.  Build you new fuzzer (conditioned on `use fuzzer`), with the
        appropriate flags:
        1.  [Inherit flag-o-matic toolchain-funcs]
        2.  [Set up flags: call asan-setup-env & fuzzer-setup-env]
        3.  [USE flags: fuzzer]
    3.  Install your binary in /usr/libexec/fuzzers/
    4.  Build the libraries your fuzzer depends on with the appropriate
        `-fsanitize` flags (optional).
*   Build and test your new fuzz target locally. Commit your changes.
*   Add the package dependency to the chromium-os-fuzzers ebuild, and commit
    the change.

That's it! The continuously running fuzzer builder on the Chrome OS waterfall
will automatically detect your new fuzz target, build it and upload it to
ClusterFuzz, which will start fuzzing it. For more details on what you can do
with ClusterFuzz, see the [Using ClusterFuzz] section.

## More Detailed Instructions

This section goes over the steps mentioned in the Quickstart Guide in more
detail. Because many of the packages that are likely to benefit most from
fuzz testing are Platform packages, some special scaffolding has been added to
streamline generating fuzz targets for platform packages. The streamlined
steps are explained in
"[Adding a Fuzz Target to a Platform Package](#Adding-a-Fuzz-Target-to-a-Platform-Package)".
If the package for which you want to create a fuzz test is not a platform
package, please follow the steps in
"[Adding a Fuzz Target to Any Other Package](#Adding-a-Fuzz-Target-to-Any-Other-Package)"
(further down this document).

### Adding a Fuzz Target to a Platform Package

Below are the steps to create a new fuzz target (fuzz test binary) in Chrome
OS, in a platform package. Note that there are slightly different steps
required, depending on whether or not your package builds with a gyp file.
The steps will tell you where these differences are.

1.  Write a new test program in your package, whose name ends in "_fuzzer" and
    which defines a function `LLVMFuzzerTestOneInput` with the signature
    below.

    ```c
    extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
        <your test code goes here>
        return 0;
    }
    ```

    The [bsdiff fuzzer] is a very simple example of how to write such a fuzz
    test.  [puffin_fuzzer] is a slightly larger example. In general,
    individual fuzz tests are not very large or difficult to write. The most
    important piece is to have the function `LLVMFuzzerTestOneInput`, which
    calls the function you want  to fuzz, and which takes the `data` and
    `size` arguments and passes them  to your function. Some important things
    to remember about fuzz targets:

    *   The fuzz target will be executed many times with different inputs in
        the same process.
    *   It must tolerate any kind of input (empty, huge, malformed, etc).
        Note: In general this is true, but you can write your fuzz target to
        simply return 0, for example, on certain types of malformed input. See
        [this fuzzer] for such an example.
    *   It must not exit() on any input.
    *   It may use threads but ideally all threads should be joined at the end
        of the function.
    *   It must be as deterministic as possible. Non-determinism (e.g. random
        decisions not based on the input bytes) will make fuzzing inefficient.
    *   It must be fast. Avoid >= cubic complexity, logging, high memory
        consumption.
    *   Ideally, it should not modify any global state (although that's not
        strict).

2.  Update the build system for your package to build your *_fuzzer binary.
    Fix any build flags that need fixing.
    1.  Packages that are built with gyp files.

        Update your gyp file to build the fuzzer binary. See [this example]
        for how to update a gyp file to build a fuzzer.  (If copying the
        example, replace the package and fuzzer name with your own).

    2.  Packages that are built without gyp files.

        If your package is not built with a gyp file, then you will need to
        update your Makefile (or whatever build system you use), in such a way
        that your fuzzer binary gets built when a special flag or argument is
        passed to the normal build command.

        NOTE 1: If testing this by hand (without the ebuild file changes in
        step 3 below), you will need to manually pass the compiler flags
        `-fsanitize=address -fsanitize=fuzzer-no-link` and the linker flags
        `-fsanitize=address -fsanitize=fuzzer` to your build. You will not
        need to pass these flags manually once you have updated the ebuild
        file.

        NOTE: `asan` and `fuzzer` *do not support* `-Wl,-z,-defs` or
        `--no-undefined`. Make sure you are not passing those flags to the
        build of your fuzzer binary. **IF YOUR BUILD SYSTEM EVER COMBINES
        *CFLAGS* AND *LDFLAGS* INTO A SINGLE FLAGS VARIABLE, MAKE SURE LDFLAGS
        COMES AFTER CFLAGS WHEN BUILDING YOUR FUZZER!!!**

3.  Update your package ebuild file:
    1.  Update the ebuild file to build the new binary when the fuzzer
        use-flag is being used. For platform packages built with gyp files,
        you should skip the build step and go directly to the next step
        for installing the fuzzer binary.

        1.  Find the `src_compile()` function in your ebuild file.   If
            there isn't one, add one:

            ```bash
            src_compile() {
            }
            ```

        2.  Add the call to actually build your fuzz target.

            Find the line in your `src_compile` function that actually builds
            your package (the command will probably look like `emake` or `make`
            or `cmake`). This is the command that is meant by
            'original build command' below. Copy the original build command
            and add whatever flags or arguments you need in
            order to make it build just your fuzzer binary (see step 2 above).
            Replace the original build command in the src_compile function with
            a conditional statement similar to the one below, so that when
            `USE="fuzzer"` is used to build the package, it will build your
            fuzzer binary, otherwise it will build the package normally.

            ```bash
            if use fuzzer ; then
                 <modified build command>
            else
                 <original build command>
            fi
            ```

    2.  Install your binary in /usr/libexec/fuzzers/

        In your ebuild file, find the `src_install()` function. Add a
        statement to install your fuzzer target:

        ```bash
        platform_fuzzer_install "${S}"/OWNERS "${OUT}"/<your_fuzzer>
        ```

    3.  Build the libraries you fuzzer depends on with the appropriate
        `-fsanitize` flags (optional).

        When building with libfuzzer and/or asan, it is **strongly
        recommended** that you also build any libraries on which your package
        depends (see `RDEPEND` list in your ebuild file) with libfuzzer and
        asan. This is because if those libraries are not built with these
        flags, then calls into those libraries won't get tested. In addition,
        doing this helps avoid spurious errors when the library headers use
        the STL (Standard Template Library).

        To get a complete list of the libraries your fuzzer depends on, use
        the `ldd` command on your fuzzer binary.

        ```bash
        $ ldd <your_binary>
        ```

        This will give you a complete list of all the libraries, .so's, etc.
        that your fuzzer depends on. You may not want to build all of these
        with fuzzing enabled -- it is really for you to decide, based on how
        your package interacts with the libraries, but when in doubt you
        should probably instrument.

        Note:  Running the fuzzer may reveal some signs that you should have
        instrumented a library, such as if the coverage is very low (eg <100)
        after running for more than a couple minutes or if ASAN detects
        container overflows that you believe are spurious.

        For every library that you decide you want to build with fuzzing
        enabled, you will need to repeat steps 3a & 3b above (building the
        whole library, rather than just a fuzz test).

4.  Build and test your new fuzz target locally.

    To build your new fuzzer, once you have updated the ebuild file, it should
    be sufficient to build it with `USE="asan fuzzer"`:

    ```bash
    # Run build_packages to build the package and its dependencies.
    $ USE="asan fuzzer" ./build_packages --board=${BOARD} --skip_chroot_upgrade <your-package>
    # If you make more changes to your fuzzer or build, you can rebuild the package by:
    $ USE="asan fuzzer" emerge-${BOARD} <your-package>
    ```

    You should verify that your fuzzer was built and that it was installed in
    /usr/libexec/fuzzers (make sure the owners file was installed there as
    well). To run your fuzzer locally, you first run this script (outside your
    chroot) to set up your environment properly:

    ```bash
    $ ./path-to-chroot/chromite/bin/cros_fuzz_test_env --chromeos_root=/path-to-chroot --board=${BOARD}
    ```

    Then run your fuzzer:

    ```bash
    $ sudo chroot /path-to-chroot/chroot/build/${BOARD}
    $ ASAN_OPTIONS="log_path=stderr" ./usr/libexec/fuzzers/<your_fuzzer>
    ```

    **NOTE**:  You should also verify that your package still builds correctly
    without `USE="fuzzer"`.

    Once you are happy with your new fuzz target, commit your changes.

5.  Add the package dependency to the chromium-os-fuzzers ebuild. Inside your
    chroot:

    ```bash
    $ cd ~/trunk/src/third_party/chromiumos-overlay/virtual/chromium-os-fuzzers
    ```

    Edit `chromium-os-fuzzers-1.ebuild`. In that file, find the `RDEPEND` list
    and add your package/fuzzer (you can look at the other packages there, to
    see how it's done). Don't forget to uprev the ebuild symlink. Commit the
    change.

### Adding a Fuzz Target to Any Other Package

Steps to create a new fuzz target (fuzz test binary) in Chrome OS:

1.  Write a new test program in your package, whose name ends in "_fuzzer" and
    which defines a function `LLVMFuzzerTestOneInput` with the signature
    below.

    ```c
    extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
        <your test code goes here>
        return 0;
    }
    ```

    The [bsdiff fuzzer] is a very simple example of how to write such a fuzz
    test.  [puffin_fuzzer] is a slightly larger example. In general,
    individual fuzz tests are not very large or difficulty to write. The most
    important piece is to have the function `LLVMFuzzerTestOneInput`, which
    calls the function you want to fuzz, and which takes the `data` and `size`
    arguments and passes them to your function. Some important things to
    remember about fuzz targets:

    *   The fuzz target will be executed many times with different inputs in
        the same process.
    *   It must tolerate any kind of input (empty, huge, malformed, etc).
        Note: In general this is true, but you can write your fuzz target to
        simply return 0, for example, on certain types of malformed input. See
        [this fuzzer] for such an example.
    *   It must not exit() on any input.
    *   It may use threads but ideally all threads should be joined at the end
        of the function.
    *   It must be as deterministic as possible. Non-determinism (e.g. random
        decisions not based on the input bytes) will make fuzzing inefficient.
    *   It must be fast. Avoid >= cubic complexity, logging, high memory
        consumption.
    *   Ideally, it should not modify any global state (although that's not
        strict).

2.  Update the build system for your package to build your *_fuzzer binary.
    Fix any build flags that need fixing.

    The exact instructions here are going to vary widely, depending on your
    package and the build system in your package (make, ninja, scons, etc.).
    In general, you will need to be able to invoke your normal build
    command, passing a special flag or argument or environment variable, so
    that it will build your fuzzer binary and only your fuzzer binary. This
    will involve updating gyp files or Makefiles or whatever other build files
    your package uses.

    NOTE 1: If testing this by hand (without the ebuild file changes in step 3
    below), you will need to manually pass the compiler flags
    `-fsanitize=address -fsanitize=fuzzer-no-link` and the linker flags
    `-fsanitize=address -fsanitize=fuzzer` to your build. You will not need
    to pass these flags manually once you have updated the ebuild file.

    NOTE 2: `asan` and `fuzzer` *do not support* `-Wl,-z,-defs` or
    `--no-undefined`. Make sure you are not passing those flags to
    the build of your fuzzer binary.  **IF YOUR BUILD SYSTEM EVER COMBINES
    *CFLAGS* AND *LDFLAGS* INTO A SINGLE FLAGS VARIABLE, MAKE SURE LDFLAGS
    COMES AFTER CFLAGS WHEN BUILDING YOUR FUZZER!!!**

1.  Update your package ebuild file:
    1.  Add `asan` and `fuzzer` to `IUSE` flags list.

        In all probability your package ebuild already contains an IUSE
        definition. Look for a line starting `IUSE="..."`, and add `asan` and
        `fuzzer` to the list. If your file does not already contain such a
        line, add one near the top:

        ```bash
        IUSE="asan fuzzer"
        ```

        See [this ebuild] for a good example.

    2.  Update the ebuild file to build the new binary when the fuzzer
        use-flag is being used:
        1.  Find the `inherit` line in  your ebuild (near the top of the
            file). Make sure that flag-o-matic and toolchain-funcs are in the
            inherit list. If your file does not have a line that starts with
            `inherit `, add one near the top (after the `EAPI` line and before
            the `KEYWORDS` line):

            ```bash
            inherit flag-o-matic toolchain-funcs
            ```

        2.  Find the `src_compile()` function in your ebuild file.   If
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
                …
            }
            ```

        4.  Find the line in your `src_compile` function that actually builds
            your package (the command will probably look like `emake` or
            `make` or `cmake`). This is the command that is meant by 'original
            build command' below. Copy the original build command and add
            whatever flags or arguments you need in order to make it build
            just your fuzzer binary (see step 2 above). Replace the original
            build command in the src_compile function with a conditional
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

    3.  Install your binary in /usr/libexec/fuzzers/

        In your ebuild file, find the `src_install()` function. Add a
        conditional statement to install your fuzzer target.:

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

    4.  Build the libraries you fuzzer depends on with the appropriate
        `-fsanitize` flags (optional).

        When building with libfuzzer and/or asan, it is **strongly
        recommended** that you also build any libraries on which your package
        depends (see `RDEPEND` list in your ebuild file) with libfuzzer and
        asan. This is because if those libraries are not built with these
        flags, then calls into those libraries won't get tested. In addition,
        doing this helps avoid spurious errors when the library headers use
        the STL (Standard Template Library).

        To get a complete list of the libraries your fuzzer depends on, use
        the `ldd` command on your fuzzer binary.

        ```bash
        $ ldd <your_binary>
        ```

        This will give you a complete list of all the libraries, .so's, etc.
        that your fuzzer depends on. You may not want to build all of these
        with fuzzing enabled -- it is really for you to decide, based on how
        your package interacts with the libraries, but when in doubt you
        should probably instrument.

        Note:  Running the fuzzer may reveal some signs that you should have
        instrumented a library, such as if the coverage is very low (eg <100)
        after running for more than a couple minutes or if ASAN detects
        container overflows that you believe are spurious.

        For every library that you decide you want to build with fuzzing
        enabled, you will need to repeat steps 3a & 3b above (building the
        whole library, rather than just a fuzz test).

4.  Build and test your new fuzz target locally.

    To build your new fuzzer, once you have updated the ebuild file, it should
    be sufficient to build it with `USE="asan fuzzer"`:

    ```bash
    # Run build_packages to build the package and its dependencies.
    $ USE="asan fuzzer" ./build_packages --board=${BOARD} --skip_chroot_upgrade <your-package>
    # If you make more changes to your fuzzer or build, you can rebuild the package by:
    $ USE="asan fuzzer" emerge-${BOARD} <your-package>
    ```

    You should verify that your fuzzer was built and that it was installed in
    /usr/libexec/fuzzers (make sure the owners file was installed there as
    well). To run your fuzzer locally, you first run this script (outside your
    chroot) to set up your environment properly:

    ```bash
    $ ./path-to-chroot/chromite/bin/cros_fuzz_test_env --chromeos_root=/path-to-chroot --board=${BOARD}
    ```

    Then run your fuzzer:

    ```bash
    $ sudo chroot /path-to-chroot/chroot/build/${BOARD}
    $ ASAN_OPTIONS="log_path=stderr" ./usr/libexec/fuzzers/<your_fuzzer>
    ```

    **NOTE:**  You should also verify that your package still builds correctly
    without `USE="fuzzer"`.

    Once you are happy with your new fuzz target, commit your changes.

5.  Add the package dependency to the chromium-os-fuzzers ebuild. Inside your
    chroot:

    ```bash
    $ cd ~/trunk/src/third_party/chromiumos-overlay/virtual/chromium-os-fuzzers
    ```

    Edit `chromium-os-fuzzers-1.ebuild`. In that file, find the `RDEPEND` list
    and add your package/fuzzer (you can look at the other packages there, to
    see how it's done). Don't forget to uprev the ebuild symlink. Commit the
    change.


## Getting Help with Modifying Ebuild Files

Some ebuild files are more complex or confusing than others. There are
several links in the [References] section of this document that might help you
with understanding/editing your ebuild file. If you are still having
difficulties editing your ebuild file and need more help, please file a bug in
crosbug, and assign it to the "Tools>ChromeOS-Toolchain" component, and  send
an email to
[chromeos-fuzzing@google.com](mailto:chromeos-fuzzing@google.com). We
will try to help you figure this out.

## Using ClusterFuzz

As already mentioned, ClusterFuzz will pick up any fuzzer written using the
above steps, run the fuzzer, and file bugs for any crashes found. ClusterFuzz
runs fuzzers as soon as the [builder] completes a build and uploads it to the
Google Cloud Storage bucket
(`gs://chromeos-fuzzing-artifacts/libfuzzer-asan/amd64-generic-fuzzer/`).

ClusterFuzz has many features such as statistics reporting that you may find
useful. Below are links to some of the more important ones:

*   [Fuzzer Statistics] - Statistics from fuzzer runs, updated daily. Ignore the
    columns `edge_cov`, `func_cov`, and `cov_report` as these are not supported
    for Chrome OS. Graphs of stats can viewed by changing the "Group by" drop
    down to "Time" and specifying the fuzzer you are interested in, rather than
    "libFuzzer".
*   [Crash Statistics] - Statistics on recent crashes.
*   [Fuzzer Logs] - Logs output by your fuzzer each time ClusterFuzz runs
    it. This is usually a good place to debug issues with your fuzzer.
*   [Fuzzer Corpus] - Testcases produced by the fuzzer that libFuzzer has deemed
    "interesting" (meaning it causes unique program behavior).

## See also:

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

### Useful "go" links

[go/ClusterFuzz](https://sites.google.com/corp/google.com/clusterfuzz/home)

[go/libfuzzer-Chrome](https://chromium.googlesource.com/chromium/src/+/master/testing/libfuzzer/README.md)

[go/fuzzing](https://g3doc.corp.google.com/security/fuzzing/g3doc/index.md?cl=head)

[go/whyfuzz](https://docs.google.com/document/d/1jNDjMBrXyCalNDQsHGmXP-Xc2M1_mmE4eptdOBE0tfA/edit#heading=h.6icp69vxzbq)

[libFuzzer]: https://llvm.org/docs/LibFuzzer.html

[ClusterFuzz]: https://sites.google.com/corp/google.com/clusterfuzz/home

[this system]: https://chromium.googlesource.com/chromium/src/+/master/testing/libfuzzer/README.md

[security]: https://bugs.chromium.org/p/chromium/issues/list?can=1&q=reporter:clusterfuzz@chromium.org%20-status:duplicate%20-status:wontfix%20type=bug-security

[non-security]: https://bugs.chromium.org/p/chromium/issues/list?can=1&q=reporter%3Aclusterfuzz%40chromium.org+-status%3Aduplicate+-status%3Awontfix+-type%3Dbug-security&sort=modified

[Inherit flag-o-matic toolchain-funcs]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/chromeos-base/libchrome/libchrome-395517.ebuild?q=toolchain-funcs+package:%5Echromeos_public$&dr=C&l=15

[Set up flags: call asan-setup-env & fuzzer-setup-env]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/chromeos-base/libchrome/libchrome-395517.ebuild?q=asan-setup-env+package:%5Echromeos_public$&dr=C

[USE flags: fuzzer]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/chromeos-base/libchrome/libchrome-395517.ebuild?q=IUSE+fuzzer+package:%5Echromeos_public$&dr=C

[Using ClusterFuzz]: #using-clusterfuzz

[bsdiff fuzzer]: https://android.googlesource.com/platform/external/bsdiff/+/master/bspatch_fuzzer.cc

[puffin_fuzzer]: https://android.googlesource.com/platform/external/puffin/+/master/src/fuzzer.cc

[this fuzzer]: https://chromium.googlesource.com/chromium/src/+/master/url/gurl_fuzzer.cc

[this example]: https://chromium.googlesource.com/chromiumos/platform2/+/master/midis/midis.gyp

[this ebuild]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/dev-util/puffin/puffin-9999.ebuild

[References]: #References

[builder]: https://build.chromium.org/p/chromiumos/builders/amd64-generic-fuzzer

[Fuzzer Statistics]: https://clusterfuzz.com/v2/fuzzer-stats/by-fuzzer/2018-05-01/2018-05-01/fuzzer/libFuzzer/job/libfuzzer_asan_chromeos

[Crash Statistics]: https://clusterfuzz.com/v2/crash-stats?block=day&days=7&end=423713&fuzzer=libFuzzer&group=platform&job=libfuzzer_asan_chromeos&number=count&sort=total_count

[Fuzzer Logs]: https://console.cloud.google.com/storage/browser/chromeos-libfuzzer-logs/libfuzzer

[Fuzzer Corpus]: https://console.cloud.google.com/storage/browser/chromeos-libfuzzer-logs
