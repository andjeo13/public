# Инструкция: exit-VPS с тремя независимыми транспортами

Как поднять с нуля VPS-выход для обхода блокировок: **VLESS+Reality** (TCP), **Hysteria2**
(UDP с обфускацией) и **AmneziaWG** (UDP, модифицированный WireGuard) на одной машине.
Схема работает в проде с мая 2026 на двух узлах одновременно; версии и грабли ниже — с живой
установки, не из документации.

Все адреса, порты, домены маскировки и ключи в тексте — заглушки вида `<...>`. Подставить
свои **до** начала работ: копировать чужие SNI и порты бессмысленно, именно они и есть
фингерпринт узла.

## Что получится

Один VPS за рубежом, с которого можно выйти в интернет тремя разными способами. Критерии
приёмки:

1. С российского канала открыты и отвечают: `<REALITY_PORT>/tcp`, `<HY2_PORT>/udp`,
   `<AWG_PORT>/udp`.
2. Клиент на Reality получает fake-сертификат домена маскировки — то есть при пассивном
   наблюдении соединение выглядит как поход на этот домен.
3. `https://<ваш_ip>:<HY2_PORT>` в браузере отдаёт не ошибку, а содержимое сайта-маскировки.
4. AmneziaWG: `awg show` показывает свежий handshake **и** `curl` через туннель отдаёт
   не-российский IP (это два разных проверяемых факта, см. раздел 6.4).
5. Падение любого из трёх сервисов не задевает остальные два: отдельные процессы, отдельные
   порты, отдельные ключи.
6. SSH на нестандартном порту, пароли отключены, firewall default deny, свои адреса в
   `ignoreip` fail2ban.
7. Повторный прогон любого шага ничего не ломает.

## Чего здесь нет

| Не входит | Где это решается |
|---|---|
| Клиентская часть (Clash / sing-box / Karing, правила маршрутизации, whitelist-домены) | отдельный документ |
| Подключение NAS к этому серверу и маршрутизация его приложений | [TrueNAS SCALE](vpn-exit-truenas-scale-routing-2026-10.md) |
| Роутер как шлюз на весь дом (OpenWrt, прозрачное проксирование, watchdog с авто-переключением) | отдельный документ |
| Обратный доступ в домашнюю сеть / выход в РФ через свой канал | отдельный документ |
| Телеметрия и мониторинг узла | раздел 8, только чеклист |

Три транспорта — это не избыточность. Опыт двух лет: режут их **по отдельности и
на отдельных маршрутах**. Мобильный оператор может рвать TLS-handshake, но пропускать UDP
с обфускацией; домашний провайдер — наоборот, фильтровать UDP к конкретной автономной
системе. Один транспорт = один отказ до полной потери связи.

---

## 1. Выбор VPS

### Требования

| Параметр | Минимум | Почему |
|---|---|---|
| vCPU / RAM | 1 / 1 GB | Reality+Hy2+AwG на троих едят ~150 MB; упор в канал, не в CPU |
| Трафик | 2–3 TB/мес | семейный fallback-профиль — 150–300 GB/мес, нужен запас ×10 |
| Канал | 1 Gbit/s | важнее заявленной ширины — что при переборе трафика soft-cap, а не отключение |
| IPv4 | выделенный, не NAT | shared-IP ломает входящие порты |
| ОС | Debian 12+ / Ubuntu 24.04 | свежий systemd, ядро ≥6.1 |
| Биллинг | без 3DS-препятствий | см. ниже |

IPv6 не обязателен, но если его нет — см. грабли №1, иначе половина сайтов отвалится.

### Что решает выбор провайдера

Три фактора, в порядке важности:

1. **Префикс не должен быть в блок-листах.** Это проверяется только замером с целевого
   канала и только после получения IP. Дешёвый VPS в блоке бесполезен за любые деньги.
2. **Оплата без геморроя.** Хостинг с российским биллингом принимает карту и не блокирует
   аккаунт через полгода. Европейские хостеры дешевле на 30–50%, но регистрация с
   российскими данными у части из них закрыта, а обход через родственников в ЕС — риск
   бана аккаунта вместе со всем стеком через 3–6 месяцев. Платить чуть дороже за
   легитимный канал оплаты выгоднее, чем экономить на нестабильном доверии.
3. **Страна и автономная система.** Ping до Хельсинки из Москвы ~30 мс, до Амстердама
   ~90 мс, до Сингапура ~230 мс. Для мессенджера решает только RTT. Второй узел брать
   **у другого провайдера в другой стране и другой AS** — иначе диверсификации нет.

### Проверка IP до начала работ

Как только IP известен — до того, как что-то настраивать:

