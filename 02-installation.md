# 02 - Installation

## Ubuntu / Debian

```bash
sudo apt update
sudo apt install nginx -y
```

Start and enable at boot:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

Open the firewall (if UFW is active):

```bash
sudo ufw allow 'Nginx Full'     # ports 80 and 443
sudo ufw status
```

## CentOS / RHEL / Rocky / AlmaLinux

```bash
sudo dnf install nginx -y
sudo systemctl enable --now nginx

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## macOS (Homebrew)

```bash
brew install nginx
brew services start nginx
```

Config lives in `/opt/homebrew/etc/nginx/` on Apple Silicon (`/usr/local/etc/nginx/` on Intel).

## Docker

```bash
docker run -d --name my-nginx -p 8080:80 nginx:stable
```

Mount your own config and content:

```bash
docker run -d --name my-nginx -p 8080:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:stable
```

## Verify it works

```bash
curl -I http://localhost
```

You should see `HTTP/1.1 200 OK` and a `Server: nginx` header. In a browser, opening `http://your-server-ip` shows the "Welcome to nginx!" page.

## Important files and folders (Debian/Ubuntu)

| Path | Purpose |
|------|---------|
| `/etc/nginx/nginx.conf` | Main configuration |
| `/etc/nginx/sites-available/` | Site configs (all of them) |
| `/etc/nginx/sites-enabled/` | Symlinks to the enabled sites |
| `/etc/nginx/conf.d/*.conf` | Extra config snippets, auto-included |
| `/var/www/html` | Default web root |
| `/var/log/nginx/access.log` | Access log |
| `/var/log/nginx/error.log` | Error log |

On RHEL-family systems there is no `sites-available`; use `/etc/nginx/conf.d/`, and the default root is `/usr/share/nginx/html`.

## Essential commands

```bash
sudo nginx -t                 # test configuration syntax
sudo nginx -T                 # test and dump the full config
sudo systemctl reload nginx   # apply config with zero downtime
sudo systemctl restart nginx  # full restart (drops connections)
sudo systemctl stop nginx
nginx -v                      # version
nginx -V                      # version and compile options
```

**Golden rule:** always run `nginx -t` before `reload`.

## Stable vs mainline

- **Stable**: only critical bug fixes, recommended for production.
- **Mainline**: newest features, also considered reliable.

The official repository at nginx.org usually has newer packages than your distribution.

## Next

Continue with [03 - HTTP](03-http.md).