# FastAPI full-stack: containerization proof

Work in progress. A hands-on exercise: take a real, widely used open-source app, containerize it properly and measure the difference, to back up a Docker/containerization service.

## Upstream

Based on [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) (MIT), commit `b1125d8839482cb7bbae603ddc6d0081bf87532e` from 2026-10-15, copied on 2026-10-23. The original copyright and license are kept in `LICENSE`. The original README is `README.upstream.md`. Everything not listed under "Changes" is the upstream authors' work.

## Changes

Each change was measured before and after. Raw numbers and method are in [`docs/medicoes.md`](docs/medicoes.md).

| # | Change | Result |
| --- | --- | --- |
| 1 | `--no-dev` on both `uv sync` calls | Dependency layer 319 MB to 142 MB. 14 fewer Python packages, including pytest, mypy and ruff. |
| 2 | `FROM python:3.14-slim` | Image 1.88 GB to 439 MB on disk. Same Python 3.14.9. Login through the API still returns 200. |
| 3 | Healthcheck uses Python's `urllib` instead of `curl` | `curl` is not in the slim image, and the old check failed 5 times in a row. The new one passes, and exits with code 1 when nothing listens on the port. |
| 4 | Non-root user (`uid` 10001) | Image size unchanged (439 MB, the new layer is 41 kB). The image's `User` is the number `10001:10001`. Migrations, healthcheck and login verified as the new user, and writing to the app code is denied. |
| 5 | Multi-stage build for the backend | `uv` stays in the build stage. 442 MB to 362 MB on disk and 106 MB to 82.8 MB compressed, measured against a one-stage build on the same base. Migrations, healthcheck and login verified. |
| 6 | Root `.dockerignore` extended | Build context 2568 kB to 1484 kB (42% smaller). The image is identical: both builds report the same config digest. `backend/.dockerignore` is not read, because the build context is the repo root. |
| 7 | `compose.yml`: `prestart` service, restart policies, `start_period`, Adminer pinned and opt-in | With one `up -d backend` on an empty database, the old file left the backend `healthy` while login returned HTTP 500 (`relation "user" does not exist`). Now a `prestart` service applies the migrations and creates the first user and exits with 0, the backend waits for it, and login returns 200. A failing `prestart` makes `up` exit with an error and the backend never starts. `db`, `backend` and `proxy` restart after a crash (in a test where the container exits by itself it was restarted 10 times) and stay stopped after `docker compose stop`. With a 60 s startup the backend went `unhealthy` before `start_period: 30s` and stayed `starting` until `healthy` after it. Adminer is pinned to `6.3.0` and only starts with `--profile tools`. |
| 8 | Both base images pinned by digest (`python:3.14-slim` in the two stages that use it, and `oven/bun:1`; `uv` was already pinned to `0.9.26`) | The same tag no longer gives a different base on each build. After the Python pin: Python 3.14.10, backend `healthy`, login 200. Image 362 MB to 359 MB on disk and 82.8 MB to 82.5 MB compressed. The cause of the 3 MB was not isolated (the base had moved again, and the `oven/bun` tag may also have moved). Pinning stops automatic security patches, so the digests have to be reviewed from time to time. |
| 9 | Backend: `security_opt: no-new-privileges:true` | `NoNewPrivs` in `/proc/1/status` reads 1, measured before and after. Backend `healthy`, login 200. |
| 10 | Backend: `cap_drop: ALL` | `CapBnd` in `/proc/1/status` is all zeros. The process already ran as non-root with no effective, permitted, inheritable or ambient capabilities, so this removes the bounding set and nothing visible changes in what the process holds. |
| 11 | Backend: `read_only: true` | `touch /tmp/x` fails with `Read-only file system`. The backend starts, is `healthy`, and login returns 200, with no `tmpfs` added. Only startup, the healthcheck and the login were tested. |

The original image is 2.11 GB on disk and the current one is 359 MB. Changes 1 and 2 share one commit but were measured in separate builds. The `python:3.14-slim` tag moved from Python 3.14.9 to 3.14.10 between changes 4 and 5, so change 5 is compared against a one-stage image rebuilt on the new base. It moved again while change 6 was being measured, which is why change 6 is checked by comparing two builds on the same base. By the time of change 8 the tag pointed to a fourth digest. Changes 7, 9, 10 and 11 only change `compose.yml`, not the image. Change 8 does change it (362 MB to 359 MB).

Idle memory ranged from 476 to 505 MiB across all runs, for both the original and the current image. The difference between them (about 2% at 6 minutes) is smaller than the spread between samples of the same image, so no memory benefit is claimed. Memory was not measured again after changes 8 to 11.

## Tags

- `baseline`: the repository before any containerization change. The Dockerfile and `compose.yml` are the upstream ones.
- `slim-base`: after changes 1 to 3.
- `non-root-user`: after change 4.
- `multi-stage`: after change 5.
- `dockerignore`: after change 6.
- `compose`: after change 7.
- `pinned-bases`: after change 8.
- `no-new-privileges`: after change 9.
- `cap-drop`: after change 10.
- `read-only`: after change 11.

