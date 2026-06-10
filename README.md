# smtp5.cogburnbros.com

SMTP relay server that forwards mail to Postmarkapp.

## Host

- **Hypervisor**: pve1.cogburnbros.net (Proxmox VE), QEMU VM
- **OS**: Ubuntu 22.04.5 LTS (Jammy)
- **Kernel**: 5.15.0-181-generic
- **CPU**: 2 vCPUs
- **RAM**: 4 GB (260 MB used)
- **Hostname**: smtp5.cogburnbros.com
- **IP**: 107.1.158.14/27

## Services

### smtprelay (primary)

Go binary, runs as systemd service under `smtpadmin`.

Forked at [cogburnbros/context-smtprelay](https://github.com/cogburnbros/context-smtprelay) because many of our copiers send messages without a body, which Postmark's API rejects. The fork injects a blank body so these messages are accepted.

- **Listen**: `starttls://0.0.0.0:587` and `tls://0.0.0.0:465`
- **TLS enforced**: yes (`local_forcetls = true`)
- **TLS certs**: managed by Caddy via Let's Encrypt (DNS-01 via DNSimple), stored at `/home/smtpadmin/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/smtp5.cogburnbros.com/`
- **Auth**: bcrypt-hashed passwords in `/home/smtpadmin/.config/smtprelay/allowed_users.txt`
- **Relay**: all mail forwards to `smtp.postmarkapp.com:587` via STARTTLS
- **Allowed networks**: `0.0.0.0/0` (UFW restricts actual access)
- **Allowed senders**: restricted per-user to `@smtp5.cogburnbros.com` and `@cogburnbros.com`
- **Binary**: `/home/smtpadmin/bin/smtprelay` (statically linked Go ELF)
- **Config**: `/home/smtpadmin/.config/smtprelay/smtprelay.ini`

### Caddy (TLS certs only)

Caddy v2.7.6 with DNSimple plugin, runs as systemd service under `smtpadmin`.

- **Config**: `/home/smtpadmin/.config/caddy/Caddyfile`
- **Caddyfile**: serves `https://smtp5.cogburnbros.com:1234` with DNS-01 challenge via DNSimple
- **DNSimple token**: in `/home/smtpadmin/.config/caddy/caddy.env`
- **Purpose**: obtains/renews Let's Encrypt certs that smtprelay uses
- **Binary**: `/home/smtpadmin/bin/caddy` (symlink to `caddy-v2.7.6-dnsimple`)

### Other enabled services

- `ssh` (OpenSSH 8.9p1)
- `ufw` (firewall)
- `fail2ban` (auto-ban for SSH and SMTP auth failures)
- `unattended-upgrades` (security updates)
- `apparmor`
- `qemu-guest-agent` (for Proxmox clean shutdown/snapshots)

## Auth Users

SMTP auth accounts are in `/home/smtpadmin/.config/smtprelay/allowed_users.txt` on the server. Don't maintain a copy here.

### Managing Credentials

SMTP auth credentials are stored in `/home/smtpadmin/.config/smtprelay/allowed_users.txt`.

Format: `username bcrypt-hash [email[,email[,...]]]`

The third field restricts allowed sender addresses:
- Omit = user can send from any address
- `@domain.com` = user can send from any address at that domain
- `user@domain.com` = exact match only

**To add or change a user:**

1. SSH in as smtpadmin
2. Generate a bcrypt hash (must be in the source dir):
   ```
   cd ~/Code/smtprelay
   go run cmd/hasher.go 'the-password'
   ```
3. Edit `~/.config/smtprelay/allowed_users.txt` and add/update the line
4. Restart: `sudo systemctl restart smtprelay`

**To remove a user:**

1. Delete the line from `allowed_users.txt`
2. Restart: `sudo systemctl restart smtprelay`

Always back up `allowed_users.txt` before editing. The file is owned by `smtpadmin:smtpadmin`.

## UFW Firewall

Default deny incoming, allow outgoing. Ports open to the world:

- **22/tcp** - SSH (key-only auth, no passwords)
- **465/tcp** - SMTPS (TLS, auth required)
- **587/tcp** - SMTP STARTTLS (auth required)

Previously had 128 per-IP allow rules for SMTP and SSH. Simplified 2026-06-08 since smtprelay enforces TLS + bcrypt auth + sender domain restriction, and SSH is key-only. Old ruleset backed up at `/etc/ufw/user.rules.bak.2026-06-08`.

## fail2ban

fail2ban watches for repeated auth failures and inserts UFW deny rules automatically. Bans expire on their own.

Three jails configured:

| Jail | Trigger | Ban duration | What it watches |
|------|---------|--------------|-----------------|
| sshd | 3 fails / 10min | 24h | Invalid users, root brute force, bad MAC/host key negotiation |
| smtprelay | 5 fails / 10min | 24h | Bad SMTP AUTH (wrong user or wrong password) |
| recidive | 3 bans / 7 days | 30d | IPs that keep coming back after bans expire |

Office network `107.1.158.0/27` is whitelisted from all jails. localhost too.

Config files (drop-in only, no stock edits):

- `/etc/fail2ban/jail.d/smtp5.conf` - jail definitions
- `/etc/fail2ban/filter.d/smtprelay.conf` - custom filter matching `WRN auth error` log lines

Useful commands:

```bash
sudo fail2ban-client status          # all jails
sudo fail2ban-client status sshd     # per-jail detail
sudo fail2ban-client set sshd banip 1.2.3.4   # manual ban
sudo fail2ban-client set sshd unbanip 1.2.3.4 # manual unban
```

## Mail Flow

```
Internal apps/devices → smtp5:587 (STARTTLS) or :465 (TLS)
  → auth via allowed_users.txt
  → relay to smtp.postmarkapp.com:587
  → Postmark delivers to mailbox (Google Workspace or otherwise)
```

Email delivery paths at Cogburn Bros:
1. **Google Workspace** - primary mail
2. **Postmarkapp** - transactional/relay via smtp5
3. **KnowBe4** - phishing simulation (sends fake phishing emails through its own infrastructure)

## SSH Access

Key-only auth (password auth disabled). The `cogburnadmin1` ed25519 private key is in the **IT 1Password Vault**.

## Backups

VM is backed up regularly to PBS (Proxmox Backup Server) via the hypervisor. Before making risky changes, trigger a manual backup from pve1 so you can restore the VM if something goes wrong.

Nothing known.

## Completed (2026-06-10)

### fail2ban

Installed fail2ban with three jails (sshd, smtprelay, recidive). All bans use UFW action so denied IPs are visible in `ufw status`. Bans auto-expire.

Custom smtprelay filter matches `WRN auth error` log lines:
- `error="User not found"` - unknown SMTP username
- `error="Password invalid"` - wrong password for known username

Both patterns confirmed via live test with `openssl s_client`.

Also removed 83 static per-IP UFW deny rules that were added earlier in the session. fail2ban now handles all auto-banning.

### SMTP relay opened to the world

Ports 465 and 587 were opened to all IPs (UFW `allow` rules without source restriction). Previously SMTP was restricted to known office/scanner IPs. This is safe because smtprelay enforces TLS + bcrypt auth + sender domain restriction per user.

## Completed (2026-06-08)

### Kernel / Reboot

- Upgraded kernel from 5.15.0-171 to 5.15.0-181 via `apt dist-upgrade`
- Rebooted, `/var/run/reboot-required` cleared

### Package Updates

- Full `apt dist-upgrade` applied (63 packages)
- 0 upgradable remaining

### qemu-guest-agent

- Installed and active. Proxmox can now do clean shutdowns and snapshot freeze/thaw.

### cloud-init

- Disabled via `/etc/cloud/cloud-init.disabled`

### snapd

- Removed lxd and core20 snaps, purged snapd, autoremoved dependencies

### SSH Hardening

Drop-in config at `/etc/ssh/sshd_config.d/hardening.conf`:

```
PasswordAuthentication no
ChallengeResponseAuthentication no
HostKey /etc/ssh/ssh_host_ed25519_key
MACs umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
```

Also cleaned main `sshd_config` (commented out `PasswordAuthentication yes` and `X11Forwarding yes`).
Removed DSA, RSA, and ECDSA host keys (backed up then deleted).

| Setting | Before | After | Status |
|---------|--------|-------|--------|
| `PasswordAuthentication` | yes | no | Fixed |
| `HostKey` | RSA+ECDSA+Ed25519 | Ed25519 only | Fixed |
| `MACs` | includes hmac-sha1, non-ETM | ETM only | Fixed |
| `ClientAliveInterval` | 0 | 300 | Fixed |
| `ClientAliveCountMax` | 3 | 2 | Fixed |
| `LoginGraceTime` | 120 | 30 | Fixed |
| `X11Forwarding` | yes | no | Fixed |
| `AllowTcpForwarding` | yes | no | Fixed |
| `ChallengeResponseAuthentication` | not set | no | Fixed |
| `ssh_host_dsa_key` | present | removed | Fixed |

## Key File Paths

| Path | Purpose |
|------|---------|
| `/home/smtpadmin/bin/smtprelay` | smtprelay binary |
| `/home/smtpadmin/bin/smtprelay-install` | serviceman install script |
| `/home/smtpadmin/.config/smtprelay/smtprelay.ini` | smtprelay config |
| `/home/smtpadmin/.config/smtprelay/allowed_users.txt` | SMTP auth credentials |
| `/home/smtpadmin/.config/smtprelay/allowed_users.2026-04-02.txt` | backup of auth file |
| `/home/smtpadmin/bin/caddy` | Caddy binary (symlink) |
| `/home/smtpadmin/bin/caddy-install` | serviceman install script |
| `/home/smtpadmin/.config/caddy/Caddyfile` | Caddy config |
| `/home/smtpadmin/.config/caddy/caddy.env` | DNSimple API token |
| `/home/smtpadmin/.local/share/caddy/certificates/` | Let's Encrypt certs |
| `/home/smtpadmin/bin/mail-test` | curl-based SMTP test script |
| `/home/smtpadmin/bin/backups/` | backup dir |
| `/var/log/smtprelay/` | log dir (empty, logs go to journald) |
| `/etc/fail2ban/jail.d/smtp5.conf` | fail2ban jail definitions |
| `/etc/fail2ban/filter.d/smtprelay.conf` | custom smtprelay auth failure filter |

### Tailing Logs

```bash
sudo journalctl -xefu smtprelay
```
