# ADR-006: Flint 2 vs MikroTik RB4011 — сравнение роутеров

- **Статус:** черновик
- **Дата:** 2026-04-10
- **Связанные ADR:** ADR-001 (выбор роутера)

## Контекст

Flint 2 (GL-MT6000) работает как основной VPN-шлюз с апреля 2026 года. Накоплен значительный объём кастомизации (iptables, xray, WireGuard, stubby, TG-бот, domain routing ~81k доменов, watchdog, reverse-proxy WG и т.д.). Рассматриваем MikroTik RB4011 как альтернативу — стоит ли мигрировать.

## Сравнение железа

| | GL.iNet Flint 2 (MT6000) | MikroTik RB4011iGS+5HacQ2HnD-IN |
|---|---|---|
| **CPU** | MediaTek MT7986 4× ARM Cortex-A53 @ 2.0 GHz | Annapurna Labs AL21400 4× ARM Cortex-A15 @ 1.4 GHz |
| **RAM** | 1 GB DDR4 | 1 GB DDR3 |
| **Storage** | 8 GB eMMC | 512 MB NAND |
| **Ethernet** | 2× 2.5G + 4× 1G | 10× 1G (5+5) + 1× SFP+ |
| **Wi-Fi** | Wi-Fi 6 AX6000 (2.4+5 GHz) | Wi-Fi 5 AC (2.4+5 GHz) |
| **USB** | USB 3.0 | нет |
| **PoE** | нет | вход PoE (18-57V) |
| **Размер** | 225×152×47 мм | 228×120×29 мм (rackmount) |
| **Цена** | ~$90 / ~10k ₽ | ~$170-200 / ~20-25k ₽ |

**Железо — ничья с перевесом по разным аспектам:**
- Flint 2: быстрее CPU (2.0 vs 1.4 GHz), Wi-Fi 6, 2.5G Ethernet, больше storage (8 GB)
- MikroTik: больше портов (10x 1G + SFP+), PoE, rack-mount формат, более «серверный» вид

## Сравнение софта

| | GL.iNet Flint 2 (OpenWrt) | MikroTik RB4011 (RouterOS v7) |
|---|---|---|
| **ОС** | OpenWrt (полный Linux) | RouterOS v7 (проприетарная) |
| **Shell** | ash/bash, полноценный Linux | RouterOS CLI (свой синтаксис) |
| **Пакеты** | opkg (тысячи пакетов) | нет менеджера пакетов |
| **Контейнеры** | нет (но Docker через chroot теоретически) | Docker-контейнеры (RouterOS v7, ограниченно: ARM, 512 MB) |
| **WireGuard** | ✅ native (kernel module) | ✅ native (RouterOS v7) |
| **WG throughput** | ~900 Mbps | ~300-500 Mbps (ARM Cortex-A15 медленнее) |
| **VLESS/Xray** | ✅ native (бинарь в /usr/bin) | ⚠️ Только через container (сложно, ограничения по RAM/storage) |
| **Tailscale** | ✅ (бинарь, init.d) | ⚠️ Через container или userspace (экспериментально) |
| **cron + скрипты** | ✅ полноценный cron, bash, curl, jq, python | ⚠️ RouterOS scheduler + RouterOS scripting (другой язык, ограничения) |
| **iptables/nftables** | ✅ полный контроль | ❌ Свой firewall engine (мощный, но другой подход) |
| **ipset** | ✅ native | ✅ Address Lists (аналог) |
| **DNS** | dnsmasq (полный контроль, ipset auto-fill) | DNS встроенный (менее гибкий для domain-based VPN routing) |
| **Stubby/DoT** | ✅ | ✅ (DoH/DoT в v7) |
| **SNMP/мониторинг** | базовый | ✅ Профессиональный (SNMP, The Dude, Netflow, Traffic Flow) |
| **BGP/OSPF/MPLS** | через пакеты (ограниченно) | ✅ Полная поддержка (enterprise routing) |
| **Web GUI** | GL.iNet GUI + LuCI | Winbox + WebFig (очень мощные) |
| **Стабильность** | хорошая (зависит от пакетов) | отличная (RouterOS славится стабильностью) |
| **Сообщество** | OpenWrt (огромное) | MikroTik (большое, профессиональное) |
| **Обновления** | вручную (sysupgrade) | автоматические (RouterOS auto-upgrade) |

## Оценка под текущие задачи

### Что сейчас работает на Flint 2 и насколько переносимо на MikroTik

| Функция | Flint 2 | MikroTik — возможно? | Сложность переноса |
|---|---|---|---|
| **WireGuard → Amnezia** | ✅ штатно | ✅ RouterOS v7 native | 🟢 Легко |
| **VLESS Reality (xray)** | ✅ бинарь | ⚠️ Container (512 MB, ARM) | 🔴 Сложно, ненадёжно |
| **Watchdog WG↔xray** | ✅ shell script | ⚠️ RouterOS scripting (другой язык) | 🟡 Средне (переписать) |
| **Domain routing (81k доменов)** | ✅ dnsmasq ipset auto-fill | ⚠️ Address Lists + DNS static (менее удобно) | 🟡 Средне |
| **Автообновление списков (cron+curl)** | ✅ bash/curl/jq | ⚠️ RouterOS fetch + scripting (ограничения) | 🟡 Средне |
| **TG-бот (listener + commands)** | ✅ bash + curl | 🔴 RouterOS не может polling TG API нативно | 🔴 Очень сложно |
| **Stubby DoT** | ✅ отдельный демон | ✅ встроенный DoH/DoT | 🟢 Легко |
| **Reverse-proxy WG (wg-rev)** | ✅ ip+wg+iptables | ✅ WG+firewall NAT | 🟢 Легко |
| **Split DNS (dnsmasq)** | ✅ address=/domain/ip | ⚠️ DNS static entries (менее удобно для bulk) | 🟡 Средне |
| **DHCP уведомления в TG** | ✅ dhcp-script | ⚠️ DHCP lease script (возможно, но другой подход) | 🟡 Средне |
| **Tailscale** | ✅ native | ⚠️ Container/userspace | 🔴 Сложно |
| **sysupgrade persistence** | ✅ /etc/sysupgrade.conf | ✅ RouterOS сохраняет всё автоматически | 🟢 Лучше |

