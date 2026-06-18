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

# Router sing-box only (transport changes) / nfqws only
ansible-playbook -i inventory.ini router.yml --tags owrt_singbox
ansible-playbook -i inventory.ini router.yml --tags owrt_nfqws

# Edge inbounds for the home→edge leg (hy2-in/shadowtls-in) + firewall
ansible-playbook -i inventory.ini site.yml --tags ru_singbox,firewall

# Reverse-SSH management tunnel — gateway side then router side
ansible-playbook -i inventory.ini site.yml   --tags de_mgmt
ansible-playbook -i inventory.ini router.yml --tags owrt_mgmt

# Users + edge sing-box only (fastest for user sync)
ansible-playbook -i inventory.ini site.yml --tags users,ru_singbox

# Syntax check (CI equivalent)
yamllint -d relaxed . && ansible-playbook --syntax-check site.yml

# Dry-run a role
ansible-playbook -i inventory.ini site.yml --tags de_ocserv --check --diff
```

## Architecture

Two VPS nodes + one GL-MT6000 router, deployed via two playbooks: `site.yml` (VPS) and `router.yml` (OpenWrt).

**Edge VPS** (`ru_vps`): public ingress. Runs nginx (TLS fronting/ACME), `mtg` (Telegram MTProto FakeTLS on :443), `sing-box` (HTTP proxy :2080, SOCKS :2081 with auth, local loopback :1080/:1081 for mtg), `gost-tgproxy` (TCP bridge loopback :9443 → api.telegram.org:443 via sing-box SOCKS). All proxy traffic chains to gateway via Shadowsocks 2022 on :4433. `tapi.home12.ru` is a Telegram Bot API reverse proxy: Zabbix → nginx:8443 → gost:9443 → sing-box:1081 → gateway → api.telegram.org.

**Gateway VPS** (`de_vps`): egress + VPN concentrator. Runs `ocserv` (OpenConnect/AnyConnect VPN on :443 TCP/UDP), `unbound` (DNS for VPN clients via DoT), `sing-box` (ss-in :4433, restricted by firewall to edge IP only). VPN subnet: `10.99.0.0/24`, DNS: `10.99.0.1`.

**Router** (`routers`): GL-MT6000 with OpenWrt + `sing-box` TUN (`singtun0`), deployed by the **`owrt_singbox`** + **`owrt_nfqws`** roles (the previous `flint_*` roles were dropped — over-debugged and unreliable; rebuilt clean as `owrt_*`). Split routing (by sniffed SNI + geoip): RU IPs/domains go direct, everything else (incl. RKN-blocked sites by default) tunnels to gateway via the `proxy` outbound. **The home→edge leg uses hysteria2** — raw SS2022 `router-in` :8388 was getting cut by the ISP TSPU (TCP handshake passes, first ciphertext block dropped → blackhole), so edge now exposes two always-on inbounds: `hy2-in` (hysteria2 over QUIC/UDP + obfs salamander, **primary**) and `shadowtls-in` (shadowtls v3 → internal SS2022, TCP fallback). Both route into the existing `de-ss` chain on edge. **First pass is hysteria2-only on the router**: `owrt_singbox` renders just the hysteria2 outbound as tag `proxy` and asserts `router_primary_transport == hysteria2`; the shadowtls outbound on the router is a planned 2nd pass (edge already listens on `shadowtls-in`, so switching later = add the outbound + redeploy the router only, edge untouched). Brutal is intentionally OFF (no `up_mbps`/`down_mbps`) — fixed-rate CC produces an anomalous flat traffic pattern that aids behavioral detection. `dpi_bypass_domains` (default empty) is an opt-in third category: listed domains go direct WAN + nfqws DPI-desync instead of the tunnel. Split-DNS: `*.ru` + `geosite-ru` + `dpi_bypass_domains` resolve via Yandex **DoT** (`dns-local`, anti-spoofing, direct WAN); **everything else resolves via DoH over the tunnel** (`dns-remote` = 1.1.1.1, `detour: proxy`) — real IPs resolved at the gateway, no FakeIP. A cron watchdog (`/usr/bin/singbox-watchdog`) restarts `sing-box` if the process dies.

## Key design decisions

**vpns0 is lazy**: ocserv creates `vpns0` only on first client connect. The route `10.99.0.0/24 dev vpns0` is added by `connect-script` (`route-up.sh`), not by Ansible tasks. nftables rules referencing `vpns0` are safe to load before any client connects.

**Unified users**: `vpn_users` in `group_vars/all.yml` is used for both `ocserv` (VPN passwords) and `sing-box` (proxy auth). The `proxy_users` alias points to the same list. The `users` role syncs `/etc/ocserv/ocpasswd` from this list.

**sing-box chain**: Edge → Gateway uses Shadowsocks 2022 with multiplexing. The :4433 inbound on gateway is firewall-restricted to edge IP only, so it's not an open relay.

**mtg routing**: mtg on edge forwards through the local sing-box SOCKS (:1081) which then chains to gateway — so Telegram traffic also exits through the gateway IP.

**unbound DNSSEC fix**: Removes Ubuntu's conflicting `root-auto-trust-anchor-file.conf` and uses `trust-anchor-file: "/usr/share/dns/root.key"` explicitly.

**Router RU DNS over TLS**: `dns-local` uses Yandex DoT by IP (`type: tls`, `server: 77.88.8.8`, `server_port: 853`, `tls.server_name: common.dot.dns.yandex.net`) instead of plain UDP:53. It carries **no** `detour` — in sing-box 1.12+ a DNS server without a detour already dials direct (DNS transports bypass outbound routing and use the auto-route-aware direct dialer), so it still goes out the RU WAN. Adding `detour: direct` is now a FATAL error (`detour to an empty direct outbound makes no sense`) and crash-loops sing-box; only `dns-remote` keeps an explicit `detour: proxy` to force it through the tunnel. RKN/TSPU spoof plain DNS responses for blocked domains (instagram/facebook/discord) on-path; a poisoned IP would send nfqws desync packets into a blackhole. DoT by IP is spoof-proof and needs no bootstrap; the RU location still yields a directly-reachable CDN node.

**Caveat — Yandex DoT censors Meta**: queried from the router's RU IP (`dns-local`, direct WAN), Yandex returns **NXDOMAIN** for Meta domains (`instagram.com`/`facebook.com`/`cdninstagram.com`/`fbcdn.net`) per RKN — DoT prevents on-path spoofing, not the resolver's own policy. So Meta domains must NOT sit in `dpi_bypass_domains` with `dns-local` resolution: either let them go through the tunnel (`final: proxy`, resolved at the gateway — the default since `dpi_bypass_domains: []`), or resolve them via `dns-remote` if kept on nfqws-direct. Symptom of the misconfig: `DNS_PROBE_FINISHED_NXDOMAIN` in the browser.

**Router fail-open + watchdog (no hard kill-switch)**: a hard kill-switch was rejected for a home network. Tunnel-down ≠ sing-box-down: while sing-box is alive, `direct`/`dpi-bypass-direct` (RU + zapret) keep working and only the `final: proxy` (geo-blocked) category fails — and it fails closed at the app level (no leak), no firewall rule needed. The only leak window is a rare process crash, which a 1-minute cron watchdog (`router_singbox_watchdog`) recovers automatically. `mtu_fix: '1'` on the `proxy` zone clamps MSS for TCP inside the tunnel.

**Reverse-SSH management tunnel (`de_mgmt` + `owrt_mgmt`)**: the router is behind CGNAT/TSPU with no inbound reachability, so to reach it *from* a VPS the router must initiate. It runs a persistent `ssh -R 127.0.0.1:{{ mgmt_reverse_port }}:127.0.0.1:22` to the **gateway**, supervised by procd (`/etc/init.d/mgmt-tunnel`, infinite respawn; `ServerAlive*` + `ExitOnForwardFailure` detect a dead link). The gateway was chosen over the edge for two reasons: (1) the router already reaches it through the existing tunnel — the gateway's public IP is foreign, so sing-box routes the ssh via `proxy`/hy2 and TSPU sees only QUIC (a direct WG/SSH to the gateway would likely be cut like SS2022 was); (2) terminating on the edge and routing it through the hy2 tunnel (which itself lands on the edge) would be a **loop**. From the gateway: `ssh -p {{ mgmt_reverse_port }} root@127.0.0.1`. The `rtun` user on the gateway is `nologin` and its key is `restrict,port-forwarding`; a `Match User rtun` sshd drop-in caps it to `PermitListen 127.0.0.1:<port>` and reaps dead sessions via `ClientAlive*` so the loopback port is free on reconnect. The keypair is generated on the control node into `./secrets/` (gitignored) so playbook order (`site.yml`→`router.yml`) doesn't matter. **Tradeoff accepted**: the channel rides the data-plane tunnel, so it shares fate with router sing-box. A full out-of-band channel (separate obfuscated transport + routing carve-out + own watchdog) was rejected as ~3× the surface, against the project's "no over-debugged roles" lesson; the lockout risk it would cover (self-inflicted bad config) is instead handled by the rollback-guard below.

**Router rollback-guard (`/usr/bin/singbox-guard`)**: deploying a config that crash-loops sing-box is the one failure the watchdog can't fix (and it takes the management tunnel down with it). The guard is the single (re)start entry point — used by the `restart sing-box` handler **and** the watchdog: it `sing-box check`s the config, restarts, and if the process doesn't come up it restores `config.json.lkg` (last-known-good) and restarts; every successful start re-snapshots LKG. Deploys also stage to `config.json.new` and validate it **before** overwriting the live config, so a known-bad config never disrupts a working tunnel; the play fails fast instead. "Up" = process alive (fail-open semantics: sing-box alive ⇒ RU/zapret work, only geo-blocked category fails, no leak).

**tgproxy gost bridge**: nginx cannot `proxy_pass` via SOCKS5 natively. `gost` (single static binary) listens on `127.0.0.1:9443` and dials `api.telegram.org:443` through sing-box SOCKS `127.0.0.1:1081`. nginx uses `proxy_ssl_server_name on` + `proxy_ssl_name api.telegram.org` so that TLS SNI is correct even though the TCP connection goes to localhost. mtg domain-fronting forwards all non-Telegram :443 traffic to `127.0.0.1:8443` as raw TCP; nginx does SNI-based virtual hosting there, so no firewall changes are needed.

## Variables

All secrets and host-specific config live in `group_vars/all.yml` (gitignored). Key vars:
- `singbox_server_password` — SS2022 secret for edge→gateway chain (base64, 32 bytes)
- `router_ss_secret` — separate SS2022 secret for the legacy router→edge SS2022 `router-in` :8388 (do not reuse the chain secret)
- `router_primary_transport` — intended active home→edge transport on the router. Edge always listens on both inbounds; **for now `owrt_singbox` only implements `hysteria2`** and asserts this var equals `hysteria2` (shadowtls outbound on the router is a planned 2nd pass). Keep it `hysteria2` until then
- `hy2_port` / `hy2_password` / `hy2_obfs_password` — hysteria2 inbound (edge) / outbound (router). UDP. TLS uses the edge LE cert (`/etc/letsencrypt/live/{{ domain }}/`); obfs salamander. No `up_mbps`/`down_mbps` (Brutal off, by design)
- `shadowtls_port` / `shadowtls_password` / `shadowtls_handshake_server` — shadowtls v3 (TCP fallback). Already wired on edge (`shadowtls-in`); the router outbound lands in the 2nd pass. `shadowtls_handshake_server` is the masquerade TLS1.3 site, shared by edge `handshake.server` and (future) router `tls.server_name`
- `shadowtls_ss_method` / `shadowtls_ss_secret` — internal SS2022 under shadowtls (NEW key — not `router_ss_secret`, not the :4433 chain secret)
- `vpn_users` — list of `{name, password}` for both VPN and proxy
- `ocserv_domain` / `ocserv_acme_email` — gateway domain (must resolve to gateway IP before first run)
- `tgproxy_domain` — subdomain for Telegram API reverse proxy (default: `tapi.home12.ru`; must resolve to edge IP before first run)
- `gost_version` — gost v2 binary version (default in role: `2.11.5`)
- `ru_dns_dot_ip` / `ru_dns_dot_sni` — router RU DNS-over-TLS upstream (defaults in `owrt_singbox/defaults`: `77.88.8.8` / `common.dot.dns.yandex.net`)
- `owrt_disable_hw_offload` — disable MediaTek HW flow offload on the router so NFQUEUE/nfqws can see packets (default `true` in `owrt_nfqws/defaults`; also shown in `group_vars/all.yml.example`)
- `dpi_bypass_domains` — opt-in list of RKN-blocked domains routed direct WAN + nfqws DPI-desync instead of the tunnel (default `[]` → all blocked sites tunnel to gateway). Do not put Meta domains here with `dns-local` resolution (see caveat above). `nfqws_youtube_domains` must be a subset; `nfqws_routing_mark`/`nfqws_queue_num`/`nfqws_fwmark`/`nfqws_args` tune the desync
- `router_singbox_watchdog` — enable the cron watchdog that restarts router `sing-box` on crash (default in role: `true`)

## Post-deploy verification

Gateway:
```bash
systemctl status ocserv unbound sing-box --no-pager
ss -lntup | egrep '(:443|:4433|:53)'
nft list ruleset
```

Edge:
```bash
systemctl status nginx mtg sing-box gost-tgproxy --no-pager
ss -lntup | egrep '(:443|:2080|:2081|:9443)'
curl --proxy http://USER:PASSWORD@EDGE_IP:2080 https://ifconfig.me
curl https://tapi.home12.ru/bot<TOKEN>/getMe
```

Router:
```bash
service sing-box status && ip a show singtun0
sing-box check -c /etc/sing-box/config.json   # render is from owrt_singbox (hysteria2 proxy outbound)
nslookup instagram.com 127.0.0.1          # default (dpi_bypass_domains=[]): REAL Meta IP, resolved via dns-remote (DoH over tunnel) at gateway
                                          # if on nfqws-direct via dns-local: would be NXDOMAIN (Yandex censors Meta) — see caveat
