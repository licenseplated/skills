---
name: structurizr
description: Help users write Structurizr DSL for C4 software-architecture diagrams and navigate the homelab `structurizr.oddkin.co` server-mode deployment. Use when the user mentions Structurizr, C4 model diagrams, software architecture diagramming, the `workspace { model { views { } } }` DSL, or asks about workspace navigation, view types (System Context / Container / Component / Deployment / Dynamic), or publishing diagrams. Authoritative docs at https://docs.structurizr.com — link there rather than recalling syntax from memory when the user wants the definitive reference.
---

# Structurizr

## The deployment

- **URL**: https://structurizr.oddkin.co — production-mode `structurizr/structurizr` (the Java/Maven on-premises server, not `structurizr/lite`).
- **Source fork**: `git.oddkin.co/mmz/structurizr` (mirror of `github.com/structurizr/structurizr`). Image built by `.forgejo/workflows/build.yml` and published to `git.oddkin.co/mmz/structurizr:{sha,latest}`.
- **Auth**: behind oauth2-proxy gating on the `structurizr-users` group in the Keycloak `rumi` realm. All realm users are auto-added to that group by the realm-config-job. Sign-in is passkey-only (browser-passkey-required flow).
- **Workspaces**: client-side encrypted by default. First sign-in to `/workspace/1` shows the "Workspace 0001" placeholder with a Decrypt button; new workspaces get a per-workspace password set in the UI.
- **Storage**: per-pod PVC at `/usr/local/structurizr` (5 Gi). Workspaces survive pod restarts but not PVC deletion.

## Authoritative docs

Always link to https://docs.structurizr.com for definitive reference rather than reciting from memory.

| Topic                       | URL                                          |
| --------------------------- | -------------------------------------------- |
| DSL language reference      | https://docs.structurizr.com/dsl/language    |
| DSL basics + defaults       | https://docs.structurizr.com/dsl/basics      |
| Identifiers (flat/hierarchical) | https://docs.structurizr.com/dsl/identifiers |
| Implied relationships       | https://docs.structurizr.com/dsl/implied-relationships |
| `!include` composition      | https://docs.structurizr.com/dsl/includes    |
| Cookbook (views, styles, deployment) | https://docs.structurizr.com/dsl/cookbook/ |
| Pattern catalog (AWS, K8s, …) | https://docs.structurizr.com/dsl/patterns/ |
| UI: diagrams                | https://docs.structurizr.com/ui/diagrams/    |
| UI: documentation           | https://docs.structurizr.com/ui/documentation/ |
| UI: ADRs / decisions        | https://docs.structurizr.com/ui/decisions/   |
| UI: exploration / table view | https://docs.structurizr.com/ui/explorations/ |
| UI: quick navigation        | https://docs.structurizr.com/ui/quick-navigation |
| Usage overview              | https://docs.structurizr.com/usage           |

## Quick start: minimal DSL

```structurizr
workspace "Example" "A starter C4 workspace" {
    model {
        user = person "Customer"
        ss   = softwareSystem "Banking System" {
            web = container "Web App"  "React SPA"     "TypeScript"
            api = container "API"       "Spring Boot"  "Java"
            db  = container "Database"  "PostgreSQL"   "Database"
        }
        user -> web "Visits"
        web  -> api "Calls" "JSON/HTTPS"
        api  -> db  "Reads/writes" "JDBC"
    }
    views {
        systemContext ss { include *; autolayout lr }
        container     ss { include *; autolayout lr }
        styles {
            element "Person"   { shape Person }
            element "Database" { shape Cylinder }
        }
        theme default
    }
}
```

Paste into the **DSL** tab in the workspace UI and click *Save* — the **Diagrams** tab will then show System Context and Container views.

## Push a DSL update to the homelab server

The workspace API uses HMAC auth (per-workspace `apiKey` + `apiSecret`) — *separate* from the OIDC gate at oauth2-proxy. For our deployment, `oauth2-proxy` still gates `/api/*` with OIDC, so the push goes via a `kubectl port-forward` that bypasses the gate. `structurizr-cli` (installed via `brew install structurizr-cli`) handles the HMAC signing.

Workspace credentials live at Vault path `kv/structurizr/workspace-1`:
- `api_key` — 40-char hex
- `api_secret` — 64-char hex
- `url` — `https://structurizr.oddkin.co/api` (informational; the push uses the port-forward URL)

The matching bcrypt hashes are already in the pod's `/usr/local/structurizr/1/workspace.properties` (no setup work needed for re-pushes).

```sh
# 1. Port-forward in a separate terminal (or backgrounded)
kubectl -n structurizr port-forward structurizr-0 18080:8080 &

# 2. Pull credentials (claude-admin AppRole, creds in ~/.env)
set -a; . ~/.env; set +a
export VAULT_TOKEN=$(vault write -field=token auth/approle/login \
  role_id="$VAULT_APPROLE_ROLE_ID" secret_id="$VAULT_APPROLE_SECRET_ID")
KEY=$(vault kv get -field=api_key    -mount=kv structurizr/workspace-1)
SECRET=$(vault kv get -field=api_secret -mount=kv structurizr/workspace-1)

# 3. Push the DSL — structurizr-cli parses + HMAC-signs + PUTs
structurizr-cli push -id 1 -key "$KEY" -secret "$SECRET" \
  -url http://localhost:18080/api \
  -workspace ~/yardidev/junk-drawer/jira/yvm/architecture/STRUCTURIZR.dsl
```

`structurizr-cli push` drops a `structurizr-<id>-<ts>.json` rollback artifact next to the DSL — the directory's `.gitignore` excludes that pattern.

For a raw `curl` (no structurizr-cli) or details on the HMAC scheme, the oauth2-proxy bypass option, and rollback handling, see [REFERENCE.md](REFERENCE.md) §Pushing.

## Workspace UI tabs

- **Diagrams** — visual views (System Context, Container, Component, Deployment, Dynamic, Filtered, …). The primary surface.
- **Documentation** — Markdown/AsciiDoc per element. Define with `documentation` blocks in the model.
- **Decisions** — Architecture Decision Records (ADRs) per element. `!adrs path/to/dir/` in the workspace block.
- **Explore** — tabular view of elements + relationships; useful for audit/cleanup.
- **Inspections** — built-in validation (missing descriptions, orphan elements, etc.).
- **DSL** — edit the source. Re-rendered on Save.
- **JSON** — raw workspace JSON. Read-only here; for programmatic edits use the API.
- **Settings** — workspace name, description, encryption password, branding.

See [REFERENCE.md](REFERENCE.md) for deeper DSL syntax + common patterns.
