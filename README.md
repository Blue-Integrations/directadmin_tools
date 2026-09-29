# /root/bin

Root-only commands for this DirectAdmin host. Both mail through the msmtp account in `/root/.config/da-hits/msmtprc`. Run them as root.

| Command | What it does |
| --- | --- |
| `da-hits` | Monthly web-hit digest from Webalizer (AWStats if that data is present) |
| `da-load` | Load, hot files, today's access logs, mail logs, OpenLiteSpeed |

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
da-load mail
da-load lsws
da-load -e ops@example.com
```

Sections: `load`, `hits`, `mail`, `lsws`. `lightspeed` and `ols` are aliases for `lsws`.

| Section | Source |
| --- | --- |
| `load` | Load average, memory, disk, top CPU processes |
| `hits` | Webalizer month-to-date URL table, plus today's vhost access logs in `/var/log/httpd/domains/*.log` |
| `mail` | Exim queue, `/var/log/exim/mainlog`, `/var/log/exim/paniclog`, `/var/log/mail.log` |
| `lsws` | `lshttpd` processes, `lswsctrl status`, `/var/log/openlitespeed/error_log` |

A site is flagged when month-to-date hits are at least 100,000 and at least triple the previous month. A single URL is flagged at 100,000 hits. `mailer.php` and `send-mail.php` are called out even when they sit below the top of the table.

| Option | Meaning |
| --- | --- |
| `-d`, `--domain DOMAIN` | Limit the hits section and today's access logs |
| `-n`, `--lines N` | Sample depth, 1–200 (default 12) |
| `-t`, `--threshold N` | Hit count that raises an alert (default 100000) |
| `-e`, `--email ADDR` | Email the report. Repeatable, or comma-separated |
| `-s`, `--subject TEXT` | Subject override |
| `-q`, `--quiet` | No stdout |

`lswsctrl status` can report "not running" while workers are up, because it looks for a pid file. The process list in the report is the check to trust. The server-wide OpenLiteSpeed access log is usually empty; per-domain hits are in `/var/log/httpd/domains/`.

```bash
/root/bin/da-load-mail
```
