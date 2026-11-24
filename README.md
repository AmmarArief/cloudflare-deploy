# Cloudflare auto-deploy

Set up **push to GitHub → live on Cloudflare** with no manual `wrangler deploy`, and fix the pipeline when it breaks.

Ships as an [Agent Skill](https://agentskills.io) (`SKILL.md`), so a coding agent can do the setup end to end: pick the mechanism, write the workflow, register the secrets, run the deploy, and prove the site serves the new build.

## Contents

| Path | What it is |
|---|---|
| [`SKILL.md`](./SKILL.md) | The runbook: decision table, token scopes, secret commands, deployment checks, real pitfalls |
| [`templates/opennext-worker.yml`](./templates/opennext-worker.yml) | Next.js on Workers via `@opennextjs/cloudflare`, with type check, tests, D1 migrations |
| [`templates/worker.yml`](./templates/worker.yml) | Plain Worker: `wrangler deploy` |
| [`templates/pages.yml`](./templates/pages.yml) | Static site on Cloudflare Pages |

## Use it

Copy a template to `<repo>/.github/workflows/deploy.yml`, then register the two secrets in that repo:

```bash
gh secret set CLOUDFLARE_API_TOKEN --repo <owner>/<repo>
gh secret set CLOUDFLARE_ACCOUNT_ID --body <account-id> --repo <owner>/<repo>
```

Push to `main` and the workflow builds, migrates, deploys.

### As an agent skill

```bash
cp SKILL.md ~/.omp/agent/skills/cloudflare-deploy/SKILL.md
```

Register the name in `~/.omp/agent/config.yml` under `skills.includeSkills`, plus the directory `~/.omp/agent/skills/cloudflare-deploy/`. Other agents read the same `SKILL.md` from their own skills directory.

## Prompt for a new project

Paste this into your agent after cloning a repo that is not deployed by GitHub Actions yet. Replace the two placeholders.

```
Pakai skill cloudflare-deploy untuk repo ini.

Target deploy: <NAMA_PROJECT> → <DOMAIN>
Account ID: <ACCOUNT_ID>
Token Cloudflare sudah ada di secret repo lain; jangan minta token baru.

Yang harus kamu lakukan:
1. Baca stack repo ini, lalu pilih mekanisme sesuai SKILL.md
   (Workers Builds / GitHub Actions / Pages). Jelaskan pilihanmu dalam satu baris.
2. Kalau GitHub Actions: ambil template yang cocok dari
   https://github.com/AmmarArief/cloudflare-deploy/tree/main/templates,
   tulis ke .github/workflows/deploy.yml, sesuaikan nama DB D1 dan script
   build repo ini.
3. Daftarkan CLOUDFLARE_API_TOKEN dan CLOUDFLARE_ACCOUNT_ID sebagai secret repo.
4. Jangan commit sebelum build lokal sama dengan command di workflow lolos.
5. Push, pantau sampai run hijau, lalu buktikan URL live balas 200 dan
   isinya build baru.
```

Untuk repo yang sudah punya workflow tapi gagal deploy:

```
Pakai skill cloudflare-deploy. Deploy repo ini gagal di run <RUN_ID>.
Cari penyebabnya dari log, perbaiki, lalu buktikan dengan push-triggered run
yang hijau dan URL live yang menampilkan perubahan.
```


## What the pipeline covers

- Build cache (`pnpm`), frozen lockfile
- Type check and tests before any deploy
- OpenNext bundle for Next.js, or direct `wrangler deploy`
- Remote D1 migrations before the new version takes traffic
- Failure surfaces early: missing token dies at the first Cloudflare step, not after a silent deploy

## License

MIT
