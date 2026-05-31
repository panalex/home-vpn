# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

```bash
ansible-galaxy collection install -r requirements.yml
cp inventory.ini.example inventory.ini
cp group_vars/all.yml.example group_vars/all.yml
# Edit inventory.ini and group_vars/all.yml with real IPs, domains, passwords
```

Generate secrets:
```bash
openssl rand -base64 32          # Shadowsocks 2022 secret
LC_ALL=C tr -dc 'A-Za-z0-9_@%+=:.,-' </dev/urandom | head -c 32; echo  # user passwords
```

## Commands

```bash
# Full VPS deploy
ansible-playbook -i inventory.ini site.yml

# Router only
ansible-playbook -i inventory.ini router.yml

# Users + edge sing-box only (fastest for user sync)
ansible-playbook -i inventory.ini site.yml --tags users,ru_singbox

# Syntax check (CI equivalent)
yamllint -d relaxed . && ansible-playbook --syntax-check site.yml

# Dry-run a role
ansible-playbook -i inventory.ini site.yml --tags de_ocserv --check --diff
```

## Architecture

Two VPS nodes + one GL-MT6000 router, deployed via two playbooks: `site.yml` (VPS) and `router.yml` (OpenWrt).

**Edge VPS** (`ru_vps`): public ingress. Runs nginx (TLS fronting/ACME), `mtg` (Telegram MTProto FakeTLS on :443), `sing-box` (HTTP proxy :2080, SOCKS :2081 with auth, local loopback :1080/:1081 for mtg). All proxy traffic chains to gateway via Shadowsocks 2022 on :4433.

**Gateway VPS** (`de_vps`): egress + VPN concentrator. Runs `ocserv` (OpenConnect/AnyConnect VPN on :443 TCP/UDP), `unbound` (DNS for VPN clients via DoT), `sing-box` (ss-in :4433, restricted by firewall to edge IP only). VPN subnet: `10.99.0.0/24`, DNS: `10.99.0.1`.

**Router** (`routers`): GL-MT6000 with OpenWrt + `sing-box` TUN (`singtun0`). FakeIP DNS + split routing: RU IPs/domains go direct, everything else tunnels to gateway via SS2022 :8388.

## Key design decisions

**vpns0 is lazy**: ocserv creates `vpns0` only on first client connect. The route `10.99.0.0/24 dev vpns0` is added by `connect-script` (`route-up.sh`), not by Ansible tasks. nftables rules referencing `vpns0` are safe to load before any client connects.

**Unified users**: `vpn_users` in `group_vars/all.yml` is used for both `ocserv` (VPN passwords) and `sing-box` (proxy auth). The `proxy_users` alias points to the same list. The `users` role syncs `/etc/ocserv/ocpasswd` from this list.

**sing-box chain**: Edge → Gateway uses Shadowsocks 2022 with multiplexing. The :4433 inbound on gateway is firewall-restricted to edge IP only, so it's not an open relay.

**mtg routing**: mtg on edge forwards through the local sing-box SOCKS (:1081) which then chains to gateway — so Telegram traffic also exits through the gateway IP.

**unbound DNSSEC fix**: Removes Ubuntu's conflicting `root-auto-trust-anchor-file.conf` and uses `trust-anchor-file: "/usr/share/dns/root.key"` explicitly.

## Variables

All secrets and host-specific config live in `group_vars/all.yml` (gitignored). Key vars:
- `singbox_server_password` — SS2022 secret for edge→gateway chain (base64, 32 bytes)
- `router_ss_secret` — separate SS2022 secret for router→gateway (do not reuse the chain secret)
- `vpn_users` — list of `{name, password}` for both VPN and proxy
- `ocserv_domain` / `ocserv_acme_email` — gateway domain (must resolve to gateway IP before first run)

## Post-deploy verification

Gateway:
```bash
systemctl status ocserv unbound sing-box --no-pager
ss -lntup | egrep '(:443|:4433|:53)'
nft list ruleset
```

Edge:
```bash
systemctl status nginx mtg sing-box --no-pager
ss -lntup | egrep '(:443|:2080|:2081)'
curl --proxy http://USER:PASSWORD@EDGE_IP:2080 https://ifconfig.me
```

Router:
```bash
service sing-box status && ip a show singtun0
```

## Known gaps (from docs/architecture.md)

- Secrets are plaintext in `group_vars/all.yml` — no Ansible Vault or SOPS yet
- No healthchecks (ocserv login test, DNS-through-VPN test, proxy egress IP)
- IPv6 is disabled globally (`common_disable_ipv6: true`), not routed
- No rate limiting on public proxy endpoints
- No fail2ban/dynamic nftables sets for SSH/ocserv brute-force
