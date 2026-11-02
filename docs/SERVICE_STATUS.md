# Состояние сервиса olcRTC

**Снимок:** 2026-11-02 20:17 UTC
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
