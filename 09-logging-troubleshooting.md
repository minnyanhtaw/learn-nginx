# 09 - Logging and Troubleshooting

## Log files

| File | Content |
|------|---------|
| `/var/log/nginx/access.log` | One line per request |
| `/var/log/nginx/error.log` | Problems, config errors, upstream failures |

```nginx
access_log /var/log/nginx/example.access.log;
error_log  /var/log/nginx/example.error.log warn;
```

Error log levels, from most to least verbose: `debug`, `info`, `notice`, `warn`, `error`, `crit`, `alert`, `emerg`.

## Default log format

```
203.0.113.5 - - [20/Oct/2026:12:20:01 +0000] "GET /index.html HTTP/1.1" 200 612 "-" "Mozilla/5.0"
```

Fields: client IP, user, time, request line, status, bytes sent, referrer, user agent.

## Custom log format with timing

```nginx
log_format timed '$remote_addr [$time_local] "$request" '
                 '$status $body_bytes_sent '
                 'rt=$request_time urt=$upstream_response_time '
                 'cache=$upstream_cache_status';

access_log /var/log/nginx/access.log timed;
```

- `$request_time`: total time to handle the request.
- `$upstream_response_time`: time spent waiting for the backend.

JSON format (good for ELK, Loki, Datadog):

```nginx
log_format json escape=json
  '{"time":"$time_iso8601","ip":"$remote_addr","method":"$request_method",'
  '"uri":"$request_uri","status":$status,"bytes":$body_bytes_sent,'
  '"rt":$request_time,"ua":"$http_user_agent"}';
```

## Log rotation

Package installs include `/etc/logrotate.d/nginx`. Test it with:

```bash
sudo logrotate -d /etc/logrotate.d/nginx
```

## Handy log analysis one-liners

```bash
# Live view
sudo tail -f /var/log/nginx/access.log

# Top 10 client IPs
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Status code counts
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Most requested URLs
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# All 5xx responses
awk '$9 ~ /^5/' /var/log/nginx/access.log
```

For visual analysis, try GoAccess:

```bash
sudo apt install goaccess
goaccess /var/log/nginx/access.log --log-format=COMBINED
```

## Status page (stub_status)

```nginx
location = /nginx_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```

```bash
curl http://127.0.0.1/nginx_status
```

Shows active connections, accepted, handled and total requests. Prometheus users can add `nginx-prometheus-exporter`.

## Troubleshooting checklist

1. **Syntax**: `sudo nginx -t`
2. **Is it running?** `systemctl status nginx`
3. **Is the port open?** `sudo ss -tlnp | grep nginx`
4. **Error log**: `sudo tail -n 50 /var/log/nginx/error.log`
5. **Which config is actually loaded?** `sudo nginx -T | less`
6. **Test locally**: `curl -I http://localhost`, `curl -H "Host: example.com" http://127.0.0.1`
7. **Firewall and DNS** reachable from outside

## Common errors and fixes

| Error | Typical cause and fix |
|-------|----------------------|
| `bind() to 0.0.0.0:80 failed (98: Address already in use)` | Another service (Apache?) uses port 80. Stop it or change the port |
| `403 Forbidden` | Wrong file permissions, missing index file, or `autoindex off` on a directory |
| `404 Not Found` | Wrong `root`/`alias`, or missing `try_files` for SPA routes |
| `502 Bad Gateway` | Backend down or wrong address. Check the app and the socket path |
| `504 Gateway Timeout` | Backend slow. Raise `proxy_read_timeout` or fix the app |
| `413 Request Entity Too Large` | Raise `client_max_body_size` |
| `too many open files` | Raise `worker_rlimit_nofile` and system limits |
| `nginx: [emerg] unknown directive` | Typo or missing module |
| `upstream sent too big header` | Increase `proxy_buffer_size` |
| Changes not visible | Forgot `reload`, or the browser/CDN cache |

## Permissions quick fix

```bash
sudo chown -R www-data:www-data /var/www/mysite
sudo find /var/www/mysite -type d -exec chmod 755 {} \;
sudo find /var/www/mysite -type f -exec chmod 644 {} \;
```

The whole path to the file must be traversable (execute bit) by the Nginx user.

## Debug logging

```nginx
error_log /var/log/nginx/debug.log debug;
```