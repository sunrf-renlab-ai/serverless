# Browser fallbacks — when UI scripting is unavoidable

## The rule

**Don't.** Every service in this stack has a Management API or MCP that replaces what you'd do in the browser. The 5% of cases that genuinely need a browser:

1. First-time API key / PAT generation (no API can issue an API token).
2. Email verification link clicks.
3. OAuth Authorize flows.
4. Reading a value the UI shows once (Render DB password, etc.).

Everything else: API.

## Approaches, ranked

### 1. Tell the user (preferred)

Copy-pasteable instructions beat any automation. 30 s of user time, 100% reliable.

> Open https://supabase.com/dashboard/account/tokens
> Click **Generate new token** → name "deploy"
> Copy `sbp_…` back here.

### 2. Inject JS via osascript (when scripting saves real time)

Prereq: Chrome → View → Developer → **Allow JavaScript from Apple Events** (one-time per machine).

```bash
cat > /tmp/click.js <<'JS'
const btn = [...document.querySelectorAll('button')]
  .find(b => b.textContent.trim() === 'Deploy');
if (!btn) throw new Error('no Deploy button');
if (btn.disabled) throw new Error('disabled — likely anti-bot delay');
btn.click();
'ok'
JS

osascript -e 'on run argv
  set js to (read POSIX file (item 1 of argv))
  tell application "Google Chrome" to tell front window to execute active tab javascript js
end run' /tmp/click.js
```

Runs against the real DOM with real cookies. Sites can't distinguish this from DevTools.

### 3. Playwright + stealth (almost never the answer)

If even real-Chrome-via-osascript is blocked, the service has a Management API. Use that instead.

## What JS alone cannot do

Needs OS-level Accessibility (which page-context JS lacks):

- Real keyboard events (`isTrusted: true` keystrokes)
- Mouse-path simulation (some anti-bots check non-instant cursor movement)
- Trusted clicks during anti-bot disable periods (GitHub OAuth Authorize button, 2-4 s post-load)
- System dialogs (file picker, permission prompts)

For these: ask the user. A 30-second human interrupt beats a 30-minute Playwright-stealth detour.

## Patterns that consistently work

```js
// Find a button by exact text
const btn = [...document.querySelectorAll('button')]
  .find(b => b.textContent.trim() === 'Deploy');

// Multi-line env paste (Vercel only — relies on Vercel's special paste handler)
const dt = new DataTransfer();
dt.setData('text/plain', envBlock);
input.dispatchEvent(new ClipboardEvent('paste',
  { clipboardData: dt, bubbles: true, cancelable: true }));

// Submit a form whose visible button has nested span structure
// (GitHub OAuth account picker is the worst offender)
document.querySelector('form[action^="/login/oauth/authorize"]')?.submit();

// Read a value shown once (Render DB password, Vercel build secret)
copy(document.querySelector('[data-test=db-password]').textContent);
```

## Network caveats

If you're behind a TUN-mode proxy (Clash / Mihomo with fakeip):
- osascript-launched processes inherit system proxy.
- Direct TCP to non-HTTPS ports (5432, 6379) fails.
- HTTPS works.
- Chrome's network stack uses the same proxy as the system.

When "works in DevTools, fails from osascript":
1. JS-from-Apple-Events not enabled in Chrome.
2. Wrong window — use `front window`, not all windows.
3. JS throws silently — wrap in try/catch + return error.

## Bottom line

Every minute you script a UI is a minute you could read API docs. The API path is shorter 90% of the time and survives dashboard redesigns.
