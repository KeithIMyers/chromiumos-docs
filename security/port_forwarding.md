# Port forwarding and tunneling in Chrome OS

## `localhost` to Crostini

Chrome OS will forward ports from `localhost` into [Crostini]. This allows
developers to use Chrome to access their development environment inside
[Crostini].

[cicerone] will [ask](https://chromium.googlesource.com/chromiumos/platform2/+/master/vm_tools/cicerone/service.cc#853)
[chunnel] to tunnel all ports listening in the [Crostini] container, except:

*   Privileged (<1024) since [chunnel] lacks `CAP_NET_BIND_SERVICE`.
*   2222 (SFTP for the Chrome OS Files app) and 5355 (mDNS) which are
    [blocklisted](https://chromium.googlesource.com/chromiumos/platform2/+/master/vm_tools/cicerone/service.cc#71).

Moreover, tunneled ports are [locked down](https://chromium.googlesource.com/chromiumos/platform2/+/master/patchpanel/firewall.cc#337)
to reject traffic from non-`chronos` UIDs.

[Crostini]: https://chromium.googlesource.com/chromiumos/docs/+/master/containers_and_vms.md
[cicerone]: https://chromium.googlesource.com/chromiumos/platform2/+/master/vm_tools/cicerone/
[chunnel]: https://chromium.googlesource.com/chromiumos/platform2/+/master/vm_tools/chunnel/
