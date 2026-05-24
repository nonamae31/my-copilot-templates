# Dockerfile Rules — SaveFood Backend (ASP.NET Core)

## Full Annotated Template

```dockerfile
# ═══════════════════════════════════════════════════════════════════
# SaveFood Backend — Multi-Stage Dockerfile
# Target: Render.com (Linux, x64, port 8080)
# ═══════════════════════════════════════════════════════════════════

# ── Stage 1: Build ──────────────────────────────────────────────────
# Use full SDK image only for building — never in final image
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

# Copy csproj files first → enables Docker layer caching for `dotnet restore`
# If source code changes but .csproj doesn't, restore is skipped (fast builds)
COPY ["SaveFood.API/SaveFood.API.csproj", "SaveFood.API/"]

# Restore NuGet packages
RUN dotnet restore "SaveFood.API/SaveFood.API.csproj"

# Copy all source code after restore (separate layer for cache efficiency)
COPY . .

# Move into project directory and publish Release build
WORKDIR "/src/SaveFood.API"
RUN dotnet publish "SaveFood.API.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# ── Stage 2: Runtime ────────────────────────────────────────────────
# Use only the ASP.NET runtime — much smaller than SDK (~200MB vs ~700MB)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

WORKDIR /app

# Render.com routes external traffic to port 8080 by default
EXPOSE 8080

# Set production environment — disables dev exception pages, enables prod logging
ENV ASPNETCORE_ENVIRONMENT=Production

# Bind to all interfaces on port 8080 (required for container networking)
ENV ASPNETCORE_URLS=http://+:8080

# ── Security: Non-Root User ──────────────────────────────────────────
# Running as root inside Docker is a security vulnerability
# Even if container is compromised, attacker has limited OS privileges
RUN addgroup --system appgroup \
    && adduser --system --ingroup appgroup --no-create-home appuser

# Copy published output from build stage (only release artifacts)
COPY --from=build /app/publish .

# Give ownership to appuser before switching
RUN chown -R appuser:appgroup /app

# Switch to non-root user — all subsequent commands and the app run as this user
USER appuser

# Start the application
ENTRYPOINT ["dotnet", "SaveFood.API.dll"]
```

---

## Rules Checklist

### Image Selection
- [x] Build stage: `mcr.microsoft.com/dotnet/sdk:8.0` (includes MSBuild, NuGet, etc.)
- [x] Runtime stage: `mcr.microsoft.com/dotnet/aspnet:8.0` (ASP.NET runtime only, no SDK)
- [x] Never use `:latest` — always pin major.minor version
- [x] For Alpine variant (smallest image): use `mcr.microsoft.com/dotnet/aspnet:8.0-alpine` — but test thoroughly, some globalization features require extra config

### Layer Caching
- [x] `COPY *.csproj` → `RUN dotnet restore` must happen BEFORE `COPY . .`
- [x] This way, if only `.cs` files change (not `.csproj`), the restore layer is cached
- [x] Each `RUN` command that installs or restores should be its own layer unless chaining with `&&`

### Build Flags
- [x] Always publish with `-c Release` (not Debug)
- [x] Use `--no-restore` in publish if restore was already done (avoids double-restore)
- [x] `/p:UseAppHost=false` prevents generating a native executable wrapper (not needed in containers)

### Security
- [x] `addgroup` + `adduser` → `chown` → `USER` — all four steps are required
- [x] `--no-create-home` saves disk space
- [x] Verify with: `docker run --rm savefood-api whoami` → should return `appuser`

### Secrets
- [x] ZERO secrets in Dockerfile
- [x] Connection strings → Render env vars
- [x] JWT secrets → Render env vars
- [x] OAuth client secrets → Render env vars
- [x] `ENV` in Dockerfile is for config (ASPNETCORE_ENVIRONMENT, ASPNETCORE_URLS), not secrets

### Port
- [x] `EXPOSE 8080` matches Render's expected port
- [x] `ASPNETCORE_URLS=http://+:8080` binds Kestrel to that port
- [x] Never use `EXPOSE 443` in a container without TLS termination setup (Render handles TLS externally)

---

## Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| Using `:latest` tag | Pin to `sdk:8.0` |
| Copying `.csproj` after `COPY . .` | Always copy project file and restore BEFORE copying source |
| Running as root | Add `appuser` and `USER appuser` |
| Including `appsettings.Development.json` secrets | Use `.dockerignore` to exclude it |
| Publishing with Debug config | Use `-c Release` |
| Not setting `ASPNETCORE_URLS` | Kestrel defaults to 5000/5001, not 8080 |

---

## .dockerignore (place at backend root)

```dockerignore
**/.git
**/.vs
**/.vscode
**/bin
**/obj
**/TestResults
**/.env
**/.env.*
**/appsettings.Development.json
**/appsettings.*.json
!**/appsettings.json
**/*.user
**/Dockerfile*
**/*.md
```
