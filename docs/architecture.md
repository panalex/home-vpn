# Архитектура проекта

## Назначение

Проект автоматизирует разворачивание домашней двухузловой VPN/proxy-инфраструктуры на Ubuntu 24.04.

Целевой результат:

- полнотуннельный OpenConnect/AnyConnect VPN для ноутбуков и мобильных клиентов;
- браузерный HTTP/SOCKS proxy для выборочной маршрутизации через FoxyProxy;
- egress через отдельный gateway-узел;
- воспроизводимый Ansible-deploy после reboot или переустановки сервера.

## Узлы

| Логическое имя | Inventory group | Назначение | Основные сервисы |
|---|---|---|---|
| Edge VPS | `ru_vps` | публичная точка входа для браузерного прокси и home→edge туннеля | nginx, sing-box |
| Gateway VPS | `de_vps` | VPN concentrator, WDTT relay и внешний egress | ocserv, unbound, sing-box, wdtt |
| Роутер | `routers` | прозрачный proxy для домашней сети + DPI-обход | sing-box TUN, nfqws |

Названия групп исторические. В публичной версии их можно трактовать как `edge_vps` и `gateway_vps`.

## Потоки трафика

### 1. VPN-клиент -> Gateway -> Internet

```mermaid
sequenceDiagram
    participant C as VPN client
    participant O as ocserv on gateway
    participant U as unbound on gateway
    participant I as Internet

    C->>O: OpenConnect TLS/DTLS :443
    O->>O: creates vpns0 lazily
    O->>O: connect-script adds route 10.99.0.0/24 dev vpns0
    C->>U: DNS to 10.99.0.1:53
    U->>U: DoT forward to upstream resolvers
    C->>I: full tunnel via gateway NAT
```

### 2. Browser -> Edge proxy -> Gateway -> Internet

```mermaid
sequenceDiagram
    participant B as Browser/FoxyProxy
    participant E as sing-box on edge
    participant G as sing-box on gateway
    participant I as Internet

    B->>E: HTTP proxy :2080 or SOCKS :2081 with auth
    E->>G: edge→DE single transport: shadowtls :8943 (active) / legacy SS2022 :4433 (manual fallback)
    G->>I: outbound direct
```

### 4. WireGuard client -> WDTT -> Gateway -> Internet

```mermaid
sequenceDiagram
    participant W as WireGuard client
    participant T as VK TURN relay
    participant D as wdtt-server on gateway :56000
    participant I as Internet

    W->>T: DTLS to VK TURN servers
    T->>D: relayed DTLS datagram
    D->>D: unwrap → WireGuard :56001
    W->>I: full tunnel via gateway NAT (10.66.66.0/24)
```

Преимущество: DTLS через публичные TURN-серверы VK пробивает большинство NAT без проброса портов на клиентской стороне.

### 5. Роутер: DPI-обход через nfqws (опционально)

> По умолчанию `dpi_bypass_domains` пуст — все РКН-заблокированные сайты идут **через туннель** до gateway (категория `final: proxy`), мимо nfqws и ТСПУ. Схема ниже работает только для доменов, явно добавленных в `dpi_bypass_domains`.

```mermaid
sequenceDiagram
    participant H as Home device
    participant FL as GL-MT6000
    participant I as Internet (RKN-blocked site)

    H->>FL: TCP/UDP to blocked domain
    FL->>FL: sing-box marks packet (routing_mark=100) → dpi-bypass-direct outbound
    FL->>FL: nftables NFQUEUE → nfqws desync (disorder2)
    FL->>I: direct WAN (no VPN tunnel)
```

nfqws перехватывает пакеты с `routing_mark={{ nfqws_routing_mark }}` через NFQUEUE, блокирует QUIC (UDP/443) и десинхронизирует TCP-хэндшейки (disorder2, split). Список доменов задаётся в `dpi_bypass_domains` (по умолчанию пуст). Шаблон `owrt_singbox/config.json.j2` оборачивает правила dpi-bypass в `{% if dpi_bypass_domains %}`, поэтому пустой список даёт валидный конфиг без этой категории.

