---
name: savefood-cicd
description: >
  Skill for setting up CI/CD, Docker, and deployment configuration for the SaveFood project.
  Triggers whenever the user mentions: Dockerfile, Docker, CI/CD, deployment, Render, Vercel,
  environment variables, CORS, vercel.json, ASP.NET Core containerization, or any task related
  to building/deploying the SaveFood frontend (React/Vercel) or backend (ASP.NET Core/Render).
  Always use this skill when any infrastructure, DevOps, or deployment topic arises for SaveFood,
  even if the user only asks a partial question like "how do I add CORS?" or "fix my Dockerfile".
---

# SaveFood — CI/CD & Docker Skill

> **Role**: You are a senior DevOps/Cloud Engineer responsible for the SaveFood platform.
> Follow every rule in this skill **exactly and completely**. No shortcuts. No approximations.
> When in doubt, reference the relevant section below before generating any file or command.

---

## 1. Project Architecture Overview

```
SaveFood/
├── frontend/          # React + TypeScript + Tailwind  → Deploy on Vercel
└── backend/           # ASP.NET Core + SQL Server      → Docker → Deploy on Render
```

| Layer    | Stack                          | Deploy Target | Config Files                        |
|----------|--------------------------------|---------------|-------------------------------------|
| Frontend | React 18, TypeScript, Tailwind | Vercel        | `vercel.json`, `.env.production`    |
| Backend  | ASP.NET Core 8, EF Core        | Render (Docker) | `Dockerfile`, `docker-compose.yml`, `.env` |

---

## 2. Dockerfile — ASP.NET Core (Multi-Stage, Optimized, Secure)

> **Read `references/dockerfile-rules.md` for full annotated template and detailed rules.**

### Non-negotiable rules (summary):

1. **Always use multi-stage build**: `build` stage (SDK image) → `runtime` stage (aspnet runtime image only).
2. **Pin exact image tags**: Never use `:latest`. Use `mcr.microsoft.com/dotnet/sdk:8.0` and `mcr.microsoft.com/dotnet/aspnet:8.0`.
3. **Restore dependencies separately** before copying source — leverages Docker layer cache.
4. **Non-root user**: Create and switch to `appuser` in the runtime stage before `ENTRYPOINT`.
5. **Minimize runtime image**: Copy only the published output (`--configuration Release`), never copy `.sln`, test projects, or dev tools.
6. **Expose port 8080** (Render default). Do NOT hardcode port 443 or 5000.
7. **Set `ASPNETCORE_ENVIRONMENT=Production`** in the runtime stage via `ENV`.
8. **No secrets in Dockerfile**: Connection strings, JWT secrets, OAuth keys must NEVER appear in any `ENV` instruction inside the Dockerfile.

Quick reference template:
```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["SaveFood.API/SaveFood.API.csproj", "SaveFood.API/"]
RUN dotnet restore "SaveFood.API/SaveFood.API.csproj"
COPY . .
WORKDIR "/src/SaveFood.API"
RUN dotnet publish -c Release -o /app/publish --no-restore

# ── Stage 2: Runtime ────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:8080

# Security: non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
COPY --from=build /app/publish .
RUN chown -R appuser:appgroup /app
USER appuser

ENTRYPOINT ["dotnet", "SaveFood.API.dll"]
```

---

## 3. Environment Variables — Security & Management

> **Read `references/env-management-rules.md` for full rules on all secret types.**

### Golden rules:

| Rule | Detail |
|------|--------|
| **Never commit secrets** | `.env`, `.env.production`, `appsettings.Production.json` → always in `.gitignore` |
| **Render secrets** | Set via Render Dashboard → Environment → Environment Variables. Never in Dockerfile. |
| **Vercel secrets** | Set via Vercel Dashboard → Project Settings → Environment Variables. Never in `vercel.json`. |
| **Local dev** | Use `.env.local` (frontend) and `dotnet user-secrets` or `.env` file (backend). Both gitignored. |
| **Prefix convention** | All frontend env vars exposed to React **must** be prefixed `VITE_` (if using Vite) or `REACT_APP_` (CRA). |

### Required backend env vars (set on Render):
```
ConnectionStrings__DefaultConnection=Server=...;Database=SaveFood;...
JwtSettings__SecretKey=<strong-random-256bit>
JwtSettings__Issuer=https://savefood-api.onrender.com
JwtSettings__Audience=savefood-client
ASPNETCORE_ENVIRONMENT=Production
ALLOWED_ORIGINS=https://savefood.vercel.app,https://www.savefood.vn
```

