---
name: auto-web
description: "Use when deployed web or backend infrastructure must be checked against the repository — when the user asks whether prod matches main, mentions deployment drift, infra drift, TLS or DNS or CDN or env-var parity, asks to verify a deploy, or says \"/auto-web\"; or when about to claim a deployment is current, an environment is configured correctly, or infrastructure state is safe to change."
---

# /auto-web

Determine whether what is actually running matches what is in this repository — and remediate the
gaps in the correct order, with the read-only work done first.

**Two governing rules.**

1. **Observe before you touch.** Every phase below is read-only until an explicit approval gate. The
   dangerous failure here is not missing drift; it is "fixing" drift by overwriting live state that
   was the correct side.
2. **Never print a secret.** Enumerate names, never values. The commands below are chosen for that
   property, and the ones that violate it are called out.

## Step 1 — Discover The Infrastructure

Find every deploy target the repo describes:

```bash
rg --files --hidden --no-ignore \
  -g '{Dockerfile*,docker-compose*,*.tf,*.tfvars,Pulumi.*,*.bicep,template.y*ml,serverless.y*ml}' \
  -g '{vercel.json,netlify.toml,fly.toml,render.yaml,railway.*,app.yaml,Procfile,heroku.yml,wrangler.*}' \
  -g '{*.k8s.y*ml,kustomization.y*ml,Chart.y*ml,ansible.cfg,inventory*,Caddyfile,nginx.conf}' \
  -g '!.git' -g '!node_modules' -g '!.terraform' -g '!vendor'
```

Put the exclusions **after** the inclusions and use the bare-directory form (`-g '!.git'`, not
`-g '!.git/**'`) — the glob order matters and the trailing-slash form leaks paths back in.

Also read: CI deploy jobs and their environment declarations, `.env.example`, SSH config and deploy
scripts, DNS-as-code, and reverse-proxy config. Then ask the user which environments exist and which
you may touch.

**Stack is server-side state, not repo state.** On Heroku, `heroku.yml` overrides the `Procfile`'s
process definitions **only when the app's stack is `container`** — confirm with `heroku stack -a <app>`
rather than inferring from which files exist.

## Step 2 — Credentials, Safely

Detect the secret manager in use, then fetch **without echoing**: `op read`, `vault kv get -field=…`,
`sops -d`, `aws secretsmanager get-secret-value --query SecretString --output text`, `doppler secrets get --plain`.

Name-only enumeration is what an audit needs:

```bash
gh secret list                                  # safe: names only
gh variable list --json name,updatedAt          # bare `gh variable list` PRINTS VALUES
kubectl -n prod get secret app-env -o json | jq -r '.data | keys[]'
kubectl -n prod describe secret app-env         # masks values, shows key names + byte counts
docker compose config --no-interpolate --no-env-resolution   # topology with ${VARS} left unexpanded
```

**Never** `kubectl config view --raw` in any combination — redaction is the default and `--raw`
defeats it; `--minify` only narrows scope. **Never** `docker compose config` bare (it interpolates
`.env`) and never `--environment`.

If credentials are missing, ask for the specific least-privilege credential and name the scope. Do not
hunt the filesystem. Record non-secret connection details in the repo; record nothing else.

## Step 3 — Version Alignment

What SHA is actually live?

```bash
curl -fsS https://app.example.com/version            # or /health, /__build
curl -fsSI https://app.example.com/ | grep -i 'x-app-version\|x-nextjs-deployment-id'
crane digest --platform linux/amd64 ghcr.io/org/app:prod    # pin the platform or digests differ
```

Then place that SHA in your history — **fetch first, and handle the unknown-commit case explicitly**:

```bash
git fetch --all --tags --prune
git cat-file -e "$LIVE_SHA^{commit}" 2>/dev/null \
  || echo "UNKNOWN COMMIT $LIVE_SHA — built from a fork, a deleted branch, or a dirty tree"
git merge-base --is-ancestor "$LIVE_SHA" HEAD && echo "live is an ancestor: behind" || echo "live is NOT an ancestor: divergent"
```

A live SHA that is not an ancestor of your branch is the finding that matters — it means production is
running something this branch does not contain.

If the app exposes no build signal, that is itself a finding: recommend stamping one
(`-ldflags "-X main.commit=$GIT_SHA"`, `deploymentId: process.env.GIT_SHA` in `next.config.js`, the
`org.opencontainers.image.revision` label) so the next audit is one curl.

## Step 4 — Infrastructure Drift, Read-Only

```bash
terraform init -input=false -backend=true -lockfile=readonly   # NEVER -migrate-state / -reconfigure
terraform plan -detailed-exitcode        # 0 = no changes, 1 = error, 2 = changes present
pulumi preview --diff
kubectl diff -f manifests/
helm diff upgrade <release> <chart>
ansible-playbook site.yml --check --diff
```

**`terraform plan` is not unconditionally read-only.** Before calling it so, check what the config
actually does during refresh:

```bash
rg -nE 'data "external"|data "http"|provisioner|terraform_data|null_resource' *.tf modules/
```