> **Домены Meta нельзя класть в `dpi_bypass_domains` с резолвом через Яндекс.** См. предупреждение в split-DNS ниже.

## Порты

### Edge

| Порт | Протокол | Bind | Сервис | Назначение |
|---:|---|---|---|---|
| 22 | TCP | public | sshd | управление |
| 80 | TCP | public | nginx/certbot | ACME HTTP-01 |
| 8388 | TCP/UDP | public | sing-box `router-in` | legacy SS2022 leg home→edge (режется ТСПУ) |
| 39443 | UDP | public | sing-box `hy2-in` | leg home→edge **primary**: hysteria2 + obfs salamander (LE-серт edge) |
| 8843 | TCP | public | sing-box `shadowtls-in` | leg home→edge **fallback**: shadowtls v3 → внутр. SS2022 |
| 1080 | TCP | loopback | sing-box | локальный HTTP proxy |
| 1081 | TCP | loopback | sing-box | локальный SOCKS proxy |
| 2080 | TCP | public | sing-box | HTTP proxy с auth |
| 2081 | TCP | public | sing-box | SOCKS proxy с auth |

### Gateway

| Порт | Протокол | Bind | Сервис | Назначение |
|---:|---|---|---|---|
| 22 | TCP | public | sshd | управление |
| 80 | TCP | public | certbot standalone | ACME HTTP-01 для ocserv |
| 443 | TCP/UDP | public | ocserv | OpenConnect/AnyConnect VPN |
| 8943 | TCP | public, restricted by firewall (edge IP only) | sing-box `shadowtls-in` | нога edge→DE **active**: shadowtls v3 → внутр. SS2022 |
| 4433 | TCP | public, restricted by firewall | sing-box `ss-in` | нога edge→DE **manual fallback**: legacy SS2022 ingress с edge-узла |
| 53 | TCP/UDP | VPN interface | unbound | DNS для VPN-клиентов |
| 56000 | UDP | public | wdtt-server | WireGuard over DTLS/VK TURN |
| 56001 | UDP | loopback | wdtt-server | внутренний WireGuard listener |

## Адресация VPN

По умолчанию (ocserv):

- VPN network: `10.99.0.0/24`
- gateway/DNS: `10.99.0.1`
- ocserv device base: `vpns`
- runtime interface: `vpns0`

WDTT WireGuard:

- subnet: `10.66.66.0/24`
- пользователи и ключи управляются через `passwords.json` в `{{ wdtt_config_dir }}`; доступны Telegram-команды при наличии бота

Особенность: `vpns0` появляется только после подключения первого клиента. Поэтому постоянный Ansible task вида `ip route replace 10.99.0.0/24 dev vpns0` неидемпотентен на холодном сервере. В проекте маршрут добавляется best-effort hook-скриптом `route-up.sh`, подключенным через `connect-script` в `ocserv.conf`.

## DNS

`unbound` слушает локально на gateway-узле и обслуживает VPN-клиентов. Upstream DNS идет по DoT на публичные резолверы.

Для VPN-клиентов `ocserv` пушит DNS `10.99.0.1` и `tunnel-all-dns = true`.

Важный фикс для Ubuntu/Debian: проект удаляет конфликтующий distro-файл `root-auto-trust-anchor-file.conf` и использует `trust-anchor-file: "/usr/share/dns/root.key"`, чтобы избежать конфликта `auto-trust-anchor-file`.

## Firewall/NAT

Firewall реализован через nftables.

Основные требования:

- разрешить входящие публичные сервисы только на нужных портах;
- разрешить `input` DNS с `vpns0` на `:53`;
- разрешить `forward` из `vpns0` в egress-интерфейс;
- разрешить `forward` established/related обратно в `vpns0`;
- masquerade для `10.99.0.0/24` на gateway egress-интерфейс;
- ограничить sing-box inbound на gateway только edge-IP;
- не допустить open proxy без auth.

