# Acceptance checklist

A template for the checks agreed with a client before the work starts. Nothing is delivered until every agreed check passes. What "works" means is specific to each application, so the checks in section 2 are always decided together with the client.

## 1. Agreed before starting

- [ ] The client's own definition of "the app works", written down (section 2).
- [ ] How the application is run today (command, environment, database).
- [ ] What stays out of scope: secrets management, registry credentials, infrastructure changes.
- [ ] A way back: the current image or Dockerfile is kept untouched under a name or tag.

## 2. Checks that prove the app still works

Decided with the client. Each one needs a command, an expected result and who runs it.

| # | Check | Command | Expected |
| --- | --- | --- | --- |
| 1 | The app starts | | The container reaches `healthy` within a time we agree |
| 2 | The main flow works | | |
| 3 | Data survives | | |

## 3. Checks on the image

- [ ] Size on disk and compressed, before and after.
- [ ] The process does not run as root (`id` inside the container).
- [ ] No development tools in the final image.
- [ ] The base image is named and pinned.
- [ ] No secrets in the image layers (`docker history`).

## 4. Evidence delivered

- [ ] A table with before and after for every measured value.
- [ ] One commit per change, so any change can be reverted on its own.
- [ ] How to build and run, in the README.

## Example: this repository

| # | Check | Command | Result |
| --- | --- | --- | --- |
| 1 | The backend becomes healthy | `docker compose -f compose.yml ps` | `healthy` |
| 2 | Migrations run from an empty database | `docker compose -f compose.yml run --rm backend bash scripts/prestart.sh` | Five migrations, first user created |
| 3 | A real login works | The login command in the README | `200` |
| 4 | The code is importable from a neutral folder | `docker run --rm -w / backend:multistage python -c "import app; print(app.__file__)"` | `/app/backend/app/__init__.py` |
| 5 | The process is not root | `docker compose -f compose.yml exec backend id` | `uid=10001(appuser)` |
| 6 | The code cannot be changed | `docker compose -f compose.yml exec backend sh -c "touch /app/backend/app/teste"` | `Permission denied` |
| 7 | `uv` is not in the final image | `docker compose -f compose.yml exec backend sh -c "command -v uv \|\| echo sem uv"` | `sem uv` |