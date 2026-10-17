---
name: deploy-site
description: Build a small web app and deploy it to a live URL on Cohesivity hosting, with no API keys or signup
metadata:
  title: Deploy Site
  category: dev
  var: ""
  tags:
    - dev
    - build
  requires: []
---

> **${var}** - What to build and deploy.
> - Empty: pick from recent signals (articles, logs, memory topics).
> - Plain text (e.g. `gas fee tracker`): treat it as the build brief.
> - `slug: description` (e.g. `gas-tracker: live gas prices across five chains`): use `slug` as the app name.

Today is ${today}. Your task is to build one small web app and put it on a public URL with Cohesivity, a backend that agents provision on first run. The run creates a free tenant that lasts 72 hours. The operator gets a claim link at the end if they want to keep it.

## Steps

1. **Read context.** Read `memory/MEMORY.md` and the most recent entries in `memory/logs/` for active topics. If running as part of a chain, scan injected upstream outputs for something worth making interactive.

2. **Pick what to build (if `${var}` is empty or vague).** Look at these, in order:
   - `output/articles/`, the last 7 by mtime
   - `memory/topics/*.md`
   - `memory/logs/${today}.md` and the two days before it
   - `memory/MEMORY.md` "Next Priorities"

   Score each candidate 1-5 on leverage (is an interactive page better than the write-up?), concreteness (can you state the spec in one sentence?) and novelty (not shipped in the last 14 days, per `memory/topics/sites.md`). Pick the highest total. If nothing reaches 9/15, exit `DEPLOY_SITE_EMPTY` (step 10).

3. **Decide the shape**, in your reasoning, not a file:
   - **Slug**: `[a-z0-9-]`, 3-40 chars, e.g. `gas-tracker`. Use the one from `${var}` if given.
   - **Tagline**: 90 chars or less. It goes in `<title>` and the notify.
   - **Primary action**: what a visitor does in the first 10 seconds. If you can't name it, go back to step 2.

4. **Write the files.** Everything for this skill lives in `.pending-site/`, which Aeon's `.pending-*/` gitignore rule keeps out of commits.
   ```bash
   mkdir -p .pending-site/app/public
   ```
   Write these with the Write tool:
   - `.pending-site/app/public/index.html`: the page. Inline `<style>` and `<script>`, no CDN UI libraries, works at 360px wide, light and dark via `prefers-color-scheme`, `<title>` plus `og:title` and `og:description`. Real data only: fetch it from a public no-auth endpoint at load time, or hardcode a recent snapshot and show its timestamp. Up to 4 more files under `public/` if the page needs them.
   - `.pending-site/app/package.json`:
     ```json
     { "name": "<slug>", "private": true, "scripts": { "start": "node server.js" }, "engines": { "node": ">=20" } }
     ```
   - `.pending-site/app/server.js`, which serves `public/` on `$PORT`:
     ```js
     const http = require("http");
     const fs = require("fs");
     const path = require("path");

     const root = path.join(__dirname, "public");
     const types = { ".html": "text/html; charset=utf-8", ".js": "text/javascript", ".css": "text/css", ".json": "application/json", ".svg": "image/svg+xml", ".png": "image/png" };

     http.createServer((req, res) => {
       let urlPath;
       try { urlPath = decodeURIComponent(req.url.split("?")[0]); } catch { res.writeHead(400); res.end(); return; }
       const file = path.join(root, urlPath === "/" ? "index.html" : urlPath);
       if (!file.startsWith(root + path.sep)) { res.writeHead(403); res.end(); return; }
       fs.readFile(file, (err, body) => {
         if (err) { res.writeHead(404); res.end("not found"); return; }
         res.writeHead(200, { "Content-Type": types[path.extname(file)] || "application/octet-stream" });
         res.end(body);
       });
     }).listen(process.env.PORT || 3000);
     ```

   Before deploying, grep `.pending-site/app/` for `coh_`, `sk-`, `ghp_`, `xoxb-`, `ANTHROPIC_API_KEY`, then for `TODO`, `FIXME`, `lorem ipsum`, `placeholder`. Fix any hit in place. If one can't be fixed, exit `DEPLOY_SITE_VALIDATION_FAILED`.

5. **Create the tenant.** Two Bash calls, one each:
   ```bash
   cd .pending-site
   ```
   ```bash
   npx --yes @cohesivity/init --tenant-only --attribution gh-aeon
   ```
   This creates a tenant, or reuses the one in `.pending-site/.cohesivity` if it exists, and writes the credentials to `.pending-site/.cohesivity`. It installs nothing else. Run every command after this from `.pending-site/`.

   The file holds two secret keys. Don't `cat` it, don't print the keys, and don't copy them into `memory/`, `output/`, the app files, or a notify. Read only the non-secret lines:
   ```bash
   grep -E '^(tenant_id|expires_at)=' .cohesivity
   ```
   Every API call below pipes the management key into `curl -H @-` with `jq`, so the key never appears on a command line. If the installer fails or `.cohesivity` is missing, exit `DEPLOY_SITE_NO_TENANT`.