## Ansible ownership

| Компонент | Где задается |
|---|---|
| Список пользователей VPN/proxy | `group_vars/all.yml: vpn_users` |
| ocserv config | `roles/de_ocserv/templates/ocserv.conf.j2` |
| ocserv route hook | `roles/de_ocserv/templates/route-up.sh.j2` |
| unbound config | `roles/de_unbound/templates/unbound.conf.j2` |
| nftables ruleset | `roles/firewall/templates/nftables.conf.j2` |
| ocserv NAT | `roles/de_ocserv/templates/ocserv-nat.nft.j2` |
| sing-box edge/gateway | `roles/ru_singbox`, `roles/de_singbox` |
| wdtt service + NAT | `roles/de_wdtt/templates/wdtt.service.j2`, `wdtt-nat.nft.j2` |
| wdtt пользователи | `roles/de_wdtt/templates/passwords.json.j2` |
| nfqws binary + init | `roles/owrt_nfqws/tasks/main.yml` (качает zapret с GitHub) |
| nfqws nftables rules | `roles/owrt_nfqws/templates/dpi-bypass.sh.j2` |
| router sing-box config (split-DNS/routing) | `roles/owrt_singbox/templates/config.json.j2` |
| router DNS DoT + watchdog vars | `roles/owrt_singbox/defaults/main.yml` |

## Что важно показать на интерактивной карте сети

Слои карты:

1. **Nodes** — edge VPS, gateway VPS, clients, upstream DNS, Internet.
2. **Public ingress** — SSH, ACME, browser proxy, home→edge туннель, ocserv.
3. **Encrypted tunnels** — browser/home chain edge -> gateway, VPN client -> gateway.
4. **DNS plane** — VPN client -> unbound -> DoT upstream.
5. **Firewall plane** — input, forward, NAT, edge-IP restriction.
6. **Ansible ownership** — какая роль владеет каким сервисом и файлом.
7. **Runtime state** — `vpns0` появляется только при подключенном клиенте.

## Маршрутизация на роутере: три категории

| Категория | Правило | Выход |
|---|---|---|
| RU IP / geosite-ru | sing-box `geoip:ru`, `geosite:ru` | прямой WAN (без туннеля) |
| `dpi_bypass_domains` (опц., по умолч. пусто) | sing-box → `dpi-bypass-direct` outbound с `routing_mark` | прямой WAN + nfqws NFQUEUE десинхронизация |
| всё остальное (вкл. заблокированные сайты по умолчанию) | sing-box → outbound `proxy` | туннель до gateway VPS (через edge) |

`dpi_bypass_domains` и `nfqws_routing_mark` задаются в `group_vars/all.yml`. По умолчанию `dpi_bypass_domains: []` — РКН-заблокированные сайты (youtube, instagram, x.com…) идут через туннель. nfqws-direct (средняя строка) — opt-in для отдельных доменов: экономит bandwidth туннеля и сохраняет RU-CDN-локацию, но требует рабочей стратегии десинхронизации под конкретный ТСПУ.

### Транспорт leg home→edge

Категория `proxy` (туннель) физически идёт роутер → **edge** одним из транспортов, и уже на edge все они маршрутизируются в активный outbound плеча edge → gateway (`de_uplink_transport`, см. ниже). **Edge всегда слушает оба** новых inbound (`hy2-in` + `shadowtls-in`). Роли роутера — **`owrt_singbox`** (sing-box TUN) + **`owrt_nfqws`** (DPI-десинк); прежние `flint_*` сняты как нестабильные и переписаны с нуля как `owrt_*`.

> **Ручной переключатель `router_primary_transport`** (по образцу `de_uplink_transport` на edge). `owrt_singbox` рендерит соответствующий outbound с тегом `proxy`: `hysteria2` (primary) или пару `shadowtls` v3 → внутренний SS2022 (`proxy-stls` + `proxy`, TCP-fallback); `assert` допускает только эти два значения. Edge всегда слушает оба inbound, поэтому переключение = смена переменной + `ansible-playbook router.yml` (edge не трогаем). Роутерный shadowtls-outbound идёт **без multiplex** — симметрично edge-инбаунду `shadowtls-ss-in` (у него mux нет); mux на одном конце = blackhole обратного канала.