```sh
# С любой машины на целевом канале (российский домашний или мобильный):
ping -c3 <NEW_IP>
nc -zvw3 <NEW_IP> 22          # SSH хостера жив?
```

Если IP не отвечает вообще, а соседи по `/24` (`.1`, `.50`, `.200` того же префикса)
пингуются — это точечный блок на `/32`, и лечится он сменой IP в панели того же хостера
(та же VM, тот же префикс, чистый адрес). Если не отвечает весь `/24` — менять провайдера.
Прецедент: в августе 2026 узел потерял доступность со всех российских каналов при живых
соседях по префиксу; смена IP в панели + reboot решили за 20 минут.

Проверять с **двух** разных каналов (домашний провайдер и мобильный оператор): фильтрация
бывает маршрутной, а не общей.

---

## 2. Базовая подготовка хоста

### 2.1 DNS

На Debian пакет `resolvconf` конфликтует с `systemd-resolved` и ломает резолвинг
при каждом обновлении. Проще прибить статику:

```sh
apt-get purge -y resolvconf 2>/dev/null
rm -f /etc/resolv.conf
printf 'nameserver 1.1.1.1\nnameserver 9.9.9.9\n' > /etc/resolv.conf
chattr +i /etc/resolv.conf   # опционально: чтобы DHCP-клиент не перезаписал
```

### 2.2 Пакеты

```sh
apt-get update && apt-get upgrade -y
apt-get install -y ufw fail2ban curl jq dnsutils ca-certificates gnupg \
                   unzip xz-utils nftables conntrack ncat
```

### 2.3 sysctl

```sh
cat > /etc/sysctl.d/99-vpn-exit.conf <<EOF
# BBR — заметно лучше cubic на длинных и потерянных маршрутах
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# Форвардинг нужен для AmneziaWG (раздел 5). Без него туннель поднимется,
# но транзитный трафик клиента не пойдёт.
net.ipv4.ip_forward = 1

# Буферы под Hysteria2: QUIC на гигабитном канале упирается в дефолтные 212 KB
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
EOF
sysctl --system
```

Проверка: `sysctl net.ipv4.tcp_congestion_control net.ipv4.ip_forward` → `bbr`, `1`.

Если RAM меньше 2 GB — добавить swap, иначе сборка Go-бинарей (раздел 5) упадёт по OOM:

```sh
fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness = 10' > /etc/sysctl.d/99-swap.conf && sysctl --system
```

### 2.4 SSH

**Порядок критичен: не закрывать текущую сессию, пока новый доступ не подтверждён из
второго окна.** Сначала положить ключ, потом включить новый порт, потом проверить, и только
потом отключать пароли.

```sh
# 1. С локальной машины — свой ключ на сервер
ssh-keygen -t ed25519 -f ~/.ssh/<alias> -C "<alias>"
ssh-copy-id -i ~/.ssh/<alias>.pub root@<NEW_IP>

# 2. На сервере — hardening отдельным drop-in файлом (не правя основной sshd_config)
cat > /etc/ssh/sshd_config.d/99-hardening.conf <<EOF
Port 22
Port <SSH_PORT>
PermitRootLogin prohibit-password
PasswordAuthentication no
KbdInteractiveAuthentication no
DebianBanner no
EOF
sshd -t && systemctl reload ssh
```

Два `Port` одновременно — это окно на переход. Из **второго** терминала:

```sh
ssh -i ~/.ssh/<alias> -p <SSH_PORT> root@<NEW_IP> 'echo ok'
```

Получив `ok` — убрать `Port 22` из drop-in, `sshd -t && systemctl reload ssh`. Не раньше.

`<SSH_PORT>` имеет смысл выносить с 22: не ради безопасности (ключ и так не брутфорсится),
а чтобы отвязать журнал от фонового шума сканеров и не ловить баны по пустякам. Любой
порт >1024, не совпадающий с портами VPN-сервисов.

Запись в `~/.ssh/config` на клиенте:

```
Host <alias>
  HostName <NEW_IP>
  Port <SSH_PORT>
  User root
  IdentityFile ~/.ssh/<alias>
  IdentitiesOnly yes
```

### 2.5 Firewall

```sh
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow <SSH_PORT>/tcp    comment 'SSH'
ufw allow <REALITY_PORT>/tcp comment 'Reality'
ufw allow <HY2_PORT>/udp     comment 'Hysteria2'
ufw allow <AWG_PORT>/udp     comment 'AmneziaWG'
ufw --force enable
ufw status verbose
```

Перед каждой правкой firewall — бэкап и явная команда откатa:

```sh
mkdir -p /root/backups
iptables-save > /root/backups/fw.before-$(date +%s)
# откат: iptables-restore < /root/backups/fw.before-<ts>
```

