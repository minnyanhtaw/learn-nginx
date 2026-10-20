# 03 - Serving HTTP

This chapter covers the core of Nginx: `server` blocks and `location` blocks.

## A minimal static website

1. Create the content:

```bash
sudo mkdir -p /var/www/mysite
echo "<h1>Hello from mysite</h1>" | sudo tee /var/www/mysite/index.html
sudo chown -R www-data:www-data /var/www/mysite
```

2. Create `/etc/nginx/sites-available/mysite`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name mysite.com www.mysite.com;

    root /var/www/mysite;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

3. Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## How Nginx picks a server block

For each request Nginx checks:

1. **IP and port** match the `listen` directive.
2. **Host header** is compared with `server_name`.
3. If nothing matches, the **default server** is used (the first one, or the one marked `default_server`).

```nginx
server {
    listen 80 default_server;
    server_name _;
    return 444;          # close the connection for unknown hosts
}
```

`server_name` match order: exact name, then leading wildcard (`*.example.com`), then trailing wildcard, then regex, then default.

## Location blocks

A `location` matches the request URI.

| Syntax | Meaning | Priority |
|--------|---------|----------|
| `location = /path` | Exact match | 1 (highest) |
| `location ^~ /path` | Prefix match, stop searching regex | 2 |
| `location ~ regex` | Case-sensitive regex | 3 |
| `location ~* regex` | Case-insensitive regex | 3 |
| `location /path` | Plain prefix match (longest wins) | 4 |

Example:

```nginx
location = / {
    # only the exact URL /
}

location /images/ {
    root /data;                  # /images/a.png -> /data/images/a.png
}

location ~* \.(jpg|jpeg|png|gif|css|js)$ {
    expires 30d;                 # cache static assets
    add_header Cache-Control "public";
}

location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

## root vs alias

```nginx
location /static/ {
    root /var/www/app;       # /static/a.css -> /var/www/app/static/a.css
}

location /files/ {
    alias /srv/downloads/;   # /files/a.zip  -> /srv/downloads/a.zip
}
```

`root` appends the URI; `alias` replaces the matched part. With `alias`, keep trailing slashes consistent.

## try_files

Checks files in order, uses the last item as fallback:

```nginx
# Single Page App (React, Vue, Angular)
location / {
    try_files $uri $uri/ /index.html;
}
```

## Redirects and rewrites

```nginx
# Permanent redirect
return 301 https://$host$request_uri;

# Redirect old path
location = /old-page {
    return 301 /new-page;
}

# Rewrite internally
rewrite ^/blog/(.*)$ /articles/$1 last;
```

Prefer `return` over `rewrite` when possible; it is simpler and faster.

## Custom error pages

```nginx
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

location = /50x.html {
    root /usr/share/nginx/html;
    internal;
}
```

## Directory listing

```nginx
location /downloads/ {
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
}
```

## Basic authentication

```bash
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

```nginx
location /admin/ {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

## Useful variables

| Variable | Value |
|----------|-------|
| `$host` | Host header |
| `$uri` | Normalized request path |
| `$request_uri` | Original URI with query string |
| `$args` | Query string |
| `$remote_addr` | Client IP |
| `$scheme` | `http` or `https` |
| `$request_method` | GET, POST, ... |
| `$status` | Response status code |

## Next

Continue with [04 - HTTPS](04-https.md).