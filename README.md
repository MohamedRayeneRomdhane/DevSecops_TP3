# DevSecOps TP3

A small PHP app used to practice a DevSecOps pipeline (CI, linting, security checks, Docker, deployment).

The app does one thing: it **generates a PNG image** showing some text and the current date/time.

---

## What's inside

| Path | What it is |
|---|---|
| `public/index.php` | Entry point. Returns the generated image. |
| `public/info.php` | Shows `phpinfo()` (debug only, see [Security notes](#security-notes)). |
| `public/font/consolas.ttf` | Font used to draw the text. |
| `src/ImageCreator.php` | The class that draws the image. |
| `tests/` | PHPUnit tests (`Unit` and `Feature` suites). |
| `Docker/Dockerfile` | Docker image (PHP 8.2 + Apache). |
| `.circleci/config.yml` | CI/CD pipeline. |
| `phpcs.xml` | Code style rules (PSR-12 + PHP compatibility). |
| `.env.example` | Example environment variables. |

---

## Requirements

- PHP **8.2+** with the **GD** extension
- [Composer](https://getcomposer.org/)
- Docker (optional)

---

## Run it locally

```bash
# 1. Install dependencies
composer install

# 2. Create your env file
cp .env.example .env

# 3. Start the PHP built-in server
php -S localhost:8000 -t public
```

Open <http://localhost:8000> and you should see the image.

### Customize the image

Edit the variables at the top of `public/index.php`:

```php
$yourColor  = [128, 128, 128];       // border color (RGB)
$yourColor2 = [60, 80, 57];          // background color (RGB)
$yourText   = "DEVOPS";              // big text (date is added automatically)
$yourText2  = "Une superbe image";   // small text
```

### Environment variables

| Variable | Description |
|---|---|
| `APP_SECRET` | Optional. If set, it is printed on the image. Used to show how secrets reach the app. |

---

## Run with Docker

```bash
docker build -f Docker/Dockerfile -t devsecops-tp3 .
docker run --rm -p 8080:80 devsecops-tp3
```

Open <http://localhost:8080>.

> `vendor/` is excluded from the image (see `.dockerignore`). Run `composer install` inside the container, or mount `vendor/` as a volume.

---

## Tests and code quality

```bash
# Run unit tests
./vendor/bin/phpunit --testsuite=Unit

# Run all tests
./vendor/bin/phpunit

# Check code style (needs phpcs + PHPCompatibility installed)
./vendor/bin/phpcs --standard=phpcs.xml --ignore=vendor/ .
```

---

## CI/CD pipeline (CircleCI)

There are two workflows.

### 1. `main_workflow`

```
build-setup ──┬── lint-phpcs
              ├── security-check-dependencies
              └── test-phpunit

hold (manual approval) ──┬── deploy-ssh-production   (main / master)
                         └── deploy-ssh-staging      (release/*)
```

| Job | What it does |
|---|---|
| `debug-info` | Prints environment info (for debugging the CI). |
| `build-setup` | Installs Composer dependencies and caches them. |
| `lint-phpcs` | Checks code style. Report saved as `phpcs-report.txt`. |
| `security-check-dependencies` | Scans `composer.lock` for known vulnerabilities. Report saved as `security-report.json`. |
| `test-phpunit` | Runs the `Unit` test suite. |
| `hold` | Waits for someone to click **Approve** in CircleCI. |
| `deploy-ssh-staging` | SSH into the staging server, `git pull`, `composer install`, restart PHP-FPM. |

### 2. `container_workflow`

| Job | What it does |
|---|---|
| `build-docker-image` | Builds the Docker image and pushes it to GitHub Container Registry (`ghcr.io/<user>/<repo>:<branch>`). |

Runs on `main`, `master`, `develop`, `feature/*`, `release/*`, `hotfix/*`, `bugfix/*`.

### CircleCI environment variables to set

| Variable | Used by |
|---|---|
| `GHCR_USERNAME` | Docker push login |
| `GHCR_PAT` | Docker push login (GitHub token with `write:packages`) |
| `STAGING_SSH_FINGERPRINT` | SSH key for staging |
| `STAGING_SSH_USER` | SSH user on staging |
| `STAGING_SSH_HOST` | Staging server address |
| `STAGING_DEPLOY_DIRECTORY` | Project folder on the staging server |

---

## Known issues / TODO

- **`deploy-ssh-production` is not defined.** It is used in `main_workflow` but has no job definition, so CircleCI will reject the config until it is added (or removed from the workflow).
- **Dockerfile path case.** CI uses `docker/Dockerfile` but the folder is `Docker/`. This works on Windows/macOS but fails on Linux (the CI runner).
- **`lint-phpcs` never fails.** Style errors (exit codes 1 and 2) are turned into success. Good for reporting, but it won't block bad code.
- **`phpunit.xml` coverage source** points to `./app`, but the code lives in `./src`.

---

## Security notes

- `public/info.php` exposes full server configuration. Remove it or block it outside of development.
- `APP_SECRET` is drawn on the image. That's intentional for the exercise, but never do this with a real secret.
- Never commit `.env` (it is already in `.gitignore`).
