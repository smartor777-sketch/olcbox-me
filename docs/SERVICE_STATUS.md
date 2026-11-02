# Состояние сервиса olcRTC

**Снимок:** 2026-11-02 20:17 UTC
**Обновлено:** 2026-11-02 21:13 UTC — добавлен раздел 8 «Docker и фаервол».
**Клиент:** [olcbox-me](https://github.com/smartor777-sketch/olcbox-me)
**Итог:** сервис работает, сбойных юнитов — 0.

> В документе нет секретов: криптоключ olcRTC, пароли claims, VK call URL и SSH-доступы не публикуются.

## 1. Сводка

| Компонент | Состояние | Детали |
| --- | --- | --- |
| `olcrtc.service` | ✅ active (running) | Jitsi `meet.riddlerx.org`, комната `panel.kuban-forum.ru`, peers: 0 |
| `vk-eternal-call.service` | ✅ active (running) | Playwright + Chromium headless shell, звонок удерживается |
| Systemd | ✅ 0 failed, 0 activating | 50 запущенных сервисов |
| HTTP-панели | ✅ 200 | panel / sleep / tg / save / save4box, 0.19–0.25 с |
| PostgreSQL | ✅ | `accepting connections` |
| Redis | ✅ | `PONG` |
| Диск | ✅ | 28 GB из 99 GB (29%) |
| RAM | ✅ | 8.4 GB из 16 GB, available 7.6 GB |
| Load average | ⚠️ высокий, но стабильный | 8.04 / 9.34 / 10.26 |

Высокий load average — структурный: ~63 процесса `mtproto-proxy` и несколько `uvicorn`. CPU не насыщен, запас есть. Это не сбой и не утечка.

## 2. Конфигурация

### 2.1 Сервер olcRTC

| Параметр | Значение |
| --- | --- |
| Режим | `srv` |
| Провайдер | `jitsi` |
| Транспорт | `datachannel` |
| Комната | `https://meet.riddlerx.org/panel.kuban-forum.ru` |
| Jitsi-хост | `meet.riddlerx.org` (RU, 78.36.1.154) |
| DNS | `8.8.8.8:53` |
| Liveness | interval 10s, timeout 5s, failures 3 |
| Конфиг | `/etc/olcrtc/server.yaml` |
| Данные | `/var/lib/olcrtc/data` |
| Юнит | `olcrtc.service` (`Restart=always`, `RestartSec=5s`, hardening включён) |
| Запущен с | 2026-11-02 19:43:41 UTC |
| Клиентов (peers) | 0 — нормально, пока никто не подключён |
| Криптоключ | хранится в `server.yaml`, в репозиторий не включён |

Подтверждение из лога:

```text
jitsi: MUC joined meet.riddlerx.org/panel.kuban-forum.ru; waiting for peer …
Link connected
```

Сервисные эндпоинты хоста:

| Эндпоинт | Статус |
| --- | --- |
| `POST /http-bind` | 200, анонимная BOSH-сессия |
| `/xmpp-websocket` | живой (501 на plain GET — норма для WS-хендлера) |
| `/colibri-ws/` | 200 |
| `/config.js`, `/external_api.js` | 200 |
| TLS | Let's Encrypt, до 2027-01-09 |

### 2.2 Клиентские профили

- 8 пользователей, по 4 файла на каждого: `*_olcrtc.json`, `.txt`, `.uri`, `.png`.
- Путь: `/var/lib/nyxpanel/users/<user>/`.
- Все `room_id` указывают на `meet.riddlerx.org`.
- QR-коды перегенерированы 2026-11-02 (`qrencode`), владелец `nyxpanel:nyxpanel`.
- Ключ и claims-пароли содержатся в клиентских файлах и здесь не дублируются.

### 2.3 vk-eternal-call

| Параметр | Значение |
| --- | --- |
| Назначение | удерживать VK-звонок для продления WDTT-хеша |
| Playwright | 1.62.0 (`/opt/pw-venv`) |
| Браузер | Chromium headless shell build 1234 (`/root/.cache/ms-playwright/chromium_headless_shell-1234`) |
| Состояние | active (running), `Restart=always`, `RestartSec=10` |
| Автозапуск | `enabled` |
| Лог | `Call alive. Sleeping 60s...` |

Холодный старт проверен вручную: после `systemctl restart` браузер поднимается за ~2 с, страница звонка открывается, цикл продолжается. Браузер лежит на диске (не в tmpfs), поэтому после перезагрузки сервис поднимется сам.

## 3. Инфраструктура

| Метрика | Значение |
| --- | --- |
| Сбойные юниты | 0 |
| Запущенные сервисы | 50 |
| Load average | 8.04 / 9.34 / 10.26 |
| RAM | 8.4 GB / 16 GB, available 7.6 GB |
| Диск `/` | 28 GB / 99 GB (29%) |
| PostgreSQL | принимает соединения |
| Redis | `PONG` |
| Панели | HTTP 200, 0.19–0.25 с |

## 4. История переключений Jitsi-хоста

| Дата | Хост | Результат |
| --- | --- | --- |
| до 2026-11-02 | `meet.egovm.ru` | ❌ `host-unknown`, хост перестал обслуживать домен |
| 2026-11-02 | `meet.mamba.group` | ⚠️ MUC join проходит, но data-channel между участниками не ретранслируется → у клиента `wait for peer` и таймаут |
| 2026-11-02 | `meet.riddlerx.org` | ✅ туннель работает, проверен end-to-end |

Причина сбоя на `mamba`: клиент и сервер заходят в комнату, JVB открывает SCTP, но сообщения между endpoint'ами не доходят. Через ~10–15 с клиент падает по `wait for peer`, сервер видит `jitsi bridge closed`.

Бэкапы перед изменениями:

- `/etc/olcrtc/server.yaml.bak-egovm-20261102`
- `/etc/olcrtc/server.yaml.bak-mamba-20261102`
- `/root/nyxpanel-users-olcrtc-backup-20261102.tar.gz`
- `/root/nyxpanel-users-olcrtc-backup-mamba-20261102.tar.gz`
- шаблоны: `*.bak-egovm-20261102`, `*.bak-mamba-20261102`

## 5. Проверка Jitsi-хостов

### 5.1 Методика

Проверка не ограничивается `curl` на главную страницу. Для каждого хоста поднимается **реальная пара** `olcrtc srv` + `olcrtc cnc` в отдельной комнате, затем через локальный SOCKS5 выполняются запросы:

```bash
curl -x socks5h://127.0.0.1:PORT http://example.com
curl -x socks5h://127.0.0.1:PORT https://ifconfig.me
```

Критерий «работает» — оба запроса вернули `200`. Дополнительно стабильные кандидаты проходят 2-минутный тест: 4 запроса с паузами.

### 5.2 Официальный список olcRTC

| Хост | Вход в MUC | Туннель | Причина отказа |
| --- | --- | --- | --- |
| `meet.riddlerx.org` | ✅ | ✅ | — |
| `meet.mamba.group` | ✅ | ❌ | нет ретрансляции data-channel |
| `meet.mirox.chat` | ✅ | ❌ | нет ретрансляции data-channel |
| `conf.expressmoney.com` | ✅ | ❌ | нестабильный ICE/TURN, `i/o timeout` |
| `meet.egovm.ru` | ❌ | ❌ | `host-unknown` |
| `m.catonmoon.com` | ❌ | ❌ | нет анонимного XMPP-логина |
| `conference.ct.placetime.team` | — | ❌ | недоступен |
| `jitsy.amateusfox.online` | — | ❌ | недоступен |
| `meet.ecopsy.com` | — | ❌ | недоступен |
| `webinar.knomary.dev` | — | ❌ | недоступен |
| `meet.playform.ru` | — | ❌ | недоступен |
| `webinar.devknomarylms.ru` | — | ❌ | недоступен |
| `zgn-y-vc01.zignotch.com` | — | ❌ | недоступен |
| `conf.hyperia.space` | — | ❌ | недоступен |
| `jitsi.etudevs.ru` | — | ❌ | недоступен |

**Рабочий из официального списка — только `meet.riddlerx.org`.**

### 5.3 Публичные инстансы вне списка

Источник: [список Framatalk](https://framatalk.org/abc/en/info) (~100 инстансов).

Воронка:

| Этап | Хостов |
| --- | --- |
| Всего проверено | 100 |
| Доступны и пускают анонимно (BOSH `authid`) | 28 |
| Держат туннель (HTTP + HTTPS 200 через SOCKS) | 15 |
| Прошли 2-минутный стресс-тест | 5 |

Рабочие хосты:

| Хост | Страна | RU-доступ | TTFB из RU | Сертификат | Стресс-тест |
| --- | --- | --- | --- | --- | --- |
| `meet.systemli.org` | 🇩🇪 | 200 | **0.83 с** | 2027-01-18 | ✅ |
| `meet.vpsfree.cz` | 🇨🇿 | 200 | **0.98 с** | 2027-01-30 | ✅ |
| `jitsi.milkywan.fr` | 🇫🇷 | 200 | 1.31 с | 2026-12-24 | ✅ |
| `meet.hostpoint.ch` | 🇨🇭 | 200 | 2.05 с | 2027-02-13 | ✅ |
| `framatalk.org` | 🇫🇷 | 200 | 2.02 с | 2027-01-15 | ✅ |
| `conf.underworld.fr` | 🇫🇷 | 200 | 4.31 с | — | разово ✅ |
| `allo.bim.land` | 🇫🇷 | 200 | — | — | разово ✅ |
| `fairmeeting.net` | 🇦🇹 | 200 | — | — | разово ✅ |
| `konferenz.netzbegruenung.de` | 🇩🇪 | 200 | — | — | разово ✅ |
| `meet.greenmini.host` | 🇳🇱 | 200 | — | — | разово ✅ |
| `meet.lug-stormarn.de` | 🇩🇪 | 200 | 1.09 с | — | ❌ 000 во втором прогоне |
| `visio.colibris-outilslibres.org` | 🇫🇷 | 200 | — | — | ⚠️ TURN-варнинги |
| `video.devloprog.org` | 🇫🇷 | 200 | — | — | ⚠️ TURN-варнинги |
| `meet.ffmuc.net` | 🇩🇪 | 200 | 19.69 с | — | ⚠️ TURN-варнинги, очень медленный |
| `jitsi.free.org` | 🇫🇷 | 200 | — | — | ⚠️ TURN-варнинги |

### 5.4 Рекомендованные запасные

| Приоритет | Хост | Почему |
| --- | --- | --- |
| Основной | `meet.riddlerx.org` | RU-хостинг, работает сейчас, единственный из официального списка |
| Запасной №1 | `meet.vpsfree.cz` | Чехия, близко по сети, 0.98 с, стабилен под нагрузкой |
| Запасной №2 | `meet.systemli.org` | Германия, самый быстрый отклик (0.83 с), стабилен |
| Запасной №3 | `jitsi.milkywan.fr` | Франция, 1.31 с, стабилен |
| Аварийный | `meet.hostpoint.ch`, `framatalk.org` | Стабильны, но коммерческий/редиректор |

Не рекомендуются: `mamba`, `mirox`, `expressmoney` (не ретранслируют data-channel или нестабильны), `egovm`, `catonmoon` (не пускают).

## 6. Runbook: переключение на другой Jitsi-хост

1. Проверить новый хост тестовой парой `olcrtc srv` + `cnc` и запросами через SOCKS (см. 5.1).
2. Сделать бэкап и заменить комнату:

```bash
cp -a /etc/olcrtc/server.yaml /etc/olcrtc/server.yaml.bak-$(date +%F)
sed -i 's|https://OLD.HOST/panel.kuban-forum.ru|https://NEW.HOST/panel.kuban-forum.ru|' \
  /etc/olcrtc/server.yaml
systemctl restart olcrtc
journalctl -u olcrtc -n 20 --no-pager | grep -E "MUC joined|Link connected"
```

3. Обновить клиентские файлы и QR:

```bash
grep -rl "OLD.HOST" /var/lib/nyxpanel/users | xargs sed -i 's/OLD\.HOST/NEW.HOST/g'
for u in /var/lib/nyxpanel/users/*/*_olcrtc.uri; do
  qrencode -t PNG -o "${u%.uri}.png" -r "$u"
  chown nyxpanel:nyxpanel "${u%.uri}.png"
done
```

4. Обновить шаблоны, чтобы старый хост не вернулся при установке:

```text
/opt/nyxpanel/server/install.sh
/opt/nyxpanel/server/proxy_manager_dev.sh
/opt/nyxpanel/bin/convert_yaml_to_json.py
/opt/nyxpanel/bin/Merlin_olcrtc.yaml
/opt/nyxpanel/config/tmp_olcrtc_client_test.yaml
```

`/opt/nyxpanel/bin/proxy_manager.sh` читает `room.id` из `/etc/olcrtc/server.yaml` динамически — для новых пользователей править его не нужно.

5. Проверить end-to-end в реальной комнате тестовым claims-пользователем:

```bash
# клиентский yaml с room.id прод-комнаты и socks-портом 10899
olcrtc /tmp/client-test.yaml &
sleep 20
curl -x socks5h://127.0.0.1:10899 http://example.com
journalctl -u olcrtc -n 10 --no-pager | grep "peers count"   # ожидаем 1
```

6. Раздать клиентам новые конфиги/QR (в клиенте один `room_id`, автопереключения нет).

## 7. Ограничения и риски

- **Один `room_id` на клиента.** Автоматический failover между Jitsi-хостами невозможен: при смене хоста все пользователи должны переимпортировать конфиг или отсканировать QR.
- **Публичные Jitsi-инстансы — волонтёрские.** Постоянный туннель создаёт заметную нагрузку; использовать запасные хосты только на время аварии.
- **Хосты живут недолго.** Список выше — снимок на 2026-11-02. Перед переключением всегда перепроверять тестовой парой.
- **Секреты не публикуются.** Криптоключ и claims-пароли хранятся только на сервере и в клиентских конфигах.
- **Часы сервера.** Во время работ наблюдался скачок системного времени (~6 ч) с последующей NTP-синхронизацией — на сервисы не повлияло, но стоит учитывать при разборе логов.

## 8. Docker и фаервол

### 8.1 Почему UFW сам по себе не защищает Docker

UFW пишет правила фильтрации входящих соединений в цепочку `INPUT` таблицы `filter`. Туда попадают только пакеты, адресованные самому хосту. Docker меняет схему:

1. Включает `net.ipv4.ip_forward = 1` и поднимает мосты `docker0` / `br-xxxx`.
2. Для проброса портов добавляет DNAT-правила в `nat/PREROUTING`.
3. Пакет снаружи на опубликованный порт попадает в `PREROUTING` → цепочку `DOCKER` → **DNAT**, где адрес назначения на лету меняется с публичного IP на адрес контейнера (`172.x.x.x`).
4. После DNAT пакет уже не «локальный» — он идёт транзитом и уходит в `FORWARD`, а не в `INPUT`.
5. В `FORWARD` Docker ставит `DOCKER-USER` и `DOCKER-FORWARD` **выше** цепочек UFW, поэтому правила UFW до такого пакета не доходят.

Отсюда правило: `ufw status` может выглядеть идеально, а база в контейнере — быть открыта в интернет. Дополнительные ловушки:

- **IPv6.** С Docker 27 `ip6tables` управляется по умолчанию, DNAT дублируется и в IPv6. Правила, добавленные только в `iptables`, IPv6 не закрывают.
- **`docker-proxy` слушает `[::]`.** Если IPv6 в UFW выключен, пакет по IPv6 может зайти в userspace-сокет `docker-proxy` и оттуда — в IPv4-контейнер.
- **Опасный совет `"iptables": false`.** Docker перестаёт трогать netfilter, но заодно не создаёт `MASQUERADE` — контейнеры теряют выход в интернет, `docker build` падает на `apt-get update`. На рабочих серверах так делать нельзя.

### 8.2 Фактическое состояние прода

Проверено 2026-11-02, только чтение:

| Что проверено | Результат |
| --- | --- |
| Docker | 29.9.2, Compose v5.6.1, legacy iptables backend (`/etc/docker/daemon.json` отсутствует) |
| Контейнеры | 2: `telegram-bot-api` и `cobalt`, оба running |
| Публикация портов | `127.0.0.1:8082->8082` и `127.0.0.1:9000->9000` |
| `ss -tulpn` | только `127.0.0.1`; Postgres и Redis хоста — `127.0.0.1` и `[::1]` |
| `iptables -t nat -S DOCKER` | DNAT **только на `127.0.0.1`** для портов 8082 и 9000 |
| `raw/PREROUTING` | есть DROP-правила Docker 28+ для loopback-пакетов не с `lo` |
| `ip6tables -t nat -S DOCKER` | пусто — IPv6-DNAT нет |
| Внешний интерфейс | `ens3` (публичный `87.120.186.100`) |
| UFW | active; `DEFAULT_INPUT_POLICY=DROP`, `DEFAULT_FORWARD_POLICY=ACCEPT` |
| `DOCKER-USER` до правки | пуст (v4 и v6) |

**Вывод на момент проверки:** наружу базы и контейнеры не опубликованы, сценарий «UFW закрыт, а Docker открыт» не воспроизводится. Но пустой `DOCKER-USER` при `DEFAULT_FORWARD_POLICY=ACCEPT` означал, что случайное `ports: "5432:5432"` (без `127.0.0.1`) сразу открыло бы порт в интернет.

### 8.3 Предохранитель DOCKER-USER

Добавлен на прод, чтобы даже случайная публикация порта на `0.0.0.0` не стала доступна снаружи.

Файлы:

| Путь | Назначение |
| --- | --- |
| `/usr/local/sbin/docker-fw-guard.sh` | идемпотентный скрипт установки правил |
| `/etc/systemd/system/docker-fw-guard.service` | oneshot-юнит, `After=docker.service`, `PartOf=docker.service`, `enabled` |

Скрипт сам определяет внешний интерфейс по `ip route show default` (сейчас `ens3`), поэтому правило не привязано к имени интерфейса жёстко. Правила переустанавливаются после рестарта Docker благодаря `PartOf`.

Содержимое цепочки `DOCKER-USER` (одинаково для IPv4 и IPv6):

```text
-A DOCKER-USER -m conntrack --ctstate RELATED,ESTABLISHED -j RETURN
-A DOCKER-USER -m conntrack --ctstate INVALID -j DROP
-A DOCKER-USER -i ens3 -o docker0 -j DROP
-A DOCKER-USER -i ens3 -o br-+ -j DROP
-A DOCKER-USER -j RETURN
```

Почему именно так:

- `RELATED,ESTABLISHED` пропускается первым — контейнеры продолжают ходить в интернет и получать ответы.
- `DROP` ограничен парой `-i ens3 -o br-+` / `-o docker0`, то есть только новыми входящими пакетами с публичного интерфейса **в Docker-мосты**. VPN- и WDTT-маршрутизация (`vpn0`, `vpn7`, `wdtt0`, `wdtt-raw`) не затрагивается.
- Трафик из контейнеров наружу идёт как `-i br-+ -o ens3` и под правило не попадает.
- Опубликованные на `127.0.0.1` порты обрабатываются через `INPUT`/`docker-proxy` и тоже не затрагиваются.

Проверка предохранителя:

1. Временно добавлен DNAT `87.120.186.100:65530 → 172.18.0.2:9000` (cobalt).
2. С внешней машины (`dev`) запрос `curl http://87.120.186.100:65530/` завершился таймаутом (`000`, 8 с).
3. Счётчик правила `-i ens3 -o br-+ -j DROP` вырос на 7 пакетов / 420 байт — пакеты поймал именно предохранитель, а не фаервол хостера.
4. Временное DNAT-правило удалено, текущая конфигурация — только loopback-публикации.

Если в будущем понадобится открыть контейнер наружу осознанно, правило разрешения нужно ставить **до** `DROP` и матчить оригинальный порт через `--ctorigdstport` (порт до DNAT):

```bash
# пример: разрешить 195.88.54.10 -> контейнерный порт 5432
iptables -I DOCKER-USER 3 -i ens3 -s 195.88.54.10/32 -p tcp \
  -m conntrack --ctorigdstport 5432 -j RETURN
```

Но приоритетный способ — не публиковать порт на `0.0.0.0` вообще: `127.0.0.1:5432:5432`, доступ через SSH-туннель, либо `internal: true` для сети с БД.

### 8.4 Чек-лист аудита Docker-портов

```bash
# что и куда опубликовано
docker ps --format '{{.Names}}\t{{.Ports}}'
ss -tulpn | grep docker-proxy

# DNAT (что реально проброшено)
iptables -t nat -S DOCKER
ip6tables -t nat -S DOCKER

# защита транзита
iptables -t filter -S DOCKER-USER
ip6tables -t filter -S DOCKER-USER

# проверка с внешней машины
nmap -Pn -T4 --open -p 1-65535 87.120.186.100
```

Если в `nmap` появились `3306`, `5432`, `6379`, `27017`, `9200` или `2375` — порты нужно немедленно закрывать и проверять контейнеры на компрометацию.

### 8.5 Что сознательно не менялось

- **`DEFAULT_FORWARD_POLICY=ACCEPT` в UFW.** Ужесточение до `DROP` может задеть VPN/WDTT-маршрутизацию (`qeli-nat`, `wdtt0`, `wdtt-raw`), поэтому без отдельного окна и тестов не трогалось. Docker-трафик уже закрыт предохранителем.
- **Нативный nftables backend Docker 29+.** Помечен как experimental; текущий legacy-iptables + `DOCKER-USER` работает и проверен.
- **Внешний фаервол хостера.** Если панель провайдера даёт Cloud Firewall / Security Groups, его стоит включить как первый рубеж — пакеты будут отсекаться ещё до сетевого стека виртуалки.
