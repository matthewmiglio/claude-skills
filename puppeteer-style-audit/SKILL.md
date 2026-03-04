---
name: puppeteer-style-audit
description: >
  Perform a comprehensive visual and CSS audit of any website using Puppeteer MCP tools.
  Use when the user wants to audit, review, or check the styling, layout, responsiveness,
  or visual quality of a web page or local web project. Supports live URLs and local repos
  (auto-detects framework and starts a dev server).
argument-hint: "<repo path or URL>"
disable-model-invocation: true
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Edit
  - Grep
  - Glob
  - mcp__puppeteer__puppeteer_navigate
  - mcp__puppeteer__puppeteer_screenshot
  - mcp__puppeteer__puppeteer_evaluate
  - mcp__puppeteer__puppeteer_click
  - mcp__puppeteer__puppeteer_hover
  - mcp__puppeteer__puppeteer_fill
---

# Puppeteer Style Audit

Perform a comprehensive visual and CSS audit of a website. Checks layout, typography, images, navigation, forms, and responsiveness across three viewports.

## Process

### Step 1 — Parse Input

Read `$ARGUMENTS` and determine the target:

- **URL** (starts with `http`): use directly, skip to Step 3.
- **Repo path** (local directory): proceed to Step 2.
- **No argument**: check if CWD contains a web project (`package.json`, `index.html`, `app.py`, `manage.py`). If found, use CWD. Otherwise ask the user for a target.

### Step 2 — Start Dev Server (repo path only)

Detect the framework and start the appropriate dev server in background.

**Framework detection** — check these files at the repo root:

| Check | Framework | Command | Default Port |
|---|---|---|---|
| `package.json` has `"next"` | Next.js | `npm run dev` | 3000 |
| `package.json` has `"vite"` | Vite | `npm run dev` | 5173 |
| `package.json` has `"react-scripts"` | CRA | `npm start` | 3000 |
| `package.json` has `"vue"` | Vue CLI | `npm run serve` | 8080 |
| `app/__init__.py` or `app.py` | Flask | see below | 5000 |
| `manage.py` | Django | `python manage.py runserver` | 8000 |
| `index.html` at root (no package.json) | Static | `python -m http.server 8080` | 8080 |

**Flask-specific**: source `.env` if it exists, then try `python run.py` first; fall back to `flask run --port 5000`.

**Startup procedure**:
1. `cd` to the repo path.
2. If `package.json` exists and `node_modules/` does not, run `npm install` first.
3. Start the server in background using Bash `run_in_background`.
4. Wait up to 30 seconds for the server, polling with `curl -s -o /dev/null -w "%{http_code}" http://localhost:<PORT>/` every 2 seconds.
5. Record the PID and port for cleanup.
6. If startup fails after 30s, report the error and stop.

Set `BASE_URL=http://localhost:<PORT>`.

### Step 3 — Discover Pages

Try these strategies in order until you have a page list:

**Strategy A — Sitemap**: Navigate to `<BASE_URL>/sitemap.xml`. If it returns XML, extract all `<loc>` URLs.

**Strategy B — Crawl homepage links**: Navigate to `<BASE_URL>`, then run `puppeteer_evaluate` with:
```js
JSON.stringify(
  [...new Set(
    [...document.querySelectorAll('a[href]')]
      .map(a => a.href)
      .filter(h => h.startsWith(location.origin) && !h.includes('#'))
  )]
)
```

**Strategy C — Framework route inspection**:
- Flask: `grep -r "@app.route\|@bp.route\|@blueprint.route" <repo>` to extract route paths
- Next.js: list files under `pages/` or `app/` directories
- Django: grep `urlpatterns` in `urls.py` files

Always include the homepage. **Safety limit: max 20 pages.** If more are found, keep the first 20 and note the truncation.

### Step 4 — Audit Each Page

**Viewports:**
- Mobile: 375 x 812
- Tablet: 768 x 1024
- Desktop: 1440 x 900

For each page + viewport combination:

