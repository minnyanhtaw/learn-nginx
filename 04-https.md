# 04 - HTTPS (TLS/SSL)

HTTPS encrypts traffic between the browser and your server. Nginx can terminate TLS, so your backend apps can stay on plain HTTP internally.

## How the TLS handshake works (simplified)

```
Client                                  Nginx
  |  ---- ClientHello (TLS versions, ciphers) ---->  |
  |  <--- ServerHello + Certificate ---------------  |
  |  ---- verify certificate, key exchange -------->  |
  |  <--- Finished ---------------------------------  |
  |  ======== encrypted HTTP traffic ==============   |
```

The certificate proves your identity; it is signed by a Certificate Authority (CA) the browser trusts.

## Option 1: Free certificate with Let's Encrypt (recommended)

Prerequisites: a domain name pointing to your server, ports 80 and 443 open.

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot will:

1. Prove you control the domain (HTTP-01 challenge).
2. Download the certificate to `/etc/letsencrypt/live/example.com/`.
3. Edit your Nginx config to use it.
4. Optionally add an HTTP to HTTPS redirect.

Test automatic renewal (certificates last 90 days):

```bash
sudo certbot renew --dry-run
systemctl list-timers | grep certbot
```

## Option 2: Manual HTTPS configuration

```nginx
# Redirect all HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    ssl_stapling on;
    ssl_stapling_verify on;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;

    root /var/www/example;
    index index.html;
}
```

Notes:

- Use `fullchain.pem` (certificate plus intermediates), not just `cert.pem`.
- `http2 on;` is the syntax for recent Nginx versions (1.25.1+). On older versions use `listen 443 ssl http2;`.
- HSTS tells browsers to always use HTTPS. Only enable it once you are sure HTTPS works everywhere.

## Self-signed certificate (development only)

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/selfsigned.key \
  -out /etc/ssl/certs/selfsigned.crt \
  -subj "/CN=localhost"
```

Browsers will show a warning because no trusted CA signed it.

## TLS termination in front of an app

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

`X-Forwarded-Proto` lets the app know the original request was HTTPS.

## Testing your setup

```bash
curl -I https://example.com
openssl s_client -connect example.com:443 -servername example.com
```

Online: SSL Labs Server Test (ssllabs.com/ssltest).

## Common problems

| Symptom | Likely cause |
|---------|--------------|
| `ERR_CERT_COMMON_NAME_INVALID` | Certificate does not include that domain |
| `ERR_SSL_PROTOCOL_ERROR` | Missing `ssl` on the `listen` line |
| Certbot challenge fails | Port 80 blocked or DNS not pointing to this server |
| Mixed content warnings | Page loads `http://` resources; switch to `https://` or relative URLs |
| Certificate expired | Renewal timer not running; check `certbot renew` |

## Next

Continue with [05 - Reverse Proxy](05-reverse-proxy.md).