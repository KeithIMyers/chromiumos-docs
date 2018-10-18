# Debug Button Shortcuts

Chrome OS devices provide a variety of advanced keyboard and button shortcuts
that are useful for debugging. We document those here, as well as highlight
some differences between devices with and without keyboards.

[TOC]

## Devices With Keyboards

See the [official support page] for non-debug keyboard shortcuts.

Note that some of the below debug keyboard shortcuts are implemented at a very
low level and are not keyboard-layout aware. When in doubt, assume debug
shortcuts refer to the physical mapping of a US keyboard layout.

Shortcuts with an asterisk (**\***) also work with tablets/detachables with a
keyboard attached.

| Functionality                         | Shortcut                                                 |
|---------------------------------------|----------------------------------------------------------|
| Clean shutdown                        | Power button, hold for 3 - 8 seconds                     |
| Hard shutdown                         | Power button, hold for 8 seconds                         |
| Screenshot capture                    | `Ctrl + Switch-Window` **\***                            |
| File feedback / bug report            | `Alt + Shift + I` **\***                                 |
| Powerwash                             | `Ctrl + Alt + Shift + R` from login screen **\***        |
| EC reset                              | `Power + Refresh`                                        |
| Recovery mode                         | See [recovery mode] section                              |
| Developer mode                        | See [recovery mode] section                              |
| Battery cutoff                        | `Power + Refresh`, hold down, remove power for 5 seconds |
| Warm AP reset                         | `Alt + Volume-Up + R`                                    |
| Restart Chrome                        | `Alt + Volume-Up + X + X` **\***                         |
| Kernel panic/reboot                   | `Alt + Volume-Up + X + X + X` **\***                     |
| Reboot EC but don't boot AP           | `Alt + Volume-Up + Down-Arrow`                           |
| Force EC hibernate                    | `Alt + Volume-Up + H`                                    |
| Override USB-C port used for charging | `Left-Ctrl + Right-Ctrl + Search + (0 or 1 or 2)` **\*** |
| Virtual terminal switching            | `Ctrl + Alt + F1 (or F2, F3, F4)` **\***                 |

*Notes:*

*   AP = Application Processor; EC = Embedded Controller.
*   EC hibernate is supported only on devices with select Embedded Controllers.
*   `Alt + Volume-Up` is treated as the [Magic SysRq] key. When in Developer
    mode, you can enable SysRq key combinations as documented in the Linux
    kernel docs.
*   Overriding the charging port is only supported on Samus.

### Recovery Mode

*   **Recovery mode**: Hold `Esc + Refresh` and press `Power`.
    *   From here, follow the on-screen instructions to recover your device
        using external media.
*   **Developer mode**: To enter [developer mode], first enter recovery mode,
    then press `Ctrl + D`, followed by `Enter` to accept.

Some devices do not support `Esc + Power + Refresh` for entering Recovery Mode:

*   Chromeboxes have a recovery button. To enter Recovery Mode, hold down the
    recovery button and press `Power`.
*   Older devices have a physical developer mode switch. You can read up on
    your particular device under the [device-specific developer information]
    table.

_Note_: An additional, deeper reset (with forced memory retraining) can be
triggered on some devices on the way to recovery mode by holding `Left-Shift`
during the normal recovery sequence (i.e., `Left-Shift + Esc + Refresh +
Power`).

## Devices Without Keyboards

### Shortcuts

Starting in 2018, keyboardless Chrome OS devices are being built, in tablet and
detachable form factors. Without a keyboard, some typical keyboard-based
debugging shortcuts can't be supported. Below are the shortcuts for performing
various system and debugging tasks with only the limited physical buttons and
touch screen found on tablets or detachable devices.

Note that some of the following behaviors work on [convertible] devices when
used in tablet mode.

| Functionality                     | Shortcut                                                       |
|-----------------------------------|----------------------------------------------------------------|
| Screen off                        | Power button, short press                                      |
| Clean shutdown                    | Power button, hold for 3 seconds, select option in menu        |
| Hard shutdown                     | Power button, hold for 10 seconds                              |
| Screenshot capture                | `Power + Volume-Down`, short press                             |
| File feedback                     | Power button, hold for 3 seconds, select option in menu        |
| EC reset                          | `Power + Volume-Up`, hold for 10 seconds                       |
| Recovery mode                     | See [firmware UI] section                                      |
| Developer mode                    | See [firmware UI] section                                      |
| Battery cutoff                    | `Power + Volume-Up`, hold down, remove power for 5 seconds     |
| Warm AP reset                     | See [EC debug mode] section                                    |
| Restart Chrome                    | See [EC debug mode] section                                    |
| Kernel panic/reboot               | See [EC debug mode] section                                    |

*Notes:*

*   Filing feedback from the power button menu is currently (as of 2018-10-19)
    only [supported](https://crbug.com/845558#c8) on canary (for all users) and
    dev/beta/stable (for Google employees) release channels. All users can
    still file feedback via the browser options menu (`Help > Report an
    issue...`).
*   Battery cutoff shortcut is supported only on devices with a [smart battery]
    (e.g., not Scarlet, Dru).


### Firmware User Interface

The [boot firmware screen] is accessible through keyboard shortcuts on devices
with keyboards; on keyboardless devices, the following methods are supported:

*   **Recovery mode**: `Power + Volume-Up + Volume-Down`, hold for 10 seconds.
    *   From here, follow the on-screen instructions to recover your device
        using external media.

To exercise more advanced options, like entering **developer mode**, continue
to the following.

*   Press `Volume-Up + Volume-Down` simultaneously; now you should see a
    developer menu. This menu can be navigated with the volume buttons, and
    menu items can be selected with the power button.
*   For example, to enter Developer Mode, press Volume-Up until you reach
    "Confirm Enabling Developer Mode," then press the Power button.

_Note_: An additional, deeper reset (with forced memory retraining) can be
triggered on some devices on the way to recovery mode by holding `Power +
Volume-Up + Volume-Down` for 30 seconds (until the LED starts flashing).

### EC Debug Mode

Not all debugging shortcuts are implemented with a simple combination of
buttons—some are found by utilizing multi-button sequences, initiated after
entering a special EC state called "debug mode". To enter debug mode, press
`Volume-Up + Volume-Down` for 10 seconds. Release once the charging LED starts
flashing. Once in debug mode, any of the following actions can be taken, using
`Volume-Up` (`U`) and `Volume-Down` (`D`) buttons.

Supported codes:

| Functionality         | Sequence   |
|-----------------------|------------|
| Restart Chrome        | `UD`       |
| Kernel panic/crash    | `UUD`      |
| Warm reset            | `DU`       |

Debug mode is cancelled when a sequence finishes, or after 10 seconds without a
volume button press.


[official support page]: https://support.google.com/chromebook/answer/183101
[recovery mode]: #Recovery-Mode
[Magic SysRq]: https://www.kernel.org/doc/html/latest/admin-guide/sysrq.html
[developer mode]: https://www.chromium.org/chromium-os/poking-around-your-chrome-os-device
[convertible]: https://en.wikipedia.org/wiki/Laptop#Convertible
[firmware UI]: #Firmware-User-Interface
[EC debug mode]: #EC-Debug-Mode
[smart battery]: http://sbs-forum.org/specs/sbdat110.pdf
[device-specific developer information]: https://www.chromium.org/chromium-os/developer-information-for-chrome-os-devices
[boot firmware screen]: https://www.chromium.org/chromium-os/poking-around-your-chrome-os-device
