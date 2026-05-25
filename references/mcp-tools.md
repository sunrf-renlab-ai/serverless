# MCP tool catalog

What's available via MCP per service, and the curl fallback when MCP isn't.

## Detect what's connected

```bash
claude mcp list 2>&1 | grep -E "supabase|vercel|github"
```

Connected entries show ✓ status. Missing or ✗ → install / re-auth.

## Install a missing MCP (Claude Code)

```bash
claude plugin install supabase@claude-plugins-official
claude plugin install vercel@claude-plugins-official
claude plugin install github@claude-plugins-official
# restart Claude Code; first call to the tool triggers auth
```

GitHub MCP requires `GITHUB_PERSONAL_ACCESS_TOKEN` env. Resolve from gh:

```bash
jq '.env.GITHUB_PERSONAL_ACCESS_TOKEN = "'"$(gh auth token)"'"' \
  ~/.claude/settings.json > /tmp/s.json && mv /tmp/s.json ~/.claude/settings.json
# restart Claude Code
```

---

## Supabase MCP (full coverage)

Tool prefix: `mcp__plugin_supabase_supabase__`

| Tool | What it does | curl equivalent |
|---|---|---|
| `list_organizations` | List orgs you belong to | `GET /v1/organizations` |
| `list_projects` | List projects in an org | `GET /v1/projects` |
| `create_project` | New project | `POST /v1/projects` |
| `get_project` | Project metadata | `GET /v1/projects/<ref>` |
| `get_project_url` | Project URL | `GET /v1/projects/<ref>` |
| `get_publishable_keys` | Browser-safe `sb_publishable_…` | `GET /v1/projects/<ref>/api-keys` |
| `apply_migration` | Versioned SQL migration | `POST /v1/projects/<ref>/database/query` |
| `execute_sql` | Run arbitrary SQL | same as above |
| `list_tables` | Public tables | `pg_tables` SQL |
| `list_migrations` | History | `GET /v1/projects/<ref>/database/migrations` |
| `list_extensions` | Postgres extensions | `pg_extension` SQL |
| `get_logs` | Service logs | `GET /v1/projects/<ref>/analytics/endpoints/logs.all` |
| `get_advisors` | RLS + perf warnings | `GET /v1/projects/<ref>/advisors` |
| `generate_typescript_types` | DB-derived TS types | `GET /v1/projects/<ref>/types/typescript` |
| `deploy_edge_function` | Deno function deploy | `POST /v1/projects/<ref>/functions/deploy` |
| `list_edge_functions` | List functions | `GET /v1/projects/<ref>/functions` |
| `get_edge_function` | Function source | `GET /v1/projects/<ref>/functions/<slug>` |
| `create_branch` / `merge_branch` / `rebase_branch` / `reset_branch` / `delete_branch` / `list_branches` | DB preview branches (Pro+) | `/v1/projects/<ref>/branches/...` |
| `pause_project` / `restore_project` | Pause/wake | `/v1/projects/<ref>/{pause,restore}` |
| `search_docs` | Search Supabase docs | n/a |
| `get_cost` / `confirm_cost` | Cost preview before destructive op | n/a |

**The secret key** (`sb_secret_…`) is NOT exposed by MCP. Get from dashboard → **Settings → API → Data API**, or `GET /v1/projects/<ref>/api-keys` with `?reveal=true`.

---

## Vercel MCP (deploy + read; env vars not exposed)

Tool prefix: `mcp__plugin_vercel_vercel__`

| Tool | What it does | curl equivalent |
|---|---|---|
| `list_teams` | Your teams | `GET /v2/teams` |
| `list_projects` | Projects in a team | `GET /v9/projects` |
| `get_project` | Project metadata | `GET /v9/projects/<id>` |
| `deploy_to_vercel` | Deploy current repo | `POST /v13/deployments` |
| `get_deployment` | Deployment details | `GET /v13/deployments/<id>` |
| `list_deployments` | Recent deployments | `GET /v6/deployments` |
| `get_deployment_build_logs` | Build log stream | `GET /v3/deployments/<id>/events` |
| `get_runtime_logs` | Function logs | `GET /v3/deployments/<id>/runtime-logs` |
| `check_domain_availability_and_price` | Domain shop | `GET /v4/domains/status` |
| `search_vercel_documentation` | Docs search | n/a |
| `get_access_to_vercel_url` | Bypass token for private deploys | n/a |
| `web_fetch_vercel_url` | Fetch with bypass token applied | n/a |
| `*_toolbar_*` | Comment/thread tools (vercel toolbar) | n/a |

