# Environment Variable Management — SaveFood

## Taxonomy of Secrets

| Secret | Backend (Render) | Frontend (Vercel) | Local Dev |
|--------|-----------------|-------------------|-----------|
| DB Connection String | ✅ Render Env Var | ❌ Never | `.env` (gitignored) |
| JWT Secret Key | ✅ Render Env Var | ❌ Never | `.env` (gitignored) |
| JWT Issuer/Audience | ✅ Render Env Var | ❌ Never | `appsettings.Development.json` |
| Google OAuth Client ID | ✅ Render Env Var | ✅ `VITE_GOOGLE_CLIENT_ID` | `.env.local` |
| Google OAuth Client Secret | ✅ Render Env Var | ❌ Never | `.env` (gitignored) |
| API Base URL | ❌ N/A | ✅ `VITE_API_BASE_URL` | `.env.local` |
| ALLOWED_ORIGINS | ✅ Render Env Var | ❌ N/A | Optional |

---

## Backend — Render Environment Variables

### Setting via Render Dashboard
1. Go to Render Dashboard → Your Service → **Environment** tab
2. Add each key/value pair individually
3. Render injects these as OS environment variables at runtime

### Accessing in ASP.NET Core

**Connection string** (mapped from env var to `ConnectionStrings` section):
```csharp
// In appsettings.json (template only, no real value):
{
  "ConnectionStrings": {
    "DefaultConnection": ""
  }
}

// Render env var name: ConnectionStrings__DefaultConnection
// Double underscore (__) maps to nested JSON in ASP.NET Core configuration
```

**JWT settings**:
```csharp
// Render env var names:
// JwtSettings__SecretKey
// JwtSettings__Issuer
// JwtSettings__Audience
// JwtSettings__ExpirationMinutes

// In Program.cs:
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("JwtSettings"));
```

**ALLOWED_ORIGINS** (flat env var, not nested):
```csharp
var origins = Environment.GetEnvironmentVariable("ALLOWED_ORIGINS")?
    .Split(',', StringSplitOptions.RemoveEmptyEntries)
    ?? Array.Empty<string>();
```

---

## Frontend — Vercel Environment Variables

### Naming Convention (Vite)
- All env vars exposed to React **MUST** be prefixed with `VITE_`
- Non-prefixed vars are server-side only and NOT accessible in browser JS

```bash
# .env.local (local dev, gitignored)
VITE_API_BASE_URL=http://localhost:8080
VITE_GOOGLE_CLIENT_ID=your-dev-client-id.apps.googleusercontent.com

# Vercel Dashboard (production)
VITE_API_BASE_URL=https://savefood-api.onrender.com
VITE_GOOGLE_CLIENT_ID=your-prod-client-id.apps.googleusercontent.com
```

### Accessing in React/TypeScript
```typescript
// Type-safe access
const API_BASE = import.meta.env.VITE_API_BASE_URL as string;
const GOOGLE_CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID as string;

// Always validate at app startup
if (!API_BASE) throw new Error("VITE_API_BASE_URL is not set");
```

### env.d.ts — Type declarations
```typescript
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_GOOGLE_CLIENT_ID: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## Local Development Setup

### Backend
```bash
# backend/.env (gitignored — DO NOT COMMIT)
ConnectionStrings__DefaultConnection=Server=localhost,1433;Database=SaveFood;User Id=sa;Password=YourLocalPass123!;TrustServerCertificate=True
JwtSettings__SecretKey=local-dev-secret-minimum-32-chars-long
JwtSettings__Issuer=https://localhost:8080
JwtSettings__Audience=savefood-client
ASPNETCORE_ENVIRONMENT=Development
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

Load in `Program.cs`:
```csharp
// Automatically loaded by ASP.NET Core from environment
// For Docker Compose, specify env_file in docker-compose.yml
```

### Frontend
```bash
# frontend/.env.local (gitignored — DO NOT COMMIT)
VITE_API_BASE_URL=http://localhost:8080
VITE_GOOGLE_CLIENT_ID=dev-client-id
```

---

## .env.example Files (COMMIT THESE)

Always provide `.env.example` files showing keys without values:

**`backend/.env.example`**:
```bash
ConnectionStrings__DefaultConnection=Server=localhost,1433;Database=SaveFood;User Id=sa;Password=CHANGE_ME;TrustServerCertificate=True
JwtSettings__SecretKey=CHANGE_ME_minimum_32_chars
JwtSettings__Issuer=https://localhost:8080
JwtSettings__Audience=savefood-client
ASPNETCORE_ENVIRONMENT=Development
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

**`frontend/.env.example`**:
```bash
VITE_API_BASE_URL=http://localhost:8080
VITE_GOOGLE_CLIENT_ID=CHANGE_ME
```

---

## Security Audit Checklist

- [ ] `.env` files are in `.gitignore`
- [ ] `appsettings.Production.json` is in `.gitignore` (or doesn't exist)
- [ ] No secrets in `Dockerfile` `ENV` instructions
- [ ] No secrets in `vercel.json`
- [ ] No secrets in `docker-compose.yml` (only variable references like `${SA_PASSWORD}`)
- [ ] No API keys or tokens in frontend source code (only `import.meta.env.VITE_*`)
- [ ] JWT secret is minimum 32 characters, randomly generated
- [ ] Different secrets for dev vs production
