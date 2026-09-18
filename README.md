# Hack Sims — Supplies and Warehouse Management

Laravel 13 + Filament 5 + Filament Shield. Everything runs in Docker, so you
don't have to install PHP, Postgres, Node or anything else on your machine.

## Setup

You need Docker Desktop. That's it.

```sh
git clone <repo-url> hack-sims-app
cd hack-sims-app
docker compose up -d --build
```

Go make coffee. The first run takes a few minutes because it builds the PHP
image and pulls every dependency. After that it's about 30 seconds.

When it's done, open **http://localhost/admin** and log in:

```
admin@example.com
password
```

That's the whole setup. You don't need to run migrations, generate a key, seed
anything, or build assets. It already happened.

## What you get

| Where | What |
|---|---|
| http://localhost/admin | The Filament panel |
| http://localhost:8080 | Adminer, for poking at the database |
| http://localhost:5173 | Vite. You don't visit this, the app uses it for hot reload |

Four logins, all with the password `password`:

| Email | Role |
|---|---|
| admin@example.com | `super_admin`, can do everything |
| requester@example.com | Requesting Unit |
| custodian@example.com | Warehouse Staff |
| approver@example.com | Supply Officer |

The last three start with no permissions. Give them some in the panel under
**Roles**.

## Working on it

Edit files normally. Blade, CSS and JS hot-reload on save. PHP changes show up
on the next request.

Anything you'd normally type as `php artisan`, `composer` or `npm`, run through
the container instead:

```sh
docker compose exec php php artisan make:model Item -m
docker compose exec php composer require some/package
docker compose exec php npm install some-package
```

Don't bother running `npm` on your own machine. The container keeps
`node_modules` in a Docker volume and never looks at yours, so anything you
install locally is just ignored.

Useful bits:

```sh
docker compose logs -f setup   # see what the bootstrap did
docker compose logs -f php     # Laravel errors land here
docker compose down            # stop everything
docker compose down -v         # stop and wipe the database
```

Re-running `docker compose up -d --build` is always safe. It won't duplicate
your data or reset your database.

## Something broke

**Port 80 is already in use.** Put this in `.env` and run `docker compose up -d`
again:

```
APP_PORT=8000
```

Same idea for `ADMINER_PORT`, `VITE_PORT`, `FORWARD_DB_PORT` and
`FORWARD_REDIS_PORT` if those clash too.

**The site won't load / you get a 502.** The bootstrap probably failed. Look at
`docker compose logs setup` — the error will be near the bottom.

**You want a clean slate.** `docker compose down -v` then
`docker compose up -d --build`. This deletes the database.

**Changed `vite.config.js` and nothing happened.** `docker compose restart vite`.

## Starting a fresh project instead

If you're setting this up on a brand new Laravel app rather than cloning this
repo, `install.sh` does the whole thing:

```sh
composer create-project laravel/laravel my-app
cd my-app
curl -O https://raw.githubusercontent.com/<user>/<repo>/main/install.sh
chmod +x install.sh
./install.sh
```

It writes the Docker config, installs Filament and Shield, and starts the stack.
Run it again any time and it'll skip whatever is already done. Files it replaces
get copied to `.docker-setup-backup/` first.

Flags: `--no-up` writes the config but doesn't start containers, `--force`
overwrites files it would otherwise leave alone.

## Notes for the curious

- The `setup` container does the bootstrap and exits. `php`, `nginx` and `vite`
  wait for it to finish, which is why the first page you load already works.
- Database settings live in `docker-compose.yml`, not `.env`. Laravel won't
  override a real environment variable, so editing `DB_HOST` in `.env` does
  nothing. Passwords and ports do come from `.env` though.
- `node_modules` lives in a Docker volume, not your folder. Rollup ships
  platform-specific binaries and a macOS build won't run inside Linux.
- Web fonts aren't downloaded during the build. The Laravel starter fetches
  Instrument Sans from fonts.bunny.net, which fails the whole setup on a
  restricted network. `vite.config.js` shows how to turn it back on.
