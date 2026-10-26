---
name: app-platform-publish
description: Publish a website or app to DigitalOcean App Platform and return its public URL. Use this whenever the user asks to publish, deploy, ship, host, or go live with something you built. Use it INSTEAD of the `publish` skill / `apps` tool, which needs Docker and does not work here. Covers pushing the code to GitHub, creating the App Platform app, waiting for the deploy, and fixing failures from the build logs.
---

# Publish to App Platform

Flow: **build and test in the sandbox → make it deployable → push to GitHub → create the App Platform app → wait for ACTIVE → check the URL.**

Never use `apps` action `publish`, `docker`, or `doctl` — none of them work on this computer. Everything goes through
two shared credentials, which you call via the core's credential broker. You never see the tokens.

Use one explicit `sandbox_id` for every command, and select the credential in `execute.credentials` for any command
that calls the broker (`digitalocean`, `github`, or `github-git`). Core supplies `AGENT_CREDENTIAL_TOKEN` to those
commands only. Do not use `rm -rf`; make a fresh directory with `mktemp -d` instead.

## What the computer has

Node 24 + npm, Python 3.12 + pip, git, gh (not logged in; do not use it), curl, gcc, make. **No** docker, doctl, jq,
wget, unzip. Parse JSON with `node -e` or `python3 -c`.

## Credentials (what each can do)

| Credential | Host | Allowed |
| --- | --- | --- |
| `digitalocean` | `api.digitalocean.com` | GET/POST/PUT/DELETE under `/v2/apps` and `/v2/regions` |
| `github` | `api.github.com` | GET/POST/PATCH under `/user` and `/repos/jnaidu-do` |
| `github-git` | `github.com` (git push/clone) | GET/POST under `/jnaidu-do/` |

Everything is created under the GitHub account `jnaidu-do`. A `403 path_not_allowed` / `method_not_allowed` means the
request is outside that — do not try variations; tell the user it is not permitted.

### Broker helper

Paste this once per command (shell state does not persist). It prints the HTTP status to stderr and the response body
to stdout:

```bash
bk() { # bk <credential> <METHOD> <url> [json-body]
  p=$(node -e 'const [c,m,u,b]=process.argv.slice(1);console.log(JSON.stringify({credential:c,method:m,url:u,headers:{accept:"application/json","content-type":"application/json"},...(b?{body:b}:{})}))' "$1" "$2" "$3" "${4:-}")
  curl -fsS -X POST "$AGENT_API_URL/v1/credentials/broker" -H "x-agent-capability: $AGENT_CREDENTIAL_TOKEN" \
    -H "content-type: application/json" -d "$p" |
  node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{const r=JSON.parse(s);console.error("HTTP "+r.status);console.log(r.body)})'
}
```

## 1. Make it deployable (check before you push)

The app runs as a normal container behind App Platform, not in this sandbox.

- **Listen on `$PORT`**, bound to `0.0.0.0` (App Platform sets `PORT`; default 8080). Never hardcode another port.
- **Plain HTML/CSS/JS or a framework that builds to static files** (Vite, CRA, Astro): deploy as a **static site**
  (`build_command`, `output_dir`). Prefer this over running a server for a static page.
- **Anything with a server**: a **service** with `build_command`, `run_command`, `http_port`. Standard layouts work
  with no config: Node needs `package.json` with a `start` script; Python needs `requirements.txt` and a run command
  (e.g. `gunicorn app:app --bind 0.0.0.0:$PORT`). A `Dockerfile` in the repo also works (set `dockerfile_path`), but you
  cannot test-build it here, so keep it simple.
- **No persistent disk.** Anything written to the filesystem is lost on every deploy. Do not use SQLite files, uploads
  to disk, or JSON files for state. If the app needs a database, add a managed Postgres (see below); otherwise keep it
  stateless.
- Run it locally first: `npm install`, start it with `PORT=8080`, and `curl localhost:8080`. Only publish what works.
- Add a `.gitignore` with `node_modules` etc. Do not commit secrets. Never print tokens.

### If the app uses Postgres

App Platform's `${db.DATABASE_URL}` includes `sslmode=require`, and Node `pg` then rejects the managed CA
(`SELF_SIGNED_CERT_IN_CHAIN`). Strip `sslmode` from the URL **and** set `ssl: { rejectUnauthorized: false }`:

```js
function normalizeDatabaseUrl(raw) {
  const u = new URL(raw);
  u.searchParams.delete("sslmode"); u.searchParams.delete("ssl");
  return u.toString();
}
const pool = new Pool({ connectionString: normalizeDatabaseUrl(process.env.DATABASE_URL), ssl: { rejectUnauthorized: false } });
```

Add `"databases": [{"name":"db","engine":"PG","production":false,"version":"16"}]` to the spec and
`{"key":"DATABASE_URL","scope":"RUN_TIME","value":"${db.DATABASE_URL}"}` to each component that needs it. Retry the
first connection for ~30 s: the database can still be starting. Only add a database if the app really needs one (it
costs extra).

## 2. Push to GitHub

Pick a repo name `qm-<short-kebab-name>` (lowercase). If the name is taken (422), add a short suffix.

