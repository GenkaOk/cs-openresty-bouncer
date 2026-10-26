<p align="center">
<img src="https://github.com/crowdsecurity/cs-openresty-bouncer/raw/main/docs/assets/crowdsec_openresty.png" alt="CrowdSec" title="CrowdSec" width="280" height="300" />
</p>
<p align="center">
<img src="https://img.shields.io/badge/build-pass-green">
<img src="https://img.shields.io/badge/tests-pass-green">
</p>
<p align="center">
&#x1F4DA; <a href="#installation/">Documentation</a>
&#x1F4A0; <a href="https://hub.crowdsec.net">Hub</a>
&#128172; <a href="https://discourse.crowdsec.net">Discourse </a>
</p>



# CrowdSec OpenResty Bouncer

A lua bouncer for OpenResty.

## How does it work ?

This bouncer leverages OpenResty lua's API, namely `access_by_lua_file`.

New/unknown IP are checked against crowdsec API, and if request should be blocked, a **403** is returned to the user, and put in cache.

# Installation

Please follow the [official documentation](https://doc.crowdsec.net/docs/bouncers/openresty).

## This fork

Fork of the upstream bouncer based on **v1.2.3** that adds **Yandex SmartCaptcha**
(`CAPTCHA_PROVIDER=smartcaptcha`). The actual patch lives in the forked
[lua-cs-bouncer](https://github.com/GenkaOk/lua-cs-bouncer) (`captcha.lua`), this
repository only gained a `LUA_LIB_REPO`/`LUA_LIB_VERSION` build argument so the image
can be built straight from the forks.

```bash
docker build \
  --build-arg BUILD_ENV=git \
  --build-arg LUA_LIB_REPO=https://github.com/GenkaOk/lua-cs-bouncer.git \
  --build-arg LUA_LIB_VERSION=smartcaptcha \
  -t smartcaptcha/openresty:1.2.3-smartcaptcha .
```

Build, deployment, verification, update and rollback procedure:
[docs/smartcaptcha.md](docs/smartcaptcha.md) (Russian).
