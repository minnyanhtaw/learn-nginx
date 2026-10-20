# 01 - Introduction to Nginx

## What is Nginx?

Nginx (pronounced "engine-x") is a high-performance **web server**, **reverse proxy**, **load balancer** and **HTTP cache**. It was created by Igor Sysoev in 2004 to solve the "C10K problem": how to handle 10,000+ concurrent connections on one machine.

Today it powers a large share of the busiest sites on the internet.

## What can Nginx do?

| Role | What it means |
|------|---------------|
| Web server | Serves static files (HTML, CSS, JS, images) very fast |
| Reverse proxy | Sits in front of your app (Node, Python, PHP, Java) and forwards requests to it |
| Load balancer | Spreads traffic across several backend servers |
| HTTP cache | Stores responses so the backend does less work |
| TLS terminator | Handles HTTPS so your app does not have to |
| API gateway (basic) | Rate limiting, auth, routing by path |

## Why is it fast? The architecture

Apache's classic model uses one process or thread per connection. Nginx uses an **event-driven, asynchronous, non-blocking** model.

```
                +------------------+
                |   Master process |   reads config, manages workers
                +--------+---------+
                         |
        +----------------+----------------+
        |                |                |
  +-----v-----+    +-----v-----+    +-----v-----+
  | Worker 1  |    | Worker 2  |    | Worker N  |
  | event loop|    | event loop|    | event loop|
  +-----------+    +-----------+    +-----------+
   thousands of connections handled by each worker
```

- **Master process**: reads the configuration, binds to ports, starts and supervises workers. Runs as root so it can bind to ports 80/443.
- **Worker processes**: do the real work. Each one runs an event loop (epoll on Linux) and handles thousands of connections without blocking.
- Typically `worker_processes auto;` creates one worker per CPU core.

Result: low memory use, high concurrency, predictable performance.

## Request workflow (high level)

1. Client opens a TCP connection to the server (port 80 or 443).
2. A worker accepts the connection.
3. Nginx reads the request line and headers.
4. Nginx picks a **server block** (by port and `Host` header).
5. Nginx picks a **location block** (by URI).
6. The location decides what to do: serve a file, proxy to a backend, redirect, or return a fixed response.
7. The response goes back to the client and the event is logged.

See the animated workflow page that comes with this guide for a visual version.

## Configuration structure

Nginx is configured with plain text files. The main file is `/etc/nginx/nginx.conf`.

```nginx
# Global context
user  www-data;
worker_processes  auto;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    server {
        listen 80;
        server_name example.com;

        location / {
            root /var/www/html;
            index index.html;
        }
    }
}
```

Key terms:

- **Directive**: a single setting ending with `;` (for example `listen 80;`).
- **Context / block**: a group of directives inside `{ }` (`events`, `http`, `server`, `location`, `upstream`).
- Inheritance: inner blocks inherit from outer blocks unless they override.

```
main
 |-- events
 |-- http
      |-- upstream
      |-- server
           |-- location
```

## Nginx vs Apache (short version)

| | Nginx | Apache |
|---|---|---|
| Model | Event-driven | Process/thread per connection (prefork/worker) or event MPM |
| Static files | Excellent | Good |
| Per-directory config | No `.htaccess` | `.htaccess` supported |
| Dynamic content | Passes to external app (PHP-FPM, uWSGI, Node) | Can embed modules (mod_php) |
| Typical use | Front door, proxy, static, load balancing | Traditional hosting with `.htaccess` |

They are often combined: Nginx in front, Apache behind.

## Next

Continue with [02 - Installation](02-installation.md).