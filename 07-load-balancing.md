# 06 - Load Balancing

A load balancer distributes incoming requests across several backend servers to improve capacity and availability.

```
                      +--> App server 1 (10.0.0.11:3000)
Clients --> Nginx ----+--> App server 2 (10.0.0.12:3000)
                      +--> App server 3 (10.0.0.13:3000)
```

## Basic upstream

Define a group of servers in the `http` context, then proxy to it by name.

```nginx
upstream backend {
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
    server 10.0.0.13:3000;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

The default method is **round robin**: requests go to each server in turn.

## Balancing methods

| Directive | How it chooses | Good for |
|-----------|----------------|----------|
| (default) | Round robin | Equal servers, short requests |
| `least_conn;` | Server with fewest active connections | Long or uneven requests |
| `ip_hash;` | Hash of client IP, same client goes to same server | Simple sticky sessions |
| `hash $request_uri consistent;` | Hash of any key | Cache servers |
| `random two least_conn;` | Pick two at random, choose the less loaded | Large clusters |

```nginx
upstream backend {
    least_conn;
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
}
```

## Weights, backups and health behavior

```nginx
upstream backend {
    server 10.0.0.11:3000 weight=3;                    # gets 3x the traffic
    server 10.0.0.12:3000 weight=1;
    server 10.0.0.13:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.14:3000 backup;                      # used only if others are down
    server 10.0.0.15:3000 down;                        # temporarily disabled
}
```

- `max_fails` and `fail_timeout`: after N failed attempts within the time window, the server is marked unavailable for `fail_timeout`.
- This is **passive** health checking: Nginx notices failures from real traffic. Active health checks (`health_check`) are available in the commercial Nginx Plus.

## Retry on failure

```nginx
location / {
    proxy_pass http://backend;
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries 2;
}
```

Careful: retrying non-idempotent requests (POST) can cause duplicate actions. By default Nginx does not retry POST after the request was sent.

## Keepalive connections to backends

Reuse connections to reduce overhead:

```nginx
upstream backend {
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
    keepalive 32;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

## Zero-downtime deployments

1. Mark one server `down` and reload Nginx.
2. Deploy to that server and test it.
3. Re-enable it, then repeat for the next server.

`systemctl reload nginx` starts new workers with the new config while old workers finish existing requests.

## TCP/UDP load balancing (stream module)

For non-HTTP services such as databases or DNS:

```nginx
stream {
    upstream mysql_read {
        server 10.0.1.21:3306;
        server 10.0.1.22:3306;
    }

    server {
        listen 3307;
        proxy_pass mysql_read;
    }
}
```

The `stream` block sits at the top level, next to `http`.

## Session persistence

Apps with in-memory sessions need the same user to reach the same server. Options:

1. `ip_hash;` (simple, but users behind one NAT share a server)
2. Store sessions in Redis or a database (best; makes servers stateless)
3. Sticky cookies (Nginx Plus, or a third-party module)

## Next

Continue with [07 - Caching and Compression](07-caching-compression.md).