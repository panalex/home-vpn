# home-vpn-ansible

Ansible-проект для двух VPS и домашнего роутера GL-MT6000 (Flint 2). Edge VPS — публичный ingress/relay, Gateway VPS — egress/VPN-концентратор, роутер — прозрачный proxy для всей домашней сети.

Проект разворачивает:

- браузерный HTTP/SOCKS-прокси на edge-узле с аутентификацией для FoxyProxy и похожих клиентов;
- цепочку `sing-box` edge → gateway;
- OpenConnect/AnyConnect VPN на gateway-узле через `ocserv`;
- локальный DNS для VPN-клиентов через `unbound` с DoT-forwarding;
- nftables firewall/NAT для обоих VPS-узлов;
- единый список пользователей для прокси и VPN;
- WDTT-сервер на gateway: WireGuard через VK TURN / DTLS (:56000), позволяет пробить NAT без проброса портов;
- sing-box TUN на роутере GL-MT6000 (OpenWrt, роли `owrt_singbox` + `owrt_nfqws`): трафик домашней сети прозрачно уходит в gateway через edge — первый хоп роутер→edge идёт по **hysteria2** (QUIC/UDP + obfs salamander; «голый» SS2022 leg резался ТСПУ), дальше нога edge→gateway идёт **одним TCP-транспортом** — shadowtls v3 (по умолчанию; голый legacy SS2022 :4433 — ручной fallback через `de_uplink_transport: ss`); трансграничный RU→DE QUIC режется сильнее, поэтому hysteria2 на этом плече снят; российские IP/домены идут напрямую;
- `nfqws` (zapret) на роутере: опциональная DPI-десинхронизация для отдельных РКН-заблокированных сайтов с прямым выходом через WAN без туннеля (домены задаются в `dpi_bypass_domains`; по умолчанию список пуст — все заблокированные сайты идут через туннель).

Репозиторий подготовлен для публичной публикации: реальные IP, домены, пароли, `.git`-история и локальные vars исключены. Перед использованием создайте собственные `inventory.ini` и `group_vars/all.yml` из `.example`-файлов.

## Архитектура

```mermaid
flowchart LR
    U1[VPN clients\nUbuntu / Android / Windows]
    U2[Browser clients\nFoxyProxy]
    U4[Home LAN\nall devices]

    FL[GL-MT6000 Flint 2\nOpenWrt + sing-box TUN\nowrt_singbox / singtun0]
    RU[Edge VPS\nnginx + sing-box\nHTTP :2080 / SOCKS :2081]
    DE[Gateway VPS\nocserv + unbound + sing-box\nVPN :443 / shadowtls :8943 + ss-in :4433]
    NET[Internet]
    DNS[Cloudflare / Quad9 DoT]

    U4 -->|all traffic via TUN| FL
    FL -->|RU IP/domains| NET
    FL -.->|dpi_bypass_domains: direct + nfqws — опц., по умолч. пусто| NET
    FL ==>|non-RU: hysteria2 obfs UDP :39443 — active| RU
    FL -.->|fallback: shadowtls v3 TCP :8843 — edge готов, роутер 2-й проход| RU
    FL -.->|legacy SS2022 :8388 — режется ТСПУ| RU
    U2 -->|HTTP proxy auth| RU
    RU ==>|edge→DE: shadowtls v3 TCP :8943 — active| DE
    RU -.->|manual fallback: legacy SS2022 :4433| DE
    U1 -->|OpenConnect :443 tcp/udp| DE
    U5[WireGuard clients] -->|DTLS :56000 via VK TURN| DE
    DE --> NET
    DE -->|DoT :853| DNS
```

Подробная техническая карта: [`docs/architecture.md`](docs/architecture.md).

## Требования

- Control node: Ansible Core 2.15+.
- Managed hosts (VPS): Ubuntu 24.04.
- Managed host (роутер): GL-MT6000 с OpenWrt и пакетом `sing-box`.
- На gateway-узле домен для `ocserv` должен указывать на публичный IP gateway-узла.
- На edge-узле домен должен указывать на публичный IP edge-узла.
- DNS A-записи должны быть готовы до первого запуска Let's Encrypt.

Установить коллекции:

```bash
ansible-galaxy collection install -r requirements.yml
```

Включает `community.openwrt` — требуется для управления роутером.

## Быстрый старт

```bash
cp inventory.ini.example inventory.ini
cp group_vars/all.yml.example group_vars/all.yml
```

Отредактируйте:

- `inventory.ini` — реальные адреса, пользователи SSH, домен edge-узла и email;
- `group_vars/all.yml` — домен gateway-узла, секрет `sing-box`, пользователи, пароли, порты.

Сгенерировать секреты sing-box — каждый отдельной командой `openssl rand -base64 32`:

