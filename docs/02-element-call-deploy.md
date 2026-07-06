# Element Call Deploy

Краткая инструкция для репозитория `element-clients`.

Пример доменов:

```text
room.example.local -> standalone Element Call
hs.example.local   -> Matrix / Synapse homeserver
core.example.local -> LiveKit service / MatrixRTC focus
```

## 1. Проверить DNS

Перед развертыванием убедиться, что DNS-записи указывают на нужные серверы:

```bash
dig +short room.example.local
dig +short hs.example.local
dig +short core.example.local
```

`room.example.local` должен указывать на сервер, где будет работать Element Call и Nginx.

## 2. Скопировать конфиг Element Call

В репозитории `element-clients`:

```bash
cd /srv/element-clients

cp config/element-call/config.example.json config/element-call/config.json
nano config/element-call/config.json
```

В файле заменить:

```text
REPLACE_WITH_MATRIX_DOMAIN -> hs.example.local
REPLACE_WITH_LIVEKIT_SERVICE_DOMAIN -> core.example.local
```

Должно получиться примерно так:

```json
{
  "default_server_config": {
    "m.homeserver": {
      "base_url": "https://hs.example.local",
      "server_name": "hs.example.local"
    }
  },

  "livekit": {
    "livekit_service_url": "https://core.example.local"
  },

  "media_devices": {
    "enable_audio": true,
    "enable_video": false
  },

  "matrix_rtc_mode": "compatibility"
}
```

Проверить JSON:

```bash
jq . config/element-call/config.json
```

## 3. Запустить Element Call

Проверить compose-конфиг:

```bash
docker compose config
```

Запустить контейнеры:

```bash
docker compose pull
docker compose up -d
```

Проверить локально:

```bash
docker compose ps
curl -I http://127.0.0.1:18082/
curl -s http://127.0.0.1:18082/config.json | jq
```

## 4. Скопировать Nginx config

Скопировать Nginx-конфиг в `conf.d`:

```bash
cp nginx/element-call.example.conf /etc/nginx/conf.d/element-call.conf
nano /etc/nginx/conf.d/element-call.conf
```

В файле заменить:

```text
REPLACE_WITH_ELEMENT_CALL_DOMAIN -> room.example.local
```

Итоговый HTTP-only конфиг до certbot:

```nginx
server {
    listen 80;
    server_name room.example.local;

    # --- Element Call ---
    location / {
        proxy_pass http://127.0.0.1:18082;

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
certbot --nginx -d room.example.local
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
curl -I http://room.example.local
curl -I https://room.example.local
```

Проверить, что Element Call отдает нужный config:

```bash
curl -s https://room.example.local/config.json | jq
```

Проверить важные поля:

```bash
curl -s https://room.example.local/config.json | jq '.default_server_config'
curl -s https://room.example.local/config.json | jq '.livekit'
curl -s https://room.example.local/config.json | jq '.media_devices'
```

Проверить контейнер:

```bash
docker compose ps
docker compose logs -f element-call
```

После этого открыть в браузере:

```text
https://room.example.local
```

## Важно

`media_devices.enable_video: false` означает, что камера выключена по умолчанию в standalone Element Call.

`org.matrix.msc4143.rtc_foci` в этот config не добавляется. Он должен оставаться в `.well-known/matrix/client` на `hs.example.local`.
