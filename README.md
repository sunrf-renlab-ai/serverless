# zero-cost-deploy

A Claude Code skill (and human-readable playbook) for shipping a web app to production for **$0/month** on Vercel + Render + Supabase + Upstash free tiers.

> **MCP-first.** Uses the official Supabase / Vercel / GitHub MCP servers when available, falls back to bare HTTPS Management APIs for Render + Upstash (no MCP yet).

Encodes every form, every secret-handling pattern, every weird-network workaround, and every gotcha that has actually broken a real first-time deploy.

## Who this is for

- Indie hackers shipping a side project on a Friday night.
- Open-source maintainers who want a public demo without a hosting bill.
- Teams prototyping before committing to paid infra.
- **AI agents (this is a skill)** that need a reliable deploy playbook.

## Install — Claude Code

```bash
git clone git@github.com:sunrf-renlab-ai/zero-cost-deploy.git ~/.claude/skills/zero-cost-deploy
```

Claude Code auto-discovers skills under `~/.claude/skills/`. Next time you say "deploy this" / "ship to prod" / "免费上线", the skill activates.

## Install — Cursor / Cline / Continue / other MCP clients

Same clone, then either:
- Symlink the directory into your client's skill/instruction path
- Or point your client's instruction file at `SKILL.md` and the references

The skill itself uses MCP tools (`mcp__plugin_supabase_supabase__*`, etc.) — install the equivalent MCP servers in your client.

## Use without Claude (humans only)

Read `SKILL.md` top to bottom — it's the playbook. Then:

1. `scripts/gen-secrets.sh` → generate randoms.
2. Walk through ① Supabase → ② Upstash → ③ Render → ④ Vercel → ⑤ smoke test.
3. `scripts/rotate-secrets.md` if anything leaked to terminal output, chat, or commits.

Each `references/<service>.md` is a deeper dive when the SKILL.md summary isn't enough.

## What's in here

```
zero-cost-deploy/
├── SKILL.md                      # Skill entry — playbook, dense, MCP-first
├── README.md                     # you are here
├── LICENSE                       # MIT
├── references/
│   ├── mcp-tools.md              # MCP tool catalog per service + curl fallbacks
│   ├── supabase.md               # Management API, RLS, project pausing, key formats
│   ├── upstash.md                # Region selection, REST API, rate-limit pattern
│   ├── render.md                 # Blueprint quirks, Dockerfile patterns, cold starts
│   ├── vercel.md                 # Hobby limits, paste trick, Edge vs Node, custom domain
│   ├── management-apis.md        # Cheat sheet for every service's REST surface
│   ├── browser-fallbacks.md      # Last-resort UI scripting (rare, mostly "don't")
│   └── gotchas.md                # Catalog of real breakage + fixes
├── scripts/
│   ├── gen-secrets.sh            # pre-flight: generate random secrets
│   ├── supabase-push-migration.sh# push SQL via Management API (HTTPS-only)
│   ├── supabase-verify-tables.sh # verify tables + RLS landed
│   ├── upstash-create-redis.sh   # create Redis DB without fighting the UI
│   ├── verify-deploy.sh          # smoke test a fresh deploy
│   ├── vercel-add-domain.sh      # add custom domain + poll verification
│   └── rotate-secrets.md         # step-by-step rotation playbook
└── templates/
    ├── render.yaml               # Blueprint with safe defaults
    ├── Dockerfile                # multi-stage Bun image, fits 512 MB
    ├── vercel.json               # Hobby-safe (daily-only) cron + security headers
    ├── .env.example
    └── workflows/
        ├── ci.yml                # typecheck + lint + test on push/PR
        └── scheduled.yml         # sub-daily crons via GitHub Actions
```

## The stack — at a glance

| Service | What it does | Free-tier cap | MCP |
|---|---|---|---|
| **Supabase Free** | Postgres + Auth + RLS + Storage | 500 MB DB · 50K MAU · pauses after 7 days idle | ✅ |
| **Vercel Hobby** | Frontend, short API routes, daily cron | 100 GB BW · 60 s timeout · daily cron · no commercial use | ✅ (env vars excluded) |
| **Render Free** | Long-running services, WebSocket, > 60 s work | 512 MB RAM · sleeps after 15 min · 750 hr/month | ❌ |
| **Upstash Free** | Redis for rate-limit, locks, ephemeral state | 10K commands/day · 256 MB | ❌ |
| **GitHub Actions** | CI + sub-daily schedules | Unlimited on public repos | ✅ |
| **Sentry Free** (opt) | Errors + sourcemaps | 5K errors/month | n/a |

Monthly bill: **$0**. Optional custom domain: ~$10/year.

## Why this exists

Every "deploy your app for free" guide on the internet handwaves the broken parts. They tell you to "click the deploy button" — they don't tell you about:

- Vercel's Hobby plan rejecting sub-daily crons after you've configured your whole app around them.
- Render's Blueprint silently saving empty environment variables, leaving your service crashlooping.
- Clash/Mihomo fakeip blocking Supabase's `db.<ref>.supabase.co:5432` so `supabase db push` mysteriously times out.
- GitHub's OAuth Authorize button looking enabled but ignoring clicks for 2-4 seconds after page load.
- Vercel CLI in GitHub Actions failing with `Must not contain "***"` because a trailing newline in your secret made GitHub's redaction regex eat half the command line.
- The new `sb_publishable_*` / `sb_secret_*` key format breaking SDKs older than 2 months.

Each entry in `references/gotchas.md` is a real "I lost an hour to this" moment. The patterns in SKILL.md are the workarounds that work.

## Universal patterns it teaches you

1. **MCP > Management API > UI automation.** Use MCP tools where they exist; fall back to bare HTTPS API; only touch UIs as a last resort.
2. **ClipboardEvent for paste-aware React forms.** The only synthetic event React's value tracker accepts (Vercel env vars).
3. **OAuth identity vs GitHub App permissions are separate.** Render/Vercel both do this; confusing them is the most common "no repos found" cause.
4. **Network-aware fallbacks.** TUN-mode proxies break raw TCP. Use HTTPS APIs.
5. **Anti-bot delays exist on real buttons.** GitHub OAuth, hCaptcha, Stripe — don't fight them, fall back to manual click.

## When NOT to use this stack

- Sub-second cold start matters (Render Free sleeps; 30–60 s wake).
- Commercial monetization on Vercel Hobby (banned by ToS — use Cloudflare Pages instead).
- Postgres > 500 MB or > 50K MAU (upgrade Supabase or shard).
- Heavy compute > 512 MB RAM (upgrade Render or use Cloudflare Workers for stateless).
- Multi-region writes (everything here is single-region on free).

See `SKILL.md` "Skip when" for the full list.

## Contributing

Hit a gotcha that isn't documented? Open a PR adding it to `references/gotchas.md`. Pattern that should be in the universal patterns section? Same.

The bar: every entry must be a real issue you (or someone you trust) actually hit. No theoretical "what if X happens" — only "X happened, here's the symptom and the fix."

## License

MIT. See `LICENSE`.
