# GSC Indexing Issues - Redirect Fix

## Problem

Google Search Console shows indexing failures for adinfinity.gr with three key issues:

1. **Mixed protocols** — Google crawled both `http://adinfinity.gr` and `https://www.adinfinity.gr`
2. **Mixed www/non-www variants** — Duplicate crawling of canonical and www versions
3. **Redirect validation failures** — Chains of ineffective redirects confusing Google's crawler

Root cause: Next.js `redirects()` array cannot enforce protocol changes (HTTP→HTTPS); only the hosting/edge layer can.

## Solution Implemented

### 1. Enhanced `proxy.ts` (Edge Middleware)

Updated [adifinity/proxy.ts](adifinity/proxy.ts) to enforce canonical domain at the edge:

- **Canonical form**: `https://adinfinity.gr` (no www, always HTTPS)
- **Status code**: 308 (preserves request method for POST/PUT, unlike 301)
- **Covered cases**:
  - `http://adinfinity.gr` → `https://adinfinity.gr` ✓
  - `https://www.adinfinity.gr` → `https://adinfinity.gr` ✓
  - `http://www.adinfinity.gr` → `https://adinfinity.gr` ✓
  - Preserves pathname and query params (except legacy Joomla params)

### 2. Added Invitations Subdomain Proxy

Created [adinfinity-invitations/proxy.ts](adinfinity-invitations/proxy.ts) with same redirect logic for:
- `http://invitations.adinfinity.gr` → `https://invitations.adinfinity.gr`
- `https://www.invitations.adinfinity.gr` → `https://invitations.adinfinity.gr`

### 3. Cleaned up next.config.mjs

Removed ineffective header-based protocol redirects from [adifinity/next.config.mjs](adifinity/next.config.mjs):
- Kept only application-level redirects (URL consolidation, old routes)
- Protocol normalization is now handled entirely by proxy.ts

### 4. Added Test Suite

Created [adifinity/__tests__/proxy.test.ts](adifinity/__tests__/proxy.test.ts) with test cases for:
- Protocol upgrade (HTTP → HTTPS)
- WWW stripping
- Query param preservation
- Joomla URL cleanup
- Legacy `.html` extension removal

## How It Works

```
Request → Vercel Edge (proxy.ts runs) → Canonical domain check
         ↓
    Not canonical? → 308 redirect to https://adinfinity.gr/...
         ↓
    Canonical? → Continue to Next.js app routing
```

## What Google Will See

Before: 
```
Crawled: http://adinfinity.gr (HTTP error)
Crawled: https://www.adinfinity.gr (indirect canonical)
Crawled: https://adinfinity.gr (canonical)
→ Validation failures, mixed signals
```

After:
```
Crawl request (any variant) → 308 redirect to https://adinfinity.gr
→ Google follows 308 → Always sees canonical form
→ Single indexable URL per route
```

## Deployment

Both Next.js apps compile successfully:
- ✅ `npm run build` in adifinity/
- ✅ `npm run build` in adinfinity-invitations/

The edge middleware (proxy.ts) is **automatically active on Vercel** — no config needed. When deployed:

1. Google's crawlers will follow 308 redirects to canonical form
2. GSC will stop showing "Page with redirect" notices for variants
3. Validation failures will clear as crawl normalizes

## Next Steps for Google Search Console

After deployment, in Google Search Console:

1. **Request indexing** for the canonical URLs:
   - `https://adinfinity.gr/`
   - `https://adinfinity.gr/services`
   - etc.

2. **Remove URL parameters** for old variants (if GSC still shows them as crawled):
   - Mark `http://adinfinity.gr` as "Not selected"
   - Mark `https://www.adinfinity.gr` as "Not selected"

3. **Monitor Coverage**:
   - Watch for "Valid pages with warnings" to clear
   - "Excluded" count should not increase (unless intentional)

## Files Changed

- ✅ `adifinity/proxy.ts` — Enhanced protocol/www normalization
- ✅ `adifinity/next.config.mjs` — Removed ineffective header-based redirects
- ✅ `adinfinity-invitations/proxy.ts` — New proxy for subdomain
- ✅ `adinfinity-invitations/next.config.ts` — Added redirects() placeholder
- ✅ `adifinity/__tests__/proxy.test.ts` — New test suite
