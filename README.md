# docker-cal-diy

[![Docker Hub](https://img.shields.io/badge/Docker%20Hub-zenjoy%2Fcal--diy-lightgrey?style=flat)](https://hub.docker.com/r/zenjoy/cal-diy)
[![License](https://img.shields.io/github/license/zenjoy/docker-cal-diy)](https://github.com/zenjoy/docker-cal-diy/blob/main/LICENSE)

Container images for [cal.diy](https://github.com/calcom/cal.diy), the MIT community fork of Cal.com.

```sh
docker pull zenjoy/cal-diy:main
docker pull ghcr.io/zenjoy/cal-diy:main
```

## Why this repo exists

Cal.com went closed-source in April 2026 and its official image is frozen at v6.2.0. `cal.diy`
continues as an MIT fork but publishes no container image of its own, so we build it ourselves.

This repo holds no application code. It pins an upstream commit and builds upstream's **unmodified**
`Dockerfile` with `calcom/cal.diy` as the build context.

## Bumping the pin

`versions.env` is the single source of truth:

```
CAL_DIY_REF=<40-char commit sha of calcom/cal.diy>
```

Renovate watches `main` of the upstream repo (`git-refs` datasource) and opens a PR to bump the sha.
To bump by hand:

```sh
gh api repos/calcom/cal.diy/commits/main --jq .sha
```

Merging a change to `versions.env` on `main` triggers a build. You can also run the
**Build and push container images** workflow manually; its optional `ref` input overrides
`versions.env` for that run (handy to test an upstream branch without committing).

## Tags

| Tag                   | Meaning                                      |
| --------------------- | -------------------------------------------- |
| `<yyyymmdd>-<sha7>`   | Immutable build tag — use this in Kubernetes |
| `sha-<sha7>`          | Alias for the upstream commit                |
| `main`                | Latest successful build                      |

amd64 only. Builds take ~45-60 min (Next.js monorepo, `MAX_OLD_SPACE_SIZE=6144`).

## Runtime notes

Upstream's entrypoint is `scripts/start.sh`, which on every boot:

1. Rewrites the baked-in URL placeholder with the runtime `NEXT_PUBLIC_WEBAPP_URL`
   (static Next.js output is patched in place, so the container filesystem must be writable).
2. Waits for `DATABASE_HOST` (set it, or `wait-for-it.sh` hangs/errors).
3. Runs `prisma migrate deploy` — deploy one replica at a time, strategy `Recreate`.
4. Seeds the app store (`seed-app-store.ts`), then `yarn start`.

Because of step 1, `NEXT_PUBLIC_WEBAPP_URL` is **not** a build arg here — set it at runtime.

Other facts from upstream's Dockerfile: listens on port `3000`, runs as `root` (`node:20` base, no
`USER`), and has a `HEALTHCHECK` hitting `http://localhost:3000`. Required runtime env includes
`DATABASE_URL`, `DATABASE_DIRECT_URL`, `NEXTAUTH_SECRET`, `NEXTAUTH_URL`,
`CALENDSO_ENCRYPTION_KEY` (never rotate this one) and `NEXT_PUBLIC_WEBAPP_URL`.

## Container signatures

All images are signed with [Cosign](https://docs.sigstore.dev/cosign/overview/) keyless signatures:

```sh
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp https://github.com/zenjoy/docker-cal-diy/.github/workflows/ \
  zenjoy/cal-diy:main
```

## License

This repo is MIT ([LICENSE](LICENSE)). The image content is `calcom/cal.diy`, also MIT.
