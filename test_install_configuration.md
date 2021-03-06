# Configuring Installation and Tests with GN

## Introduction

This guide describes how to configure tests and installation.
It assumes that you are familiar with [platform2 primer].
Using GN can reduce the amount of code in ebuild.

[TOC]

## Install configuration

### Installing Targets defined in BUILD.gn

A file built by a GN target, either by `executable`, `shared_library` or `static_library`, can be installed by specifying `install_path`.

A target that install the output must be a dependency of `group("all")`.

#### Variable

*   install_path
    *   An install destination path.

#### Installing executable targets

*   `install_path` must end in `bin` or `sbin`.
*   When you specify the `install_path` simply as `bin`,
    the binary will be installed into `/usr/bin`.

    ```gn
    executable("executable_target") {
        sources = [ "source.cc" ]
        install_path = "bin"
    }
    ```

    is equivalent to

    ```bash
    dobin ${OUT}/executable_target
    ```

*   If you want to change the destination, you can specify the absolute path.

    ```gn
    executable("executable_target") {
        sources = [ "source.cc" ]
        install_path = "/sbin"
    }
    ```

    is equivalent to

    ```bash
    into /
    dosbin ${OUT}/executable_target
    ```

#### Installing Shared Library Targets

Installing shared library requires specifying `install_path` in the shared_library target.

*   The `install_path` must end in `lib`.
*   When you specify the `install_path` simply as `lib`, `/usr/lib` will be used.

    ```gn
    shared_library("libtarget") {
        sources = [ "source.cc" ]
        install_path = "lib"
    }
    ```

    is equivalent to

    ```bash
    dolib.so ${OUT}/lib/libtarget
    ```

*   If you want to change the destination, you can specify the absolute path.

    ```gn
    shared_library("libtarget") {
        sources = [ "source.cc" ]
        install_path = "/usr/local/lib"
    }
    ```

    is equivalent to

    ```bash
    into /usr/local
    dolib.so ${OUT}/lib/libtarget
    ```

#### Installing Static Library Targets

Installing static libraries requires specifying `install_path` in the static_library target.

*   The `install_path` must end in `lib`.
*   When you specify the `install_path` simply as `lib`,
    it installs into `/usr/lib`.

    ```gn
    static_library("libtarget") {
        sources = [ "source.cc" ]
        install_path = "lib"
    }
    ```

    is equivalent to

    ```bash
    dolib.a ${OUT}/libtarget
    ```

*   If you want to change the destination, you can specify the absolute path.

    ```gn
    static_library("libtarget") {
        sources = [ "source.cc" ]
        install_path = "/usr/local/lib"
    }
    ```

    is equivalent to

    ```bash
    into /usr/local
    dolib.a ${OUT}/libtarget
    ```

### Installing files not defined in BUILD.gn

You can install files that are not generated by GN and ninja by using
`install_config` target.

#### Variables

*   sources (required)
    *   A list of files to be installed.
*   install_path (optional)
    *   An install destination path.
    *   default: `/`
*   recursive (optional)
    *   A boolean, indicating whether to install files recursively.
    *   default: `false`
*   options (optional)
    *   A string of options for installing files.
    *   default: `"-m0644"`
*   outputs (optional)
    *   A list of new file names, if sources should be renamed too.
    *   When not specified, original names are used.
*   symlinks (optional)
    *   A list of symbolic links to be created.
    *   When install_path is specified, links are created in
        `${install_path}/${symlink}`

#### Usage

*   To install files, add `install_config` into dependency tree of `group("all")`.

    ```gn
    install_config("install_init") {
        sources = [ "init/initialize.conf" ]
        install_path = "/etc/init"
    }
    ```

    is equivalent to

    ```bash
    insinto /etc/init
    doins init/initialize.conf
    ```

*   To install files recursively, set `recursive` to `true`.

    ```gn
    install_config("install_rec") {
        sources = [ "source_directory" ]
        install_path = "/usr/local"
        recursive = true
    }
    ```

    is equivalent to

    ```bash
    insinto /usr/local
    doins -r source_directory
    ```

*   When you want to change owner or permission, specify `options`.

    ```gn
    install_config("install_exe") {
        sources = [ "source" ]
        install_path = "/usr/local"
        options = "-m0755"
    }
    ```

    is equivalent to

    ```bash
    insinto /usr/local
    insopts -m0755
    doins source
    ```

*   When you want to install multiple files, specify sources all together.

    ```gn
    install_config("install_init") {
        sources = [
            "init/initialize1.conf",
            "init/initialize2.conf",
        ]
        install_path = "/etc/init"
    }
    ```

    is equivalent to

    ```bash
    insinto /etc/init
    doins init/initialize.conf
    ```

*   When you want to use `newins` command, specify new file name as `outputs`.

    ```gn
    install_config("install_policy") {
        sources = [ "policy/configuration.policy" ]
        outputs = [ "newfilename.policy" ]
        install_path = "/usr/share/policy"
    }
    ```

    is equivalent to

    ```bash
    insinto /usr/share/policy
    newins policy/configuration.policy newfilename.policy
    ```

*   `outputs` are also used for installing multiple files.

    ```gn
    install_config("install_multiple_new") {
        sources = [
            "old.policy",
            "another_old.policy",
        ]
        outputs = [
            "new1.policy",
            "new2.policy",
        ]
        install_path = "/usr/share/policy"
    }
    ```

    is equivalent to

    ```bash
    insinto /usr/share/policy
    newins old.policy new1.policy
    newins another_old.policy new2.policy
    ```

*   `symlinks` parameter is similar to `outputs`, but it creates symbolic links by `dosym`.

    ```gn
    install_config("install_sym") {
        sources = [ "source" ]
        symlinks = [ "symlink" ]
    }
    ```

    is equivalent to

    ```bash
    dosym source symlink
    ```

*   When `install_path` is specified, it is added to the head of `symlinks`.

    ```gn
    install_config("install_sym") {
        sources = [ "source" ]
        symlinks = [ "symlink" ]
        install_path = "/path/to/install"
    }
    ```

    is equivalent to

    ```bash
    dosym source /path/to/install/symlink
    ```

## Tests configuration

*   A unit test executable can be run in the test phase by specifying `run_test = true` in the
    executable target.

    ```gn
    executable("test_target") {
        sources = [ "source.cc" ]
        run_test = true
    }
    ```

    is equivalent to

    ```bash
    platform_test "run" "${OUT}/test_target"
    ```

*   When you want to specify config, you can use `test_config` variable.

    ```gn
    executable("test_target") {
        sources = [ "source.cc" ]
        run_test = true
        test_config = {
            run_as_root = true
            gtest_filter = ".RunAsRoot*-"
        }
    }
    ```

    is equivalent to

    ```bash
    platform_test "run" "${OUT}/test_target" "1" "*.RunAsRoot*-"
    ```

[platform2 primer]: platform2_primer.md