Для правок, после которых можно потерять доступ, — сторож с авто-откатом. Запускать
**до** применения изменения:

```sh
( sleep 300; ufw --force reset; ufw default allow incoming; ufw --force enable ) &
# подтвердив доступ из второй сессии: kill %1
```

### 2.6 fail2ban

Дефолтный `sshd` jail и **сразу** свои адреса в белый список:

```sh
cat > /etc/fail2ban/jail.local <<EOF
[DEFAULT]
# Критично: с banaction = ufw бан режет ВСЕ порты хоста, а не только SSH.
# Свой адрес в бане = узел недоступен целиком, включая VPN.
ignoreip = 127.0.0.1/8 <YOUR_HOME_IP> <YOUR_OTHER_HOSTS>
bantime  = 1h
findtime = 10m
maxretry = 3

[sshd]
enabled = true
port    = <SSH_PORT>
EOF
systemctl restart fail2ban && fail2ban-client status sshd
```

Прецедент: домашний адрес улетел в бан после серии подключений за одну сессию настройки.
Симптом — «хост недоступен из дома, но открыт со всех остальных машин», и выглядит это как
проблема маршрутизации. Признак бана, а не упавшего сервиса: `Connection refused`
(это REJECT), а не timeout. Разбан: `fail2ban-client set sshd unbanip <ip>`.

Домашний адрес обычно динамический — при смене провайдерского IP строку надо обновлять.
Держать запасной путь входа (другой VPS как jump-хост) полезно именно для таких случаев.

---

## 3. Слой 1 — VLESS + Reality (xray)

Основной TCP-транспорт. Reality не требует своего домена и сертификата: сервер при
рукопожатии притворяется чужим сайтом, и клиент видит настоящий сертификат этого сайта.
Для пассивного наблюдателя соединение неотличимо от похода на домен маскировки.

### 3.1 Установка

```sh
bash -c "$(curl -fsSL https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
xray version   # проверено на 26.3.x
```

### 3.2 Ключи

```sh
xray x25519            # PrivateKey (на сервер) + Password/PublicKey (клиенту)
xray uuid              # UUID клиента
openssl rand -hex 8    # shortId
```

Сохранить в `/etc/xray/creds.env` (`chmod 600`) и в свой парольный менеджер. **Ключи
уникальны для каждого узла** — не переиспользовать между серверами: компрометация одного
не должна раскрывать остальные.

### 3.3 Выбор домена маскировки

Требования к кандидату: TLS 1.3 + HTTP/2, домен не заблокирован в вашей стране, его
посещение выглядит обыденно, и сам сервер отдаёт `X25519` в key_share. Хорошие категории:
CDN-эндпоинты крупных вендоров, домены обновлений браузеров, аналитические поддомены.
Плохие: что-то, к чему ваш IP не имеет повода обращаться, и что-то уже заблокированное.

Проверка кандидата:

```sh
echo Q | openssl s_client -connect <CANDIDATE>:443 -servername <CANDIDATE> -tls1_3 2>&1 \
  | grep -E 'Protocol|Cipher|subject='
curl -sI --http2 https://<CANDIDATE>/ | head -1   # ждём HTTP/2
```

На разных узлах — **разные** домены маскировки: одинаковый SNI на нескольких ваших IP
позволяет связать их в одну группу.

### 3.4 Конфиг

`/usr/local/etc/xray/config.json`:

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "tag": "vless-reality-in",
      "listen": "0.0.0.0",
      "port": <REALITY_PORT>,
      "protocol": "vless",
      "settings": {
        "clients": [
          { "id": "<UUID>", "flow": "xtls-rprx-vision" }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "<MASQ_DOMAIN>:443",
          "xver": 0,
          "serverNames": ["<MASQ_DOMAIN>"],
          "privateKey": "<REALITY_PRIVATE_KEY>",
          "shortIds": ["<SHORT_ID>"]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    }
  ],
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": { "domainStrategy": "UseIPv4" }
    },
    { "tag": "block", "protocol": "blackhole" }
  ],
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      { "type": "field", "ip": ["geoip:private"], "outboundTag": "block" },
      { "type": "field", "protocol": ["bittorrent"], "outboundTag": "block" }
    ]
  }
}
```

Три неочевидных места:

- **`domainStrategy: UseIPv4` в freedom-outbound** — обязательно, если у VPS нет глобального
  IPv6. Без него xray отдаёт имя хоста в ядро, резолвер предпочитает AAAA, ядро пытается
  подключиться по IPv6 — и получает молчаливый таймаут. Симптом: «всё работает, кроме
  YouTube» (и любых других AAAA-only доменов). Проверить наличие IPv6: `ip -6 addr show scope global`
  — если пусто, флаг нужен.
- **`geoip:private` в blackhole** — иначе ваш узел становится сканером внутренней сети
  хостера, и это быстро заканчивается abuse-репортом.
- **blackhole для bittorrent** — главная причина жалоб хостеру. Дешевле отрезать на входе.

```sh
xray -test -config /usr/local/etc/xray/config.json
systemctl restart xray && systemctl enable xray
systemctl is-active xray
```

---

## 4. Слой 2 — Hysteria2

UDP-транспорт на QUIC с обфускацией salamander. Нужен ровно для того случая, когда
DPI режет TLS-рукопожатие: трафик salamander не похож на ClientHello, и фильтры,
настроенные на TLS, его не видят. На мобильных операторах это регулярно единственный
работающий канал.

Reality (TCP) и Hysteria2 (UDP) могут жить **на одном номере порта** — это разные
протоколы транспортного уровня, конфликта нет. Чем меньше открытых портов, тем меньше
поводов для вопросов.

### 4.1 Бинарь

```sh
HY_VER=v2.12.2   # проверено на 2.8.1 и 2.12.2
curl -fsSL -o /usr/local/bin/hysteria \
  "https://github.com/apernet/hysteria/releases/download/app%2F${HY_VER}/hysteria-linux-amd64"