crontab -l | grep singbox-watchdog         # watchdog installed
uci show firewall | grep mtu_fix           # MSS clamp on proxy zone
ls -l /etc/sing-box/config.json.lkg        # rollback-guard last-known-good snapshot exists
/etc/init.d/mgmt-tunnel status             # reverse-SSH tunnel running (procd)
```

Gateway — reach the router through the reverse tunnel:
```bash
ss -lnt | grep 127.0.0.1:2222              # rtun reverse port bound (loopback only)
ssh -p 2222 root@127.0.0.1                 # lands on the router
```

Edge — confirm the home→edge leg inbounds are listening:
```bash
ss -lnup | grep <hy2_port>                 # hysteria2 (UDP) — primary
ss -lntp | grep <shadowtls_port>           # shadowtls (TCP) — fallback (edge always on; router outbound is a 2nd pass)
```

## Known gaps (from docs/architecture.md)

- Secrets are plaintext in `group_vars/all.yml` — no Ansible Vault or SOPS yet
- No healthchecks (ocserv login test, DNS-through-VPN test, proxy egress IP) — only the router `sing-box` has a process watchdog
- IPv6 is disabled globally (`common_disable_ipv6: true`), not routed
- No rate limiting on public proxy endpoints
- No fail2ban/dynamic nftables sets for SSH/ocserv brute-force
