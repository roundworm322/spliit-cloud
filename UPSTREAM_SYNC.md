# Syncing `direct-balances` with upstream

`direct-balances` is a small patch stack on top of an upstream release tag of
[antonio-ivanovski/spliit-cloud](https://github.com/antonio-ivanovski/spliit-cloud).
It adds direct (un-simplified, pairwise) balances — upstream issue #167 — and
fork-only CI. The upstream tag it is based on is recorded in `UPSTREAM_TAG`;
`.github/workflows/direct-images.yml` reads that file to tag images as
`<tag>-direct`.

## What the fork changes

| Area     | Files                                                                                                                                                                               |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Domain   | `packages/domain/src/balances.ts` (`getDirectSettlements`, `getDirectSettlementsByCurrency`, shared per-expense normalisation), `packages/domain/src/direct-balances.test.ts`       |
| API      | `apps/api/src/trpc/routers/groups/balances/list.procedure.ts` (+ test), `apps/api/src/trpc/outputs/balances.ts` (`direct` field), `apps/api/src/trpc/outputs/offline.ts` (omits it) |
| Web      | `apps/web/src/app/groups/[groupId]/balances/*` (Simplified / Direct picker), `apps/web/src/router/schemas.ts` (`settlementView` search param), `SimpleBalancesCard.test.tsx`        |
| i18n     | `Balances.settlementView.*` in `apps/web/src/messages/en-US.json` and `es.json` (other locales fall back to English)                                                                |
| CI       | Deleted `release.yml`, `android-release.yml`, `android-update-metadata.yml` (publish/deploy). Added `direct-images.yml`. Kept `ci.yml`, `android.yml`, `pullfrog.yml`.              |
| Metadata | `UPSTREAM_TAG`, this file                                                                                                                                                           |

## Rebase onto a new upstream tag

```bash
# One-time setup
git remote add upstream https://github.com/antonio-ivanovski/spliit-cloud.git

# 1. Pick the new release tag (latest vX.Y.Z)
git fetch upstream --tags --force
git tag --list 'v*.*.*' --sort=-v:refname | head -1
NEW=vX.Y.Z
OLD="$(cat UPSTREAM_TAG)"

# 2. Replay only the fork's commits (OLD..direct-balances) onto NEW
git switch direct-balances
git pull --ff-only origin direct-balances
git branch "direct-balances-backup-$OLD"
git rebase --onto "$NEW" "$OLD" direct-balances
```

### Resolving conflicts

During a rebase, `--ours` is the upstream side and `--theirs` is the fork
commit being replayed.

- **Deleted workflows** (`release.yml`, `android-*.yml`): a modify/delete
  conflict when upstream edits them. Keep them deleted: `git rm <file>`.
- **New upstream workflows**: after the rebase, check
  `.github/workflows/` for anything new that publishes or deploys
  (`packages: write`, `docker/build-push-action`, `deploy`, release creation,
  `git push`) and delete it in a follow-up commit.
- **`apps/web/src/messages/*.json`**: never hand-merge. Take upstream's file
  (`git checkout --ours <file>`), finish the rebase, then re-add the strings
  with the CLI:

  ```bash
  cat > /tmp/direct-en.json <<'EOF'
  {
    "Balances.settlementView.label": "Show debts as",
    "Balances.settlementView.simplified": "Simplified",
    "Balances.settlementView.direct": "Direct",
    "Balances.settlementView.directDescription": "What each person owes the people they actually shared expenses with, without combining debts across the group.",
    "Balances.settlementView.offlineHint": "Direct debts need a connection. Showing simplified payments while offline."
  }
  EOF
  cat > /tmp/direct-es.json <<'EOF'
  {
    "Balances.settlementView.label": "Mostrar deudas como",
    "Balances.settlementView.simplified": "Simplificados",
    "Balances.settlementView.direct": "Directos",
    "Balances.settlementView.directDescription": "Lo que cada persona debe a las personas con las que realmente compartió gastos, sin combinar deudas en todo el grupo.",
    "Balances.settlementView.offlineHint": "Las deudas directas necesitan conexión. Sin conexión se muestran los pagos simplificados."
  }
  EOF
  bun i18n add --file /tmp/direct-en.json
  bun i18n set es --file /tmp/direct-es.json
  ```

- **`packages/domain/src/balances.ts`**: the fork moves the per-expense
  normalisation from `getBalances()` / `getBalancesByCurrency()` into
  `getLedgerExpenseShares()` / `getNativeExpenseShares()`. If upstream changed
  that normalisation, apply their change inside those helpers so both the
  simplified and direct views keep using it.
- **Balances UI / API**: keep upstream's version and re-apply the
  `settlementView` / `direct` additions on top.

### Verify, record the tag, push

```bash
echo "$NEW" > UPSTREAM_TAG
git commit -am "chore: rebase direct-balances onto $NEW"

bun install
bun run prisma-generate
bun run check
bun run test          # API unit tests need Postgres on localhost:5432 (postgres/1234)

git push --force-with-lease origin direct-balances
```

The push runs `direct-images.yml`, which publishes
`ghcr.io/<owner>/spliit-cloud-api` and `ghcr.io/<owner>/spliit-cloud-web` as
`$NEW-direct` and `direct-latest`. Delete the backup branch once the images
work.

## Deploying the images

Only `api` and `web` are rebuilt. The fork adds no Prisma migrations and does
not touch the worker, so keep the upstream `spliit-cloud-migrate` and
`spliit-cloud-worker` images at the same release (`SPLIIT_TAG` = the content of
`UPSTREAM_TAG`) and override just `api` and `web`, e.g. with a
`compose.direct.yaml`:

```yaml
services:
  api:
    image: ghcr.io/<owner>/spliit-cloud-api:vX.Y.Z-direct
  web:
    image: ghcr.io/<owner>/spliit-cloud-web:vX.Y.Z-direct
```

```bash
SPLIIT_TAG=vX.Y.Z docker compose -f compose.yaml -f compose.direct.yaml up -d
```

GHCR packages start private; make them public in the package settings or log
in with `docker login ghcr.io` on the host.
