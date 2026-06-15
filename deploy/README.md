# Деплой airona-landing (`aironaai.ru`) на droplet 209.38.247.138

## Контекст

На сервере уже работает:
- **Traefik** (контейнер `n8n-compose-traefik-1`) на портах 80/443 — рулит routing'ом и автоматически выдаёт SSL через Let's Encrypt
- **`airona-landing-web-1`** — старый вебинарный лендинг на домене `airona-ai.ru` (с дефисом). Не трогаем
- **`aiironbot-bot`** — backend API на `bot.aironaai.ru`

Мы поднимем **новый отдельный контейнер** для домена `aironaai.ru` (без дефиса), который вписывается в существующий Traefik через labels.

## Шаг 1. Настроить DNS, чтобы aironaai.ru указывал на сервер

Чтобы Traefik смог получить Let's Encrypt сертификат, домен должен резолвиться в `209.38.247.138`.

**Вариант A — быстрый (без Cloudflare):**
В панели nic.ru добавить A-записи:
- `aironaai.ru` → `209.38.247.138`
- `www.aironaai.ru` → `209.38.247.138`

Через 5–30 минут пропагация пройдёт.

**Вариант B — с Cloudflare (как изначально хотели):**
1. Cloudflare → Add site → `aironaai.ru` → Free
2. Добавить A-записи в Cloudflare:
   | Type | Name | IPv4 | Proxy |
   |---|---|---|---|
   | A | `@` | `209.38.247.138` | ⚪ DNS only |
   | A | `www` | `209.38.247.138` | ⚪ DNS only |
3. На nic.ru заменить NS-сервера на 2 от Cloudflare
4. **Важно:** оставить proxy ВЫКЛЮЧЕННЫМ (DNS only) до первого выпуска SSL — иначе Let's Encrypt не пройдёт challenge. Потом можно включить.

Проверить пропагацию:
```bash
dig aironaai.ru +short    # должен показать 209.38.247.138
```

## Шаг 2. Клонировать репо на сервер

```bash
ssh root@209.38.247.138
cd /opt
git clone https://github.com/iamnikitina/-airona-landing.git ./aironaai-landing
cd /opt/aironaai-landing
```

Обратите внимание: репо называется `-airona-landing` (с дефисом в начале — артефакт GitHub). Целевую папку называем `aironaai-landing` (без дефиса), чтобы не путать с существующей `/opt/airona-landing`.

## Шаг 3. Запустить контейнер

```bash
cd /opt/aironaai-landing
docker compose -f deploy/docker-compose.yml up -d
```

Traefik автоматически:
- увидит новый контейнер с labels
- запросит SSL у Let's Encrypt для `aironaai.ru` и `www.aironaai.ru`
- начнёт принимать трафик

Прогресс выпуска SSL:
```bash
docker logs n8n-compose-traefik-1 2>&1 | grep -i aironaai | tail -20
```

Сертификат выпускается за 30–60 секунд после первого запроса к домену.

## Шаг 4. Проверка

С локального Мака:
```bash
curl -I https://aironaai.ru/
curl -I https://www.aironaai.ru/    # должен редиректить на https://aironaai.ru/
```

В браузере: открыть `https://aironaai.ru/` — должен открыться лендинг.

## Обновление сайта потом

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
cd /opt/aironaai-landing
git pull
# nginx-контейнер ничего перезагружать не нужно — файлы примонтированы по volume,
# изменения видны сразу
```

Если меняли `deploy/docker-compose.yml` или `deploy/nginx.conf` — перезапустить:
```bash
docker compose -f deploy/docker-compose.yml restart
```

## Если что-то не так

**SSL не выпускается / 404 от Traefik:**
- Проверить что DNS уже резолвится: `dig aironaai.ru +short` с **сервера** (не с Мака — кеш может отличаться)
- Проверить что Cloudflare proxy ВЫКЛЮЧЕН (серое облачко), если используете CF
- Логи Traefik: `docker logs n8n-compose-traefik-1 --tail 50 | grep -i error`

**CORS-ошибки в браузере при попытке оплаты:**
- Попросить партнёра-бэкендера добавить в whitelist `https://aironaai.ru` и `https://www.aironaai.ru`

**Контейнер не стартует:**
```bash
docker compose -f deploy/docker-compose.yml logs --tail 50
```

## Чек-лист после деплоя

- [ ] DNS aironaai.ru → 209.38.247.138 (проверка `dig`)
- [ ] Контейнер `aironaai-landing-web-1` запущен и healthy
- [ ] Traefik получил Let's Encrypt cert (нет ошибок в логах)
- [ ] Открывается `https://aironaai.ru/`
- [ ] `https://www.aironaai.ru/` редиректит на apex
- [ ] Партнёр добавил CORS для нашего домена на backend
- [ ] Партнёр починил обработку 302 от Prodamus
