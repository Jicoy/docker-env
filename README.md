# docker-env

One script that turns a fresh Laravel app into a running Docker stack with
Filament and Shield already set up.

You need Docker. Nothing else has to be on your machine.

## How to use it

Make a Laravel project first:

```sh
composer create-project laravel/laravel my-app
cd my-app
```

Then run the script inside it:

```sh
curl -fL -O https://raw.githubusercontent.com/Jicoy/docker-env/main/install.sh
chmod +x install.sh
./install.sh
```

That's the whole thing. Go make coffee, the first run takes a few minutes.

When it finishes, open **http://localhost/admin** and log in:

```
admin@example.com
password
```

You don't have to run migrations, generate a key, seed anything or build assets.
It already happened.

## What it sets up

| | |
|---|---|
| http://localhost/admin | Filament panel |
| http://localhost:8080 | Adminer, for poking at the database |
| http://localhost:5173 | Vite. You don't visit this, the app uses it for hot reload |

Containers: PHP 8.5-FPM, nginx, Postgres 16, Redis, Adminer, Vite.

Four logins, all with the password `password`:

| Email | Role |
|---|---|
| admin@example.com | `super_admin`, can do everything |
| requester@example.com | `requester` |
| custodian@example.com | `custodian` |
| approver@example.com | `approver` |

The last three start with no permissions. Give them some in the panel under
**Roles**, or rename them in `database/seeders/ShieldRoleSeeder.php`.

## What it actually does

1. Writes `.docker/` and `docker-compose.yml`
2. Adds env defaults, Vite config, seeders, and the Shield wiring on your `User`
   model
3. Builds the image and waits for Postgres
4. Installs Filament 5 and Shield, scaffolds the admin panel
5. Migrates, generates permissions and policies, seeds the four logins
6. `npm install` and `npm run build`
7. Hands `storage/` and `bootstrap/cache/` to `www-data` at 775

Options:

```sh
./install.sh --no-up    # write the config but don't start containers
./install.sh --force    # overwrite files it would otherwise leave alone
```

Run it as many times as you like. It skips whatever is already done, and
anything it replaces is copied to `.docker-setup-backup/` first.

## Day to day

Edit files normally. CSS and JS hot-reload, PHP shows up on the next request.

Run `artisan`, `composer` and `npm` through the container:

```sh
docker compose exec php php artisan make:model Item -m
docker compose exec php composer require some/package
docker compose exec php npm install some-package
```

```sh
docker compose logs -f setup   # what the bootstrap did
docker compose logs -f php     # Laravel errors
docker compose down            # stop
docker compose down -v         # stop and wipe the database
```

## If something breaks

**`line 1: 404:: command not found`** — the download failed and curl saved the
error page as the script. Use `-fL` like the command above, don't drop the `-f`.

**Port 80 is already in use.** Add this to `.env`, then `docker compose up -d`:

```
APP_PORT=8000
```

Same for `ADMINER_PORT`, `VITE_PORT`, `FORWARD_DB_PORT`, `FORWARD_REDIS_PORT`.

**502 or the site won't load.** The bootstrap failed. `docker compose logs setup`
and look near the bottom.

**Clean slate.** `docker compose down -v` then `docker compose up -d --build`.
This deletes the database.

**Changed `vite.config.js` and nothing happened.** `docker compose restart vite`.

## Notes

- A one-shot `setup` container does the bootstrap and exits. `php`, `nginx` and
  `vite` wait for it, which is why the first page you load already works.
- Database and Redis settings come from `docker-compose.yml`, not `.env`.
  Laravel won't override a real environment variable, so editing `DB_HOST` in
  `.env` does nothing. Passwords and ports do come from `.env`.
- `node_modules` lives in a Docker volume, not your folder. Rollup ships
  platform-specific binaries and a macOS build won't run inside Linux.
- Web fonts aren't downloaded during the build. Laravel's starter fetches
  Instrument Sans from fonts.bunny.net, which fails the whole setup on a
  restricted network. `vite.config.js` shows how to turn it back on.
- Container names are `hack-sims-*`. Change them in `docker-compose.yml` if you
  want, or if you need two of these running side by side.
