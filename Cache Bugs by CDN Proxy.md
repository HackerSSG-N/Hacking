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