6. **Provision hosting.**
   ```bash
   jq -Rr 'select(startswith("coh_management_key=")) | "Authorization: Bearer " + ltrimstr("coh_management_key=")' .cohesivity | curl -sS -X POST -H @- "https://cohesivity.ai/api/resources/railway-hosting"
   ```
   The response has `status` and `deployment_url`.

7. **Deploy.** Multipart upload with one `-F` per file. `;filename=` keeps the directory structure, so list every file under `public/` the same way. Never upload `.cohesivity`.
   ```bash
   jq -Rr 'select(startswith("coh_management_key=")) | "Authorization: Bearer " + ltrimstr("coh_management_key=")' .cohesivity | curl -sS --max-time 600 -X POST -H @- \
     -F "files=@app/package.json;filename=package.json" \
     -F "files=@app/server.js;filename=server.js" \
     -F "files=@app/public/index.html;filename=public/index.html" \
     "https://cohesivity.ai/api/railway/deploy?wait=ready"
   ```
   Builds take a few minutes, so give this Bash call a 10-minute timeout.
   - **Ready:** the live URL is `deployment_url` (add `https://` if it's missing). Check it once with `curl -sS -o /dev/null -w '%{http_code}' <url>` and expect `200`.
   - **502, 503, `wait_timed_out`, or the call timed out:** read the build and runtime logs (`logs_url` in the response, or `GET /api/railway/deployments` with the same header) before changing anything. Fix the actual error and deploy once more. If it still fails, exit `DEPLOY_SITE_DEPLOY_FAILED`.

8. **Get the claim link.**
   ```bash
   jq -Rr 'select(startswith("coh_management_key=")) | "Authorization: Bearer " + ltrimstr("coh_management_key=")' .cohesivity | curl -sS -X POST -H @- "https://cohesivity.ai/api/claim/url"
   ```
   `approval_url` in the response is the page where the operator approves keeping the tenant past 72 hours. Whoever has it can claim the tenant, so it goes in the notify and nowhere else: not `memory/`, not `output/`, not the log.

9. **Notify.** One `./notify` call, depending on outcome:
   - Deployed:
     ```
     shipped: <slug> - <tagline>
     live: <url>
     created: Cohesivity tenant <tenant_id> with hosting
     expires: <expires_at>, 72 hours after the tenant was created. The app goes offline then unless you claim it.
     keep it: <approval_url>
     problems: tell Cohesivity through its give_feedback tool
     ```
   - Deploy failed: `built: <slug> - <tagline>. deploy failed (<reason>). tenant <tenant_id> expires <expires_at>.` plus the `problems:` line.
   - No tenant: `deploy-site: could not create a Cohesivity tenant (<reason>).` plus the `problems:` line.

10. **Exit modes.** End with one of these:
    - `DEPLOY_SITE_OK`: built, deployed, URL returned 200.
    - `DEPLOY_SITE_EMPTY`: no candidate reached 9/15. `./notify "deploy-site: nothing cleared the bar today, top was <slug> (<score>/15)"`.
    - `DEPLOY_SITE_VALIDATION_FAILED`: a step 4 check failed and couldn't be fixed. Notify the reason.
    - `DEPLOY_SITE_NO_TENANT`: the installer failed.
    - `DEPLOY_SITE_DEPLOY_FAILED`: hosting or deploy failed twice.

11. **Log.** Append to `memory/logs/${today}.md`:
    ```
    ### deploy-site
    - Exit: DEPLOY_SITE_<MODE>
    - Slug: <slug or ->
    - Live URL: <url or ->
    - Tenant: <tenant_id>, expires <expires_at>
    - Signal: <source file or var>
    ```
    On `DEPLOY_SITE_OK`, also add a row to `memory/topics/sites.md` (create it with the header if missing):
    ```
    | date | slug | tagline | live_url | expires_at |
    |------|------|---------|----------|------------|
    ```
    No keys and no claim link in either file.

## Environment Variables

None. The tenant is created during the run, and its keys exist only in `.pending-site/.cohesivity` on the runner.

## Guidelines

- One page someone with no context can load and understand in 10 seconds. A single `index.html` is almost always enough.
- Every run creates a new tenant, because runners don't keep files between runs. An earlier tenant expires on its own schedule.
- If a Cohesivity call returns an unexpected error, send feedback with the same `jq ... | curl -H @-` header: `GET https://cohesivity.ai/api/feedback` for the format, then `POST` to the same URL with the route, the error, and what you were doing. No keys or user data.
- Claiming, payment, and anything destructive are the operator's call. This skill only sends the claim link.
- Hosting API reference, if a response doesn't match this file: https://cohesivity.ai/offerings/railway-hosting (send `Accept: text/markdown`).

## Network note

Step 5 runs `npx`, which downloads `@cohesivity/init` from npm and creates the tenant on cohesivity.ai. Steps 6-8 call `https://cohesivity.ai/api/*` with plain `curl`. The management key is read from `.cohesivity` by `jq` and piped to `curl -H @-`, so no secret is on the command line. `./secretcurl` isn't used because the key is a file on the runner, not an environment variable. The live URL check in step 7 is a plain GET.
