# 08 - Security

## Hide the version number

```nginx
http {
    server_tokens off;
}
```

## Security headers

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
```

Test your Content-Security-Policy carefully; a strict policy can break scripts, fonts or images from other domains.

Note: `add_header` in an inner block replaces headers inherited from outer blocks, so repeat them where needed (or use an `include` snippet).

## Rate limiting

Limit requests per client IP to slow down brute force and abusive bots.

```nginx
# http context: 10 requests per second per IP, 10 MB state
limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;

server {
    location / {
        limit_req zone=perip burst=20 nodelay;
        limit_req_status 429;
        proxy_pass http://backend;
    }

    location /login {
        limit_req zone=perip burst=5;
    }
}
```

- `rate`: sustained rate.
- `burst`: queue of extra requests allowed above the rate.
- `nodelay`: process burst requests immediately instead of spacing them out.

## Limit connections

```nginx
limit_conn_zone $binary_remote_addr zone=conn_perip:10m;

server {
    limit_conn conn_perip 20;
}
```

## Allow or deny by IP

```nginx
location /admin/ {
    allow 203.0.113.10;
    allow 10.0.0.0/8;
    deny  all;
}
```

Rules are checked in order; the first match wins.

## Block bad requests

```nginx
# Block hidden files, except .well-known (needed by Let's Encrypt)
location ~ /\.(?!well-known) {
    deny all;
}

# Block unwanted user agents
if ($http_user_agent ~* (sqlmap|nikto|masscan)) {
    return 403;
}

# Allow only some HTTP methods
if ($request_method !~ ^(GET|HEAD|POST)$) {
    return 405;
}
```

Use `if` sparingly; inside `location` it has surprising behavior. Simple `return` statements as above are the safe use.

## Limit request size and timeouts

```nginx
client_max_body_size 10m;
client_body_timeout 12s;
client_header_timeout 12s;
send_timeout 10s;
large_client_header_buffers 4 8k;
```

These reduce the effect of slow-connection attacks (slowloris).

## Restrict TLS

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_session_tickets off;
```

Disable SSLv3, TLS 1.0 and 1.1.

## Run with least privilege

- Workers run as an unprivileged user (`user www-data;` or `user nginx;`).
- Keep web content readable but not writable by that user where possible.
- Keep private keys readable only by root (`chmod 600`).
- Keep Nginx updated.

## Use fail2ban for repeated abuse

Fail2ban can read `/var/log/nginx/error.log` or `access.log` and ban IPs through the firewall. The default `nginx-http-auth` and `nginx-limit-req` filters work out of the box:

```bash
sudo apt install fail2ban
```

## Web Application Firewall

ModSecurity with the OWASP Core Rule Set can be added as a module, or place a cloud WAF/CDN in front of Nginx.

## Next

Continue with [09 - Logging and Troubleshooting](09-logging-troubleshooting.md).