chmod +x /usr/local/bin/hysteria
hysteria version
```

### 4.2 Сертификат и креды

TLS-сертификат self-signed: проверять его клиент всё равно не будет (`insecure: true`),
защита держится на пароле obfs, а не на доверии к сертификату. CN берётся тот же, что
домен маскировки — чтобы совпадал с тем, что увидят в ответе на зонд.

```sh
mkdir -p /etc/hysteria
openssl req -x509 -nodes -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
  -keyout /etc/hysteria/key.pem -out /etc/hysteria/cert.pem \
  -subj "/CN=<HY2_MASQ_DOMAIN>" -days 36500
chmod 600 /etc/hysteria/key.pem

openssl rand -base64 24   # пароль auth
openssl rand -base64 24   # пароль obfs
```

### 4.3 Конфиг

`/etc/hysteria/server.yaml` (`chmod 600`):

```yaml
listen: :<HY2_PORT>

tls:
  cert: /etc/hysteria/cert.pem
  key: /etc/hysteria/key.pem

auth:
  type: password
  password: <HY2_AUTH>

obfs:
  type: salamander
  salamander:
    password: <HY2_OBFS>

# Кто постучится без знания obfs-пароля, увидит настоящий сайт, а не ошибку сервиса.
masquerade:
  type: proxy
  proxy:
    url: https://<HY2_MASQ_DOMAIN>/
    rewriteHost: true

quic:
  initStreamReceiveWindow: 8388608
  maxStreamReceiveWindow: 8388608
  initConnReceiveWindow: 20971520
  maxConnReceiveWindow: 20971520
```

Домен маскировки Hysteria2 имеет смысл взять **не тот же**, что у Reality: два разных
фронта на одном IP хуже коррелируются между собой.

### 4.4 systemd

```sh
cat > /etc/systemd/system/hysteria-server.service <<EOF
[Unit]
Description=Hysteria2 Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/hysteria server -c /etc/hysteria/server.yaml
Restart=on-failure
RestartSec=5
# Привилегии только на bind низкого порта, без запуска от root
User=hysteria
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF

useradd -r -s /usr/sbin/nologin hysteria 2>/dev/null
chown -R hysteria:hysteria /etc/hysteria
systemctl daemon-reload
systemctl enable --now hysteria-server
systemctl is-active hysteria-server
```

### 4.5 Проверка маскировки

Снаружи, с клиентской машины:

```sh
curl -sk -o /dev/null -w '%{http_code} %{http_version}\n' https://<NEW_IP>:<HY2_PORT>/
# ждём 200 и содержимое сайта-маскировки, а не ошибку
```

---

## 5. Слой 3 — AmneziaWG (userspace)

WireGuard с добавленными junk-пакетами и изменёнными заголовками рукопожатия: обычный WG
определяется DPI по характерной первой посылке, AmneziaWG — нет. Нужен как второй UDP-канал
на случай, если начнут троттлить именно QUIC, и как транспорт для «постоянного» туннеля
(роутер, телефон) в отличие от прокси-режима.

**Userspace-реализация (`amneziawg-go`), а не модуль ядра** — сознательный выбор:
kernel-модуль собирается через DKMS и молча отваливается после каждого обновления ядра,
если не стоит метапакет `linux-headers-amd64`/`linux-headers-generic`. На VPS с
автообновлениями это мина с отложенным сроком. Userspace к смене ядра безразличен.
Цена — чуть выше нагрузка на CPU, на гигабитном канале незаметно.

### 5.1 Сборка

Версии пинить и проверять контрольную сумму — это бинарь, через который пойдёт весь трафик.

```sh
GO_VER=1.22.5
curl -fsSL -o /tmp/go.tgz https://go.dev/dl/go${GO_VER}.linux-amd64.tar.gz
# Сумму взять с https://go.dev/dl/ для своей версии и подставить:
echo '<SHA256>  /tmp/go.tgz' | sha256sum -c -
rm -rf /usr/local/go && tar -C /usr/local -xzf /tmp/go.tgz
export PATH=/usr/local/go/bin:$PATH

