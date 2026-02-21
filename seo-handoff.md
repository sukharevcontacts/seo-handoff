````markdown
# ✅ ИТОГ ТЕКУЩЕГО ЭТАПА — SEO ДЛЯ SPA (ПОЛНЫЙ HANDOFF v3)

---

# 📦 Архитектура проекта

## Front

* Чистый SPA (HTML + JS)
* Главный шаблон: `templates/webapp.html`
* SEO управляется динамически через `seo.js`
* Deep routing реализован через `navigation.js`
* Canonical, Title, Description обновляются через JS
* H1 управляется SEO-движком

---

## Backend

* Python (uvicorn)
* Отдаёт:
  * HTML shell
  * API
  * sitemap.xml
* Реализован fallback выбора магазина по региону
* Исправлена логика Product SEO при отсутствии cookie

---

## Reverse Proxy

* Отдельная машина: `srv-revproxy`
* Nginx выполняет:
  * TLS termination
  * маршрутизацию
  * prerender routing
  * bot detection
  * SEO routing
  * редиректы доменов
  * нормализацию HEAD-запросов
  * контроль X-Prerendered
  * таймаут-стабилизацию

---

# 🌐 URL Архитектура

Региональная структура:

```
/region/:region/
/region/:region/category/:slug
/region/:region/category/:slug/:subslug
/region/:region/product/:id
/region/:region/farmer/:id
```

Особенности:

* Без hash
* Без query мусора
* Canonical = чистый путь
* Единый https домен
* Prerender только для GET

---

# 🖥 Инфраструктура

## Reverse Proxy

* nginx
* docker
* docker-compose
* prerender контейнер

## Backend адреса

```
10.0.44.11:8030  ← основной прод backend
10.0.44.15:8010  ← другой сервис
10.0.44.16:7006  ← старый тестовый backend
```

---

# 🌍 Домены

## Основной домен

```
https://koptorg.ru
https://www.koptorg.ru
```

Nginx:

```
/etc/nginx/conf.d/koptorg.ru.conf
```

Routing:

```
80  → redirect HTTPS
443 → proxy_pass http://10.0.44.11:8030
```

---

## Старый домен

```
коопторгъ.рф
xn--c1ammaafio9e.xn--p1ai
```

### Статус:

* Делает редирект на koptorg.ru
* Требует отдельный SSL сертификат
* Ошибка была: NET::ERR_CERT_COMMON_NAME_INVALID
* Причина: отдавался сертификат koptorg.ru
* Нужно: отдельный server block + отдельный cert + 301 redirect

---

# 🔥 Prerender

Контейнер:

```
Image: tvanro/prerender-alpine
Port: 127.0.0.1:3000
```

Проверка:

```
curl http://127.0.0.1:3000/render?url=https://koptorg.ru/region/nso
```

Работает.

---

## Интеграция в nginx

### Исправленная production-конфигурация

В `http {}`:

```
map $request_method $upstream_method {
    default $request_method;
    HEAD    GET;
}
```

В `location /`:

```
location / {

    # HEAD не должен уходить в prerender
    if ($request_method = HEAD) { set $do_prerender_main 0; }

    proxy_method $upstream_method;

    proxy_connect_timeout 30s;
    proxy_send_timeout    120s;
    proxy_read_timeout    120s;
    send_timeout          120s;

    proxy_pass $proxy_target_main;

    proxy_hide_header X-Prerendered;
    add_header X-Prerendered $hdr_prerendered always;
}
```

---

## Исправленные проблемы (16 февраля)

### Было

* HEAD → backend = 405
* HEAD → prerender = 400
* Дубли X-Prerendered
* В Яндекс.Вебмастере = 504 Gateway Timeout
* Prerender иногда давал 499

### Стало

* HEAD проксируется как GET к backend
* HEAD никогда не уходит в prerender
* X-Prerendered ровно один
* 504 устранены
* 499 сведены к редким ретраям
* Массовый прогон 20 URL под ботом = 200

---

# 🧠 SEO Реализация

## SEO-движок (seo.js)

Генерация:

* Title
* Description
* Canonical
* H1

Источники:

* Category registry
* Product data
* Region
* Farmer

---

## Проверка через prerender (реально для бота)

```
curl -A YandexBot https://koptorg.ru/region/nso/product/3255
curl -A YandexBot https://koptorg.ru/region/nso/farmer/188233
```

Проверено:

* Title корректный
* Description корректный
* Canonical совпадает
* Нет "Каталог" вместо товара

