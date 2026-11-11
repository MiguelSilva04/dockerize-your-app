# FastAPI full-stack: containerization proof

Work in progress. A hands-on exercise: take a real, widely used open-source app, containerize it properly and measure the difference, to back up a Docker/containerization service.

## Upstream

Based on [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) (MIT), commit `b1125d8839482cb7bbae603ddc6d0081bf87532e` from 2026-10-15, copied on 2026-10-23. The original copyright and license are kept in `LICENSE`. The original README is `README.upstream.md`. Everything not listed under "Changes" is the upstream authors' work.

## Changes

Each change was measured before and after. Raw numbers and method are in [`docs/medicoes.md`](docs/medicoes.md), which is in Portuguese for now.

| # | Change | Result |
| --- | --- | --- |
| 1 | `--no-dev` on both `uv sync` calls | Dependency layer 319 MB to 142 MB. 14 fewer Python packages, including pytest, mypy and ruff. |
| 2 | `FROM python:3.14-slim` | Image 1.88 GB to 439 MB on disk. Same Python 3.14.9. Login through the API still returns 200. |
| 3 | Healthcheck uses Python's `urllib` instead of `curl` | `curl` is not in the slim image, and the old check failed 5 times in a row. The new one passes, and exits with code 1 when nothing listens on the port. |
| 4 | Non-root user (`uid` 10001) | Image size unchanged (439 MB, the new layer is 41 kB). The image's `User` is the number `10001:10001`. Migrations, healthcheck and login verified as the new user, and writing to the app code is denied. |
| 5 | Multi-stage build for the backend | `uv` stays in the build stage. 442 MB to 362 MB on disk and 106 MB to 82.8 MB compressed, measured against a one-stage build on the same base. Migrations, healthcheck and login verified. |
| 6 | Root `.dockerignore` extended | Build context 2568 kB to 1484 kB (42% smaller). The image is identical: both builds report the same config digest. `backend/.dockerignore` is not read, because the build context is the repo root. |

The original image is 2.11 GB on disk and the current one is 362 MB. Changes 1 and 2 share one commit but were measured in separate builds. The `python:3.14-slim` tag moved from Python 3.14.9 to 3.14.10 between changes 4 and 5, so change 5 is compared against a one-stage image rebuilt on the new base. It moved again while change 6 was being measured, which is why change 6 is checked by comparing two builds on the same base.

Idle memory ranged from 476 to 505 MiB across all runs, for both the original and the current image. The difference between them (about 2% at 6 minutes) is smaller than the spread between samples of the same image, so no memory benefit is claimed.

## Tags

- `baseline`: the repository before any containerization change. The Dockerfile and `compose.yml` are the upstream ones.
- `slim-base`: after changes 1 to 3.
- `non-root-user`: after change 4.
- `multi-stage`: after change 5.
- `dockerignore`: after change 6.

Compare: [`baseline` to `slim-base`](https://github.com/MiguelSilva04/dockerize-your-app/compare/baseline...slim-base), [`slim-base` to `non-root-user`](https://github.com/MiguelSilva04/dockerize-your-app/compare/slim-base...non-root-user), [`non-root-user` to `multi-stage`](https://github.com/MiguelSilva04/dockerize-your-app/compare/non-root-user...multi-stage), and [`multi-stage` to `dockerignore`](https://github.com/MiguelSilva04/dockerize-your-app/compare/multi-stage...dockerignore).

## How to build and run

Needs Docker with Compose, and Python 3 to generate secrets (any other way of generating random strings works). Tested on Docker Desktop for Windows with PowerShell.

The base `compose.yml` refuses to start with the template's default secrets, so set real ones first. They live only in your shell session and are never written to a file.

PowerShell:

```powershell
$env:SECRET_KEY = python -c "import secrets; print(secrets.token_urlsafe(32))"
$env:POSTGRES_PASSWORD = python -c "import secrets; print(secrets.token_urlsafe(24))"
$env:FIRST_SUPERUSER_PASSWORD = python -c "import secrets; print(secrets.token_urlsafe(16))"
```

Bash (the same Python one-liner, not tested here):

```bash
export SECRET_KEY=$(python3 -c "import secrets; print(secrets.token_urlsafe(32))")
export POSTGRES_PASSWORD=$(python3 -c "import secrets; print(secrets.token_urlsafe(24))")
export FIRST_SUPERUSER_PASSWORD=$(python3 -c "import secrets; print(secrets.token_urlsafe(16))")
```

Then, from the repo root:

```
docker compose -f compose.yml up -d db
docker compose -f compose.yml run --rm backend bash scripts/prestart.sh
docker compose -f compose.yml up -d backend
docker compose -f compose.yml ps
```

The first `run` builds the image, applies the migrations and creates the first user. After about a minute the backend should show `healthy`. The base compose file publishes no ports, so the checks run inside the container. A real login through the API:

```
docker compose -f compose.yml exec backend python -c "import urllib.request, urllib.parse, os; d=urllib.parse.urlencode({'username': os.environ['FIRST_SUPERUSER'], 'password': os.environ['FIRST_SUPERUSER_PASSWORD']}).encode(); print(urllib.request.urlopen('http://localhost:8000/api/v1/login/access-token', d, timeout=5).status)"
```

It prints `200`. To stop and remove the containers and the database volume:

```
docker compose -f compose.yml down -v
```

## Notes

- `.env` holds upstream's development defaults (`changethis`). Not for production. The base `compose.yml` refuses to start with them, so real values have to come from the environment.
- The repo contains three `.dockerignore` files. Only the one at the root is read, because the build context is the repo root. The one in `backend/` was shown to be ignored; the one in `frontend/` was not tested.
- The upstream `.claude` and `.agents` folders were left out of this copy.
- The upstream `.github` folder (workflows and Dependabot config) was removed, because it targets the upstream repository.