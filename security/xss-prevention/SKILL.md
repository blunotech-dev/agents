---
name: xss-prevention
description: Audit frontend or backend for XSS vulnerabilities and fix them. Use this skill when the user mentions XSS, cross-site scripting, innerHTML, dangerouslySetInnerHTML, unsanitized input, output encoding, CSP headers, script injection, or asks "is this safe to render?" or "how do I sanitize user input?".
category: "Security"
---

# XSS Prevention

XSS executes attacker-controlled scripts in a victim's browser — stealing tokens, hijacking sessions, or silently making authenticated requests.

**Three types:**
- **Reflected** — payload in URL, echoed in response (`?q=<script>`)
- **Stored** — payload saved to DB, rendered to all viewers
- **DOM-based** — payload never hits the server; JS reads URL/fragment and writes to DOM

---

## Sink Inventory: Where XSS Happens

Every place user data reaches the DOM or HTTP response is a sink. Audit these first.

### Dangerous sinks (browser)

```ts
// ❌ All of these execute scripts
element.innerHTML = userInput;
element.outerHTML = userInput;
document.write(userInput);
element.insertAdjacentHTML('beforeend', userInput);

// ❌ React escape hatch
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ❌ URL sinks — javascript: protocol
element.href = userInput;          // <a href="javascript:alert(1)">
element.src = userInput;           // <img src="x" onerror="...">
location.href = userInput;

// ❌ Script execution sinks
eval(userInput);
setTimeout(userInput, 100);
new Function(userInput)();
```

### Safe alternatives

```ts
// ✅ Text only — never executes scripts
element.textContent = userInput;
element.setAttribute('data-value', userInput);

// ✅ React default — escaped automatically
<div>{userInput}</div>

// ✅ URL: validate scheme before assigning
function safeUrl(url: string): string {
  try {
    const u = new URL(url, location.origin);
    return ['https:', 'http:'].includes(u.protocol) ? url : '#';
  } catch { return '#'; }
}
element.href = safeUrl(userInput);
```

---

## When You Must Render HTML

If user-generated HTML is a product requirement (rich text editor, markdown renderer):

```ts
import DOMPurify from 'dompurify'; // browser
import { JSDOM } from 'jsdom';
const { window } = new JSDOM('');
const DOMPurify = createDOMPurify(window); // server-side

// ✅ Sanitize before inserting
element.innerHTML = DOMPurify.sanitize(userHtml);
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userHtml) }} />

// Restrict allowed tags if you know the shape
DOMPurify.sanitize(userHtml, { ALLOWED_TAGS: ['b', 'i', 'em', 'a', 'p', 'ul', 'li'] });
```

**Never sanitize on input/save — sanitize on output.** Sanitizing on save loses the original content and breaks if your sanitizer has bugs (can't re-sanitize with a fixed version).

---

## Server-Side: Output Encoding

Every templating engine has auto-escaping — ensure it's on.

```py
# Jinja2 — autoescape on by default for .html files, but be explicit
env = Environment(autoescape=True)

# ❌ Bypasses escaping
return Markup(user_input)   # only use for trusted content
{{ user_input | safe }}     # same — never for user data
```

```ts
// Express + template engines
// ❌ Never construct HTML by string concatenation
res.send(`<p>Hello ${req.query.name}</p>`);

// ✅ Use a template engine with auto-escaping, or encode explicitly
import { escape } from 'html-entities';
res.send(`<p>Hello ${escape(req.query.name)}</p>`);
```

### JSON in HTML (common overlooked vector)

```ts
// ❌ </script> in JSON breaks out of the script tag
const data = { message: '</script><script>alert(1)</script>' };
res.send(`<script>window.__data = ${JSON.stringify(data)}</script>`);

// ✅ Encode </script> sequences in JSON embedded in HTML
const safeJson = JSON.stringify(data).replace(/<\/script>/gi, '<\\/script>');
res.send(`<script>window.__data = ${safeJson}</script>`);
// Or better: put data in a data attribute, read with textContent
```

---

## DOM-Based XSS: URL/Fragment Sources

```ts
// ❌ Reads from URL, writes to DOM — no server involved
const name = new URLSearchParams(location.search).get('name');
document.getElementById('greeting').innerHTML = name;

// ✅
document.getElementById('greeting').textContent = name;

// ❌ Fragment-based (even more dangerous — never sent to server, invisible to WAFs)
document.getElementById('tab').innerHTML = location.hash.slice(1);

// ✅
document.getElementById('tab').textContent = decodeURIComponent(location.hash.slice(1));
```

Also check: `document.referrer`, `window.name`, `postMessage` handlers writing to DOM.

---

## Content Security Policy

CSP is a defense-in-depth header — it limits damage if XSS slips through. Not a substitute for fixing sinks.

```
# Strict CSP — blocks inline scripts and limits sources
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM_PER_REQUEST}';
  style-src 'self' 'nonce-{RANDOM_PER_REQUEST}';
  img-src 'self' data: https:;
  connect-src 'self' https://api.yourdomain.com;
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
```

```ts
// Express: generate nonce per request
import crypto from 'crypto';
app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString('base64');
  res.setHeader('Content-Security-Policy',
    `script-src 'self' 'nonce-${res.locals.nonce}'; object-src 'none'; base-uri 'self'`
  );
  next();
});

// In template: <script nonce="<%= nonce %>">...</script>
```

**`unsafe-inline` defeats CSP entirely.** If you need it for legacy code, add it — but know it's no protection.

**Test CSP before deploying:** use `Content-Security-Policy-Report-Only` first to catch breakage without blocking anything.

---

## Non-Obvious Vectors

| Vector | What to check |
|---|---|
| `href` / `src` attributes | Validate `javascript:` and `data:` schemes |
| `postMessage` handlers | Validate `event.origin` before acting on `event.data` |
| SVG files uploaded by users | SVGs can contain `<script>` — serve with `Content-Type: text/plain` or sanitize |
| Markdown renderers | Check if renderer outputs raw HTML; configure to escape or sanitize |
| `target="_blank"` links | Add `rel="noopener noreferrer"` — without it, opened page can access `window.opener` |
| Third-party scripts | Each one is a potential XSS vector — prefer self-hosting critical scripts |
| Error messages reflecting input | `"Invalid value: <input>"` echoed in an HTML response |

---

## Audit Checklist

- [ ] No `innerHTML`, `outerHTML`, `document.write`, `insertAdjacentHTML` with user data
- [ ] No `dangerouslySetInnerHTML` without DOMPurify wrapping it
- [ ] No `eval`, `setTimeout(string)`, `new Function(string)` with user data
- [ ] `href`/`src` assignments validate scheme (block `javascript:`, `data:`)
- [ ] URL params / fragment / `document.referrer` not written to DOM via HTML sinks
- [ ] Server templates have auto-escaping on; no `| safe` / `Markup()` on user data
- [ ] JSON embedded in `<script>` tags has `</script>` escaped
- [ ] CSP header set; no `unsafe-inline` if avoidable; nonce-based for inline scripts
- [ ] Uploaded SVGs served as `text/plain` or sanitized
- [ ] `postMessage` handlers validate `event.origin`
- [ ] Rich text: sanitize on render with DOMPurify, not on save