### Required frontend env vars (set on Vercel):
```
VITE_API_BASE_URL=https://savefood-api.onrender.com
VITE_GOOGLE_CLIENT_ID=<oauth-client-id>
```

---

## 4. CORS Policy — Backend Docker Configuration

> **Read `references/cors-rules.md` for full Program.cs configuration pattern.**

### Non-negotiable rules:

1. **Read `ALLOWED_ORIGINS` from environment variable** — never hardcode Vercel URLs in source code.
2. **Named policy**: Define a named CORS policy (e.g., `"SaveFoodCorsPolicy"`) and apply it globally.
3. **Production**: Allow only the exact Vercel production domain(s) from env var.
4. **Development**: Allow `http://localhost:3000` and `http://localhost:5173` automatically when `ASPNETCORE_ENVIRONMENT=Development`.
5. **Allowed methods**: `GET, POST, PUT, DELETE, PATCH, OPTIONS`.
6. **Allowed headers**: `Content-Type, Authorization, X-Requested-With`.
7. **`AllowCredentials()`** is required if using HttpOnly cookie auth; omit if JWT-only via Authorization header.

### Program.cs pattern (condensed):
```csharp
var allowedOrigins = builder.Configuration["ALLOWED_ORIGINS"]?
    .Split(',', StringSplitOptions.RemoveEmptyEntries) ?? [];

builder.Services.AddCors(options =>
{
    options.AddPolicy("SaveFoodCorsPolicy", policy =>
    {
        if (builder.Environment.IsDevelopment())
            policy.WithOrigins("http://localhost:3000", "http://localhost:5173");
        else
            policy.WithOrigins(allowedOrigins);

        policy.WithMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
              .WithHeaders("Content-Type", "Authorization", "X-Requested-With")
              .AllowCredentials();
    });
});

// Must appear BEFORE UseAuthorization
app.UseCors("SaveFoodCorsPolicy");
```

---

## 5. vercel.json — Frontend SPA Routing

> **Read `references/vercel-config-rules.md` for advanced routing patterns.**

### Non-negotiable rules:

1. **SPA fallback rewrite is mandatory**: All routes must fall back to `/index.html` so React Router works on page refresh.
2. **API proxy rewrite** (recommended): Proxy `/api/*` to backend URL to avoid CORS during dev and simplify frontend base URLs.
3. **Security headers**: Always add `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`.
4. **No secrets in `vercel.json`**: API keys, tokens → Vercel Dashboard only.
5. **`trailingSlash: false`**: Prevents double-routing issues with React Router.

### Canonical `vercel.json`:
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
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
      ]
    }
  ]
}
```

---

## 6. docker-compose.yml — Local Development

Use `docker-compose.yml` at project root for **local dev only** (not for Render deployment):

```yaml
version: "3.9"
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=SaveFood;User Id=sa;Password=${SA_PASSWORD};TrustServerCertificate=True
    depends_on:
      - db
    env_file:
      - ./backend/.env

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: ${SA_PASSWORD}
      ACCEPT_EULA: "Y"
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql

volumes:
  sqldata:
```

> ⚠️ `SA_PASSWORD` must be in `backend/.env` (gitignored). Never hardcode in `docker-compose.yml`.

---

## 7. .gitignore — Mandatory Entries

Ensure these are always present:
```gitignore
# Secrets & env
.env
.env.*
!.env.example
appsettings.Production.json
appsettings.Development.json

# Docker local overrides
docker-compose.override.yml

# Build artifacts
**/bin/
**/obj/
**/publish/
```

---

## 8. Render Deployment — Backend Checklist

When setting up or modifying Render deployment:

- [ ] Set **Docker** as the build environment in Render service settings
- [ ] Point to `backend/Dockerfile` 
- [ ] Set all env vars listed in Section 3 via Render Dashboard (never via Dockerfile)
- [ ] Health check path: `GET /health` → ensure a `/health` endpoint exists in the API
- [ ] Set `RENDER_PORT=8080` if Render doesn't auto-detect
- [ ] Enable auto-deploy from `main` branch

---

## Reference Files

| File | When to Read |
|------|-------------|
| `references/dockerfile-rules.md` | Writing or reviewing any Dockerfile for SaveFood backend |
| `references/env-management-rules.md` | Any task involving secrets, connection strings, API keys |
| `references/cors-rules.md` | CORS configuration, API access issues from frontend |
| `references/vercel-config-rules.md` | Routing issues, `vercel.json` setup, frontend deployment |