An `external` data source runs a program on every plan. Same discipline as grepping Ansible for
`check_mode: false`.

Also compare API contract (`oasdiff`, `graphql-inspector`) and database migration state
(`alembic current`, `prisma migrate status`, `django ... showmigrations`, `flyway info`,
`atlas schema diff`) — all in their read-only forms.

## Step 5 — The Public Surface

```bash
# TLS: validate the chain AND the hostname
echo | openssl s_client -connect HOST:443 -servername HOST -verify_hostname HOST -verify_return_error >/dev/null 2>&1
echo | openssl s_client -connect HOST:443 -servername HOST 2>/dev/null | openssl x509 -checkend 1209600 -noout   # 14 days

dig +short A HOST; dig +short CNAME HOST; dig +short CAA HOST
curl -sSL -o /dev/null -D - https://HOST/     # read the LAST header block, after redirects
```

`-verify_hostname` supersedes the `-checkhost`-and-grep workaround: it actually exits non-zero on a
mismatch.

**Headers must be read after the redirect chain.** A bare `curl -I` on a 3xx measures the redirector,
not the app. If the first status is 3xx, you have measured the wrong thing.

**`robots.txt` severity is asymmetric**: a 5xx (and Google treats **429** the same way) halts crawling
for as long as it persists, with a documented escape hatch of roughly 30 days; a 4xx is treated as
"no restrictions". A WAF rate-limiting Googlebot on `robots.txt` is therefore a crawl outage.

Also check: `Set-Cookie` attributes (`Secure`, `HttpOnly`, `SameSite`), CORS config against the real
frontend origin **including `Vary: Origin`**, sitemap versus the actual route table, cached-vs-origin
behaviour (`curl --resolve` pins host:port to an IP; `--connect-to` redirects to another host:port —
both preserve SNI), and **dangling CNAMEs** pointing at deprovisioned services, which is a subdomain
takeover rather than a cosmetic error.

## Step 6 — Environment Parity

Compare declared variables against what is actually set, per environment. Know which commands print
**values** rather than names — those are for your eyes only and never for a report:

```bash
vercel env ls           fly secrets list        heroku config -a APP
wrangler secret list    kubectl get secret ... -o json | jq -r '.data | keys[]'
```

Missing, extra, and differing-between-environments are three separate findings.

## Step 7 — Report, Then Gate

`.audit/web-report.md`: every finding as **observed vs expected**, with the command that produced it,
the environment, and whether remediation is read-only, mutating, or irreversible.

Then classify every proposed action:

| Tier | Examples |
|---|---|
| **Unattended** | Any read. Generating the report |
| **Approval per action** | `terraform apply` on a *fresh* plan, `kubectl apply`, a cache purge, a DNS change, a redeploy |
| **Never without explicit, specific instruction** | Anything on production during peak, a destructive migration, rotating a credential other systems consume, `terraform apply` on a stale plan, deleting state |

**Order of operations when you do change something:** staging before production; snapshot or backup
before any schema change; re-plan immediately before applying (a plan file goes stale the moment
anything else touches state); one change at a time with verification between.

## "The Drift Is Fine" Is A Decision, Not A Default

When live and repo disagree, **establish which side is correct before acting.** Production may be
carrying an emergency hotfix nobody committed. The repo may be ahead because a deploy failed silently.
Overwriting either one blindly is how an outage gets caused by a cleanup.

Ask: when was each side last changed, by whom, and does the live state contain anything the repo does
not? `terraform state` history, `kubectl get ... -o json | jq .metadata.managedFields`, and the
provider's audit log answer this. If live is right, the fix is a commit, not an apply.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "The deploy succeeded, so it's current" | Check the live SHA. Succeeded means the job exited zero |
| "`terraform plan` is read-only" | Not with `external`/`http` data sources or provisioners. Grep first |
| "The health endpoint returns 200" | That may be the CDN's cached copy. Check the origin |
| "Headers look fine" | You may have measured the redirector. Read the last block after `-L` |
| "The cert is valid" | Valid for *which hostname*? Use `-verify_hostname` |
| "robots.txt 404s, no big deal" | Correct — but a 5xx or 429 halts crawling entirely |
| "Live differs from repo, so live is wrong" | Live may hold an uncommitted hotfix. Establish which side is right |
| "I'll apply the plan I made earlier" | Stale plans apply stale intent. Re-plan immediately before |
| "I'll just purge the cache to be safe" | That is a mutating action with user-visible effect. Gate it |
| "I need the env values to compare" | Compare names and presence. Values do not belong in a report |

## Red Flags — Stop

- Any command that prints a secret value into output, logs, or the report
- `kubectl config view --raw`, bare `docker compose config`, bare `gh variable list`
- Calling `terraform plan` read-only without grepping for external data sources and provisioners
- Applying, purging, or redeploying without an explicit approval for that specific action
- Concluding drift direction without checking when each side last changed
- Reporting a version match from a tag rather than a digest or SHA
- Touching production before staging, or without a snapshot when schema is involved
- Treating a CDN response as evidence about the origin
