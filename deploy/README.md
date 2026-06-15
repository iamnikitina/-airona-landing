# Деплой airona-landing на droplet 209.38.247.138

## Что у нас

- Droplet с уже работающим `bot.aironaai.ru` (FastAPI/uvicorn)
- Хотим рядом поднять статичный `aironaai.ru` через nginx
- DNS переезжает с nic.ru на Cloudflare

## План

```
┌─────────────┐        ┌──────────────┐        ┌─────────────────┐
│   GitHub    │  push  │   Droplet    │  serve │  aironaai.ru    │
│  ваш репо   │ ─────► │  /var/www    │ ─────► │   (через CF)    │
└─────────────┘        └──────────────┘        └─────────────────┘
```

---

## Шаг 1. Залить код на GitHub

В этой папке (на вашем Маке) git репо уже инициализирован, первый коммит сделан. Создайте на github.com новый репозиторий (можно приватный) — назовите `airona-landing` — и не инициализируйте его README/gitignore.

Затем выполните в терминале (замените `USERNAME` на ваш логин GitHub):

```bash
cd /Users/evgeniapudokhina/projects/airona-landing
git remote add origin git@github.com:USERNAME/airona-landing.git
git push -u origin main
```

Если GitHub попросит авторизацию — настройте SSH-ключ или используйте HTTPS-вариант `https://github.com/USERNAME/airona-landing.git` (тогда нужен personal access token при push).

---

## Шаг 2. Настроить nginx на droplet

SSH на сервер:
```bash
ssh root@209.38.247.138
# или с вашим пользователем: ssh user@209.38.247.138
```

Установить (если ещё нет) git и certbot:
```bash
apt update
apt install -y nginx git certbot python3-certbot-nginx
```

Создать папку для сайта и клонировать репо:
```bash
mkdir -p /var/www
cd /var/www
git clone https://github.com/USERNAME/airona-landing.git aironaai
# Если репо приватный — нужен deploy key или PAT
```

Положить nginx-конфиг (он лежит в репо `deploy/nginx-aironaai.conf`):
```bash
cp /var/www/aironaai/deploy/nginx-aironaai.conf /etc/nginx/sites-available/aironaai.ru
ln -s /etc/nginx/sites-available/aironaai.ru /etc/nginx/sites-enabled/
nginx -t          # проверяем что синтаксис ok
systemctl reload nginx
```

**Важно:** конфиг сразу слушает 443 (HTTPS) — но сертификата ещё нет. После шага 4 certbot допишет SSL-блоки.

---

## Шаг 3. Перевести DNS на Cloudflare

1. Зайти на cloudflare.com → Sign Up → подтвердить email
2. Add a site → ввести `aironaai.ru`
3. Выбрать **Free plan**
4. Cloudflare сканирует существующие DNS-записи. Должен подхватить `bot.aironaai.ru` → 209.38.247.138 (если нет — добавим вручную)
5. **Добавить две A-записи** в Cloudflare DNS:

   | Type | Name | IPv4 | Proxy |
   |------|------|------|-------|
   | A | `@` (aironaai.ru) | `209.38.247.138` | ✅ Proxied |
   | A | `www` | `209.38.247.138` | ✅ Proxied |
   | A | `bot` | `209.38.247.138` | ❌ DNS only |

   `bot` оставляем без Cloudflare-прокси, чтобы API работал без сюрпризов (CORS, кеширование).

6. Cloudflare покажет 2 nameserver'а (типа `liz.ns.cloudflare.com` и `ivan.ns.cloudflare.com`)
7. Зайти на nic.ru → панель домена `aironaai.ru` → DNS-серверы → **заменить** на 2 от Cloudflare
8. Подождать 5 мин – 24 ч (часто пропагация за 30 мин). Cloudflare пришлёт email когда увидит обновление NS

После активации:
- Cloudflare SSL/TLS → Overview → **Full** (НЕ "Flexible"!)
- Cloudflare SSL/TLS → Edge Certificates → **Always Use HTTPS: On**

---

## Шаг 4. Выпустить SSL на droplet (Let's Encrypt)

После того как Cloudflare активен и DNS отвечает (проверка: `dig aironaai.ru` должен показывать IP Cloudflare), запустить certbot **с режимом `--cert-only`** (потому что трафик идёт через Cloudflare proxy):

Вариант A — через webroot (если Cloudflare proxy на 80 порт пропускает ACME):
```bash
# Создать папку для challenge
mkdir -p /var/www/certbot

# Временно отключить Cloudflare proxy для @ и www (поставить серое облачко в Cloudflare DNS)
# Затем:
certbot --nginx -d aironaai.ru -d www.aironaai.ru \
  --non-interactive --agree-tos -m ВАШ@EMAIL.RU

# После выпуска сертификата вернуть Cloudflare proxy (оранжевое облачко)
```

Вариант B — Cloudflare Origin Certificate (проще):
1. Cloudflare → SSL/TLS → Origin Server → Create Certificate
2. Скачать private key + certificate
3. Положить на сервер в `/etc/ssl/cloudflare/aironaai.{crt,key}`
4. В nginx-конфиге:
   ```nginx
   ssl_certificate     /etc/ssl/cloudflare/aironaai.crt;
   ssl_certificate_key /etc/ssl/cloudflare/aironaai.key;
   ```
5. `systemctl reload nginx`

Cloudflare Origin certs живут 15 лет — никаких renewals.

---

## Шаг 5. Проверка

```bash
# С локального Мака после пропагации DNS:
curl -I https://aironaai.ru/
curl -I https://www.aironaai.ru/
curl -s https://aironaai.ru/ | head -3   # должен быть наш DOCTYPE
```

В браузере: открыть `https://aironaai.ru/` → должен открыться лендинг.

---

## Шаг 6. Обновление сайта потом

Когда нужно что-то поменять:

На локалке:
```bash
cd /Users/evgeniapudokhina/projects/airona-landing
# правим файлы
git add .
git commit -m "что изменилось"
git push
```

На сервере:
```bash
ssh root@209.38.247.138
cd /var/www/aironaai
git pull
# nginx ничего перезагружать не нужно — он отдаёт файлы напрямую
```

---

## Чек-лист, что попросить у партнёра (бэкендер)

После того как `aironaai.ru` поднимется, добавить в CORS-whitelist бэкенда:
- `https://aironaai.ru`
- `https://www.aironaai.ru`

Иначе модалка оплаты будет падать с `Disallowed CORS origin`.

И починить обработку 302 от Prodamus (HTTP 502 сейчас).
