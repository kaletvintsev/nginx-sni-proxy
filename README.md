# Nginx SSL Proxy with Docker

Простой SSL proxy с маршрутизацией по SNI. Чистый nginx без дополнительных скриптов.

## Структура проекта

```
.
├── docker-compose.yml     # Docker конфигурация
├── nginx.conf.template    # Шаблон конфига nginx (с переменными окружения)
├── routes.conf            # ⭐ МАРШРУТЫ домен→IP (файл нужно создать и сконфигурировать)
└── README.md              # Эта инструкция
```

## Быстрый старт

### 1. Создайте и настройте маршруты
Создайте `routes.conf`:
```bash
touch routes.conf
```

Откройте и добавьте ваши маршруты:
```nginx
# Формат: домен IP_сервера;
your-domain.com          192.168.1.100;
another-domain.com       192.168.1.200;

default                  192.168.1.100;
```

### 2. Запустите
```bash
docker-compose up -d
```

## Управление

### Добавить/изменить домен
1. Отредактируйте `routes.conf`
2. Перезагрузите nginx:
```bash
docker-compose exec nginx-proxy nginx -s reload
```

### Или перезапустите контейнер:
```bash
docker-compose restart nginx-proxy
```

### Просмотр логов
```bash
docker-compose logs -f nginx-proxy
```

### Проверка конфигурации
```bash
docker-compose exec nginx-proxy nginx -t
```

### Остановка
```bash
docker-compose down
```

## Как это работает

- **routes.conf** - единственный файл с маршрутами (домен → IP)
- Nginx читает его через `include` в двух местах:
  - `map $ssl_preread_server_name` - для HTTPS (порт 443)
  - `map $host` - для HTTP (порт 80)
- Трафик проксируется на указанный IP с добавлением порта (:443 или :80)

## Порты

- **80** - HTTP (ACME challenge + redirect to HTTPS)
- **443** - HTTPS (SSL passthrough по SNI)

## Особенности

✅ Один файл маршрутов (`routes.conf`)  
✅ Чистый nginx, никаких скриптов  
✅ SSL passthrough без расшифровки  
✅ Поддержка Let's Encrypt (ACME challenge)  
✅ Автоматический редирект HTTP → HTTPS  
✅ Легко добавлять/удалять домены  
✅ Опциональная передача реального IP клиента (PROXY Protocol)

---

## Передача реального IP клиента на backend-серверы

По умолчанию backend-серверы видят IP прокси-сервера вместо реального IP клиента.

### HTTP трафик (порт 80)

**Уже настроено!** Прокси автоматически добавляет HTTP заголовки:
- `X-Real-IP` - реальный IP клиента
- `X-Forwarded-For` - цепочка прокси-серверов
- `X-Forwarded-Proto` - протокол (http/https)

Ваш backend просто читает эти заголовки из запроса.

### HTTPS трафик (порт 443) - PROXY Protocol

Для SSL passthrough нужен **PROXY Protocol**, так как трафик зашифрован и HTTP заголовки добавить невозможно.

#### Что такое PROXY Protocol?

Протокол передает информацию о клиенте в начале TCP соединения (до SSL handshake):
```
PROXY TCP4 1.2.3.4 10.0.0.1 54321 443\r\n
<далее идет SSL/TLS трафик>
```

#### Как включить PROXY Protocol

**Шаг 1:** Включите PROXY Protocol через переменную `PROXY_PROTOCOL_PARAM=on`


**Шаг 2:** Настройте backend-серверы на прием PROXY Protocol (Ваших приложений на который будете перенаправлять трафик)

**⚠️ КРИТИЧЕСКИ ВАЖНО!** Без настройки backend сайты **перестанут работать**!

#### Настройка Nginx на backend:

```nginx
server {
    listen 443 ssl proxy_protocol;  # ← добавьте proxy_protocol
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # Доверяем прокси-серверам (укажите IP вашего прокси)
    set_real_ip_from 172.16.0.0/12;  # Docker networks
    set_real_ip_from 192.168.0.0/16; # Локальные сети
    set_real_ip_from 10.0.0.0/8;     # Или конкретный IP прокси
    real_ip_header proxy_protocol;
    
    location / {
        # Теперь $remote_addr содержит реальный IP клиента
        access_log /var/log/nginx/access.log;
    }
}
```

#### Настройка Apache на backend:

```apache
<VirtualHost *:443>
    # Включаем PROXY Protocol (требуется mod_remoteip)
    RemoteIPProxyProtocol On
    
    # Доверяем прокси-серверам
    RemoteIPInternalProxy 172.16.0.0/12
    RemoteIPInternalProxy 192.168.0.0/16
    RemoteIPInternalProxy 10.0.0.0/8
    
    SSLEngine on
    SSLCertificateFile /path/to/cert.pem
    SSLCertificateKeyFile /path/to/key.pem
    
    # Теперь REMOTE_ADDR содержит реальный IP
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

#### Настройка Caddy на backend:

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

your-domain.com {
    tls /path/to/cert.pem /path/to/key.pem
    
    # Реальный IP доступен автоматически
    log {
        output file /var/log/caddy/access.log
    }
}
```

