# abuseipdb-block

Drop inbound traffic from IPs on the [AbuseIPDB](https://www.abuseipdb.com) blacklist,
using a single nftables set. One bash script, no daemon, no dependencies beyond
`nft` and `curl`.

- **Self-contained**: its own table (`inet abuseipdb`); it never touches rules created by
  fail2ban, Docker, WireGuard or your own firewall, and needs no `netfilter-persistent`.
- **Cheap**: ~90,000 IPs cost one set lookup per new connection and a few MB of RAM.
- **Atomic**: every update replaces the whole table in one nftables transaction; IPs that
  leave the list are unblocked automatically.
- **Safe on failure**: if the download fails or the response looks wrong, the previous
  list stays loaded; if no update succeeds for `MAXAGE` days the table is removed, as
  AbuseIPDB asks for lists that cannot be kept current.
- **Only new inbound connections** are dropped (`ct state new`): replies to connections
  opened by the host itself still work.
- **Allowlist** for your own addresses, and a one-command unblock.

## Requirements

- Linux with nftables (`nft`), conntrack, `curl`, bash 4+
- An AbuseIPDB account and [API key](https://www.abuseipdb.com/account/api)

| Plan | IPs returned | `confidenceMinimum` |
|------|--------------|---------------------|
| Free | 10,000 (most recently reported at 100%) | fixed at 100 |
| Basic | up to 100,000 | 25–100 |
| Premium | up to 500,000 | 25–100 |

The script works unchanged on any plan: the API silently truncates the list to the
plan's limit. The free plan allows 5 blacklist requests per day; the script uses one.

## Install

```bash
sudo install -m 700 abuseipdb-block /usr/local/sbin/
sudo install -m 600 abuseipdb-block.conf.example /etc/abuseipdb-block.conf
sudo editor /etc/abuseipdb-block.conf                 # set ALLOW
sudo install -m 600 /dev/null /etc/abuseipdb.key
echo 'YOUR_API_KEY' | sudo tee /etc/abuseipdb.key >/dev/null
sudo abuseipdb-block -n                               # download and verify only
sudo abuseipdb-block && sudo abuseipdb-block -s
sudo install -m 644 abuseipdb-block.cron /etc/cron.d/abuseipdb-block
```

## Usage

```
abuseipdb-block [-c FILE] [-q] [-n]   download and apply the list
abuseipdb-block -r [-q]                reload the last saved list (for boot)
abuseipdb-block -s                     status: loaded IPs, list age, dropped packets
abuseipdb-block -u IP                  unblock IP until the next update
abuseipdb-block -x                     remove the table until the next update
```

Locked out? From another network (e.g. a phone), `abuseipdb-block -u YOUR_IP`, then add
it to `ALLOW` in `/etc/abuseipdb-block.conf`.

Exit codes: 0 success, 1 update/reload failed (previous state kept), 2 usage error.

## How it works

```
table inet abuseipdb {
  set allow4 / allow6   # ALLOW, interval sets (CIDR)
  set bl4 / bl6         # the blacklist
  chain input {
    type filter hook input priority filter - 5; policy accept;
    ct state != new return
    ip saddr @allow4 return
    ip6 saddr @allow6 return
    ip saddr @bl4 counter drop
    ip6 saddr @bl6 counter drop
  }
}
```

The last good list is kept in `/var/lib/abuseipdb-block/`. Only inbound traffic to the
host is filtered; forwarded traffic (containers, VPN, routing) is not.

## License

MIT
