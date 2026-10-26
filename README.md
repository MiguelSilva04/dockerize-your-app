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

The original image is 2.11 GB on disk and the current one is 439 MB. Changes 1 and 2 share one commit but were measured in separate builds.

## Tags

- `baseline`: the repository before any containerization change. The Dockerfile and `compose.yml` are the upstream ones.
- `slim-base`: after changes 1 to 3.
- `non-root-user`: after change 4.

Compare: [`baseline` to `slim-base`](https://github.com/MiguelSilva04/dockerize-your-app/compare/baseline...slim-base), and [`slim-base` to `non-root-user`](https://github.com/MiguelSilva04/dockerize-your-app/compare/slim-base...non-root-user).

## Notes

- `.env` holds upstream's development defaults (`changethis`). Not for production. The base `compose.yml` refuses to start with them, so real values have to come from the environment.
- The upstream `.claude` and `.agents` folders were left out of this copy.
- The upstream `.github` folder (workflows and Dependabot config) was removed, because it targets the upstream repository.