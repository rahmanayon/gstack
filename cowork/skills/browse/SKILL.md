---
name: browse
description: Headless browser automation for QA testing and site dogfooding. Use when asked to "browse", "navigate to", "test this URL", "take a screenshot", "click through the app", "check this page", or "verify the deployment". Navigate any URL, interact with elements, verify page state, diff before/after actions, take annotated screenshots. Requires gstack browser binary — see Setup section.
argument-hint: "<URL or browser command>"
---

# /browse

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../../CONNECTORS.md).

Persistent headless Chromium. First call auto-starts (~3s), then ~100-200ms per command. Auto-shuts down after 30 min idle. State persists between calls (cookies, tabs, sessions).

## Usage

```
/browse <command>
/browse goto https://example.com
/browse snapshot -i
/browse screenshot /tmp/result.png
```

## Setup (required before first use)

The `/browse` skill requires the gstack browser binary. This is a one-time build (~10 seconds):

```bash
git clone https://github.com/rahmanayon/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup
```

Requirements: [Bun](https://bun.sh/) v1.0+, macOS or Linux (x64 or arm64).

**Before any browse command, check if the binary is ready:**

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/browse/dist/browse" ] && B="$_ROOT/.claude/skills/gstack/browse/dist/browse"
[ -z "$B" ] && B=~/.claude/skills/gstack/browse/dist/browse
if [ -x "$B" ]; then
  echo "READY: $B"
else
  echo "NEEDS_SETUP"
fi
```

If `NEEDS_SETUP`: Tell the user the setup command above. Ask if it's OK to run. Then run it.

## IMPORTANT

- Use the compiled binary via shell: `$B <command>`
- Browser persists between calls — cookies, login sessions, and tabs carry over.
- Dialogs (alert/confirm/prompt) are auto-accepted by default — no browser lockup.

## QA Workflows

### Test a user flow (login, signup, checkout, etc.)

```bash
# 1. Go to the page
$B goto https://app.example.com/login

# 2. See what's interactive
$B snapshot -i

# 3. Fill the form using refs
$B fill @e3 "test@example.com"
$B fill @e4 "password123"
$B click @e5

# 4. Verify it worked
$B snapshot -D              # diff shows what changed after clicking
$B is visible ".dashboard"  # assert the dashboard appeared
$B screenshot /tmp/after-login.png
```

### Verify a deployment / check prod

```bash
$B goto https://yourapp.com
$B text                          # read the page — does it load?
$B console                       # any JS errors?
$B network                       # any failed requests?
$B js "document.title"           # correct title?
$B is visible ".hero-section"    # key elements present?
$B screenshot /tmp/prod-check.png
```

### Dogfood a feature end-to-end

```bash
$B goto https://app.example.com/new-feature

# Take annotated screenshot — shows every interactive element with labels
$B snapshot -i -a -o /tmp/feature-annotated.png

# Find ALL clickable things (including divs with cursor:pointer)
$B snapshot -C

# Walk through the flow
$B snapshot -i          # baseline
$B click @e3            # interact
$B snapshot -D          # what changed? (unified diff)

# Check element states
$B is visible ".success-toast"
$B is enabled "#next-step-btn"
$B is checked "#agree-checkbox"

# Check console for errors after interactions
$B console
```

### Test responsive layouts

```bash
$B goto https://yourapp.com
$B responsive /tmp/layout    # 3 screenshots: mobile/tablet/desktop

# Manual viewports
$B viewport 375x812     # iPhone
$B screenshot /tmp/mobile.png
$B viewport 1440x900    # Desktop
$B screenshot /tmp/desktop.png
```

### Test forms with validation

```bash
$B goto https://app.example.com/form
$B snapshot -i

# Submit empty — check validation errors appear
$B click @e10                        # submit button
$B snapshot -D                       # diff shows error messages appeared
$B is visible ".error-message"

# Fill and resubmit
$B fill @e3 "valid input"
$B click @e10
$B snapshot -D                       # diff shows errors gone, success state
```

### Multi-step chain (efficient for long flows)

```bash
echo '[
  ["goto","https://app.example.com"],
  ["snapshot","-i"],
  ["fill","@e3","test@test.com"],
  ["fill","@e4","password"],
  ["click","@e5"],
  ["snapshot","-D"],
  ["screenshot","/tmp/result.png"]
]' | $B chain
```

## Quick Assertion Patterns

```bash
$B is visible ".modal"
$B is enabled "#submit-btn"
$B is disabled "#submit-btn"
$B is checked "#agree"
$B is editable "#name-field"
$B is focused "#search-input"
$B js "document.body.textContent.includes('Success')"
$B js "document.querySelectorAll('.list-item').length"
$B attrs "#logo"
$B css ".button" "background-color"
```

## Snapshot System

The snapshot is your primary tool for understanding and interacting with pages.

```
-i        --interactive           Interactive elements only (buttons, links, inputs) with @e refs
-c        --compact               Compact (no empty structural nodes)
-d <N>    --depth                 Limit tree depth
-s <sel>  --selector              Scope to CSS selector
-D        --diff                  Unified diff against previous snapshot
-a        --annotate              Annotated screenshot with red overlay boxes and ref labels
-o <path> --output                Output path for annotated screenshot
-C        --cursor-interactive    Cursor-interactive elements (@c refs)
```

All flags can be combined freely. Example: `$B snapshot -i -a -C -o /tmp/annotated.png`

After snapshot, use @refs as selectors in any command:
```bash
$B click @e3       $B fill @e4 "value"     $B hover @e1
$B html @e2        $B css @e5 "color"      $B attrs @e6
```

## Command Reference

### Navigation
| Command | Description |
|---------|-------------|
| `goto <url>` | Navigate to URL |
| `back` | History back |
| `forward` | History forward |
| `reload` | Reload page |
| `url` | Print current URL |

### Reading
| Command | Description |
|---------|-------------|
| `text` | Cleaned page text |
| `html [selector]` | innerHTML of selector or full page |
| `links` | All links as "text → href" |
| `forms` | Form fields as JSON |
| `accessibility` | Full ARIA tree |

### Interaction
| Command | Description |
|---------|-------------|
| `click <sel>` | Click element |
| `fill <sel> <val>` | Fill input |
| `type <text>` | Type into focused element |
| `hover <sel>` | Hover element |
| `press <key>` | Press key (Enter, Tab, Escape, ArrowUp/Down, etc.) |
| `select <sel> <val>` | Select dropdown option |
| `upload <sel> <file>` | Upload file(s) |
| `scroll [sel]` | Scroll element into view or page bottom |
| `wait <sel|--networkidle|--load>` | Wait for element or page state |
| `dialog-accept [text]` | Auto-accept next dialog |
| `dialog-dismiss` | Auto-dismiss next dialog |
| `cookie <name>=<value>` | Set cookie |
| `cookie-import <json>` | Import cookies from JSON file |
| `cookie-import-browser [browser] [--domain d]` | Import from real browser |
| `header <name>:<value>` | Set custom request header |
| `useragent <string>` | Set user agent |
| `viewport <WxH>` | Set viewport size |

### Inspection
| Command | Description |
|---------|-------------|
| `snapshot [flags]` | Accessibility tree with @e refs |
| `screenshot [path]` | Save screenshot |
| `console [--clear|--errors]` | Console messages |
| `network [--clear]` | Network requests |
| `js <expr>` | Run JavaScript expression |
| `eval <file>` | Run JavaScript from file |
| `is <prop> <sel>` | State check (visible/hidden/enabled/etc.) |
| `attrs <sel>` | Element attributes as JSON |
| `css <sel> <prop>` | Computed CSS value |
| `cookies` | All cookies as JSON |
| `storage [set k v]` | Read/write localStorage + sessionStorage |
| `perf` | Page load timings |
| `dialog [--clear]` | Dialog messages |

### Visual
| Command | Description |
|---------|-------------|
| `responsive [prefix]` | Screenshots at mobile/tablet/desktop |
| `diff <url1> <url2>` | Text diff between pages |
| `pdf [path]` | Save as PDF |

### Tabs
| Command | Description |
|---------|-------------|
| `tabs` | List open tabs |
| `newtab [url]` | Open new tab |
| `tab <id>` | Switch to tab |
| `closetab [id]` | Close tab |

### Server
| Command | Description |
|---------|-------------|
| `status` | Health check |
| `restart` | Restart server |
| `stop` | Shutdown server |

### Meta
| Command | Description |
|---------|-------------|
| `chain` | Run commands from JSON stdin |

## Tips

1. **Navigate once, query many times.** `goto` loads the page; then `text`, `js`, `screenshot` all hit the loaded page instantly.
2. **Use `snapshot -i` first.** See all interactive elements, then click/fill by ref. No CSS selector guessing.
3. **Use `snapshot -D` to verify.** Baseline → action → diff. See exactly what changed.
4. **Use `is` for assertions.** `is visible .modal` is faster than parsing page text.
5. **Use `snapshot -a` for evidence.** Annotated screenshots are great for bug reports.
6. **Use `snapshot -C` for tricky UIs.** Finds clickable divs that the accessibility tree misses.
7. **Check `console` after actions.** Catch JS errors that don't surface visually.
8. **Use `chain` for long flows.** Single command, no per-step overhead.
