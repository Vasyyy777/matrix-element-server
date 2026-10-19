# Приватный Matrix-сервер + Element Call (Windows)

Готовый комплект: инструкция по сборке сервера, скрипты для звонков и установщики клиентов.

## Скачать

| Что | Файл | Откуда |
|---|---|---|
| Element для Windows | `Element-Setup-x64.exe` | [Releases](https://github.com/Vasyyy777/matrix-element-server/releases) |
| Element X для Android (arm64-v8a) | `Element-X-26.09.4-arm64-v8a.apk` | [Releases](https://github.com/Vasyyy777/matrix-element-server/releases) |
| Инструкция + скрипты | `README.md`, `sources.md` | этот репозиторий |

Другие архитектуры APK (armeabi-v7a, x86_64) и сборки Element — только на официальных сайтах, ссылки в инструкции ниже.

Сборки берутся напрямую с официальных источников:
`packages.element.io` (Element для Windows) и `github.com/element-hq/element-x-android/releases` (Element X APK), ничего не пересобирается.

---

## Инструкция


# Приватный Matrix-сервер со звонками (Element Call) — инструкция

Свой мессенджер на Windows: только свои аккаунты, федерация отключена, звук и видео через Element Call (LiveKit).

## Состав

| Компонент | Что делает | Порт |
|---|---|---|
| Synapse | сервер Matrix: аккаунты, комнаты, история | 127.0.0.1:8008 |
| Caddy | HTTPS + сертификат Let's Encrypt + прокси | 443 |
| LiveKit | медиа-сервер (SFU), сам поток звонка | 7880, 7881 + UDP 50000-50500 |
| lk-jwt-service | выдаёт JWT для LiveKit, проверяет токен Synapse | 127.0.0.1:8080 |
| NSSM | превращает всё это в службы Windows | — |

Все файлы удобно держать в `C:\ProgramData\Matrix\`.

## Что скачать (только официальные источники)

- Python 3.11+ — https://www.python.org/downloads/
- Synapse — https://github.com/element-hq/synapse/releases
  (на Windows проще всего: `pip install matrix-synapse`)
- Caddy — https://caddyserver.com/download
- LiveKit — https://github.com/livekit/livekit/releases
  (нужен файл вида `livekit_<версия>_windows_amd64.exe`)
- lk-jwt-service — https://github.com/element-hq/matrix-livekit-lk-jwt-service/releases
- NSSM — https://nssm.cc/download

Клиенты:

- **Element для Windows** — https://element.io/download
  прямая ссылка: https://packages.element.io/desktop/install/win32/x64/Element%20Setup.exe
- **Element X для Android** — https://github.com/element-hq/element-x-android/releases/latest
  (нужен файл `*.apk`), либо Google Play / F-Droid
- Element X для iOS — App Store, поиск «Element X»

## Установка сервера

### 1. Synapse

```
pip install matrix-synapse
```

Минимальный `homeserver.yaml` (путь `C:\ProgramData\Matrix\synapse\homeserver.yaml`):

```yaml
server_name: "YOUR_SERVER_DOMAIN"
listeners:
  - port: 8008
    bind_address: "127.0.0.1"
    type: http
    tls: false
    resources:
      - names: [client, federation]
        compress: false
database:
  name: sqlite3
  args:
    database: "C:/ProgramData/Matrix/synapse/homeserver.db"
media_store_path: "C:/ProgramData/Matrix/synapse/media_store"
enable_registration: false
```

Инициализация и создание первого пользователя:

```
cd C:\ProgramData\Matrix\synapse
python -m synapse.app.homeserver --generate-config --server-name YOUR_SERVER_DOMAIN --config-path homeserver.yaml
register_new_matrix_user -c homeserver.yaml http://localhost:8008 -u admin -p "ВАШ_ПАРОЛЬ" --admin
```

Федерацию не открывать: порт 8448 закрыт, и `/.well-known/matrix/server` (см. Caddy) направляет делегирование на 443.

### 2. Caddy

Файл `C:\ProgramData\Matrix\Caddyfile` — полный пример ниже, в разделе «Конфиги».
Служба:

```
nssm install MatrixCaddy "C:\ProgramData\Matrix\caddy\caddy.exe" run --config "C:\ProgramData\Matrix\Caddyfile" --adapter caddyfile
```

> **Важно.** Перезапуск службы Caddy через NSSM иногда даёт «Access denied».
> Рабочий способ — горячая перезагрузка через админ-API:
> ```
> & "C:\ProgramData\Matrix\caddy\caddy.exe" reload --config "C:\ProgramData\Matrix\Caddyfile" --adapter caddyfile
> ```

> **Важно для звонков.** Если в `/.well-known/matrix/server` указать порт 8448 вместо 443,
> lk-jwt-service уйдёт в делегирование на закрытый порт и все запросы RTC-токенов
> будут падать с `M_UNAUTHORIZED`.

### 3. LiveKit

Сгенерируйте пару ключ/секрет (она же понадобится lk-jwt-service):

```
livekit generate-keys
```

Положите `livekit.exe` и `livekit.yaml` в `C:\ProgramData\Matrix\livekit\`, впишите свой API key/secret и IP сервера (пример — в разделе «Конфиги»).

Служба:

```
nssm install MatrixLiveKit "C:\ProgramData\Matrix\livekit\livekit.exe" run --config "C:\ProgramData\Matrix\livekit\livekit.yaml"
```

### 4. lk-jwt-service

Ему нужны: URL LiveKit, тот же API key/secret, и endpoint для проверки OpenID-токенов Synapse:

```
https://YOUR_SERVER_DOMAIN/_matrix/federation/v1/openid/userinfo
```

Служба:

```
nssm install MatrixLkJwt "C:\ProgramData\Matrix\lkjwt\matrix-livekit-lk-jwt-service.exe" <флаги>
```

Точные имена флагов посмотрите в `--help` утилиты и в README проекта — в разных версиях они отличаются. Ключевые параметры: слушать `127.0.0.1:8080`, livekit-url `ws://127.0.0.1:7880`, api key/secret, openid-token-endpoint.

### 5. Файлы Windows

Понадобятся открытые порты (входящие):

```powershell
New-NetFirewallRule -DisplayName "MatrixLiveKit-UDP" -Direction Inbound -Protocol UDP -LocalPort 50000-50500 -Action Allow
New-NetFirewallRule -DisplayName "MatrixLiveKit-TCP" -Direction Inbound -Protocol TCP -LocalPort 7881 -Action Allow
```

UDP-диапазон нужен LiveKit напрямую, TCP 7881 — fallback, когда UDP заблокирован провайдером или роутером.

### 6. Проверка

```
https://YOUR_SERVER_DOMAIN/_matrix/client/versions          -> JSON, Synapse отвечает
https://YOUR_SERVER_DOMAIN/.well-known/matrix/client        -> базовый URL + rtc_foci
https://YOUR_SERVER_DOMAIN/.well-known/matrix/server        -> {"m.server":"YOUR_SERVER_DOMAIN:443"}
```

Дальше на телефоне: Element X → Добавить аккаунт → ввести `YOUR_SERVER_DOMAIN` (клиент сам найдёт сервер по well-known).

## Клиенты и звонки

- Windows: Element (обычный) — регистрация по серверу и логин/пароль.
- Телефон: Element X — там звонки идут через Element Call.

**Звонок работает, если:**

1. LiveKit поднялся после перезагрузки сервера (частая беда: сервис встал — звонок без видео/звука). Лечится перезапуском службы `MatrixLiveKit`.
2. В файрволе открыт UDP 50000-50500.
3. В well-known прописан `org.matrix.msc4143.rtc_foci` с `livekit_service_url` → `/livekit/jwt`.

## Почему окно звонка не закрывалось после сброса

Коротко: Element Call закрывает окно сам только когда комната считается **личкой**.

- Element X (26.09.x) сам строит адрес виджета звонка, `.well-known` с `widget_url` он **не читает** — эта настройка в Caddyfile ни на что не влияет.
- Намерения звонка: личка → `autoLeaveWhenOthersLeft = true` (второй сбросил трубку — окно закрылось); группа → `false` (окно висит).
- В matrix-rust-sdk личка = комната есть в аккаунтных данных `m.direct` **и** в ней не больше двух участников.

Поэтому для групповых комнат есть два способа.

### Способ 1 (серверный): добавить комнаты в m.direct

Готовый скрипт — `set-direct.ps1` в файле `sources.md`. Он для каждого аккаунта находит комнаты с двумя участниками и прописывает их в `m.direct`. Побочный эффект: такие комнаты в списке выглядят как личные переписки.

### Способ 2 (на телефоне): базовый URL Element Call

Element X → Настройки → **Опции разработчика** (нужно 7 раз нажать на номер версии внизу списка) → раздел **Element Call** → **Element Call base URL**, вписать:

```
https://appassets.androidplatform.net/element-call/index.html?autoLeave=true
```

Query переживает сборку адреса в matrix-rust-sdk, а Element Call читает параметры и из query (файл `UrlParams.ts`, функция `getParam`). Делать это надо на каждом телефоне.

Проверка: позвонить из комнаты с двумя участниками, второй стороне сбросить трубку — окно на первом должно закрыться само.

## Полезные команды

```
Get-Service MatrixLiveKit, MatrixLkJwt, MatrixCaddy
Restart-Service MatrixLiveKit
& "C:\ProgramData\Matrix\caddy\caddy.exe" reload --config "C:\ProgramData\Matrix\Caddyfile" --adapter caddyfile
```

Логи: `C:\ProgramData\Matrix\livekit\livekit-err.log`, `C:\ProgramData\Matrix\caddydata\caddy.log`.
Уровень логов LiveKit — в `livekit.yaml`, поле `logging.level` (`info` в обычной работе, `debug` только для разбора проблем).

## Безопасность

- Пароли аккаунтов, LiveKit API secret и access-токены **никуда не публиковать**. В примерах ниже — только плейсхолдеры.
- Держите `homeserver.yaml`, `livekit.yaml` и файлы с токенами в закрытых каталогах.
- Федерация не нужна — не открывайте порт 8448 наружу.

---

# Конфиги

## Caddyfile

```
{
	admin 127.0.0.1:2019
	log {
		output file C:\ProgramData\Matrix\caddydata\caddy.log
		level INFO
	}
}

YOUR_SERVER_DOMAIN {
	encode gzip

	handle /.well-known/matrix/client {
		header Content-Type application/json
		header Cache-Control "public, max-age=600"
		respond `{"m.homeserver":{"base_url":"https://YOUR_SERVER_DOMAIN"},"org.matrix.msc4143.rtc_foci":[{"type":"livekit","livekit_service_url":"https://YOUR_SERVER_DOMAIN/livekit/jwt"}]}` 200
	}

	# Делегирование federation: иначе lk-jwt-service уйдёт на 8448 и упрётся в M_UNAUTHORIZED
	handle /.well-known/matrix/server {
		header Content-Type application/json
		header Cache-Control "public, max-age=600"
		respond `{"m.server":"YOUR_SERVER_DOMAIN:443"}` 200
	}

	handle /_matrix/* {
		reverse_proxy 127.0.0.1:8008
	}

	handle /_synapse/client/* {
		reverse_proxy 127.0.0.1:8008
	}

	# lk-jwt-service: префикс /livekit/jwt срезается
	handle /livekit/jwt/* {
		uri strip_prefix /livekit/jwt
		reverse_proxy 127.0.0.1:8080 {
			header_up Host {host}
			header_up X-Real-IP {remote_host}
			header_up X-Forwarded-For {remote_host}
			header_up X-Forwarded-Proto {scheme}
		}
	}

	# LiveKit SFU: префикс срезается, иначе LiveKit его не понимает
	handle /livekit/sfu* {
		uri strip_prefix /livekit/sfu
		reverse_proxy 127.0.0.1:7880 {
			flush_interval -1
			header_up Host {host}
			header_up X-Real-IP {remote_host}
			header_up X-Forwarded-For {remote_host}
			header_up X-Forwarded-Proto {scheme}
		}
	}

	handle / {
		respond "Matrix homeserver" 200
	}
}
```

## livekit.yaml

```yaml
port: 7880
keys:
  YOUR_API_KEY: YOUR_API_SECRET
rtc:
  port_range_start: 50000
  port_range_end: 50500
  tcp_port: 7881
  use_external_ip: false
  node_ip: YOUR_SERVER_IP
room:
  auto_create: false
  empty_timeout: 300
  departure_timeout: 20
webhook:
  api_key: YOUR_API_KEY
  urls:
    - https://YOUR_SERVER_DOMAIN/livekit/jwt/sfu_webhook
logging:
  level: info
```

`node_ip` — внешний IP сервера (если сервер за NAT). `use_external_ip: false` оставьте, если IP публичный и прямой.