#### 4a. Navigate and Screenshot
1. `puppeteer_navigate` to the page URL.
2. `puppeteer_screenshot` with `encoded: true`, `width` and `height` set to the viewport dimensions, and a descriptive `name` like `"home-mobile"`.
3. Visually inspect the screenshot for issues JS cannot detect: aesthetic misalignment, ugly spacing, font rendering problems, content covering other content, general visual quality.

#### 4b. Run Automated Checks
Run `puppeteer_evaluate` with the comprehensive audit script below. It returns a JSON array of issue objects.

```js
(() => {
  const issues = [];
  const add = (cat, sev, msg, el) => {
    const loc = el ? (el.tagName?.toLowerCase() + (el.id ? '#'+el.id : '') + (el.className && typeof el.className === 'string' ? '.'+el.className.trim().split(/\s+/).join('.') : '')) : '';
    issues.push({ category: cat, severity: sev, message: msg, element: loc });
  };

  // --- Layout ---
  const dw = document.documentElement.clientWidth;
  document.querySelectorAll('body *').forEach(el => {
    const r = el.getBoundingClientRect();
    if (r.width > dw + 5) add('Layout', 'high', `Element overflows viewport horizontally by ${Math.round(r.width - dw)}px`, el);
    if (r.right > dw + 5 && el.scrollWidth > el.clientWidth + 5) add('Layout', 'medium', 'Content spills outside its container', el);
  });

  // Collapsed containers (has children but zero height)
  document.querySelectorAll('div, section, main, article').forEach(el => {
    if (el.children.length > 0 && el.getBoundingClientRect().height === 0) {
      add('Layout', 'medium', 'Container has children but zero height (collapsed)', el);
    }
  });

  // --- Typography ---
  document.querySelectorAll('p, span, a, li, td, th, label, input, button, h1, h2, h3, h4, h5, h6').forEach(el => {
    const s = getComputedStyle(el);
    const fs = parseFloat(s.fontSize);
    if (fs > 0 && fs < 10) add('Typography', 'high', `Text too small: ${fs}px`, el);
  });

  // Contrast check (simplified WCAG AA)
  const luminance = (r, g, b) => {
    const [rs, gs, bs] = [r, g, b].map(c => { c /= 255; return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4); });
    return 0.2126 * rs + 0.7152 * gs + 0.0722 * bs;
  };
  const parseColor = (c) => {
    const m = c.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)/);
    return m ? [+m[1], +m[2], +m[3]] : null;
  };
  document.querySelectorAll('p, span, a, li, h1, h2, h3, h4, h5, h6').forEach(el => {
    const s = getComputedStyle(el);
    const fg = parseColor(s.color);
    const bg = parseColor(s.backgroundColor);
    if (fg && bg) {
      const l1 = luminance(...fg), l2 = luminance(...bg);
      const ratio = (Math.max(l1, l2) + 0.05) / (Math.min(l1, l2) + 0.05);
      if (ratio < 4.5 && bg[0] + bg[1] + bg[2] < 760) {
        add('Typography', 'high', `Poor contrast ratio ${ratio.toFixed(1)}:1 (need 4.5:1)`, el);
      }
    }
  });

  // --- Images ---
  document.querySelectorAll('img').forEach(img => {
    if (img.complete && img.naturalWidth === 0) add('Images', 'high', 'Broken image (failed to load)', img);
    if (img.naturalWidth > 0 && img.width > 0 && img.height > 0) {
      const natRatio = img.naturalWidth / img.naturalHeight;
      const dispRatio = img.width / img.height;
      if (Math.abs(natRatio - dispRatio) > 0.2) add('Images', 'medium', `Distorted aspect ratio (natural: ${natRatio.toFixed(2)}, displayed: ${dispRatio.toFixed(2)})`, img);
    }
    if (!img.alt && !img.getAttribute('role')?.includes('presentation')) add('Images', 'low', 'Missing alt text', img);
  });

  // Missing favicon
  if (!document.querySelector('link[rel*="icon"]')) add('Images', 'low', 'No favicon defined', null);

  // --- Navigation ---
  const fixed = [...document.querySelectorAll('*')].filter(el => {
    const p = getComputedStyle(el).position;
    return p === 'fixed' || p === 'sticky';
  });
  for (let i = 0; i < fixed.length; i++) {
    for (let j = i + 1; j < fixed.length; j++) {
      const r1 = fixed[i].getBoundingClientRect();
      const r2 = fixed[j].getBoundingClientRect();
      if (r1.left < r2.right && r1.right > r2.left && r1.top < r2.bottom && r1.bottom > r2.top) {
        add('Navigation', 'medium', 'Overlapping fixed/sticky elements', fixed[i]);
      }
    }
  }

  // --- Forms ---
  document.querySelectorAll('input, select, textarea').forEach(el => {
    if (el.type === 'hidden' || el.type === 'submit' || el.type === 'button') return;
    const id = el.id;
    const hasLabel = (id && document.querySelector(`label[for="${id}"]`)) ||
                     el.closest('label') ||
                     el.getAttribute('aria-label') ||
                     el.getAttribute('aria-labelledby');
    if (!hasLabel) add('Forms', 'medium', 'Input without associated label', el);
  });

  // --- Page-level ---
  // Footer not at bottom
  const footer = document.querySelector('footer');
  if (footer) {
    const fb = footer.getBoundingClientRect();
    const vh = window.innerHeight;
    const bodyH = document.body.scrollHeight;
    if (bodyH < vh && fb.bottom < vh - 20) {
      add('Page', 'low', 'Footer not at bottom of viewport on short page', footer);
    }
  }

  // Tables overflowing
  document.querySelectorAll('table').forEach(t => {
    if (t.scrollWidth > t.parentElement?.clientWidth + 5) {
      add('Page', 'medium', 'Table overflows its container', t);
    }
  });

  // Code blocks overflowing
  document.querySelectorAll('pre, code').forEach(el => {
    if (el.scrollWidth > el.clientWidth + 5) {
      add('Page', 'low', 'Code block overflows its container', el);
    }
  });

  // Dedup
  const seen = new Set();
  const deduped = issues.filter(i => {
    const key = i.category + i.message + i.element;
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });

  return JSON.stringify(deduped.slice(0, 50));
})()
```