apt-get install -y git make build-essential

# Демон
cd /opt && git clone https://github.com/amnezia-vpn/amneziawg-go.git
cd amneziawg-go && git checkout <TAG_OR_COMMIT>   # проверено на v0.2.18
make && install -m 0755 amneziawg-go /usr/local/bin/amneziawg-go

# Утилиты awg / awg-quick
cd /opt && git clone https://github.com/amnezia-vpn/amneziawg-tools.git
cd amneziawg-tools && git checkout <TAG_OR_COMMIT>
cd src && make && make install     # → /usr/bin/awg, /usr/bin/awg-quick
```

systemd-юнит получается из шаблона WireGuard переименованием:

```sh
cp src/systemd/wg-quick@.service /etc/systemd/system/amneziawg@.service
sed -i 's/wg-quick/awg-quick/g; s/WireGuard/AmneziaWG/g' /etc/systemd/system/amneziawg@.service
systemctl daemon-reload
```

### 5.2 Конфиг

```sh
mkdir -p /etc/amnezia/amneziawg && chmod 700 /etc/amnezia/amneziawg
cd /etc/amnezia/amneziawg
awg genkey | tee server.key | awg pubkey > server.pub
awg genpsk > peer1.psk
chmod 600 server.key peer1.psk
```

`/etc/amnezia/amneziawg/awg0.conf` (`chmod 600`):

```ini
[Interface]
Address = 10.<X>.<Y>.1/24
ListenPort = <AWG_PORT>
PrivateKey = <SERVER_PRIVATE_KEY>

# Обфускация. Набор параметров ОБЯЗАН совпадать с клиентским конфигом байт в байт,
# иначе рукопожатие не состоится. Значения — свои, не копировать из примеров:
# именно они и отличают ваш трафик от «известного шаблона AmneziaWG».
Jc = 4
Jmin = 40
Jmax = 70
S1 = 50
S2 = 100
H1 = <RANDOM_1>
H2 = <RANDOM_2>
H3 = <RANDOM_3>
H4 = <RANDOM_4>

[Peer]
# Роутер
PublicKey = <PEER1_PUBLIC_KEY>
PresharedKey = <PEER1_PSK>
AllowedIPs = 10.<X>.<Y>.4/32

[Peer]
# Телефон
PublicKey = <PEER2_PUBLIC_KEY>
PresharedKey = <PEER2_PSK>
AllowedIPs = 10.<X>.<Y>.5/32
```

`H1..H4` — четыре различных числа, обычно семизначные; `S3`/`S4` появляются в более новых
версиях — если используете, они тоже должны совпадать с клиентом. Подсеть выбрать не
совпадающую ни с домашней LAN, ни с типовыми `192.168.0/1.x`: иначе маршруты на клиенте
схлопнутся.

### 5.3 Форвардинг и NAT — три барьера

**Самое дорогое место во всей инструкции.** Скопировать конфиг и бинари недостаточно:
туннель поднимется, `awg show` покажет свежий handshake и растущие счётчики — а трафика
через него не будет.

Почему обманчиво: рукопожатие — это UDP-пакеты **в процесс** на сервере, они идут через
цепочку INPUT (которую открывает `ufw allow <AWG_PORT>/udp`) и в FORWARD не попадают
никогда. Транзит клиента — наоборот: расшифрованный пакет **выходит** с VPS наружу и
проходит FORWARD. Три независимых барьера, каждого достаточно для тихого отказа:

| # | Что | Где | По умолчанию |
|---|---|---|---|
| 1 | `net.ipv4.ip_forward=1` | `/etc/sysctl.d/` (раздел 2.3) | `0` |
| 2 | NAT MASQUERADE для подсети туннеля | блок `*nat` в `/etc/ufw/before.rules` | нет |
| 3 | FORWARD ACCEPT туннель ↔ WAN | блок `*filter` в `/etc/ufw/before.rules` | ufw `DEFAULT_FORWARD_POLICY="DROP"` |

Имя WAN-интерфейса **сначала посмотреть**, оно разное у разных хостеров
(`ens1`, `enp0s3`, `eth0`):

```sh
WAN=$(ip -4 route show default | awk '{print $5}'); echo "$WAN"
```

Патч `/etc/ufw/before.rules` — с бэкапом:

```sh
cp /etc/ufw/before.rules /root/backups/before.rules.bak-$(date +%s)
```

В начало файла, **до** существующего `*filter`:

```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.<X>.<Y>.0/24 -o <WAN> -j MASQUERADE
COMMIT
```

В существующий блок `*filter`:

```
-A ufw-before-forward -i awg0 -o <WAN> -j ACCEPT
-A ufw-before-forward -i <WAN> -o awg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