| Транспорт первого хопа (роутер → edge) | Порт на edge | Маскировка | Статус на роутере | Назначение |
|---|---|---|---|---|
| hysteria2 поверх QUIC/UDP | `hy2_port` 39443/udp | obfs salamander + TLS на LE-серте edge | ✅ активен (`owrt_singbox`) | **primary** — TCP-ориентированный ТСПУ не разбирает QUIC-поток |
| shadowtls v3 → внутренний SS2022 (detour) | `shadowtls_port` 8843/tcp | TLS1.3-хендшейк к `shadowtls_handshake_server` | ✅ реализован (`router_primary_transport: shadowtls`) | **fallback** — если провайдер режет UDP/QUIC |
| «голый» SS2022 `router-in` | `router_ss_port` 8388/tcp | нет | ⚪ legacy, не используется | режется ТСПУ по поведению/JA3 (первый шифроблок дропается) |

Brutal **выключен сознательно**: `up_mbps`/`down_mbps` не задаются ни на edge, ни на роутере — на стабильном канале фиксированная полоса даёт аномально ровный паттерн, отдельный признак для поведенческого детекта. На edge inbound'ы `router-in`, `hy2-in`, `shadowtls-ss-in` сведены в одно route-правило → активный outbound плеча edge→DE. hysteria2 TLS использует тот же LE-серт edge, что nginx; certbot deploy-hook рестартит sing-box при обновлении серта. Требуется sing-box ≥ 1.12 на роутере (apk) — hysteria2 и shadowtls v3 поддержаны (проект на 1.13.x).

### Транспорт leg edge→DE (один транспорт)

Edge → DE — **единственное звено, реально пересекающее границу**. Какое-то время оно было укреплено симметрично роутер→edge — urltest-группой `de-out` из трёх кандидатов (hysteria2 + shadowtls + legacy ss). Это **схлопнуто до одного TCP-транспорта** по двум причинам:

1. **QUIC здесь нестабилен.** Трансграничный RU→DE QUIC режется сильнее, чем TCP; hysteria2 на этом плече давал переменную надёжность, а от единственной реальной угрозы плеча (блок провайдера Fornex по IP/ASN) UDP всё равно не страхует — TCP предсказуемее. hysteria2-outbound `de-hy2` и inbound `hy2-in` на DE сняты.
2. **urltest давал DNS-петлю.** Health-check `de-out` ходил по `https://www.gstatic.com/generate_204`, для чего нужен резолв через `remote-dns`, а `remote-dns` детурил в саму группу `de-out` → циклическая зависимость. Это и была причина **периодических обрывов**.

Активный транспорт выбирается переменной `de_uplink_transport` (по образцу `router_primary_transport` на роутере). DE держит оба inbound постоянно, поэтому переключение = смена переменной + редеплой только edge.

| Транспорт edge→DE | Порт на DE | Маскировка | `de_uplink_transport` |
|---|---|---|---|
| shadowtls v3 → внутр. SS2022 (`de-shadowtls-ss`→`de-shadowtls-tls`) | `de_shadowtls_port` 8943/tcp | TLS1.3-хендшейк к `de_shadowtls_handshake_server` | `shadowtls` (**default**, active) |
| «голый» SS2022 (`de-ss`) | `singbox_server_port` 4433/tcp | нет | `ss` (manual fallback) |

