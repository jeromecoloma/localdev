# localdev

Shared local development infrastructure for PHP/Laravel projects running under Docker and Traefik.

## Directory structure

```
localdev/
├── traefik/                  # Reverse proxy (HTTP/HTTPS, TLS, routing)
├── mysql-server/             # Shared MySQL 8 instance
└── generate-localdev-php     # Scaffold script for new projects
```

## Prerequisites

- Docker with the `devnet` external network
- Traefik and MySQL stacks running (see below)

### Create the shared network (once)

```bash
docker network create devnet
```

### Start shared infrastructure

```bash
docker compose -f ~/Codes/localdev/traefik/compose.yml up -d
docker compose -f ~/Codes/localdev/mysql-server/compose.yml up -d
```

Traefik dashboard is available at `http://localhost:8080`.

---

## Scaffolding a new project

### Usage

```bash
cd /path/to/new-project
~/Codes/localdev/generate-localdev-php
```

The script prompts for a few values then writes four files into the current directory.

### Prompts

| Prompt | Options | Default |
|--------|---------|---------|
| Project type | `laravel` / `php` | `laravel` |
| PHP version | `7.4` / `8.1` / `8.4` | `8.4` |
| Web root | any path, or `.` for project root | `public` (plain PHP only) |
| App name (slug) | lowercase, digits, hyphens | — |
| Domain | | — |

**Project type** accepts shorthand: `l` for Laravel, `p` for plain PHP.

**PHP version** is selected from a numbered menu:
```
PHP version:
  1) 7.4
  2) 8.1
  3) 8.4 (default)
Choice [3]:
```

**Web root** is only prompted for plain PHP projects. Use `.` to serve from the project root (`/var/www/html`).

### Generated files

```
your-project/
├── compose.yml               # Root wrapper — includes .localdev/compose.yml
└── .localdev/
    ├── compose.yml           # Service definitions
    ├── Dockerfile            # PHP-FPM image with Composer
    └── nginx.conf            # Nginx → php-fpm via FastCGI
```

#### compose.yml (root)

A two-line wrapper that delegates to `.localdev/compose.yml` via Docker Compose `include`. Keeps the project root clean.

#### .localdev/compose.yml

**Laravel** defines three services:

| Service | Image | Role |
|---------|-------|------|
| `node` | node:22-alpine | Runs `npm install && npm run build` once, then exits |
| `web` | nginx:1.27-alpine | Serves static assets, proxies PHP to `{app}-php` |
| `{app}-php` | Custom Dockerfile | PHP-FPM; runs migrations on start |

`web` and `{app}-php` both wait for `node` to complete successfully — this prevents the Vite manifest race condition.

**Plain PHP** defines two services (no `node`):

| Service | Image | Role |
|---------|-------|------|
| `web` | nginx:1.27-alpine | Proxies PHP to `{app}-php` |
| `{app}-php` | Custom Dockerfile | PHP-FPM |

Traefik labels on `web` expose the app on both HTTP and HTTPS:

```
traefik.http.routers.{app}.rule=Host(`{domain}`)        # HTTP
traefik.http.routers.{app}s.rule=Host(`{domain}`)       # HTTPS (TLS)
traefik.http.services.{app}.loadbalancer.server.port=80
```

#### .localdev/Dockerfile

Builds a PHP-FPM image with the extensions most projects need (`pdo_mysql`, `mbstring`, `zip`, `exif`, `pcntl`, `sockets`) and Composer.

**Composer image tag:**
- PHP 7.4 → `composer:2.2` (LTS, supports PHP 7.x)
- PHP 8.1 / 8.4 → `composer:2`

**Laravel `CMD`:**
1. Runs `composer install`
2. Creates `database/database.sqlite` if missing
3. Fixes ownership and permissions on `storage/`, `bootstrap/cache/`, and `database/`
4. Runs `php artisan migrate --force`
5. Starts `php-fpm`

**Plain PHP `CMD`:** starts `php-fpm` directly.

#### .localdev/nginx.conf

Standard nginx config forwarding PHP requests to `{app}-php:9000` via a dynamic `$upstream` variable (required for Docker DNS resolution). The `root` directive points to the resolved web root. The `X-Site` header identifies the app in logs.

### Expected project layout

**Laravel:**
```
your-project/
├── compose.yml
├── .localdev/
└── www/               # Laravel application root
    ├── .env.example
    ├── .env.local      # ← your local env (not committed)
    ├── public/
    ├── storage/
    └── ...
```

**Plain PHP:**
```
your-project/
├── compose.yml
├── .localdev/
└── www/               # PHP application root
    ├── public/        # (or whichever web root you specified)
    └── ...
```

The `.localdev` services mount `../www` as `/var/www/html`.

### Starting the stack

**Laravel:**
```bash
cp www/.env.example www/.env.local   # configure DB, APP_KEY, etc.
docker compose up -d
```

**Plain PHP:**
```bash
docker compose up -d
```

### Rebuilding after dependency changes

```bash
docker compose build {app}-php   # after composer.json changes
docker compose up -d
```

### App name validation

The script enforces lowercase alphanumeric + hyphens (no spaces, no leading hyphen). The name is used as:
- Docker Compose stack name
- Traefik router and service names
- PHP-FPM service name (`{app}-php`)
- Nginx upstream and `X-Site` header
