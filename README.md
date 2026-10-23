# FastAPI full-stack: containerization proof

Work in progress. A hands-on exercise: take a real, widely used open-source app, containerize it properly and measure the difference, to back up a Docker/containerization service.

## Upstream

Based on [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) (MIT), commit `b1125d8839482cb7bbae603ddc6d0081bf87532e` from 2026-10-15, copied on 2026-10-23. The original copyright and license are kept in `LICENSE`. The original README is `README.upstream.md`. Everything not listed under "Changes" is the upstream authors' work.

## Changes

None yet. The first step is to measure the unmodified baseline, see `docs/medicoes.md`.

## Notes

- `.env` holds upstream's development defaults (`changethis`). Not for production.
- The upstream `.claude` and `.agents` folders were left out of this copy.
- `.github/workflows` was written for the upstream repository. Review it before enabling Actions here.
