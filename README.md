# DirectAdmin Loading & Monitoring tools

Root-only commands for this DirectAdmin host. Outbound mail uses msmtp. See [Mail (msmtprc)](#mail-msmtprc). Run them as root.

| Command | What it does |
| --- | --- |
| `da-hits` | Monthly web-hit digest from Webalizer (AWStats if that data is present) |
| `da-load` | Load, hot files, today's access logs, mail logs, OpenLiteSpeed |

## Mail (msmtprc)

Local Exim on this host does not deliver these reports. Root is on the `never_users` list, so a message handed to the local MTA is rejected instead of leaving the box. `da-hits` and `da-load` skip that path when msmtp is configured and submit over authenticated SMTP.

Config file: `/root/.config/da-hits/msmtprc`  
Log file: `/root/.config/da-hits/msmtp.log`  
Binary: `/usr/bin/msmtp`

The config must be mode `600` and owned by root. Both commands call:

```bash
msmtp -C /root/.config/da-hits/msmtprc -t
```

`-t` reads the recipients from the message headers. If that file is missing or unreadable, the commands fall back to `/usr/sbin/sendmail`, which is the path that gets stuck locally.

`account default : da-hits` makes this the account msmtp uses. `from` and `user` are the authenticated mailbox (the envelope sender). The message `From:` header is set by the script (`MAIL_FROM`) and can be a different address on the same domain. Put the SMTP password only in this file, not in the scripts or in this README.

```
defaults
auth           on
tls            on
tls_starttls   on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /root/.config/da-hits/msmtp.log

account        da-hits
host           mail.example.com
port           587
from           reports@o.example.com
user           reports@o.example.com
password       your-smtp-password

account default : da-hits
```

Create it like this:

```bash
install -d -m 700 /root/.config/da-hits
install -m 600 /dev/null /root/.config/da-hits/msmtprc
# edit the file, then:
msmtp -C /root/.config/da-hits/msmtprc --serverinfo
```

A successful send appends a line to `msmtp.log` with `smtpstatus=250`. Auth failures show up there as `535`.

Overrides, if the files live somewhere else:

| Variable | Default |
| --- | --- |
| `MSMTP_CONFIG` | `/root/.config/da-hits/msmtprc` |
| `MSMTP_BIN` | `/usr/bin/msmtp` |
| `MAIL_FROM` | Header From address set in the script |
| `DEFAULT_EMAIL` | Recipient used by `da-load-mail` when `-e` is omitted |

## da-hits

Current month plus a short lookback. Stats come from `~/domains/*/stats/`.

```bash
da-hits
da-hits -d example.com
da-hits -d example.com -e ops@example.com -q
da-hits -u someuser -m 3 -v
da-hits --json
```

| Option | Meaning |
| --- | --- |
| `-u`, `--user USER` | One DirectAdmin user |
| `-d`, `--domain DOMAIN` | One domain, or a subdomain label |
| `-m`, `--months N` | History depth, 1–120 (default 6) |
| `-e`, `--email ADDR` | Email the report. Repeatable, or comma-separated |
| `-s`, `--subject TEXT` | Subject override |
| `-q`, `--quiet` | No stdout. Use with `--email` or cron |
| `-v`, `--verbose` | Per-domain rows for each history month |
| `--json` | Machine-readable output. Cannot be combined with `--email` |

Cron (`/etc/cron.d/da-hits`), daily at 01:25 after the DirectAdmin tally at 00:10. One quiet email per domain, for example:

- example.com → ops@example.com
- exampleb.com → ops@exampleb.com
- examplec.com → ops@examplec.com

## da-load

One snapshot of the host. With no section, it prints all of them.

```bash
da-load
da-load load
da-load hits -d example.com
da-load topbyip
da-load topbyip -d example.com -n 20
da-load mail
da-load lsws
da-load -e ops@example.com
```

Sections: `load`, `hits`, `mail`, `lsws`, `topbyip`. `lightspeed` and `ols` are aliases for `lsws`. `topbyip` is opt-in and is left out of a plain `da-load` run.

| Section | Source |
| --- | --- |
| `load` | Load average, memory, disk, top CPU processes |
| `hits` | Webalizer month-to-date URL table, plus today's vhost access logs in `/var/log/httpd/domains/*.log` |
| `mail` | Exim queue, `/var/log/exim/mainlog`, `/var/log/exim/paniclog`, `/var/log/mail.log` |
| `lsws` | `lshttpd` processes, `lswsctrl status`, `/var/log/openlitespeed/error_log` |
| `topbyip` | Client IP counts from each `/var/log/httpd/domains/*.log`. Every IP is listed unless `-n` caps the list |

A site is flagged when month-to-date hits are at least 100,000 and at least triple the previous month. A single URL is flagged at 100,000 hits. `mailer.php` and `send-mail.php` are called out even when they sit below the top of the table.

| Option | Meaning |
| --- | --- |
| `-d`, `--domain DOMAIN` | Limit the hits section, today's access logs, and `topbyip` |
| `-n`, `--lines N` | Sample depth, 1–200 (default 12). For `topbyip`, how many IPs to list per domain |
| `-t`, `--threshold N` | Hit count that raises an alert (default 100000) |
| `-e`, `--email ADDR` | Email the report. Repeatable, or comma-separated |
| `-s`, `--subject TEXT` | Subject override |
| `-q`, `--quiet` | No stdout |

`lswsctrl status` can report "not running" while workers are up, because it looks for a pid file. The process list in the report is the check to trust. The server-wide OpenLiteSpeed access log is usually empty; per-domain hits are in `/var/log/httpd/domains/`.
