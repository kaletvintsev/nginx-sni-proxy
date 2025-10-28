# Nginx SSL Proxy with Docker

Простой и надежный SSL прокси-сервер с маршрутизацией по SNI на базе чистого Nginx. Никаких дополнительных скриптов — только конфигурационные файлы.

## Возможности

✅ Единый файл с маршрутами (домен → IP)  
✅ SSL passthrough без расшифровки трафика  
✅ Поддержка Let's Encrypt (ACME challenge)  
✅ Автоматический редирект HTTP → HTTPS  
✅ Простое добавление и удаление доменов  
✅ Передача реального IP клиента (PROXY Protocol)  
✅ Автоматический перезапуск при сбоях

## Быстрый старт

### 1. Настройте маршруты

Создайте и отредактируйте файл `routes.conf`:
```nginx
# Формат: домен IP_адрес_backend;
example.com          192.168.1.100;
api.example.com      192.168.1.101;
shop.example.com     192.168.1.102;
```

### 2. Запустите контейнер

```bash
docker compose up -d
```

Готово! Прокси запущен и работает на портах 80 (HTTP) и 443 (HTTPS).

## Основные команды

### Просмотр логов
```bash
docker compose logs -f nginx-proxy
```

### Добавление или изменение домена

1. Отредактируйте `routes.conf`
2. Примените изменения без остановки:
```bash
docker compose exec nginx-proxy nginx -s reload
```

### Проверка конфигурации
```bash
docker compose exec nginx-proxy nginx -t
```

### Перезапуск контейнера
```bash
docker compose restart nginx-proxy
```

### Остановка прокси
```bash
docker compose down
```

## Как это работает

Прокси маршрутизирует трафик на основе доменного имени:

1. **routes.conf** содержит простую таблицу маршрутизации: домен → IP backend-сервера
2. Nginx использует этот файл дважды:
   - Для **HTTPS** (порт 443): читает SNI из SSL handshake через `ssl_preread`
   - Для **HTTP** (порт 80): использует заголовок `Host`
3. Трафик проксируется на указанный IP с соответствующим портом (443 или 80)

### Порты

- **80** — HTTP (обработка ACME challenge и редирект на HTTPS)
- **443** — HTTPS (SSL passthrough с сохранением шифрования)

### Структура проекта

```
.
├── docker-compose.yml     # Конфигурация Docker
├── nginx.conf.template    # Шаблон Nginx с переменными окружения
├── routes.conf            # ⭐ Файл маршрутизации (домен → IP)
└── README.md              # Документация
```

## Передача реального IP клиента

По умолчанию backend-серверы видят IP прокси, а не реальный IP клиента.

### HTTP-трафик (порт 80)

**Уже настроено!** Прокси автоматически добавляет заголовки:
- `X-Real-IP` — реальный IP клиента
- `X-Forwarded-For` — цепочка прокси
- `X-Forwarded-Proto` — протокол (http/https)

Backend просто читает эти заголовки из входящего запроса.

### HTTPS-трафик (порт 443)

Для зашифрованного трафика используется **PROXY Protocol** — специальный протокол передачи метаданных клиента перед SSL handshake.


### PROXY Protocol

PROXY Protocol передает информацию о клиенте в начале TCP-соединения (до SSL handshake):

```
PROXY TCP4 1.2.3.4 10.0.0.1 54321 443\r\n
<далее следует SSL/TLS трафик>
```

#### Включение PROXY Protocol

**Шаг 1: Включите на прокси-сервере**

Создайте файл `.env` рядом с `docker-compose.yml`:
```bash
PROXY_PROTOCOL_PARAM=on
```

Или установите переменную при запуске:
```bash
PROXY_PROTOCOL_PARAM=on docker compose up -d
```

**⚠️ ВНИМАНИЕ:** Без настройки backend-серверов сайты **перестанут работать**!

**Шаг 2: Настройте backend-серверы**

#### Nginx на backend

```nginx
server {
    listen 443 ssl proxy_protocol;  # добавьте proxy_protocol
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # Доверяем IP прокси-сервера
    set_real_ip_from 172.16.0.0/12;  # Docker networks
    set_real_ip_from 192.168.0.0/16; # Private networks
    set_real_ip_from 10.0.0.0/8;
    real_ip_header proxy_protocol;
    
    location / {
        # $remote_addr теперь содержит реальный IP клиента
        proxy_pass http://upstream;
    }
}
```

#### Apache на backend

```apache
<VirtualHost *:443>
    # Требуется mod_remoteip
    RemoteIPProxyProtocol On
    
    RemoteIPInternalProxy 172.16.0.0/12
    RemoteIPInternalProxy 192.168.0.0/16
    RemoteIPInternalProxy 10.0.0.0/8
    
    SSLEngine on
    SSLCertificateFile /path/to/cert.pem
    SSLCertificateKeyFile /path/to/key.pem
    
    # REMOTE_ADDR теперь содержит реальный IP
</VirtualHost>
```

#### Caddy на backend

```caddy
{
    servers {
        listener_wrappers {
            proxy_protocol {
                timeout 5s
                allow 172.16.0.0/12 192.168.0.0/16 10.0.0.0/8
            }
        }
    }
}

example.com {
    tls /path/to/cert.pem /path/to/key.pem
    reverse_proxy upstream:8080
}
```

**Шаг 3: Перезапустите прокси**

```bash
docker compose down
docker compose up -d
docker compose logs -f nginx-proxy
```

В логах должна появиться информация об успешном старте Nginx.

