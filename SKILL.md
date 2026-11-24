---
name: cloudflare-deploy
description: Use when wiring a repo to auto-deploy on Cloudflare (Workers, OpenNext Next.js, or Pages) from GitHub pushes, when a Cloudflare deploy pipeline fails, or when secrets/tokens for it need fixing — covers workflow templates, token scopes, D1 migrations, and verification.
---
# Cloudflare auto-deploy from GitHub

One reusable runbook for "push to GitHub → live on Cloudflare". Applies to Workers (plain and OpenNext/Next.js) and to Pages.

## Fixed facts (this machine)

- Cloudflare account: `Marmdhn28@gmail.com's Account`
- Account ID: `6691af46b1cbea948bb45ef274a9bfd6`
- Workers subdomain: `marprojector.workers.dev`
- Reusable API token name: `githubworkers`, TTL no expiration

Token scopes that cover everything below:

| Scope | Permission |
|---|---|
| Account | Workers Scripts: Edit |
| Account | Workers Routes: Edit |
| Account | D1: Edit |
| Account | Account Settings: Read |
| Zone | Zone: Read |

Token is **never** stored in a repo. It lives in each repo's GitHub secret `CLOUDFLARE_API_TOKEN`; account id in `CLOUDFLARE_ACCOUNT_ID`.

## Choose the mechanism first

| Situation | Use |
|---|---|
| Static/Vite/Astro, no tests, no migrations | Cloudflare **Workers Builds** (dashboard → connect Git). Zero YAML, zero secrets. |
| Next.js + OpenNext, or tests/typecheck, or D1 migrations | **GitHub Actions** (templates below) |
| Many repos, want set-once | GitHub **org-level secrets** + a `ci` repo with a reusable workflow (`secrets: inherit`) |

Ask which one before writing files. Do not add a workflow when Workers Builds is enough.

## GitHub Actions: per repo

1. Add `.github/workflows/deploy.yml` (pick the matching template).
2. Register secrets:
   ```bash
   gh secret set CLOUDFLARE_API_TOKEN --repo <owner>/<repo>
   gh secret set CLOUDFLARE_ACCOUNT_ID --body 6691af46b1cbea948bb45ef274a9bfd6 --repo <owner>/<repo>
   ```
3. Commit, push, then `gh run watch <run-id> --repo <owner>/<repo> --exit-status`.
4. Prove it: push-triggered run goes green **and** the live URL returns 200 with the new content.

Secrets are per repo. There is no inheritance on personal accounts.

### Template: Next.js + OpenNext on Workers

```yaml
name: Deploy to Cloudflare Workers
on:
  push:
    branches: [main]
  workflow_dispatch:
concurrency:
  group: deploy-production
  cancel-in-progress: true
permissions:
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .node-version
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm check-types
      - run: pnpm test
      - run: pnpm exec opennextjs-cloudflare build
      - run: pnpm exec wrangler d1 migrations apply <DB_NAME> --remote   # drop if no D1
      - run: pnpm exec opennextjs-cloudflare deploy
```

### Template: plain Worker

```yaml
name: Deploy Worker
on:
  push: { branches: [main] }
  workflow_dispatch:
concurrency: { group: deploy-production, cancel-in-progress: true }
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec wrangler deploy
```

### Template: static site on Pages

```yaml
name: Deploy Pages
on:
  push: { branches: [main] }
  workflow_dispatch:
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile && pnpm build
      - run: pnpm exec wrangler pages deploy dist --project-name=<PROJECT>
```

Needs `Pages: Edit` in the token (already present in `githubworkers`) and the Pages project to exist: `wrangler pages project create <PROJECT>` once.

## Reading the current state before editing

```bash
# which worker serves which hostname
curl -s -H "Authorization: Bearer $TOK" \
  "https://api.cloudflare.com/client/v4/accounts/<ACC>/workers/domains" | jq -r '.result[] | "\(.hostname) -> \(.service)"'
# deployed scripts
curl -s -H "Authorization: Bearer $TOK" \
  "https://api.cloudflare.com/client/v4/accounts/<ACC>/workers/scripts" | jq -r '.result[] | .id'
```

Do not assume the worker name equals the hostname, and do not assume a site is served by Pages — custom domains on Workers also resolve to `AAAA 100::`.

## Pitfalls that have actually bitten

- `package.json` referencing `scripts/cf-*.mjs` that was never committed → `pnpm deploy` is dead. Replace it with `opennextjs-cloudflare build && opennextjs-cloudflare deploy`.
- Token without `D1: Edit` fails only at the migration step, after a green build. Add D1 to the **existing Account policy**, not a new policy.
- A workflow run that started before the secret was saved sees an empty `CLOUDFLARE_API_TOKEN`; fix with `gh run rerun <id>`.
- Content on the live site can come from **D1**, not from `src/**`. Before hunting a string in code, check the DB:
  ```bash
  wrangler d1 execute <DB> --remote --command "select section,item_id from portfolio_content where data like '%needle%'"
  ```
- A local clone can be dozens of commits behind `origin/main`; `git fetch` and compare before editing, or edits land on a stale base.
- A renamed GitHub account makes old `github.com/<old>` links 404 while the remote keeps working through GitHub's redirect. Update the remote URL after the first push.
- In git worktrees, commit with the `commit` namespace; never `git add`/`git commit` directly.

## Verification (required before reporting success)

1. Local build of the same command the workflow runs (`pnpm exec opennextjs-cloudflare build` or `pnpm build`).
2. `gh run view <id>` → job `✓`, and the deploy step prints `Success! Uploaded` plus a Version ID.
3. Live check: `curl -s -o /dev/null -w '%{http_code}' https://<domain>/` → 200, and the changed content is present in the HTML.
4. Push-trigger path proven separately from `workflow_dispatch`: rerun the push-triggered run, or push a trivial commit.
