# Logging on Chrome OS

## Syslog Logs (managed by rsyslog)

Services write the logs into files using syslog like a Linux. Application can call the syslog APIs, the API writes the messages into the syslog socket (/dev/log), the syslog daemon (rsyslogd) receives the message and the daemon writes it to the file.

### Locations

System logs are collected by rsyslog daemon from applications and stored in `/var/log/`. The rule of destinations is defined in the configuration files.

Logs defined in [rsyslog.chromeos](https://source.chromium.org/chromiumos/chromiumos/codesearch/+/master:src/platform2/init/rsyslog.chromeos;l=1?q=rsyslog.chromeos):

* `/var/log/messages`: general system logs.
* `/var/log/net.log`: network-related logs.
* `/var/log/boot.log`: boot messages.
* `/var/log/secure`: logs with authpriv facility. May contain sensitive information.
* `/var/log/upstart.log`: upstart logs.

Logs defined in (rsyslog.arc.conf)[https://source.chromium.org/chromiumos/chromiumos/codesearch/+/master:src/platform2/arc/scripts/rsyslog.arc.conf?q=rsyslog.arc.conf]:

* `/var/log/arc.log`: ARC-related logs gathered by rsyslogd.

Logs defined in (rsyslog.hammerd.conf)[https://source.chromium.org/chromiumos/chromiumos/codesearch/+/master:src/platform2/hammerd/rsyslog/rsyslog.hammerd.conf;l=1?q=rsyslog.ham]:

* `/var/log/hammerd.log`: network-related logs gathered by rsyslogd.

### Format

We are using a custom format for Chrome OS system log. It has microsec precision and a time zone depending on the system setting.

Example of format:

> 2020-03-10T14:02:08.470152+09:00 INFO processname[12345]: This is the log message.

The format is defined in [rsyslog.chromeos](https://source.chromium.org/chromiumos/chromiumos/codesearch/+/master:src/platform2/init/rsyslog.chromeos;drc=55172975434220f3a234cada1f76a5fb109bd2e0;l=18).

### Journald Deprecation

Jounald is deprecated and is about to be removed. Journal log has a storage and CPU overheads and it’s not suitable for, at-least, Chrome OS devices. Journal log has multiple non-important fields and has multiple wide (64 or 128bit) integers for IDs. These metadata consumes storage in devices. And journald has a dedup future which saves storage for the same value entries. But in Chrome OS use-case, we don’t have much duplicated entries and dedup needs CPU time to look up a duplicated previous entry. The impact is not ignorable especially for low-end devices. So we have decided to remove journald from Chrome OS until these problems are addressed .


## Log Rotation

Major log files are rotated by (chromeos-cleanup-logs)[https://source.corp.google.com/chromeos_public/src/platform2/init/chromeos-cleanup-logs?q=chromeos-cleanup-logs] script, which rotates logs every 24 hours and keeps 7 histories.

Chrome logs are rotated by itself (to be more specific, a new log file is generated when every process starts). And old logs are removed by chromeos-cleanup-logs script.


## Logs from Chrome processes

See the chrome side document: [Chrome Logging on Chrome OS](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/chrome_os_logging.md)