```sh
ufw reload
iptables -t nat -L POSTROUTING -n -v | grep MASQUERADE   # правило на месте
```

`iptables-persistent` на стеке с ufw не нужен — `before.rules` подхватывается при reload.

### 5.4 Запуск и две разные проверки

```sh
systemctl enable --now amneziawg@awg0
awg show awg0
```

`awg show` — проверка **control plane**: интерфейс поднят, peer-ы видны, после подключения
клиента появится `latest handshake`. Это **не** значит, что трафик идёт.

Проверка **data plane** — только с подключённого клиента:

```sh
curl -s https://api.ipify.org   # должен вернуть IP вашего VPS, а не домашний
```

Если handshake свежий, а curl висит — это раздел 5.3, все три пункта по очереди.

---

## 6. Правило однородности стека

На сервере можно поставить либо `xray-core`, либо `sing-box` — оба умеют и Reality, и
Hysteria2 (sing-box — одним процессом, что удобнее в эксплуатации). Но:

**xray-клиент и sing-box-сервер не договариваются по REALITY + `xtls-rprx-vision`.**
Сервер отвечает `REALITY: processed invalid connection`, клиент получает пустой ответ.
Это протокольное расхождение двух реализаций, не ошибка аутентификации: совпадающие UUID,
публичный ключ, shortId и SNI не помогают.

| Клиент | Сервер | Результат |
|---|---|---|
| sing-box | sing-box | работает |
| xray | xray | работает |
| xray | sing-box | **отказ рукопожатия** |

Отсюда два следствия:

1. Выбирать реализацию **под парк клиентов**, а не по вкусу. Мобильные приложения
   (Karing, sing-box, Stash) — семейство sing-box; роутер с собственным xray-балансировщиком
   — xray.
2. Если нужны оба семейства — поднимать **оба** сервера на разных портах, с
   **разными** ключами и **разными** доменами маскировки. Не пытаться «починить» сменой
   flow на одной стороне: сломается совместимость с остальными клиентами того же семейства.

Диагностика этого класса занимает часы, если не знать симптома: со стороны клиента выглядит
как «сервер молчит» или тихий откат на домен маскировки. Искать в логах сервера.

---

## 7. Опциональные сервисы на том же узле

Оба независимы от VPN: отдельные процессы, отдельные порты, падение не задевает транспорты.

### 7.1 MTProto-прокси для Telegram

Открывается одним тапом по ссылке `t.me/proxy?...`, VPN-клиент не нужен — удобно для тех,
кому незачем настраивать профили. Реализация `9seconds/mtg` v2.2.8 в режиме fake-TLS.

Что важно:

- **Порт не 443** — он занят Reality, прод-транспорт не трогаем. Разумный выбор —
  «легально TLS-ные» порты, на которых не бывает открытого текста (`993` IMAPS,
  `995` POP3S): fake-TLS на них не выглядит аномалией, а два соседних номера дают одну
  пару правил в firewall.
- **Домен фронтинга — свой, не тот же, что у Reality и Hysteria2.** Разные фронты на
  разных сервисах не дают их скоррелировать.
- **Отдельный инстанс на каждого пользователя.** `mtg` v2 держит один секрет на процесс;
  раздельные секреты = независимая ротация и видно по логам, чей канал живой.
- **Секрет в git не хранится.** Ссылка `t.me/proxy?...` содержит секрет целиком — это
  и есть пароль. Выдать: `mtg access /etc/mtg-<name>.toml`, хранить в парольном менеджере.
- Отдельный системный пользователь, конфиги `400`.

Проверка маскировки (без секрета должен отдаваться настоящий сайт):

```sh
curl -s -o /dev/null -w '%{http_code} %{http_version}\n' \
  --resolve <FRONT_DOMAIN>:<MTG_PORT>:<NEW_IP> https://<FRONT_DOMAIN>:<MTG_PORT>
# ждём 200 HTTP/2
```

Смена секрета: `mtg generate-secret --hex <FRONT_DOMAIN>` → в конфиг → restart. Старая
ссылка умирает сразу, раздать новую.

