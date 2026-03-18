---
name: setup-browser-cookies
description: Import cookies from your real browser into the headless browser session for authenticated testing. Use when asked to "import cookies", "set up browser cookies", "authenticate the browser", "log in the headless browser", or "import from Chrome/Arc/Brave". Works with Comet, Chrome, Arc, Brave, and Edge.
argument-hint: "[browser] [--domain <domain>]"
---

# /setup-browser-cookies

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../../CONNECTORS.md).

Import cookies from your real browser into the gstack headless session. Lets you test authenticated pages without logging in manually in the headless browser.

## Usage

```
/setup-browser-cookies                           ← opens interactive picker
/setup-browser-cookies chrome --domain .github.com
/setup-browser-cookies arc --domain myapp.com
/setup-browser-cookies brave --domain staging.myapp.com
```

## Setup Requirement

This skill requires the gstack browser binary. See the `/browse` skill for setup instructions.

**Verify binary:**
```bash
B=~/.claude/skills/gstack/browse/dist/browse
[ -x "$B" ] && echo "READY" || echo "NEEDS_SETUP: run cd ~/.claude/skills/gstack && ./setup"
```

## How It Works

The `cookie-import-browser` command reads encrypted cookies from your real browser's SQLite database and imports them into the headless Chromium session.

Supported browsers:
- **Comet** (Anthropic's browser)
- **Chrome**
- **Arc**
- **Brave**
- **Edge**

## Interactive Picker (recommended for first-time use)

```bash
$B cookie-import-browser
```

Opens a web UI in your browser where you can:
1. Select which browser to import from
2. Browse available domains and cookies
3. Choose which cookies to import

## Direct Import (for scripted workflows)

```bash
# Import all cookies for a domain from Chrome
$B cookie-import-browser chrome --domain .github.com

# Import from Arc for a specific domain
$B cookie-import-browser arc --domain myapp.com

# Import from Comet
$B cookie-import-browser comet --domain staging.yourapp.com
```

## After Import

Verify the import worked:
```bash
$B goto https://yourdomain.com
$B cookies                    # see all imported cookies
$B screenshot /tmp/auth-check.png   # verify you're logged in
```

Now you can use `/browse` and `/qa` to test authenticated pages:
```bash
$B goto https://github.com/settings/profile
$B snapshot -i
$B screenshot /tmp/github-profile.png
```

## Common Use Cases

### Test a private staging environment

```bash
/setup-browser-cookies chrome --domain staging.myapp.com
/qa https://staging.myapp.com
```

### Test authenticated GitHub pages

```bash
/setup-browser-cookies comet --domain .github.com
$B goto https://github.com/settings/profile
$B snapshot -i
```

### Test a logged-in production app

```bash
/setup-browser-cookies arc --domain myapp.com
$B goto https://myapp.com/dashboard
$B snapshot -i -a -o /tmp/dashboard-authenticated.png
```

## Tips

1. **Cookies expire.** If you get logged-out errors, re-run this skill.
2. **Domain matters.** Use `.domain.com` (with leading dot) to match all subdomains.
3. **Session cookies are included.** The import captures all cookies including session tokens.
4. **One import per domain.** Running the command again for the same domain replaces the previous cookies.
