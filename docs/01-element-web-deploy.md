# Element Web Deploy

Краткая инструкция для репозитория `element-clients`.

Пример доменов:

```text
web.example.local  -> Element Web
hs.example.local   -> Matrix / Synapse homeserver
```

## 1. Проверить DNS

Перед развертыванием убедиться, что DNS-записи указывают на нужный сервер:

```bash
dig +short web.example.local
dig +short hs.example.local
```

`web.example.local` должен указывать на сервер, где будет работать Element Web и Nginx.

## 2. Скопировать конфиг Element Web

В репозитории `element-clients`:

```bash
cd /srv/element-clients

cp config/element-web/config.example.json config/element-web/config.json
nano config/element-web/config.json
```

В файле заменить:

```text
REPLACE_WITH_MATRIX_DOMAIN -> hs.example.local
```

Должно получиться примерно так:

```json
{
  "default_server_name": "hs.example.local",

  "default_server_config": {
    "m.homeserver": {
      "base_url": "https://hs.example.local",
      "server_name": "hs.example.local"
    }
  }
}
```

Проверить JSON:

```bash
jq . config/element-web/config.json
```

## 3. Запустить Element Web

Проверить compose-конфиг:

```bash
docker compose config
```

Запустить контейнер:

```bash
docker compose pull
docker compose up -d
```

Проверить локально:

```bash
docker compose ps
curl -I http://127.0.0.1:18081/
curl -s http://127.0.0.1:18081/config.json | jq
```

## 4. Скопировать Nginx config

Скопировать Nginx-конфиг в conf.d:

```bash
cp nginx/element-web.example.conf /etc/nginx/conf.d/element-web.conf
nano /etc/nginx/conf.d/element-web.conf
```

В файле заменить:

```text
REPLACE_WITH_ELEMENT_WEB_DOMAIN -> web.example.local
```

Итоговый HTTP-only конфиг до certbot:

```nginx
server {
    listen 80;
    server_name web.example.local;

    # --- Element Web ---
    location / {
        proxy_pass http://127.0.0.1:18081;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Проверить и перезагрузить Nginx:

```bash
nginx -t
systemctl reload nginx
```

## 5. Создать HTTPS-сертификат

```bash
certbot --nginx -d web.example.local
```

После этого certbot сам добавит HTTPS-блок и редирект на HTTPS.

Проверить Nginx еще раз:

```bash
nginx -t
systemctl reload nginx
```

## 6. Проверка после развертывания

Проверить HTTP/HTTPS:

```bash
curl -I http://web.example.local
curl -I https://web.example.local
```

Проверить, что Element Web отдает нужный config:

```bash
curl -s https://web.example.local/config.json | jq
```

Проверить, что в конфиге есть нужный homeserver:

```bash
curl -s https://web.example.local/config.json | jq '.default_server_config'
```

Проверить контейнер:

```bash
docker compose ps
docker compose logs -f element-web
```

После этого в звонках должен использоваться новый Element Call / MatrixRTC при наличии настроенного LiveKit bridge в `.well-known/matrix/client` на `hs.example.local`.
