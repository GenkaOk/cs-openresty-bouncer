# Yandex SmartCaptcha для CrowdSec OpenResty bouncer

Форк `crowdsecurity/cs-openresty-bouncer` + `crowdsecurity/lua-cs-bouncer`, добавляющий
провайдера `smartcaptcha` в механизм `captcha`-реакции.

- база: `cs-openresty-bouncer` **v1.2.3** (ветка форка `smartcaptcha`);
- патч библиотеки: `lua-cs-bouncer` **v1.0.19** (ветка форка `smartcaptcha`), только
  `lib/plugins/crowdsec/captcha.lua`;
- образ: `smartcaptcha/openresty:1.2.3-smartcaptcha`.

Патч живёт в git-форке и попадает в образ на этапе сборки — никаких bind-mount поверх
`/usr/local/openresty/lualib/`.

## Что именно меняется в lua-cs-bouncer

| | upstream (v1.0.19) | этот форк |
|---|---|---|
| JS виджета | `api.js` провайдера | `https://smartcaptcha.cloud.yandex.ru/captcha.js` |
| класс контейнера | `<frontend_key>` | `smart-captcha` |
| поле формы с ответом | `<frontend_key>-response` | `smart-token` |
| endpoint проверки | `.../siteverify` | `https://smartcaptcha.cloud.yandex.ru/validate` |
| тело запроса | `secret`/`response`/`remoteip` | `secret`/`token`/`ip` |
| признак успеха | `success == true` | `status == "ok"` |

Токен Яндекса — непрозрачный base64, может содержать `+`, `/`, `=`, поэтому тело
собирается через `ngx.encode_args`, а не конкатенацией `k.."="..v`.

`templates/captcha.html` менять не нужно: виджет монтируется в `div`-контейнер
(он внутри `<form>`), сам кладёт hidden-input `smart-token` **внутрь контейнера**
и вызывает `data-callback` уже после заполнения input'а — ровно то, на что рассчитан
шаблон.

### Fail-open / fail-closed

Политика совпадает с upstream: сломанная капча не должна закрывать сайт всем.

| ситуация | поведение |
|---|---|
| сетевая ошибка / таймаут (2 с) | пропускаем + лог |
| пустой или нечитаемый ответ | пропускаем + лог |
| неверный/отсутствующий `SECRET_KEY` (`Authentication failed...`) | пропускаем + `ngx.ERR` |
| `status: "ok"` но `host: ""` (сбой на стороне Яндекса) | пропускаем + `ngx.ERR` |
| `status: "failed"` (робот, истёкший/повторно использованный токен) | **не** пропускаем, `ALERT Invalid captcha` |

## Сборка образа

Собирать на фронте (нужен интернет: `openresty/openresty:alpine-fat` + `luarocks
install lua-resty-http`). `BUILD_ENV=git` + свои `LUA_LIB_REPO`/`LUA_LIB_VERSION`
дают воспроизводимую сборку прямо из форков, без вложенного клона:

```bash
git clone -b smartcaptcha https://github.com/GenkaOk/cs-openresty-bouncer.git
cd cs-openresty-bouncer
docker build \
  --build-arg BUILD_ENV=git \
  --build-arg LUA_LIB_REPO=https://github.com/GenkaOk/lua-cs-bouncer.git \
  --build-arg LUA_LIB_VERSION=smartcaptcha \
  -t smartcaptcha/openresty:1.2.3-smartcaptcha .
```

Проверить, что патч реально в образе:

```bash
docker run --rm --entrypoint sh smartcaptcha/openresty:1.2.3-smartcaptcha -c \
  'grep -n "smartcaptcha" /usr/local/openresty/lualib/plugins/crowdsec/captcha.lua'
```

Репроизводимость (бэкап образа):

```bash
docker save smartcaptcha/openresty:1.2.3-smartcaptcha | gzip > /root/openresty-1.2.3-smartcaptcha.tar.gz
```

Локальная итерация по патчу (без push в форк) — вложенный клон:

```bash
git clone -b smartcaptcha https://github.com/GenkaOk/lua-cs-bouncer.git lua-cs-bouncer
docker build --build-arg BUILD_ENV=local -t smartcaptcha/openresty:dev .
```

## Деплой

`docker-compose.yaml`, сервис `nginx`:

```yaml
image: smartcaptcha/openresty:1.2.3-smartcaptcha   # было crowdsecurity/openresty:latest
```

`crowdsec/bouncers/crowdsec-openresty-bouncer.conf`:

