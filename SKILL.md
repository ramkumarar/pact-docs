---
name: api-browser-debug
description: Capture authenticated API traffic from any website using Chrome remote debugging and apitap attach. Triggers on: capture protected API, attach apitap to browser, authenticated API capture, JWT capture, login session capture, chrome remote debugging, apitap attach, capture API behind login, intercept authenticated requests, CDP capture, debug chrome port 9222.
---

# API Browser Debug — Capture Authenticated APIs with apitap

Use this when a site requires login or uses JWT/cookie-based auth that can't be captured headlessly. apitap attaches to a real Chrome session via CDP and intercepts all network traffic — no MITM proxy, no certificate setup, read-only.

## When to Use This

- The target site requires login (JWT, OAuth, session cookies)
- `apitap capture` or `apitap discover` returns no endpoints or low-confidence results
- You want to capture from your already-logged-in browser session
- The site has anti-bot protection that blocks headless Playwright

---

## Step 1 — Kill any existing Chrome on port 9222

Chrome only accepts `--remote-debugging-port` at launch time. If Chrome is already running, the flag is silently ignored and apitap attach will fail.

**Check if port 9222 is already bound:**
```cmd
netstat -ano | findstr :9222
```

**Kill Chrome if needed (saves all work first):**
```cmd
taskkill /F /IM chrome.exe
```

bash equivalent:
```bash
netstat -ano | grep 9222
taskkill //F //IM chrome.exe
```

---

## Step 2 — Launch Chrome with remote debugging

Uses a **separate profile** (`C:\temp\chrome-debug`) to avoid conflicting with your main Chrome.

**cmd:**
```cmd
start "" "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir=C:\temp\chrome-debug
```

**bash:**
```bash
start chrome --remote-debugging-port=9222 --user-data-dir="/c/temp/chrome-debug"

# If 'start chrome' doesn't resolve the binary, use the full path:
"/c/Program Files/Google/Chrome/Application/chrome.exe" --remote-debugging-port=9222 --user-data-dir="/c/temp/chrome-debug" &
```

Wait for Chrome to open before proceeding.

---

## Step 3 — Log in to the target site

Open the target site in the debug Chrome window and log in manually. apitap captures your real session tokens — no re-authentication needed in your code later.

**To test with DummyJSON** (public credentials, no account needed):

1. Navigate to `https://dummyjson.com`
2. Open DevTools (F12) → Console tab
3. Paste and run:

```js
// Login and get JWT
const login = await fetch('https://dummyjson.com/auth/login', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({ username: 'emilys', password: 'emilyspass' })
}).then(r => r.json());

console.log('Token:', login.accessToken);

// Hit a protected endpoint with the token
const me = await fetch('https://dummyjson.com/auth/me', {
  headers: { 'Authorization': `Bearer ${login.accessToken}` }
}).then(r => r.json());

console.log('Protected response:', me);
```

For any other site: log in through the UI normally and click around authenticated sections to generate API traffic.

---

## Step 4 — Attach apitap in a second terminal

```cmd
apitap attach --port 9222 --domain dummyjson.com
```

Replace `dummyjson.com` with your target domain. To capture all domains in the session (API subdomains, CDN, etc.):

```cmd
apitap attach --port 9222
```

apitap prints captured endpoints in real time as you interact with the site. Press `Ctrl+C` when done. The skill file saves to `~/.apitap/skills/<domain>.json`.

---

## Step 5 — Verify and replay

```cmd
:: List what was captured
apitap show dummyjson.com

:: Replay endpoints (credentials injected automatically)
apitap replay dummyjson.com post-auth-login
apitap replay dummyjson.com get-auth-me

:: Override parameters at replay time
apitap replay dummyjson.com get-products limit=5
```

Via MCP tools in Kiro:
```
apitap_search("dummyjson")
apitap_replay(domain="dummyjson.com", endpointId="get-auth-me")
```

---

## Adapting to Any Protected Site

| Variable | What to change |
|---|---|
| `--domain dummyjson.com` | Your target domain |
| Login step | Log in through the browser UI normally |
| `apitap show dummyjson.com` | Replace domain in all commands |
| `--user-data-dir` | Keep as-is — the debug profile persists across sessions |

The `C:\temp\chrome-debug` profile **persists between sessions** — if you logged in during a previous debug session, cookies are still there and you don't need to log in again.

---

## Troubleshooting

**`apitap attach` says "no debuggable targets"**
- Chrome wasn't started with `--remote-debugging-port=9222`
- Another Chrome instance is running without the flag — kill it and relaunch from Step 1

**Port 9222 in use but not by Chrome**
```cmd
netstat -ano | findstr :9222
:: Note the PID in the last column, then:
taskkill /F /PID <pid>
```

**Endpoints captured but tier is orange/red**
- The site uses request signing or Cloudflare bot protection
- Use `apitap replay ... --fresh` to trigger a browser-based token refresh before replaying

**No endpoints captured at all**
- Make sure you navigated to pages and triggered API calls while apitap was attached
- Try without `--domain` to capture everything and see what shows up
