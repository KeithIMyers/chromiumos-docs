# Security requirements for peripheral firmware updates and update tools

Several on- and off-device peripherals on Chrome OS require their firmwares to
be updated. This document lists the security requirements for the update
mechanism.

For the firmware update **file**:
*   The firmware update file *must* be stored in the rootfs so that it is
    covered by Chrome OS' Verified boot.
*   The peripheral *should* implement a firmware verification mechanism to
    validate the integrity of the firmware update file.
    *   Even though the intended file will be covered by Verified boot,
        Chrome OS devices should prevent malicious code from flashing unverified
        firmware to peripherals.
    *   In this context, integrity verification means a mechanism to ensure
        that:
        *   The firmware update file comes from the manufacturer.
        *   The firmware update file has not been modified.

        The normal way to ensure these two properties is to use asymmetric
        signatures. As per [Cryptographic Right Answers], use NaCl/[libsodium]
        or Ed25519.
    *   If there is no support for verifying the integrity of the firmware
        update file, the peripheral *must* support a way to block updates.
        I.e. allow updates during boot, but expose a mechanism to block updates
        immediately after boot and until the peripheral is power-cycled during
        reboot/power off.

For the firmware update **tool**:
*   The update tool *must* be open source and compiled as part of the Chrome OS
    build system.
*   The update tool *must* be sandboxed following the [sandboxing guide].
    Examples of this can be seen e.g. for [fwupdtool].

Speaking of `fwupd`, this project has become the standard for firmware updates
on Linux. If the peripheral can be updated using `fwupd` this will cover the
open-source requirement, and sandboxing can be reused straightforwardly.

[sandboxing guide]: /sandboxing.md
[fwupdtool]: https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/master/sys-apps/fwupd/files/fwupdtool-update.conf
[Cryptographic Right Answers]: https://latacora.micro.blog/2018/04/03/cryptographic-right-answers.html
[libsodium]: https://download.libsodium.org/doc/public-key_cryptography/public-key_signatures