### Где MikroTik сильнее

1. **Профессиональный мониторинг** — SNMP, Traffic Flow, Netflow, The Dude. Для домашней сети overkill, но для серьёзного мониторинга лучше.
2. **Стабильность ОС** — RouterOS крайне стабильна, обновления не ломают конфиги.
3. **Больше портов** — 10× 1G + SFP+ (можно подключить DAC 10G к свитчу).
4. **BGP/OSPF** — если когда-то понадобится dynamic routing.
5. **PoE input** — питание по Ethernet (удобно в стойке).
6. **Firewall** — RouterOS firewall (filter/nat/mangle/raw chains) мощнее и предсказуемее чем iptables на OpenWrt.

### Где Flint 2 сильнее (и критически важен для текущего setup'а)

1. **VLESS/Xray** — native, работает, протестировано. На MikroTik — контейнер с ограничениями, экспериментально.
2. **TG-бот** — полноценный shell, curl, jq. На RouterOS — невозможно в текущем виде.
3. **Автообновление 81k доменов** — dnsmasq ipset, cron, bash скрипты. На MikroTik — значительно сложнее, адрес-листы ограничены.
4. **WireGuard throughput** — 900 Mbps на MT7986 vs ~400 Mbps на AL21400.
5. **Экосистема пакетов** — opkg (curl, jq, stubby, tailscale, tcpdump, iperf3...). RouterOS не имеет пакетного менеджера.
6. **Storage** — 8 GB eMMC vs 512 MB NAND. Для логов, скриптов, обновлений — огромная разница.
7. **Wi-Fi 6** vs Wi-Fi 5 (если используется встроенный Wi-Fi).
8. **Цена** — $90 vs $170-200.
9. **Уже настроен и работает** — вся инфраструктура, скрипты, документация. Миграция = недели работы.

## Анализ

### Сценарий «Заменить Flint 2 на MikroTik 4011»

**Потери:**
- VLESS/Xray — потеря основного fallback VPN-канала (Layer 0 fallback)
- TG-бот — потеря всех оповещений и управления через Telegram
- Автообновление доменов — деградация до ручного управления или сложных RouterOS скриптов
- Tailscale — потеря или сильное усложнение
- 2-4 недели миграции и тестирования
- WireGuard медленнее (~400 vs ~900 Mbps)

**Приобретения:**
- Профессиональный мониторинг (SNMP, Netflow)
- Больше портов, SFP+
- Стабильность RouterOS
- PoE input

**Вывод:** потери значительно превышают приобретения для текущего use case.

### Сценарий «MikroTik 4011 параллельно с Flint 2»

**Роль MikroTik:**
- Managed switch (10 портов + SFP+) для домашней сети
- Профессиональный firewall / monitoring уровень
- BGP/OSPF если нужен dynamic routing (сейчас не нужен)

**Роль Flint 2:**
- VPN-шлюз (как сейчас)
- Все кастомные скрипты, xray, TG-бот

**Вывод:** возможно, но дорого и избыточно для домашней сети из 20+ устройств. MikroTik как managed switch — overkill, есть дешевле.

### Сценарий «MikroTik для нового проекта / другой локации»

Если нужен второй роутер (дача, офис, родители) — MikroTik хорош:
- Стабильный, не требует внимания
- WireGuard для связи с домашней сетью
- Профессиональный remote management (Winbox)
- Не нужны VLESS/xray/TG-бот

## Решение

**Оставить Flint 2 как основной VPN-шлюз.** MikroTik RB4011 не заменяет Flint 2 для текущих задач из-за:
1. Критической зависимости от VLESS/Xray (которые на RouterOS нативно не работают)
2. Потери TG-бота и автоматизации на shell-скриптах
3. Деградации WireGuard throughput (×2 медленнее)
4. Огромного объёма работ по миграции (2-4 недели)
5. Уже работающей и задокументированной инфраструктуры

**MikroTik RB4011 может быть полезен для:**
- Отдельной локации (дача, офис, родители) с WireGuard site-to-site
- Managed switch уровня если понадобится SFP+ или 10G uplink
- Обучения RouterOS (профессиональный навык для CV)

## Компромиссы

- Flint 2 остаётся единой точкой отказа для VPN-инфраструктуры дома
- RouterOS объективно стабильнее OpenWrt при длительной работе без обслуживания
- MikroTik лучше для enterprise-сценариев, но текущие задачи — prosumer/homelab
- Инвестиция в OpenWrt/Flint 2 (скрипты, конфиги, знание) уже сделана — переезд неоправдан

## Источники

- [MikroTik RB4011 specs](https://mikrotik.com/product/RB4011iGS_5HacQ2HnD_IN)
- [GL.iNet Flint 2 specs](https://www.gl-inet.com/products/gl-mt6000/)
- [RouterOS v7 Container](https://help.mikrotik.com/docs/spaces/ROS/pages/84738097/Container)
- [RouterOS v7 WireGuard](https://help.mikrotik.com/docs/spaces/ROS/pages/69664792/WireGuard)
- ADR-001: Выбор роутера (в этом же репо)