```bash
openssl rand -base64 32   # singbox_server_password — цепочка edge→gateway (SS2022 :4433)
openssl rand -base64 32   # router_ss_secret        — legacy SS2022 router-in :8388
openssl rand -base64 32   # hy2_password            — hysteria2 leg роутер→edge
openssl rand -base64 32   # hy2_obfs_password       — obfs salamander
openssl rand -base64 32   # shadowtls_password      — shadowtls v3 (edge готов, роутер 2-й проход)
openssl rand -base64 32   # shadowtls_ss_secret     — внутр. SS2022 под shadowtls
openssl rand -base64 32   # de_shadowtls_password   — shadowtls v3 leg edge→DE (active)
openssl rand -base64 32   # de_shadowtls_ss_secret  — внутр. SS2022 под shadowtls (edge→DE)
```

Сгенерировать пароли пользователей:

```bash
LC_ALL=C tr -dc 'A-Za-z0-9_@%+=:.,-' </dev/urandom | head -c 32; echo
```

Запуск VPS-инфраструктуры:

```bash
ansible-playbook -i inventory.ini site.yml
```

Запуск только пользовательской синхронизации:

```bash
ansible-playbook -i inventory.ini site.yml --tags users,ru_singbox
```

Запуск настройки роутера:

```bash
ansible-playbook -i inventory.ini router.yml
```

## Роли

| Роль | Узел | Назначение |
|---|---:|---|
| `common` | VPS (оба) | базовые пакеты, sysctl, nftables, BBR |
| `ru_fronting` | edge | устанавливает nginx и снимает дефолтный сайт, чтобы `ru_letsencrypt` мог выпустить/обновить LE-серт edge-домена (нужен home→edge inbound `hy2-in`) |
| `ru_letsencrypt` | edge | выпуск LE-сертификата для edge-домена |
| `ru_singbox` | edge | локальный SOCKS/HTTP, публичный auth proxy, outbound на gateway одним транспортом `de_uplink_transport` (shadowtls по умолчанию, ручной fallback — legacy ss); inbound'ы leg home→edge: hysteria2 (obfs salamander, UDP) + shadowtls v3 (TCP) + legacy SS2022 router-in |
| `de_singbox` | gateway | inbound'ы ноги edge→DE: shadowtls-in (TCP, active) + legacy ss-in `:4433` (manual fallback), оба egress'ят direct |
| `de_unbound` | gateway | локальный DNS для VPN-клиентов, DoT upstream |
| `de_ocserv` | gateway | OpenConnect/AnyConnect VPN, LE-сертификат, VPN route hook |
| `de_wdtt` | gateway | WireGuard через VK TURN; собирает `wdtt-server` из исходников Go, DTLS `:56000`, WG `:56001`, NAT `10.66.66.0/24` |
| `users` | gateway | синхронизация `/etc/ocserv/ocpasswd` из `vpn_users` |
| `firewall` | VPS (оба) | nftables input/forward rules |
| `owrt_singbox` | GL-MT6000 | sing-box TUN (`singtun0`), split-DNS (RU/блок-домены → Яндекс **DoT**, иностранные → DoH через туннель), split-routing по `geoip-ru`/`geosite-ru`, outbound `proxy` на edge — **hysteria2** (1-й проход; shadowtls-fallback на роутере 2-м проходом), cron-watchdog + `mtu_fix`. Переписан с нуля вместо снятого `flint_singbox` |
| `owrt_nfqws` | GL-MT6000 | Опциональная DPI-десинхронизация (zapret/nfqws); NFQUEUE перехватывает TCP 80/443 и блокирует QUIC для трафика с `routing_mark={{ nfqws_routing_mark }}`. Простаивает, если `dpi_bypass_domains` пуст. Переписан с нуля вместо снятого `flint_nfqws` |

## Важные runtime-особенности

`ocserv` создает интерфейс `vpns0` только после подключения первого клиента. Поэтому маршрут `10.99.0.0/24 dev vpns0` нельзя безусловно добавлять во время выполнения Ansible. В проекте это решено через `connect-script`, который выполняется после появления VPN-интерфейса.

nftables-правила с `iifname "vpns0"` можно загружать заранее: существование интерфейса для таких правил не требуется.

## Проверки после деплоя

Gateway:

```bash
systemctl status ocserv unbound sing-box --no-pager
ss -lntup | egrep '(:443|:4433|:53)'
ss -lntp  | grep 8943         # edge→DE shadowtls inbound (TCP) — active
nft list ruleset
```

После подключения VPN-клиента:

```bash
ip a show vpns0
ip route | grep '10.99.0.0/24'
dig @10.99.0.1 example.com
```

Edge:

```bash
systemctl status nginx sing-box --no-pager
ss -lntup | egrep '(:2080|:2081)'
ss -lntp  | grep 8843         # home→edge shadowtls inbound (TCP) — fallback
ss -lnup  | grep 39443        # home→edge hysteria2 inbound (UDP) — primary
```

Прокси через edge:

```bash
curl --proxy http://USER:PASSWORD@EDGE_IP:2080 https://ifconfig.me
```

Роутер GL-MT6000:

```bash
service sing-box status
ip a show singtun0
# трафик с домашнего устройства должен выходить через gateway IP:
curl https://ifconfig.me   # из браузера на LAN
```

WDTT (gateway):

```bash
systemctl status wdtt --no-pager
ss -lntup | grep ':56000'
# подключение WireGuard-клиента через wdtt-клиент (proxy-turn-vk-android)
```