**Env vars need API or dashboard.** Bulk paste via dashboard ClipboardEvent (see `references/vercel.md`); or API loop:

```bash
# Bulk add from KEY=VALUE file
while IFS='=' read -r k v; do
  [ -z "$k" ] || [ "${k#\#}" != "$k" ] && continue   # skip blanks/comments
  curl -sS -X POST -H "Authorization: Bearer $VERCEL_TOKEN" \
    -H "Content-Type: application/json" \
    "https://api.vercel.com/v10/projects/$PROJECT_ID/env" \
    -d "$(jq -nc --arg k "$k" --arg v "$v" \
      '{key:$k,value:$v,type:"encrypted",target:["production","preview","development"]}')"
done < /tmp/secrets.env
```

---

## GitHub MCP (full coverage via Copilot endpoint)

Tool prefix varies by client; in Claude Code it surfaces as `mcp__plugin_github_github__` once connected.

Covers everything `gh` does (issues, PRs, releases, Actions, repo management). Treat it as a typed `gh` wrapper.

Common ops the deploy flow uses:

| Need | MCP / `gh` |
|---|---|
| Set Actions secret | `gh secret set NAME --body "$VALUE" --repo OWNER/REPO` |
| Set Actions variable | `gh variable set NAME --body "$VALUE" --repo OWNER/REPO` |
| Trigger workflow | `gh workflow run "Deploy" --repo OWNER/REPO --ref main` |
| Check run status | `gh run list --repo OWNER/REPO --workflow deploy.yml --limit 1` |
| Tag release | `gh release create v0.1.0 --generate-notes` |

**Gotcha** — never `echo "$X" \| gh secret set --body -`. The trailing newline ends up in the value and Vercel CLI then rejects the secret with `Must not contain "***"` because GitHub redacts every `-` in subsequent runner output. Always use `--body "$X"` directly.

---

## No MCP yet — Render

Auth: `Authorization: Bearer <RENDER_API_KEY>` (dashboard.render.com/u/settings#api-keys)

Base: `https://api.render.com/v1`

| Need | curl |
|---|---|
| List services | `GET /services` |
| Get service | `GET /services/<id>` |
| Trigger deploy | `POST /services/<id>/deploys` `-d '{"clearCache":false}'` |
| Read env vars | `GET /services/<id>/env-vars` |
| Replace env vars | `PUT /services/<id>/env-vars` `-d '[{"key":"X","value":"y"}, …]'` |
| Tail logs | `GET /services/<id>/logs?ownerId=<owner>&limit=50` |

Why this matters: the Blueprint UI silent-saves empty env on certain layouts. **Always verify via API after first apply**, fix via API too.

---

## No MCP yet — Upstash

Auth: HTTP Basic with `<email>:<api_key>` (key at console.upstash.com/account/api)

Base: `https://api.upstash.com/v2`

| Need | curl |
|---|---|
| Create Redis | `POST /redis/database` `-d '{"name":"x","region":"global","primary_region":"us-east-1","tls":true}'` |
| List | `GET /redis/databases` |
| Details | `GET /redis/database/<id>` |
| Delete | `DELETE /redis/database/<id>` |
| Rotate REST token | `POST /redis/database/<id>/rotate-rest-token` |
| Stats | `GET /redis/database/<id>/stats` |

Kafka, QStash, Vector each get parallel namespaces (`/v2/kafka/…`, `/v2/qstash/…`).

---

## When MCP fails

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to connect` on `claude mcp list` | OAuth token expired or DCR not supported | `claude plugin uninstall X@claude-plugins-official && claude plugin install X@claude-plugins-official`; restart |
| `Authorization header is badly formatted` (GitHub MCP) | `GITHUB_PERSONAL_ACCESS_TOKEN` env var unset | Set in `~/.claude/settings.json` `env` (see top of this file) |
| MCP tool returns "not authorized" | Token scope insufficient | Generate token with required scope, re-set env |
| MCP times out on a long call | Server-side rate limit or single-request size | Fall back to `curl` against the same endpoint — bypasses MCP framing |

When in doubt: the underlying REST API is always there. MCP is a convenience layer.
