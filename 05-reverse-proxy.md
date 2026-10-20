# 05 - Reverse Proxy

A **reverse proxy** receives requests on behalf of one or more backend servers and forwards them. Clients only talk to Nginx and never see the backend directly.

```
Browser  --->  Nginx (:80/:443)  --->  App (127.0.0.1:3000)
         <---                    <---
```

Why use one?

- Hide and protect internal servers
- Terminate HTTPS in one place
- Serve static files directly, send only dynamic requests to the app
- Run many apps on one server under different domains or paths
- Add caching, compression, rate limiting

## Basic proxy_pass

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Without these headers, the backend only sees requests coming from `127.0.0.1` with host `127.0.0.1:3000`.

## The trailing slash matters

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;     # /api/users -> /api/users
}

location /api/ {
    proxy_pass http://127.0.0.1:3000/;    # /api/users -> /users
}
```

A URI part in `proxy_pass` (even just `/`) replaces the matched location prefix.

## Static files from Nginx, API from the app

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:4000/;
    }
}
```

## WebSocket support

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    ...
    location /ws/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;
    }
}
```

The `map` block goes in the `http` context (for example in `conf.d`).

## Timeouts and buffering

```nginx
proxy_connect_timeout 5s;
proxy_send_timeout    60s;
proxy_read_timeout    60s;

proxy_buffering on;
proxy_buffers 16 16k;
client_max_body_size 20m;     # allow larger uploads (default is 1m)
```

For streaming or server-sent events, turn buffering off: `proxy_buffering off;`.

## PHP with FastCGI (PHP-FPM)

```nginx
location ~ \.php$ {
    include fastcgi_params;
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

Adjust the socket path to your PHP version.

## Other protocols

| Directive | Backend |
|-----------|---------|
| `proxy_pass` | HTTP apps (Node, Go, Java, Python) |
| `fastcgi_pass` | PHP-FPM |
| `uwsgi_pass` | uWSGI (Python) |
| `grpc_pass` | gRPC services |

For Python you can also use Gunicorn behind `proxy_pass`.

## Multiple apps on one server

```nginx
server {
    server_name blog.example.com;
    location / { proxy_pass http://127.0.0.1:2368; }
}

server {
    server_name shop.example.com;
    location / { proxy_pass http://127.0.0.1:8000; }
}
```

## Troubleshooting 502 / 504

| Error | Meaning | Check |
|-------|---------|-------|
| 502 Bad Gateway | Backend refused or sent an invalid reply | Is the app running? Right port/socket? |
| 504 Gateway Timeout | Backend too slow | Increase `proxy_read_timeout`, check app performance |
| 413 Request Entity Too Large | Upload bigger than limit | Raise `client_max_body_size` |

On SELinux systems, a 502 can also come from blocked network connections: `sudo setsebool -P httpd_can_network_connect 1`.

## Next

Continue with [06 - Load Balancing](06-load-balancing.md).