```bash
# create the repo (public: App Platform can then clone it without extra auth)
bk github POST https://api.github.com/user/repos '{"name":"qm-hello-site","private":false}'

# push from a fresh clean directory containing the app (no node_modules)
cd "$(mktemp -d)" && cp -r /path/to/app/. . && git init -q -b main && git add . &&
git -c user.name=QM -c user.email=qm@example.com commit -qm "Initial commit" &&
git remote add origin "$AGENT_API_URL/v1/credentials/git/github-git/jnaidu-do/qm-hello-site.git" &&
git -c http.extraHeader="x-agent-capability: $AGENT_CREDENTIAL_TOKEN" push origin HEAD:refs/heads/main
```

Later updates: commit and push the same way (`HEAD:refs/heads/main`). The user asking you to publish is the approval
to create the repo and the app; ask only before deleting anything.

## 3. Create the App Platform app

```bash
bk digitalocean POST https://api.digitalocean.com/v2/apps '{"spec": {
  "name": "qm-hello-site",
  "region": "nyc",
  "services": [{
    "name": "web",
    "github": {"repo": "jnaidu-do/qm-hello-site", "branch": "main", "deploy_on_push": true},
    "build_command": "npm install",
    "run_command": "npm start",
    "http_port": 8080,
    "instance_size_slug": "basic-xxs",
    "instance_count": 1
  }]
}}'
```

Static site instead of `services`: `"static_sites": [{"name":"web","github":{...},"build_command":"npm run build","output_dir":"dist"}]`
(plain HTML: omit `build_command`, `output_dir` is `"/"`; for a single-page app add
`"catchall_document":"index.html"`).

The response has the app `id`. Keep it. App names are lowercase letters, digits, dashes, 2–32 chars.

**If the create fails with a GitHub repository-access error** ("does not have access to the repository"): the
DigitalOcean GitHub App is not granted this repo. Tell the user to open
https://github.com/apps/digitalocean/installations/select_target and grant access to the `jnaidu-do` repos, then say
done; retry the same request. If they cannot, fall back to a generic public clone source
(`"git": {"repo_clone_url": "https://github.com/jnaidu-do/qm-hello-site.git", "branch": "main"}`) and tell them
updates then need a redeploy (see below).

## 4. Wait for ACTIVE

A real build takes 3–5 minutes. Poll every 30 s, and tell the user you are waiting:

```bash
bk digitalocean GET "https://api.digitalocean.com/v2/apps/$APP_ID/deployments?per_page=1" | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{const d=JSON.parse(s).deployments[0];console.log(d.id,d.phase,d.progress?.success_steps+"/"+d.progress?.total_steps)})'
```

Phases: `PENDING_BUILD` → `BUILDING` → `DEPLOYING` → `ACTIVE`. `ERROR` or `CANCELED` means it failed; go to
"When it fails". Stop polling after about 12 minutes and tell the user where it stands.

## 5. Report the URL

```bash
bk digitalocean GET "https://api.digitalocean.com/v2/apps/$APP_ID" | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>console.log(JSON.parse(s).app.live_url))'
```

`curl -s -o /dev/null -w '%{http_code}' <live_url>` — expect 200. Give the user the full `https://…ondigitalocean.app`
URL and the repo URL. The app is **public** on the internet.

## When it fails

Read the logs yourself; do not ask the user for them. Get a log URL, then fetch it:

```bash
bk digitalocean GET "https://api.digitalocean.com/v2/apps/$APP_ID/deployments/$DEPLOYMENT_ID/logs?type=BUILD"
# the body has "live_url" (or "historic_urls"); curl that URL (no auth needed)
```

`type` is `BUILD`, `DEPLOY`, or `RUN` (static sites only have `BUILD`). Match the signal:

| Signal | Fix |
| --- | --- |
| `Missing script: start` / command not found | wrong `build_command` / `run_command`, or no `start` script |
| Container exits, health check fails | not listening on `$PORT` / not bound to `0.0.0.0`, or crash at boot |
| `SELF_SIGNED_CERT_IN_CHAIN` | Postgres SSL helper missing (see above) |
| `Cannot find module` | dependency missing from `package.json`, or committed `node_modules` mismatch |
| Vite "Blocked request. This host is not allowed" | set `preview: { allowedHosts: true }` (or serve the built `dist` as a static site) |
| Build "skipped" / ACTIVE in seconds | the source did not change; push a real commit |

Fix the code, commit and push (`deploy_on_push` redeploys automatically), then poll again. For a spec change, send
`PUT /v2/apps/$APP_ID` with `{"spec": {...}}` (get the current spec first with `GET /v2/apps/$APP_ID` and edit it).
For a generic `git` source, force a rebuild with `POST /v2/apps/$APP_ID/deployments` body `{"force_build":true}`.
Do not delete and recreate the app to fix a failure — that changes the URL. Fix forward. Give up after three
different failed attempts and report what you tried.

## Deleting

Only when the user explicitly asks, and name the app first. `DELETE /v2/apps/$APP_ID` removes the app. The GitHub repo
is left behind (this credential cannot delete repos); say so.

## Report

End with: the live URL, the repo URL, and one line on how to update it ("ask me to change X and I will push it").