`route.final`, правило inbound-группы (`router-in`/`hy2-in`/`shadowtls-ss-in`) и `detour` у `remote-dns` в `ru_singbox` указывают на **единственный активный тег** (`de-shadowtls-ss` или `de-ss`) — петля устранена. Правила `direct` (DE/32 → direct, `ssh_port` → direct) **не тронуты** — они нужны, чтобы сам транспорт до DE шёл напрямую, без петли. DE-firewall открывает 8943/tcp **только с IP edge** (рядом с легаси-правилом для :4433). shadowtls/ss-in на DE сертов не требуют, поэтому предусловие LE-серта для `de_singbox` снято; `de_ocserv` по-прежнему владеет сертом `ocserv_domain` (для самого ocserv) и идёт до `de_singbox` в `site.yml` без жёсткой зависимости.

### Split-DNS на роутере

| Категория доменов | Резолвер | Транспорт |
|---|---|---|
| `dpi_bypass_domains` + `geosite-ru` + `*.ru` | `dns-local` — Яндекс DoT (`77.88.8.8`, `detour: direct`) | DoT, RU-локация |
| всё остальное (`final`) | `dns-remote` — Cloudflare DoH (`1.1.1.1`, `detour: proxy`) | DoH через туннель DE, реальные IP с gateway |

FakeIP **не используется**: dnsmasq на роутере стоит перед sing-box на :53 и режет зарезервированный диапазон `198.18.0.0/15` своей rebind-защитой, отдавая клиенту `NXDOMAIN`. Поэтому иностранные домены резолвятся в реальные IP через DoH-туннель (`dns-remote`, `detour: proxy`) — резолв происходит со стороны gateway (DE, без РКН), а split-routing по домену продолжает работать за счёт `sniff` (SNI). RU- и заблокированные домены резолвятся через **DoT** (а не plain UDP:53): РКН/ТСПУ подделывают открытые DNS-ответы для заблокированных доменов на пути, и отравленный IP отправил бы desync-пакеты nfqws в чёрную дыру. DoT по IP спуф-устойчив и не требует bootstrap-резолва; RU-локация даёт ближайший достижимый CDN-узел. `ru_dns_dot_ip` / `ru_dns_dot_sni` — в `owrt_singbox/defaults`.

> ⚠️ **Яндекс DoT цензурит домены Meta.** `dns-local` запрашивается с RU-IP роутера (`detour: direct`), а Яндекс по требованию РКН отдаёт **NXDOMAIN** на `instagram.com` / `facebook.com` / `cdninstagram.com` / `fbcdn.net` из РФ (симптом в браузере — `DNS_PROBE_FINISHED_NXDOMAIN`). DoT защищает от подмены *на пути*, но не от политики самого резолвера. Поэтому Meta-домены нельзя резолвить через `dns-local`: либо они идут через туннель (категория `final: proxy` → резолв на gateway, по умолчанию так), либо, если нужен nfqws-direct, их надо резолвить через `dns-remote`. Это же касается любого домена, который Яндекс отдаёт как NXDOMAIN/заглушку.

### Отказоустойчивость роутера

Жёсткий kill-switch сознательно **не** используется (домашняя сеть). Падение туннеля ≠ падение sing-box: при живом процессе RU/zapret-категории работают, а геоблок-категория (`final: proxy`) отваливается сама на уровне приложения без утечки RU-IP. Единственное окно утечки — редкий краш процесса; его страхует cron-watchdog (`/usr/bin/singbox-watchdog`, раз в минуту), включаемый `router_singbox_watchdog`. На зоне `proxy` стоит `mtu_fix` (MSS-clamp для TCP внутри туннеля).

## Потенциальные доработки

- Перейти с открытых паролей в `group_vars/all.yml` на Ansible Vault или SOPS.
- Разделить `vpn_users` и `proxy_users`, если появятся разные политики доступа.
- Добавить healthchecks: ocserv login test, DNS test через VPN, proxy egress IP test (на роутере sing-box уже есть cron-watchdog).
- Добавить backup/restore для `/etc/ocserv/ocpasswd`.
- Явно описать IPv6-политику: disabled, routed или filtered.
- Добавить rate limiting для публичных proxy endpoints.
- Добавить fail2ban/nftables dynamic sets для SSH/ocserv brute force.
