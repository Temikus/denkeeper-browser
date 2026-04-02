# denkeeper-browser

Hardened Docker image running `@playwright/mcp` for [denkeeper](https://github.com/Temikus/denkeeper) browser automation. Communicates via MCP stdio transport.

## Architecture

Single-container image, two-stage Dockerfile:
- **Builder stage**: `node:22-bookworm-slim` — installs npm deps + Chromium via Playwright
- **Runtime stage**: `node:22-bookworm-slim` — copies node_modules and Chromium binary, runs as non-root user `mcp` (UID 10001)

Entrypoint: `node /app/node_modules/@playwright/mcp/cli.js` — all flags passed by caller.

## Security model

The image provides: non-root execution, Chromium-only (no Firefox/WebKit), no ambient credentials.

The caller (denkeeper sandbox runtime) provides: `--cap-drop ALL`, `--read-only`, `--security-opt no-new-privileges`, network egress only, resource limits.

`--no-sandbox` is required because Docker's isolation replaces Chromium's internal sandbox.

## Build / Test / Release

Requires [just](https://github.com/casey/just) and Docker.

```
just build            # Build image for current platform
just test             # Build + run all tests (smoke + structure)
just test smoke       # MCP smoke test only
just test structure   # Container structure tests only
just lint             # Hadolint Dockerfile lint
just check            # lint + all tests
just audit            # npm audit
just scan             # All security scans (grype + audit)
just scan grype       # Grype container vulnerability scan
just scan audit       # npm audit only
just build-multi      # Multi-arch build (amd64 + arm64)
just release <bump>   # Tag and push (patch|minor|major)
```

## CI/CD

- **ci.yml**: lint, build, smoke test, structure test, Grype vuln scan (on push/PR to main)
- **release.yml**: multi-arch build+push to GHCR, cosign signing, SLSA provenance, SBOM (on tag push)
- **security.yml**: Gitleaks secret detection, npm audit (on push/PR + weekly cron)

## Tests

- `test/smoke.sh` — MCP protocol handshake over stdio (initialize + tools/list)
- `test/structure.sh` — Container security invariants (UID, entrypoint, browsers, labels, workdir)

## Key files

- `Dockerfile` — Multi-stage build
- `package.json` — Pins `@playwright/mcp` version
- `justfile` — All build/test/release tasks
- `.github/workflows/` — CI, release, and security pipelines