---

## Meta Description

Формируется:

```
description
+ composition
+ регион
+ служебный хвост
```

Ограничение: 160 символов.

---

## Canonical

```
<link rel="canonical">
```

Логика:

```
location.origin + location.pathname
```

---

## Category Registry

Источник:

```
static/classifier/categories_registry.json
```

---

## Debug

В консоли доступны:

```
window.SEO.debugLastInput
window.__SEO_LAST_PRODUCT
window.__SEO_LAST_CATEGORY
window.__SEO_LAST_FARMER
```

---

# 🔧 Критический фикс Product SEO

## Проблема

Для:

```
/region/nso/product/:id
```

Бот получал:

* Title: Каталог
* Canonical: category

Причина:

* Нет cookie selected_store
* SPA не выбирает магазин
* Включался fallback

---

## Решение — Backend Fallback

Логика:

```
если cookie нет
→ взять регион из URL
→ назначить default store
→ передать во frontend
```

Результат:

* Бот получает корректный title
* Canonical указывает на product
* Description корректный
* Prerender возвращает правильный HTML

---

# 📄 Sitemap

```
https://koptorg.ru/sitemap.xml
```

Исправлено:

* Удалён `/region/nso/`
* Оставлен `/region/nso/category/`
* Убран порт :7026

Статус:

* В очереди обработки
* Переобход запущен вручную

---

# 🤖 Яндекс Вебмастер

## Инцидент 16 февраля

* 823 URL
* Статус: 504 Gateway Timeout
* Причина: неправильная обработка HEAD + prerender

## Решение

* Исправлен nginx
* Убран дубль X-Prerendered
* Настроен proxy_method
* Добавлены таймауты
* Проведён повторный переобход
* Новых 504 в логах нет

---

## Подтверждение прав

Метатег добавлен в:

```
templates/webapp.html
```

Подтверждено.

---

## Обход через Метрику

* Счётчик привязан
* Обход включён
* Ускоряет crawl

---

# 🌍 Google Search Console

* Планируется / подключается
* Sitemap будет добавлен

---

# 🔒 SSL

## koptorg.ru

* Сертификат GlobalSign
* SNI корректный

## коопторгъ.рф

* Требуется отдельный SSL
* Нужен server block

---

# 🗂 GitHub Хранение ТЗ

Репозиторий:

```
https://github.com/sukharevcontacts/seo-handoff
```

Документ:

```
seo-handoff.md
```

На сервере:

```
/home/pyuser/seo-handoff/
```

Обновление:

```
git pull
```

---
---

# 🆕 Дополнительные SEO-фиксы (февраль 2026)

## 🧹 Причина работ
В Яндекс.Вебмастере появились «странные» URL в обходе с параметрами `token` (и др.), например:

- `/auth/telegram/callback?token=...`
- `/region/.../category/?token=...`

Цель — убрать дубли и мусор из обхода, снизить нагрузку, сохранить чистый canonical-граф.

---

## 🤖 robots.txt (backend)

### Что сделано
`robots.txt` обновлён: добавлены запреты на служебные URL и token-параметры.

Текущее содержимое:

```
User-agent: *
Allow: /

Disallow: /auth/
Disallow: /*?token=
Disallow: /*&token=

Sitemap: https://koptorg.ru/sitemap.xml
```

Проверка:

```
curl -s "https://koptorg.ru/robots.txt"
```

---

## 🌐 Nginx: нормализация query-параметров (301 → чистый URL)

### Token cleanup (основной фикс)
Любой URL с `?token=...` режется в 301 на чистый путь, кроме callback, где token нужен для логина.

```
# token=... -> canonical (except /auth/telegram/callback)
set $strip_token 0;
if ($arg_token != "") { set $strip_token 1; }
if ($uri = "/auth/telegram/callback") { set $strip_token 0; }
if ($strip_token = 1) {
    return 301 $scheme://$host$uri;
}
```

Проверка:

```
curl -I "https://koptorg.ru/region/nso/product/3255?token=abc"
```

Ожидаемо: `301 Location: https://koptorg.ru/region/nso/product/3255`

---

### Marketing junk cleanup (utm/gclid/yclid/fbclid)
Также нормализуются «маркетинговые» параметры (чтобы не плодить дубли):

- `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content`
- `gclid`, `yclid`, `fbclid`