```ini
CAPTCHA_PROVIDER=smartcaptcha
SITE_KEY=ysc1_...    # client key
SECRET_KEY=ysc2_...  # server key
```

Новых volume не нужно; существующие bind-mount'ы (`captcha.html`, `bouncers/`) остаются.

```bash
cd /opt/docker
cp -a docker-compose.yaml docker-compose.yaml.bak.$(date +%Y%m%d_%H%M%S)
cp -a crowdsec/bouncers/crowdsec-openresty-bouncer.conf crowdsec/bouncers/crowdsec-openresty-bouncer.conf.bak.$(date +%Y%m%d_%H%M%S)
docker compose up -d nginx
sleep 4
docker ps --filter name=nginx --format "{{.Names}} {{.Status}} {{.Image}}"
docker exec nginx sh -c "tail -n 20 /var/log/nginx/error.log | grep -iE 'captcha|error|emerg'"
curl -sk -o /dev/null -w "example.com -> %{http_code}\n" https://127.0.0.1/ -H "Host: example.com"
```

## Живой тест

`127.0.0.1` не подходит: `remote_addr` будет IP docker-моста. Нужен внешний адрес:

```bash
docker exec crowdsec cscli decisions add -i <ВНЕШНИЙ_IP> -t captcha -d 5m
# открыть сайт с этого IP → виджет SmartCaptcha → решить → редирект на исходный URI
docker exec crowdsec cscli decisions delete -i <ВНЕШНИЙ_IP>
```

В `access.log` успешное решение выглядит как `POST <uri>` → `302` на исходный URI.
В `error.log` при истёкшем токене будет `SmartCaptcha validation failed: Invalid or
expired Token.`.

## Ключи и домены

- `SITE_KEY` — клиентский `ysc1_...`, `SECRET_KEY` — серверный `ysc2_...` (капабильность
  ключа проверяется на стороне Яндекса, поэтому неверный ключ даёт `Authentication failed`).
- Ключи привязаны к домену: если капча обслуживает несколько доменов (`example.com`,
  `example.org`, ...) — либо отключить «проверку домена» в настройках капчи, либо
  заводить ключи на каждый домен.
- Токен одноразовый, TTL 5 минут: повторная проверка того же токена = `failed`.
  Перезагрузка страницы капчи выдаёт новый токен — это нормально.

## Открытые вопросы / риски

- Сеть до `smartcaptcha.cloud.yandex.ru` обязательна на фронте: при таймауте капча
  fail-open (пользователь пройдёт без проверки, в логе будет ошибка).
- Образ собирается из Docker Hub-базы — сборку делать заранее, не в момент аварии.
- `data-callback` у Яндекса вызывается после заполнения `smart-token`, поэтому
  существующий шаблон с `setTimeout(..., 500)` работает; задержку можно убрать, но
  она безвредна.

## Обновление форка

```bash
# внешний репо
cd cs-openresty-bouncer
git fetch upstream && git rebase upstream/v1.2.3   # или нужный тег/ветку
# библиотека
cd lua-cs-bouncer
git fetch upstream && git rebase upstream/v1.0.19  # конфликты — только в captcha.lua
# пересобрать образ с новым тегом
docker build --build-arg BUILD_ENV=git \
  --build-arg LUA_LIB_REPO=https://github.com/GenkaOk/lua-cs-bouncer.git \
  --build-arg LUA_LIB_VERSION=smartcaptcha \
  -t smartcaptcha/openresty:1.2.3-smartcaptcha.1 .
```

Проверка после rebase (без OpenResty):

```bash
cd lua-cs-bouncer && lua tests/smartcaptcha_unit.lua
```

## Откат

```bash
cd /opt/docker
cp -a docker-compose.yaml.bak.<TS> docker-compose.yaml
cp -a crowdsec/bouncers/crowdsec-openresty-bouncer.conf.bak.<TS> crowdsec/bouncers/crowdsec-openresty-bouncer.conf
docker compose up -d nginx
```

Старый образ `crowdsecurity/openresty:latest` не трогается — откат мгновенный.

## Апстрим

На 2026-10-26 в `crowdsecurity/lua-cs-bouncer` (master `92a8617`) поддержки Yandex нет,
issues/PR по `smartcaptcha`/`yandex` отсутствуют. Есть открытые PR с обобщённым
фреймворком провайдеров (ALTCHA #156, Cap.js #137) — при их мерже патч можно
переписать на новый интерфейс.