#### 4c. Combine Findings
Merge visual observations from the screenshot with automated JS issues. Note which issues are visual-only (spotted from screenshot) vs automated.

**Error handling**: If navigation fails for a page, retry up to 3 times. If it still fails, skip that page and continue.

### Step 5 — Kill Dev Server

If a dev server was started in Step 2, kill it:
- Windows: `taskkill //F //PID <PID>`
- Unix: `kill <PID>`

### Step 6 — Present Report

Format the report as follows:

#### Summary Header
```
## Style Audit Report
- **Target**: <URL or repo path>
- **Pages audited**: <N>
- **Issues found**: <total> (🔴 <high> critical, 🟡 <medium> warning, 🔵 <low> info)
```

#### Issues by Page
For each page, group by viewport, then by category:

```
### /page-name

#### Mobile (375x812)
**Layout**
- 🔴 Element `.hero-banner` overflows viewport by 120px → *Fix: add `max-width: 100%; overflow-x: hidden` to the container*

**Typography**
- 🟡 Poor contrast 2.8:1 on `p.subtitle` → *Fix: darken text to at least #595959 on white background*

#### Desktop (1440x900)
(issues...)
```

Each issue includes:
- Severity indicator (🔴 high / 🟡 medium / 🔵 low)
- What the issue is and which element
- A concrete fix suggestion

If a page+viewport has no issues, note "No issues found" for that combination.

End with a **Top Priorities** section listing the 5 most impactful fixes.

## Safety Limits

- Max **20 pages** per audit
- Max **3 navigation retries** per page
- **30 second** dev server startup timeout
- **50 issues** cap per page+viewport combination
- Skip and continue on any individual page failure
- Always kill the dev server in Step 5, even if the audit encounters errors (use try/finally logic)
