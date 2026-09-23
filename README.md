# check_nextcloud

A Nagios/Icinga plugin for [Nextcloud](https://nextcloud.com/), written in
Python 3 with the standard library only. `health` reads the unauthenticated
`status.php`; every other mode reads the REST API of the **serverinfo** app,
which ships with Nextcloud and has to be enabled.

| Mode | Answers | Default thresholds |
|---|---|---|
| `health` | Is the instance installed, out of maintenance mode and answering? | none |
| `cpu` | What is the load average of the server? | none |
| `memory` | How much RAM and swap is used? | `-w 90 -c 95` (percent) |
| `freespace` | How much space is left in the data directory? | `-w 20G: -c 5G:` |
| `storage` | How many files, users and storages, and how big is the database? | none |
| `users` | How many users were active recently? | none |
| `apps` | How many apps have an update available? | `-w 0` |

`rotate` is not a check: it replaces the app password, see
[App password rotation](#app-password-rotation).

## Usage

```sh
check_nextcloud health    -H cloud.example.com
check_nextcloud memory    -H cloud.example.com -f /etc/icinga2/nextcloud.secret -w 85 -c 95
check_nextcloud freespace -H cloud.example.com -f /etc/icinga2/nextcloud.secret -w 50G: -c 10G:
check_nextcloud apps      -H cloud.example.com -f /etc/icinga2/nextcloud.secret
```

Common options: `-H` host name (required), `-p` port (default 443), `--no-ssl`
for plain HTTP, `-k` to skip certificate verification, `-t` timeout in seconds,
`-u` user name (default `admin`).

## App password

The serverinfo API needs an administrator. Create an app password in Nextcloud
under **Personal settings → Security → Devices & sessions**, and pass it
together with the user it belongs to (`-u`).

Never pass it with `-P` in a service definition: command lines are visible to
every user via `ps`. Put it in a file that only the monitoring user can read:

```sh
umask 077
printf '%s' 'xxxxx-xxxxx-xxxxx-xxxxx-xxxxx' > /etc/icinga2/nextcloud.secret
chown root:nagios /etc/icinga2/nextcloud.secret
chmod 0640 /etc/icinga2/nextcloud.secret
```

The password is taken from, in this order:

1. `-P` / `--password` (discouraged, see above)
2. `-f` / `--password-file`
3. `$NEXTCLOUD_APP_PASSWORD_FILE`
4. `$NEXTCLOUD_APP_PASSWORD`

## App password rotation

`check_nextcloud rotate` replaces the app password and writes the new one back
into the password file:

```sh
check_nextcloud rotate -H cloud.example.com -f /etc/icinga2/nextcloud.secret
```

Nextcloud app passwords do not expire, so every run rotates and the cron
schedule decides how often. Only app passwords can be rotated, not login
passwords. The new password is 72 characters long instead of the
`xxxxx-xxxxx-xxxxx-xxxxx-xxxxx` format, which is expected.

Run it from cron as the owner of the password file or as root, not as a check
of the monitoring system, for example weekly:

```
# /etc/cron.d/check_nextcloud-rotate
17 3 * * 0 root chronic /usr/lib/nagios/plugins/check_nextcloud rotate -H cloud.example.com -f /etc/icinga2/nextcloud.secret
```

`chronic` from moreutils keeps cron quiet unless a run fails. If a rotation
leaves a broken app password behind, every check except `health` reports it as
UNKNOWN.

How the rotation stays safe:

- A lock file next to the credential (`<file>.lock`) makes sure only one
  rotation runs at a time. The directory has to be writable for it and for the
  replacement file.
- The replacement file is created with the owner and mode of the old one
  before anything is sent to the server, then written, synced and renamed over
  the old file. Checks read either the old or the new app password, never a
  half-written one; a check that read the old one just before the switch may
  report UNKNOWN once.
- A failed rotation is never retried. When it is unclear whether the server
  rotated (timeout, server error), `rotate` exits CRITICAL and leaves the file
  alone; the next run shows whether the stored app password still works.
- Use a separate app password per monitoring host. Copies of the same file on two
  hosts break as soon as one of them rotates.

`rotate` exits `0` when it rotated, `2` when the outcome is uncertain or the
new app password does not work, and `3` when nothing was rotated (rejected app
password, missing permission, unwritable directory, another rotation running).

## Thresholds

`-w` and `-c` take a range as defined by the
[monitoring plugins guidelines](https://www.monitoring-plugins.org/doc/guidelines.html#THRESHOLDFORMAT):
`10` alerts outside 0..10, `10:` below 10, `~:10` above 10, `10:20` outside
10..20, and `@10:20` inside 10..20.

Byte thresholds accept the suffixes `K`, `M`, `G`, `T` and `P`. Free space is
checked with a lower bound, so `-w 20G:` warns once less than 20 GiB are left.
Modes with a default threshold still accept `-w ~:` to switch the alert off.

## Performance data

```
NEXTCLOUD MEMORY OK - 42.5% of the memory used (1.2 GiB of 2.8 GiB), swap 6.0% used | usage=42.54%;90;95;0;100 used=1285554176B;;;0;3021996032 swap_usage=5.98%;;;0;100 swap_used=93323264B;;;0;1560281088
```

## Exit codes

`0` OK, `1` WARNING, `2` CRITICAL, `3` UNKNOWN. An unreachable server, an
uninstalled instance and a pending database upgrade are CRITICAL; maintenance
mode is WARNING; rejected credentials and unparsable answers are UNKNOWN.
