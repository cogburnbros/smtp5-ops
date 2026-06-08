# AGENTS.md - smtp5.cogburnbros.com

## What This Server Does

SMTP relay for Cogburn Bros. Accepts mail from internal apps and devices on ports 587/465 with auth, relays everything to Postmarkapp.

## Architecture

- Go binary `smtprelay` (github.com/decke/smtprelay) under `smtpadmin` user
- Caddy (v2.7.6 + DNSimple plugin) handles Let's Encrypt cert renewal via DNS-01
- UFW firewall: SSH/465/587 open, default deny everything else
- No Postfix, no Dovecot, no traditional mail stack - just the relay binary

## Important

- The `allowed_users.txt` contains bcrypt-hashed SMTP credentials. Do not modify without backup.
- The Postmark API key is in `smtprelay.ini` as the relay password. Treat as secret.
- Caddy's DNSimple OAuth token is in `caddy.env`. Treat as secret.
- TLS certs are Caddy-managed; smtprelay reads them directly from Caddy's cert store.

## SSH Access

`ssh smtp5.cogburnbros.com` - key-only auth (password auth disabled). The `cogburnadmin1` ed25519 private key is in the **IT 1Password Vault**.

## Completed (2026-06-08)

- Kernel upgraded 5.15.0-171 → -181, rebooted
- Full apt dist-upgrade applied (63 packages, 0 remaining)
- qemu-guest-agent installed and active
- cloud-init disabled
- snapd and lxd removed
- SSH hardened (password auth off, ed25519 only, ETM MACs, ClientAlive, LoginGraceTime 30, no X11, no TCP forwarding)
- DSA/RSA/ECDSA host keys removed
- UFW simplified from 128 per-IP rules to 3 open port rules (22, 465, 587)

## Making Changes

- smtprelay: edit config, then `sudo systemctl restart smtprelay`
- Caddy: edit Caddyfile/env, then `sudo systemctl restart caddy`
- UFW: `sudo ufw` commands
- SSH: prefer drop-in at `/etc/ssh/sshd_config.d/hardening.conf`; validate with `sshd -t` before restart
- Always test SSH connectivity from a separate terminal after SSH changes