Compare: [`baseline` to `slim-base`](https://github.com/MiguelSilva04/dockerize-your-app/compare/baseline...slim-base), [`slim-base` to `non-root-user`](https://github.com/MiguelSilva04/dockerize-your-app/compare/slim-base...non-root-user), [`non-root-user` to `multi-stage`](https://github.com/MiguelSilva04/dockerize-your-app/compare/non-root-user...multi-stage), [`multi-stage` to `dockerignore`](https://github.com/MiguelSilva04/dockerize-your-app/compare/multi-stage...dockerignore), [`dockerignore` to `compose`](https://github.com/MiguelSilva04/dockerize-your-app/compare/dockerignore...compose), [`compose` to `pinned-bases`](https://github.com/MiguelSilva04/dockerize-your-app/compare/compose...pinned-bases), [`pinned-bases` to `no-new-privileges`](https://github.com/MiguelSilva04/dockerize-your-app/compare/pinned-bases...no-new-privileges), [`no-new-privileges` to `cap-drop`](https://github.com/MiguelSilva04/dockerize-your-app/compare/no-new-privileges...cap-drop), and [`cap-drop` to `read-only`](https://github.com/MiguelSilva04/dockerize-your-app/compare/cap-drop...read-only).

## Checks

These are not changes to the repo. They are checks done on the result, with the details in [`docs/medicoes.md`](docs/medicoes.md).

- **Vulnerability scan.** Trivy 0.76.2, vulnerability database updated 2026-11-24 13:06 UTC, HIGH and CRITICAL only. The original image has 852 findings (844 in Debian 13.7 packages, 825 HIGH and 19 CRITICAL, plus 8 in Python packages). The current image has 61 (53 HIGH in Debian 13.8 packages, plus 8 in Python packages, 7 HIGH and 1 CRITICAL). The counts are package and CVE pairs, they are only valid for that database date, and the two images are on different Debian point releases. Because the bases are pinned, these numbers will not improve by themselves. The CRITICAL Python finding is `certifi` 2026.6.17 (fixed in 2026.10.18). It comes from upstream's dependencies and was not changed here. Of the 8 Python findings in the current image, 5 name package versions that are not installed in the image's environment. They probably come from the BuildKit SBOM, which was not confirmed.
- **Registry.** A local AWS emulator image refused the push (`error from registry: the operation is unsupported`), so the push was tested on a private Amazon ECR repository in a personal AWS account (us-west-1). 14 layers, 82,494,554 bytes stored. The digest `sha256:6d1e93a1...` is the same as the local image ID and as the one after pulling it back. The repository was deleted afterwards.
- **Clean clone.** A fresh clone of this repo at the commit of change 11, `docker compose build --no-cache` in 88.7 s (base images already downloaded, and BuildKit cache mounts may have been reused, which was not checked), an empty database, then one `up -d backend`: `prestart` exits with 0, the backend is `healthy`, Python 3.14.10, login 200, `NoNewPrivs: 1`, `CapBnd` all zeros, and writing to `/tmp` is refused. With the session secrets not set, `prestart` exits with 1 and its log shows the app refusing `SECRET_KEY=changethis`.

## How to build and run

Needs Docker with Compose, and Python 3 to generate secrets (any other way of generating random strings works). Tested on Docker Desktop for Windows with PowerShell.

The backend refuses to run with the template's default secrets (`changethis`, set in `.env`), so set real ones first. They live only in your shell session and are never written to a file.

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
docker compose -f compose.yml up -d backend
docker compose -f compose.yml ps -a
```

The first run builds the image. The `prestart` service applies the migrations and creates the first user, then exits with code 0 (use `ps -a` to see it, a plain `ps` hides exited containers). The backend starts after that and should show `healthy` after about a minute. The base compose file publishes no ports, so the checks run inside the container. A real login through the API:

```
docker compose -f compose.yml exec backend python -c "import urllib.request, urllib.parse, os; d=urllib.parse.urlencode({'username': os.environ['FIRST_SUPERUSER'], 'password': os.environ['FIRST_SUPERUSER_PASSWORD']}).encode(); print(urllib.request.urlopen('http://localhost:8000/api/v1/login/access-token', d, timeout=5).status)"
```

It prints `200`. To stop and remove the containers and the database volume:

```
docker compose -f compose.yml down -v
```

Adminer does not start by default. To include it, add `--profile tools` to the command.

## Notes

- Always pass `-f compose.yml`. Without it, Compose also loads `compose.override.yml`, the upstream development setup (`fastapi dev`, published ports, hot reload).
- `.env` holds upstream's development defaults (`changethis`). Not for production. With these defaults the containers start, but the backend exits at startup with a validation error, so real values have to come from the environment.
- The `proxy` service mounts `/var/run/docker.sock` read-only, as in the upstream template. This gives Traefik access to the Docker API, so keep it in mind before exposing the stack.
- The repo contains three `.dockerignore` files. Only the one at the root is read, because the build context is the repo root. The one in `backend/` was shown to be ignored; the one in `frontend/` was not tested.
- The `FROM` lines are pinned by digest (change 8). Builds are repeatable, but nothing patches the base automatically any more, so someone has to review and update the digests. There is no Dependabot here, because the upstream `.github` folder was removed.
- The backend runs with `no-new-privileges`, no capabilities and a read-only root filesystem. `read_only` was only tested for startup, the healthcheck and the login. An endpoint that writes files will fail unless you give it a `tmpfs` or a volume.
- The upstream `.claude` and `.agents` folders were left out of this copy.
- The upstream `.github` folder (workflows and Dependabot config) was removed, because it targets the upstream repository.