```
set $strip_junk 0;
if ($arg_utm_source   != "") { set $strip_junk 1; }
if ($arg_utm_medium   != "") { set $strip_junk 1; }
if ($arg_utm_campaign != "") { set $strip_junk 1; }
if ($arg_utm_term     != "") { set $strip_junk 1; }
if ($arg_utm_content  != "") { set $strip_junk 1; }
if ($arg_gclid        != "") { set $strip_junk 1; }
if ($arg_yclid        != "") { set $strip_junk 1; }
if ($arg_fbclid       != "") { set $strip_junk 1; }

if ($strip_junk = 1) {
    return 301 $scheme://$host$uri;
}
```

Проверка:

```
curl -I "https://koptorg.ru/region/nso/product/3255?utm_source=test"
curl -I "https://koptorg.ru/region/nso/product/3255?gclid=123"
curl -I "https://koptorg.ru/region/nso/product/3255?fbclid=abc"
```

Ожидаемо: везде `301` на чистый URL без параметров.

---

## 🔐 Nginx: защита /auth/* и TG callback

### /auth/ (слэш) → канонический /auth (HTTPS)
Чтобы не было редиректов на http и лишних вариантов URL:

```
location = /auth/ {
    return 301 https://$host/auth;
}
```

Проверка:

```
curl -I "https://koptorg.ru/auth/"
```

---

### /auth/*: noindex + no-store + мимо prerender
На уровне nginx:

- `X-Robots-Tag: noindex, nofollow`
- `Cache-Control: no-store`
- исключение из prerender (через отдельный `location ^~ /auth/`)

Пример:

```
location ^~ /auth/ {
    add_header X-Robots-Tag "noindex, nofollow" always;
    add_header Cache-Control "no-store" always;
    proxy_pass $proxy_target_main;
}
```

Проверка:

```
curl -I "https://koptorg.ru/auth/"
```

---

### /auth/telegram/callback: HEAD → 200 (убран 405 в обходе)
Проблема: `curl -I` / HEAD давал `405 Allow: GET`.

Решение: отдельный `location` — HEAD отвечает `200`, GET идёт в backend, всё под `noindex`.

```
location = /auth/telegram/callback {
    if ($request_method = HEAD) { return 200; }

    add_header X-Robots-Tag "noindex, nofollow" always;
    add_header Cache-Control "no-store" always;

    proxy_pass $proxy_target_main;
}
```

Проверка:

```
curl -I "https://koptorg.ru/auth/telegram/callback?token=abc"
curl -X GET -I "https://koptorg.ru/auth/telegram/callback?token=abc"
```

Ожидаемо: `HEAD -> 200`, `GET -> 200/302/…` (по логике backend), но **не 405**, и всегда `X-Robots-Tag: noindex, nofollow`.

---

## ✅ Prerender: финальная проверка (важно: GET, не HEAD)

Важно: `curl -I` делает HEAD, а HEAD намеренно **не** уходит в prerender (чтобы не повторить инцидент 504/400).

Правильная проверка prerender (GET):

```
curl -s -o /dev/null -D - -A "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)"   "https://koptorg.ru/region/nso/product/3255" | sed -n '1,30p'
```

Ожидаемо: `X-Prerendered: 1`.

Проверка, что параметры режутся до prerender и для бота:

```
curl -I -A YandexBot "https://koptorg.ru/region/nso/product/3255?utm_source=test"
curl -I -A YandexBot "https://koptorg.ru/region/nso/product/3255?token=abc"
```

Ожидаемо: `301` на чистый URL.

---

# 📊 SEO Готовность

URL структура — ✅  
Title — ✅  
Description — ✅  
Canonical — ✅  
Registry — ✅  
Prerender — ✅  
Product SEO — ✅  
Fallback — ✅  
Sitemap — ✅  
Robots — ✅  
Метрика — ✅  
Обход через счётчик — ✅  
SSL основной домен — ✅  
HEAD обработка — ✅  
504 устранены — ✅  
\1Token/UTM нормализация — ✅  
Auth (noindex + HEAD fix) — ✅  
\2  

---

# ⭐ Оценка

~99% технической готовности к индексации

---

# 🚀 Следующие шаги

1. Schema.org (Product, Organization, Breadcrumb)
2. Усиление внутренней перелинковки
3. Crawl budget оптимизация
4. Внешние сигналы
5. SSL коопторгъ.рф
6. Google SEO запуск

---

# 📌 Текущее состояние

SPA + Dynamic Rendering (Prerender)  
Стабильная инфраструктура  
Корректная индексация  
Ошибки 405 / 400 / 504 устранены  
Система готова к масштабированию
````

