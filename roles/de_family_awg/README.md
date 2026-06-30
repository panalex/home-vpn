# de_family_awg

Isolated **AmneziaWG 2.0** endpoint on the DE (Fornex) node for a family
member's Keenetic Racer router (Yoshkar-Ola, Rostelecom — aggressive DPI).

It is **deliberately independent** from the sing-box stack: own UDP port, own
subnet (`10.42.0.0/24`), own DNS resolver (a second `unbound` instance), own
systemd units, own keys, and its own nftables tables/rules. Different audience,
different blast radius, different SLA — if the mother's peer key leaks,
revocation never touches the main stack.

## What it does

- Installs AmneziaWG from the Amnezia PPA **`ppa:amnezia/ppa`** on Ubuntu 24.04:
  `amneziawg` (userspace tools — `awg`/`awg-quick`) + `amneziawg-dkms` (the
  **DKMS kernel module**). Loads the `amneziawg` module, persists autoload.
  (The PPA has no separate `amneziawg-tools` package — the tools live in
  `amneziawg`.)
- Generates a server keypair, a random UDP listen port (30000–50000), and a
  **shared ASC parameter set** — once — into `secrets/awg_family.yml` (control
  node, gitignored). Reused on every later run (idempotent).
- Generates per-peer keypair + PSK into `secrets/awg_family_peers/<name>.yml`
  (one file per peer, gitignored), renders client `.conf` into
  `client_configs/awg-family/<name>.conf`.
- Runs a dedicated `unbound-family` instance bound to `10.42.0.1:53` **only**,
  forwarding all queries over DoT (Cloudflare primary, Quad9 secondary).
- Adds firewall rules via a single `/etc/nftables.d/awg-family.conf` drop-in
  and NATs `10.42.0.0/24` out the egress interface.

## Deploy

Opt-in — not wired into `site.yml`. Set `awg_family_enabled: true` in
`group_vars/all.yml`, then:

```bash
ansible-playbook -i inventory.ini family.yml
```

## Add a peer

1. Append to `awg_family_peers` in `group_vars/all.yml` (or role defaults):
   ```yaml
   awg_family_peers:
     - { name: "mama-keenetic-yola", ip: "10.42.0.10" }
     - { name: "papa-phone",         ip: "10.42.0.11" }   # new
   ```
   Give each a unique `name` and a free `10.42.0.x` (peers start at `.10`).
2. Re-run `ansible-playbook -i inventory.ini family.yml`. Only the **new**
   peer's keys are generated (existing peer files are stat-guarded); the server
   config gains a `[Peer]` block and reloads.
3. The client config lands at `client_configs/awg-family/<name>.conf` — import
   it on the Keenetic / AmneziaWG client. ASC params in it must match the
   server (they do — shared set), so never hand-edit them.

## Revoke a peer

Delete its line from `awg_family_peers`, delete
`secrets/awg_family_peers/<name>.yml`, re-run. The `[Peer]` block disappears and
the key can never be re-derived. (Manual for now — no tooling, by design.)

## Shared vs per-peer ASC (important)

AmneziaWG carries one ASC set (`Jc/Jmin/Jmax/S1–S4/H1–H4/I1`) **per interface**.
The amneziawg kernel module + tools do **not** support distinct ASC params per
peer on a single interface — every peer on `awg-family` must present the same
ASC set the server advertises. So ASC here is **shared server-wide**, generated
once and mirrored into every peer `.conf`. (Per-peer ASC would require one
interface per peer — out of scope.) Unique-per-peer material is the keypair,
PSK, and tunnel IP.

`H1..H4` are validated to be mutually distinct at generation time; the play
fails loudly on a collision.

## DPI hardening rationale

- **AmneziaWG over plain WireGuard:** WG's handshake has a fixed, fingerprintable
  layout (message-type bytes, packet sizes). AWG's ASC params randomise junk
  packets (`Jc/Jmin/Jmax`), pad the handshake (`S1–S4`), rewrite the four
  message-type magics (`H1–H4`), and inject a custom signature packet (`I1`) so
  the session doesn't match the WG signature Rostelecom's DPI looks for.
- **DoT for DNS:** the resolver forwards only over TLS:853 (Cloudflare/Quad9),
  so the ISP can't see or spoof plaintext DNS for the family's traffic, and
  unbound is reachable from the tunnel subnet only (never the public IP).

## Recovery: kernel module fails to load

If `amneziawg-dkms` fails to build/load (kernel mismatch, missing headers), the
play stops at the "module failed to load" assertion. Fall back to the userspace
implementation. `amneziawg-go` is **not** in `ppa:amnezia/ppa`, so build it from
source (needs Go):

```bash
# On the DE host:
git clone https://github.com/amnezia-vpn/amneziawg-go /tmp/awg-go
cd /tmp/awg-go && make && install -m0755 amneziawg-go /usr/bin/amneziawg-go
# Point awg-quick at the userspace backend for this interface:
export WG_QUICK_USERSPACE_IMPLEMENTATION=amneziawg-go
systemctl restart awg-quick@awg-family
```

To make it persistent, drop
`Environment=WG_QUICK_USERSPACE_IMPLEMENTATION=amneziawg-go` into a systemd
override for `awg-quick@awg-family.service`. The wire protocol and configs are
identical, so no peer reissue is needed.

## Files / where things land

| Path | What |
|------|------|
| `secrets/awg_family.yml` | server key, port, shared ASC (gitignored) |
| `secrets/awg_family_peers/<name>.yml` | per-peer key + PSK (gitignored) |
| `client_configs/awg-family/<name>.conf` | client config to hand out (gitignored) |
| `/etc/amnezia/amneziawg/awg-family.conf` | server config (DE host) |
| `/etc/unbound/unbound-family.conf` | dedicated resolver config (DE host) |
| `/etc/systemd/system/unbound-family.service` | resolver unit |
| `/etc/nftables.d/awg-family.conf` | firewall + NAT drop-in |
