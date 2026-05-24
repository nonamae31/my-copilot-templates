# vercel.json Configuration — SaveFood Frontend

## Canonical Production Config

```json
{
  "trailingSlash": false,
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "https://savefood-api.onrender.com/api/:path*"
    },
    {
      "source": "/((?!api/).*)",
      "destination": "/index.html"
    }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" }
      ]
    },
    {
      "source": "/assets/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ]
}
```

---

## Rule-by-Rule Explanation

### `trailingSlash: false`
- Prevents `/products/` and `/products` from being treated as different routes
- React Router doesn't use trailing slashes
- Set to `false` to avoid redirect loops

### Rewrite 1 — API Proxy
```json
{
  "source": "/api/:path*",
  "destination": "https://savefood-api.onrender.com/api/:path*"
}
```
- Proxies all `/api/*` calls from the frontend to the Render backend
- **Eliminates CORS issues in production** if backend CORS isn't working
- Frontend code uses relative `/api/products` paths → cleaner code
- ⚠️ If using this proxy, the `VITE_API_BASE_URL` should be `/api` in production

### Rewrite 2 — SPA Fallback
```json
{
  "source": "/((?!api/).*)",
  "destination": "/index.html"
}
```
- This is the **most critical rule** for React Router
- Without it: refreshing `/products/123` returns 404 (Vercel can't find the file)
- The regex `(?!api/)` ensures API routes are NOT caught by this fallback
- Works for all React Router routes: `/`, `/login`, `/products/:id`, `/dashboard/*`

### Security Headers
- `X-Frame-Options: DENY` → prevents clickjacking (embedding the app in iframes)
- `X-Content-Type-Options: nosniff` → prevents MIME type sniffing attacks
- `Referrer-Policy` → controls what URL info is sent to external sites
- `Permissions-Policy` → disables browser features the app doesn't need

### Cache-Control for Assets
- Vite generates hashed filenames for JS/CSS bundles: `main.a3f7d1.js`
- Since filename changes when content changes, it's safe to cache forever (`immutable`)
- Dramatically improves repeat visit performance

---

## Using API Proxy vs Direct API URL

### Option A: API Proxy (recommended for production)
```typescript
// Frontend code
const response = await fetch('/api/products'); // proxied by Vercel to Render
```
```bash
# Vercel env var
VITE_API_BASE_URL=/api
```

### Option B: Direct URL (simpler, requires correct CORS on backend)
```typescript
// Frontend code
const response = await fetch(`${import.meta.env.VITE_API_BASE_URL}/products`);
```
```bash
# Vercel env var
VITE_API_BASE_URL=https://savefood-api.onrender.com
```

> If using Option B, the `/api/:path*` rewrite in `vercel.json` is still harmless but unused.

---

## Framework Detection

Vercel auto-detects Vite/React projects. If auto-detection fails, add to `vercel.json`:

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "installCommand": "npm install",
  "framework": "vite"
}
```

---

## What NOT to put in vercel.json

```json
// ❌ WRONG — secrets in vercel.json are publicly visible in the browser
{
  "env": {
    "VITE_GOOGLE_CLIENT_ID": "your-actual-client-id"  // NEVER
  }
}
```

All secrets → Vercel Dashboard → Project Settings → Environment Variables.

---

## Vercel Deployment Checklist

- [ ] `vercel.json` exists at project root (or frontend root if monorepo)
- [ ] SPA fallback rewrite is present
- [ ] `trailingSlash: false` is set
- [ ] Security headers are configured
- [ ] All `VITE_*` env vars are set in Vercel Dashboard (not in vercel.json)
- [ ] `dist` or `build` output directory matches Vite/CRA config
- [ ] Custom domain is configured in Vercel (if using `savefood.vn`)
- [ ] Custom domain is added to `ALLOWED_ORIGINS` on Render backend

---

## Monorepo Setup

If the project is a monorepo (`/frontend` and `/backend` in same repo):

```json
{
  "root": "frontend",
  "buildCommand": "cd frontend && npm run build",
  "outputDirectory": "frontend/dist"
}
```

Or configure "Root Directory" to `frontend` in the Vercel project settings UI.
