# Cache Bugs by CDN/Proxy

## 🌩 Cloudflare
Very common in bug bounty because many companies use free/paid Cloudflare.

**Most Common Bugs:**
- Cache Deception (`/account.php/style.css`)
- Cache Key Confusion (query params ignored by origin but cached by Cloudflare)
- Cache Poisoning via `Vary` / `Accept-Encoding` headers
- Cacheable Sensitive Data (missing `Cache-Control`)
- Path Normalization tricks (`/foo/../bar`)

**Rarer but Possible:**
- Cache Poisoning with Host Header (if misconfigured “orange cloud” DNS + origin)
- CORS Preflight cached incorrectly

---

## 🛡 Akamai
Akamai is huge in enterprises, lots of custom cache rules.

**Most Common Bugs:**
- Cache Poisoning via headers like `X-Forwarded-Host`, `Forwarded`, `X-Original-URL`
- Cache Key Parameter Cloaking (Akamai ignores certain query params)
- Open Redirect → Cached → Poisoning
- Cache Bypass with case/encoding (`%2F`, `;jsessionid`)
- Sensitive Data Cached due to wrong `no-store` / `no-cache` rules

**Special Akamai Quirk:**
- Sometimes Akamai doesn’t normalize duplicate params properly → great for poisoning.

---

## ☁️ Amazon CloudFront
Many startups + enterprises use it; common misconfig.

**Most Common Bugs:**
- Cache Deception (`/dashboard.php/logo.png`)
- Cache Bypass with query strings (CloudFront strips unless explicitly allowed)
- CORS Preflight response cached incorrectly
- Sensitive API responses cached due to missing `Cache-Control`
- Path Confusion (`/api;foo/bar`)

**Special CloudFront Quirk:**
- By default, CloudFront ignores query strings and cookies unless configured → perfect for cache key confusion.

---

## 🚀 Fastly
Popular with big tech companies (Reddit, Stripe, etc.).

**Most Common Bugs:**
- Cache Poisoning via `Vary: Origin`
- Cache Poisoning via `Accept-Language`
- Edge Rules Misconfiguration (custom VCL rules)
- Web Cache Deception (sensitive pages cached as static)

**Special Fastly Quirk:**
- Very flexible VCL config → often companies forget to block caching on sensitive routes.

---

## 🌀 Varnish / Nginx / Squid / Apache Traffic Server
Self-managed caches, often misconfigured.

**Most Common Bugs:**
- Cache Poisoning via custom headers (`X-Forwarded-*`, `X-Original-*`)
- Cache Deception
- Host Header Poisoning
- Path Normalization issues (`/foo//bar`, `%2e%2e`)
- Cookies not stripped → user-specific data cached globally

---

## ⚡ Quick Takeaway
- **Cloudflare** → Focus on Cache Deception & Parameter Confusion  
- **Akamai** → Focus on Header-based Cache Poisoning  
- **CloudFront** → Focus on Query String / Cookie Caching Misconfigs  
- **Fastly** → Focus on Origin/Language Vary header abuse  
- **Self-managed caches** → Focus on classic Host Header & Path Normalization tricks  

---

# 🧰 Cache Attack Checklist (Bug Bounty Edition)

## Step 1: Identify Cache Layer
🔎 Look at response headers:
- `cf-cache-status`
- `x-cache`
- `x-varnish`
- `via`
- `akamai-cache-status`
- `age`

📝 Note which CDN/proxy is used (Cloudflare, Akamai, CloudFront, Fastly, etc.).

📌 Save baseline: check if a page is cached at all (repeat request → check `Age` or response time).

---

## Step 2: Basic Cacheability Checks
❓ Does private/auth data get cached?  
- Visit authenticated page (e.g., `/account`, `/orders`) and check `Cache-Control`.  
- If missing `no-store` or `private` → 🔴 **potential sensitive data leak**.

---

## Step 3: Test for Cache Deception
Add static file extensions to dynamic pages:
- `/account.php/style.css`
- `/settings.jsp/logo.png`

Add fake file types:
- `/dashboard.php::css`
- `/login.php/random.jpg`

✅ **Success = sensitive/private page cached as if it were static.**

---

## Step 4: Parameter/Key Confusion
Test query strings:
- `/index.php?real=1&fake=2`
- `/index.php?foo=bar&foo=baz` (duplicate params)

Add ignored parameters:
- `/profile?id=123&ignoreme=xyz`

🔴 If backend ignores param but cache keys on it → **poisoning possible**.

---

## Step 5: Header Injection (Poisoning)
Try injecting headers that CDNs may treat differently than origin:
- `Host: evil.com`
- `X-Forwarded-Host: evil.com`
- `X-Original-URL: /evil`
- `X-Forwarded-Scheme: https`
- `Forwarded: for=attacker`
- `Via: 1.1 attacker`

👉 Check if response changes and gets cached.

---

## Step 6: Response Poisoning Tricks
Check `Vary` headers:
- `Vary: User-Agent` → send weird UA → see if cached for all.
- `Vary: Accept-Encoding` → gzip vs deflate mismatch.
- `Vary: Origin` → poison cache with attacker-controlled `Origin`.

Modify content-related headers:
- `ETag`
- `Last-Modified`
- `Content-Type`
- `Charset`

---

## Step 7: Path / Encoding Abuse
Try URL manipulations:
- Double slashes: `/foo//bar`
- Encoding: `/foo%2Fbar`, `%2e%2e/`
- Semicolons: `/api;jsessionid=123`
- Unicode homoglyphs: `/рrofile` (Cyrillic `p`)
- Nested query: `/page??id=1`

---

## Step 8: Method / Protocol Confusion
- Test `GET` vs `HEAD` → does one cache poison the other?  
- Test `TRACE`, `OPTIONS`, `POST` with caching.  
- HTTP/2 downgrade → try forcing **h2c upgrade smuggling**.  

---

## Step 9: CDN-Specific Tricks
- **Cloudflare** → cache deception, query param cloaking, static file tricks.  
- **Akamai** → `X-Forwarded-*`, duplicate params, open redirects.  
- **CloudFront** → query string/cookie rules, path confusion.  
- **Fastly** → `Vary: Origin` and custom edge rules.  

---

## Step 10: Side-Channel & Timing
- Send request with + without cache → check timing differences.  
- Try snooping: request victim-only URL (`/invoice?id=123`) → check cache hit/miss.  
- Try race: poison response quickly before victim hits.  

---

## ✅ Workflow Recap (Short Form)
1. Identify CDN/proxy.  
2. Check cache headers (`Cache-Control`, `Age`, `cf-cache-status`).  
3. Try cache deception (fake extensions).  
4. Test param/key confusion.  
5. Test header injection.  
6. Abuse `Vary` headers.  
7. Path/encoding tricks.  
8. Method/protocol tricks.  
9. Apply CDN-specific payloads.  
10. Check side-channels & timing.  