### 7.2 Egress-прокси на loopback

Задача: дать какому-то сервису внутри своей сети зарубежный исходящий IP, не заворачивая
в VPN весь хост. Например, часть API отдаёт `403` на российские и серверные адреса.

Решение — `tinyproxy` в контейнере, слушающий **только** `127.0.0.1`, и SSH-туннель
с нужной машины. Снаружи порт не открывается, firewall и маршруты VPN не затрагиваются.

Ключевые места конфига:

```
Port 3128
Listen 127.0.0.1
Allow 127.0.0.1
DisableViaHeader Yes      # незачем сообщать о прослойке
ConnectPort 443           # только 443: если доступ утечёт, прокси не станет
                          # универсальным туннелем
# BindSame НЕ включать: привяжет исходящее к 127.0.0.1, и CONNECT упадёт с 500
```

Образ пинить по digest. Проверка — что прокси выпускает наружу **и** что порт закрыт
снаружи:

```sh
ssh <alias> 'curl -sS -x http://127.0.0.1:3128 -o /dev/null -w "%{http_code}\n" https://<TARGET>'
nc -z -w3 <NEW_IP> 3128 && echo "ОТКРЫТ — так быть не должно" || echo "закрыт, как и задумано"
```

---

## 8. Приёмка

С клиентской машины **на целевом канале** (не с другого VPS — он может ходить другим
маршрутом и дать ложно-положительный результат):

```sh
# Порты
nc -zvw3  <NEW_IP> <REALITY_PORT>    # TCP open
nc -zuvw3 <NEW_IP> <HY2_PORT>        # UDP open
nc -zuvw3 <NEW_IP> <AWG_PORT>        # UDP open

# Reality отдаёт сертификат домена маскировки
echo Q | openssl s_client -connect <NEW_IP>:<REALITY_PORT> -servername <MASQ_DOMAIN> 2>&1 \
  | grep -E 'subject=|issuer='

# Hysteria2 отдаёт сайт-маскировку
curl -sk -o /dev/null -w '%{http_code}\n' https://<NEW_IP>:<HY2_PORT>/
```

Дальше, подключив клиента по каждому транспорту по очереди:

| Проверка | Ожидание |
|---|---|
| `curl https://api.ipify.org` | IP вашего VPS |
| Открыть AAAA-only сайт (например, видеохостинг) | грузится — значит `UseIPv4` на месте |
| `systemctl is-active xray hysteria-server amneziawg@awg0` | три `active` |
| Остановить один сервис, проверить остальные | два других живы |
| `ufw status` | открыты ровно 4 порта, остальное deny |
| `fail2ban-client status sshd` | свой адрес не в бане, `ignoreip` на месте |

Чего в этой инструкции нет, но что стоит сделать сразу после приёмки:

- **Внешний мониторинг узла**, и обязательно не через сам узел: алерт о сломанном канале
  не должен ходить по этому каналу. Проверять именно порты транспортов, а не только ping.
- **Бэкап конфигов** в приватный репозиторий или на свой NAS. Минимум:
  `/usr/local/etc/xray/config.json`, `/etc/hysteria/server.yaml`,
  `/etc/amnezia/amneziawg/awg0.conf`, `/etc/ufw/before.rules`, `/etc/ssh/sshd_config.d/`.
  Учесть, что в них лежат приватные ключи — либо приватный репо, либо шифрование.
- **Запасной путь входа.** Если домашний адрес забанится, провайдер закроет inbound или
  сломается firewall — нужен второй способ попасть на машину: jump-хост, консоль хостера,
  обратный SSH-туннель на другую вашу машину.

---

## 9. Второй узел

Один VPS — одна точка отказа, и дело не в надёжности железа. Реальные сценарии из
практики: хостинг закрыл весь входящий трафик целиком (включая SSH), IP попал под точечный
блок, оператор начал фильтровать UDP к конкретной автономной системе. Любой из них
выключает узел полностью, независимо от количества транспортов на нём.

Второй узел ставится по этой же инструкции, но:

| Параметр | На втором узле |
|---|---|
| Провайдер, страна, AS | **другие** — иначе диверсификации нет |
| Ключи Reality, пароли Hysteria2 | **свои**, не копировать |
| Домены маскировки | **другие** |
| Реализация (xray / sing-box) | по парку клиентов (раздел 6) |
| Peer-set AmneziaWG | см. ниже |

**Трюк с AmneziaWG, который сильно упрощает переключение:** если на обоих узлах
одинаковые `PrivateKey` сервера, параметры обфускации, публичные ключи и `AllowedIPs`
пиров — то для клиента это «тот же сервер на новом адресе». Переключение сводится к
правке одной строки `Endpoint =` в клиентском конфиге и перезапуску интерфейса; состояние
рукопожатия ни одна сторона не пересчитывает. Цена — ключ сервера живёт на двух машинах;
компрометация одной раскрывает туннель к обеим. Для домашнего контура обмен оправданный,
но решение сознательное.

