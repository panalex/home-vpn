# home-vpn-ansible

Ansible-проект для двух VPS, где один узел работает как публичный ingress/relay, а второй — как egress/VPN-gateway.

Проект разворачивает:

- браузерный HTTP/SOCKS-прокси на edge-узле с аутентификацией для FoxyProxy и похожих клиентов;
- Telegram MTProto proxy через `mtg` на публичном edge-узле;
- цепочку `sing-box` edge → gateway;
- OpenConnect/AnyConnect VPN на gateway-узле через `ocserv`;
- локальный DNS для VPN-клиентов через `unbound` с DoT-forwarding;
- nftables firewall/NAT для обоих узлов;
- единый список пользователей для прокси и VPN.

Репозиторий подготовлен для публичной публикации: реальные IP, домены, пароли, `.git`-история и локальные vars исключены. Перед использованием создайте собственные `inventory.ini` и `group_vars/all.yml` из `.example`-файлов.

## Архитектура

```mermaid
flowchart LR
    U1[VPN clients\nUbuntu / Android / Windows]
    U2[Browser clients\nFoxyProxy]
    U3[Telegram clients]

    RU[Edge VPS\nnginx + mtg + sing-box\nHTTP :2080 / SOCKS :2081]
    DE[Gateway VPS\nocserv + unbound + sing-box\nVPN :443 / ss-in :4433]
    NET[Internet]
    DNS[Cloudflare / Quad9 DoT]

    U2 -->|HTTP proxy auth| RU
    U3 -->|MTProto FakeTLS :443| RU
    RU -->|sing-box chain :4433| DE
    U1 -->|OpenConnect :443 tcp/udp| DE
    DE --> NET
    DE -->|DoT :853| DNS
```

Подробная техническая карта: [`docs/architecture.md`](docs/architecture.md).

## Требования

- Control node: Ansible Core 2.15+.
- Managed hosts: Ubuntu 24.04.
- На gateway-узле домен для `ocserv` должен указывать на публичный IP gateway-узла.
- На edge-узле домен должен указывать на публичный IP edge-узла.
- DNS A-записи должны быть готовы до первого запуска Let’s Encrypt.

Установить коллекции:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Быстрый старт

```bash
cp inventory.ini.example inventory.ini
cp group_vars/all.yml.example group_vars/all.yml
```

Отредактируйте:

- `inventory.ini` — реальные адреса, пользователи SSH, домен edge-узла и email;
- `group_vars/all.yml` — домен gateway-узла, секрет `sing-box`, пользователи, пароли, порты.

Сгенерировать секрет для Shadowsocks 2022:

```bash
openssl rand -base64 32
```

Сгенерировать пароли пользователей:

```bash
LC_ALL=C tr -dc 'A-Za-z0-9_@%+=:.,-' </dev/urandom | head -c 32; echo
```

Запуск:

```bash
ansible-playbook -i inventory.ini site.yml
```

Запуск только пользовательской синхронизации:

```bash
ansible-playbook -i inventory.ini site.yml --tags users,ru_singbox
```

## Роли

| Роль | Узел | Назначение |
|---|---:|---|
| `common` | оба | базовые пакеты, sysctl, nftables, BBR |
| `ru_fronting` | edge | nginx-фронт, статическая заглушка, TLS-файлы |
| `ru_letsencrypt` | edge | выпуск LE-сертификата для edge-домена |
| `ru_singbox` | edge | локальный SOCKS/HTTP, публичный auth proxy, outbound на gateway |
| `ru_mtg` | edge | MTProto FakeTLS proxy для Telegram |
| `de_singbox` | gateway | ss-in на `:4433`, принимающий цепочку с edge |
| `de_unbound` | gateway | локальный DNS для VPN-клиентов, DoT upstream |
| `de_ocserv` | gateway | OpenConnect/AnyConnect VPN, LE-сертификат, VPN route hook |
| `users` | gateway | синхронизация `/etc/ocserv/ocpasswd` из `vpn_users` |
| `firewall` | оба | nftables input/forward rules |

## Важные runtime-особенности

`ocserv` создает интерфейс `vpns0` только после подключения первого клиента. Поэтому маршрут `10.99.0.0/24 dev vpns0` нельзя безусловно добавлять во время выполнения Ansible. В проекте это решено через `connect-script`, который выполняется после появления VPN-интерфейса.

nftables-правила с `iifname "vpns0"` можно загружать заранее: существование интерфейса для таких правил не требуется.

## Проверки после деплоя

Gateway:

```bash
systemctl status ocserv unbound sing-box --no-pager
ss -lntup | egrep '(:443|:4433|:53)'
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
systemctl status nginx mtg sing-box --no-pager
ss -lntup | egrep '(:443|:2080|:2081)'
```

Прокси через edge:

```bash
curl --proxy http://USER:PASSWORD@EDGE_IP:2080 https://ifconfig.me
```