Для Reality/Hysteria2 так делать не нужно и не стоит: клиенты всё равно держат список
узлов и выбирают живой автоматически, а общие ключи там дают только лишний риск.

---

## 10. Грабли

| Симптом | Причина | Что делать |
|---|---|---|
| Работает всё, кроме отдельных сайтов (часто видеохостинг) | у VPS нет IPv6, резолвер предпочитает AAAA, ядро молча таймаутит | `domainStrategy: UseIPv4` в freedom-outbound (3.4) |
| После обновления сломался DNS | `resolvconf` против `systemd-resolved` | статический `/etc/resolv.conf` (2.1) |
| Handshake AmneziaWG свежий, трафика нет | ip_forward / MASQUERADE / FORWARD | все три пункта 5.3 |
| `REALITY: processed invalid connection` в логах сервера | xray-клиент против sing-box-сервера | однородный стек (6) |
| Узел недоступен из дома, доступен со всех остальных машин; `Connection refused` | свой адрес в бане fail2ban — режет все порты, не только SSH | `ignoreip`, разбан, вход через jump-хост (2.6) |
| IP не отвечает ни на что, соседи по `/24` живы | точечный блок на `/32` | сменить IP в панели того же хостера + reboot (1) |
| UDP-транспорт не работает с одного канала, работает с другого | маршрутная фильтрация UDP к этой AS | второй UDP-порт, TCP-канал как резерв, второй узел в другой AS |
| TLS режется на мобильном операторе, UDP проходит | DPI по ClientHello | для этого и нужен Hysteria2 с salamander |
| Туннель отвалился после обновления ядра | DKMS-модуль без метапакета headers | userspace-реализация (5), либо поставить `linux-headers-*` метапакет |
| Жалоба хостера на abuse | торренты или сканирование через узел | blackhole для bittorrent и `geoip:private` (3.4) |
| Браузер в обход прокси показывает домашний IP | QUIC/UDP:443 мимо HTTP-прокси | клиентская сторона: TUN-режим либо запрет UDP:443 |

## 11. Карта портов

Заполнить своими значениями и держать рядом — пригодится при каждой правке firewall.

| Порт | Протокол | Сервис | Конфиг |
|---|---|---|---|
| `<SSH_PORT>` | tcp | SSH | `/etc/ssh/sshd_config.d/99-hardening.conf` |
| `<REALITY_PORT>` | tcp | VLESS+Reality | `/usr/local/etc/xray/config.json` |
| `<HY2_PORT>` | udp | Hysteria2 | `/etc/hysteria/server.yaml` |
| `<AWG_PORT>` | udp | AmneziaWG | `/etc/amnezia/amneziawg/awg0.conf` |
| `<MTG_PORT>` | tcp | MTProto (опционально) | `/etc/mtg-<name>.toml` |
| `3128` | tcp, **только loopback** | egress-прокси (опционально) | `/opt/egress-proxy/tinyproxy.conf` |

## 12. Чеклист

```
[ ] IP проверен с двух российских каналов до начала работ
[ ] DNS статический, resolvconf удалён
[ ] sysctl: bbr + ip_forward + UDP-буферы
[ ] swap, если RAM < 2 GB
[ ] ключ SSH положен, новый порт подтверждён из второй сессии, пароли отключены
[ ] ufw: default deny, открыты ровно нужные порты, бэкап правил снят
[ ] fail2ban: jail.local с ignoreip своих адресов
[ ] xray: свои ключи, свой домен маскировки, UseIPv4, blackhole private+bittorrent
[ ] hysteria2: self-signed CN = домен маскировки, salamander, masquerade, своя пара паролей
[ ] amneziawg: свои ключи, свои параметры обфускации, своя подсеть
[ ] forwarding проверен не по awg show, а по curl с клиента
[ ] все три сервиса enable, перезагрузка узла пережита
[ ] приёмка (раздел 8) пройдена с целевого канала
[ ] конфиги в бэкапе, ключи не в открытом виде
[ ] внешний мониторинг, алерт не через сам узел
[ ] запасной путь входа на машину
```

---

Версии, на которых схема проверена (октябрь 2026): xray-core 26.3.x, hysteria 2.8.1 и
2.12.2, amneziawg-go v0.2.18, amneziawg-tools v1.0.20210914, sing-box 1.13.x, mtg 2.2.8,
Debian 12 / Ubuntu 24.04. Пины обновлять осознанно: это код, через который идёт весь ваш
трафик.
