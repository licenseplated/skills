# Structurizr DSL reference

This file expands on the patterns in `SKILL.md`. Authoritative source is always
https://docs.structurizr.com/dsl — link there when the user wants definitive
syntax.

## Workspace anatomy

```structurizr
workspace "Name" "Description" {
    !identifiers hierarchical      # default is flat; hierarchical lets you write a.b.c
    !impliedRelationships true     # add implicit relationships up the parent chain

    model {
        # people, software systems, containers, components,
        # deployment environments, relationships
    }

    views {
        # system landscape, system context, container, component,
        # dynamic, deployment, filtered, image
        # plus styles, themes, branding, terminology
    }

    configuration {
        scope softwareSystem       # or "landscape"; controls which inspections fire
        properties {
            "structurizr.locale"  "en-US"
        }
    }
}
```

## Model elements

| Element            | Keyword                   | Notes |
| ------------------ | ------------------------- | ----- |
| Person             | `person "Name"`           | shape: Person |
| Software system    | `softwareSystem "Name"`   | top of a system's hierarchy |
| Container          | `container "Name"`        | a runnable/data-storing thing |
| Component          | `component "Name"`        | a unit of code within a container |
| Custom element     | `element "Name" "type"`   | for things outside the C4 hierarchy |
| Group              | `group "Name" { … }`      | logical grouping; not a parent |
| Deployment env     | `deploymentEnvironment "Name" { … }` | wraps deployment nodes |
| Deployment node    | `deploymentNode "Name"`   | infrastructure (server, k8s, …) |
| Infrastructure node| `infrastructureNode`      | load balancer, firewall, … |
| Container instance | `containerInstance c`     | reference to a model container |
| Software system instance | `softwareSystemInstance ss` | reference to a model system |

Every element takes optional `description`, `technology`, and `tags` positional args:
`container "API" "Spring Boot HTTP API" "Java,Spring,REST"`

## Relationships

```structurizr
user -> web "Visits"                              # default
user -> web "Visits" "HTTPS"                       # with technology
user -> web "Visits" "HTTPS" "Tag1,Tag2"           # with tags
user -> web "Visits" "HTTPS" {                     # block form
    perspectives {
        Security "user→web traffic"
    }
}
```

Relationship blocks support `tags`, `url`, `properties`, and per-relationship
`perspectives`. https://docs.structurizr.com/dsl/language#relationship

## View types

| View              | DSL                                       | Use |
| ----------------- | ----------------------------------------- | --- |
| System Landscape  | `systemLandscape "name" { … }`            | all people + systems in scope |
| System Context    | `systemContext ss { … }`                  | one system + its neighbors |
| Container         | `container ss { … }`                      | inside one system |
| Component         | `component containerName { … }`           | inside one container |
| Dynamic           | `dynamic ss "key" "desc" { … }`           | numbered call sequence |
| Deployment        | `deployment ss "env" "key" { … }`         | infra view of an env |
| Filtered          | `filtered baseKey include "Tag" "key"`    | derived view with tag filter |
| Image             | `image element "key" { … }`               | external image attached to an element |

Common contents of a view block:

```structurizr
container ss {
    include *               # all containers in the system
    include user            # plus explicit elements
    exclude db              # minus explicit elements
    autolayout lr           # tb, bt, lr, rl, with rank/node separation args
    title "Container view"
    description "..."
    properties {
        "structurizr.color" "#1168bd"
    }
}
```

## Styles

```structurizr
styles {
    element "Person" {
        shape Person
        background #08427b
        color #ffffff
    }
    element "Database" {
        shape Cylinder
    }
    relationship "Async" {
        style dashed
        color #888888
    }
}
```

Style selectors match on element/relationship `tags`. Built-in tags include
`Element`, `Person`, `Software System`, `Container`, `Component`,
`Infrastructure Node`, `Deployment Node`, `Relationship`. Custom tags applied
via the third element-positional arg or `tags "T1,T2"`.

Shapes: `Box`, `RoundedBox`, `Circle`, `Ellipse`, `Hexagon`, `Cylinder`,
`Pipe`, `Person`, `Robot`, `Folder`, `WebBrowser`, `MobileDeviceLandscape`,
`MobileDevicePortrait`, `Component`. https://docs.structurizr.com/dsl/cookbook/element-styles/

## Themes

```structurizr
views {
    theme https://static.structurizr.com/themes/amazon-web-services-2023.01.31/theme.json
    theme default
}
```

Multiple themes layer; later ones override earlier ones. https://docs.structurizr.com/dsl/cookbook/themes/

## Deployment views — common pattern

```structurizr
model {
    # ... model elements ...
    prod = deploymentEnvironment "Production" {
        k8s = deploymentNode "Kubernetes" "k8s" "kind" {
            apiNode = deploymentNode "api-pod" {
                apiInstance = containerInstance api
            }
            dbNode  = deploymentNode "postgres-pod" {
                dbInstance  = containerInstance db
            }
        }
    }
}
views {
    deployment ss "Production" "prod-overview" {
        include *
        autolayout
    }
}
```

Deployment nodes nest arbitrarily; `containerInstance`/`softwareSystemInstance`
references existing model elements. https://docs.structurizr.com/dsl/cookbook/deployment-views/

## Splitting workspace across files

```structurizr
workspace {
    !include model.dsl
    !include views.dsl
}
```

`!include` paths are relative to the workspace file. Supports remote URLs.
https://docs.structurizr.com/dsl/includes

## Identifiers: flat vs hierarchical

```structurizr
!identifiers hierarchical
model {
    ss = softwareSystem "S" {
        c1 = container "C" {
            comp1 = component "Comp"
        }
    }
    ss.c1.comp1 -> ss.c1.comp2   # hierarchical: scoped names
}
```

With `!identifiers flat` (default), `comp1` is global and must be unique
across the whole model.

## Inspections + scope

`configuration { scope softwareSystem }` makes the Inspections tab flag
elements outside the single-system scope. For multi-system workspaces, use
`scope landscape` instead.
https://docs.structurizr.com/usage/inspections

## API + CLI

The on-premises server exposes the same workspace REST API as the cloud
service: `GET /api/workspace/{id}`. Auth via API key + secret (set per-
workspace in **Settings → API**) and an HMAC-signed nonce.
See https://docs.structurizr.com/onpremises/api for the auth shape.

The `structurizr-cli` tool (separate Go binary) wraps `push`/`pull`/`lock`/
`unlock` against any workspace. https://docs.structurizr.com/cli

## Pushing (homelab specifics)

SKILL.md has the canonical `structurizr-cli push` recipe. Extras here.

### Raw curl (no structurizr-cli)

```sh
ID=1
URL=http://localhost:18080/api          # via kubectl port-forward
structurizr-cli export -workspace STRUCTURIZR.dsl -format json -output /tmp/sx
jq '.id='$ID /tmp/sx/STRUCTURIZR.json > /tmp/sx/workspace.json

PATH_="/api/workspace/$ID"
NONCE=$(($(date +%s%N) / 1000000))
CT="application/json; charset=UTF-8"
MD5=$(openssl md5 -binary < /tmp/sx/workspace.json | base64)
TO_SIGN=$(printf 'PUT\n%s\n%s\n%s\n%s\n' "$PATH_" "$MD5" "$CT" "$NONCE")
SIG=$(printf '%s' "$TO_SIGN" | openssl dgst -sha256 -hmac "$SECRET" -binary | base64)

curl -X PUT "$URL/workspace/$ID" \
  -H "X-Authorization: $KEY:$SIG" \
  -H "Nonce: $NONCE" \
  -H "Content-MD5: $MD5" \
  -H "Content-Type: $CT" \
  --data-binary @/tmp/sx/workspace.json
```

### HMAC contract

- Sign string: `<METHOD>\n<PATH>\n<Content-MD5 base64>\n<Content-Type>\n<Nonce>\n`
- Algorithm: HMAC-SHA256 with `apiSecret` as key.
- Headers required on the request:
  - `X-Authorization: <apiKey>:<base64(hmac)>`
  - `Nonce: <epoch ms>`
  - `Content-MD5: <base64(md5(body))>`
  - `Content-Type: application/json; charset=UTF-8`
- Server stores both creds as **bcrypt** hashes in
  `/usr/local/structurizr/<id>/workspace.properties` (`apiKey=` / `apiSecret=`).
  Pod restart picks up changes; or just let it re-read (some structurizr
  versions reload on each request).

### Skipping the port-forward (not currently active)

Adding `--skip-auth-regex=^/api/` to the structurizr-oauth2-proxy Deployment
removes the OIDC gate on `/api/*` — `curl` from anywhere with Vault access
then works against `https://structurizr.oddkin.co/api/workspace/<id>`. Trade:
HMAC becomes the sole gate. We've not flipped this on; the port-forward path
keeps the OIDC perimeter intact.

### Rollback artifacts

`structurizr-cli push` writes `structurizr-<id>-<ts>.json` next to the DSL
(the previous workspace JSON for rollback). The `.gitignore` pattern
`structurizr-[0-9]*-[0-9]*.json` excludes them.

## SVG extraction (no native exporter)

Structurizr has no server-side SVG render endpoint, and `structurizr-cli
export -format` accepts `plantuml | mermaid | dot | ilograph | json | theme
| static | fqcn` but **not** `svg` or `png`. The `static` exporter produces
an HTML site whose diagrams are rendered client-side by JS at view time,
not pre-baked.

Workaround: drive a headless browser, switch views via the URL hash, and
serialize the rendered `<svg id="v-*">` element straight out of the DOM.
That's the same SVG the human-facing UI shows.

### Selector

After the workspace loads, the diagram is rendered inline as:

```html
<svg id="v-2" width="100%" height="100%">…</svg>
```

The `v-<n>` id is per-view (incremented by the SPA on view switch). Other
SVG elements on the page are small icons (20×20). Pick the largest, or
filter on `id^="v-"`.

### Browser-side serialization snippet

```js
() => {
    const svgs = Array.from(document.querySelectorAll('svg[id^="v-"]'));
    if (!svgs.length) return null;
    const top = svgs[svgs.length - 1];          // current view if multiple exist
    const clone = top.cloneNode(true);
    clone.setAttribute('xmlns', 'http://www.w3.org/2000/svg');
    clone.setAttribute('xmlns:xlink', 'http://www.w3.org/1999/xlink');
    const bbox = top.getBoundingClientRect();
    if (bbox.width && bbox.height) {
        clone.setAttribute('width',  Math.round(bbox.width));
        clone.setAttribute('height', Math.round(bbox.height));
    }
    return new XMLSerializer().serializeToString(clone);
}
```

The `xmlns` attributes + concrete `width`/`height` make the result a
standalone openable SVG (browsers and most renderers refuse the bare
inline-form without them).

### Switching views

For each view, `page.goto("https://structurizr.oddkin.co/workspace/1/diagrams#<viewKey>")`
once the SPA is loaded swaps the rendered diagram. Pre-loading the hash
on the first goto doesn't reliably trigger a view-change — the SPA hooks
hashchange after its own init. So: settle on `/diagrams` first, then
hash-set per view, with a `wait_for_timeout(1500)` to let the redraw
finish.

The `<select id="exportViewList">` you'll find in the DOM is **not** the
visible toolbar dropdown — it's the hidden multi-select inside the
"Export" dialog, only visible when the dialog opens. Don't waste time
trying to `select_option` on it.

### Full Playwright shape (works on our oauth2-proxy-gated deployment)

```python
from playwright.sync_api import sync_playwright
import rumi_smoke_lib as rs   # the homelab passkey helper

vault = rs._vault_client(); cred = rs._vault_creds(vault)
pw = sync_playwright().start()
browser = pw.chromium.launch(headless=True)
ctx = browser.new_context(); page = ctx.new_page()
page.set_default_timeout(60000)
cdp, aid = rs._attach_virtual_authenticator(page, cred)

def settle(url):
    """Goto + handle ≤4 OIDC bounce rounds, return when on structurizr."""
    page.goto(url, wait_until="domcontentloaded")
    for _ in range(4):
        if "id.oddkin.co" in page.url:
            page.locator("button#authenticateWebAuthnButton").click(timeout=8000)
            page.wait_for_url("**structurizr.oddkin.co/**", timeout=20000)
        page.wait_for_load_state("networkidle", timeout=10000)
        if "structurizr.oddkin.co" in page.url and "id.oddkin.co" not in page.url:
            return

settle("https://structurizr.oddkin.co/workspace/1/diagrams")

GRAB = "..."  # the JS snippet above
for view in ("context", "l2-overview"):
    page.goto(f"https://structurizr.oddkin.co/workspace/1/diagrams#{view}",
              wait_until="domcontentloaded")
    page.wait_for_load_state("networkidle", timeout=15000)
    page.wait_for_timeout(2000)
    svg = page.evaluate(GRAB)
    open(f"/tmp/STRUCTURIZR-{view}.svg", "w").write(
        '<?xml version="1.0" encoding="UTF-8"?>\n' + svg)

rs._vault_write_signcount(vault, cred, rs._read_signcount(cdp, aid))
ctx.close(); browser.close(); pw.stop()
```

The auth bounce loop matters: structurizr's client-side JS makes an
internal API call after the initial page load that triggers a *second*
OIDC redirect through oauth2-proxy on the structurizr OIDC client
(separate from the oauth2-proxy gate). Handle ≥2 rounds.

### Counter drift

Each browser session's CDP virtual authenticator increments signCount on
every assertion. Keycloak's `webauthn4j` strict-counter check rejects any
assertion whose count ≤ stored counter. Before a long extraction run,
sync Vault's `kv/rumi/passkey.sign_count` to Keycloak's stored counter
(see rumi-keycloak-passkey-vault notes); the smoke wrapper auto-persists
post-assertion.

## Common gotchas

- **Empty workspace JSON**: the Diagrams tab shows nothing if there are no
  views defined, even with a populated model.
- **Implied relationships**: with `!impliedRelationships true`, a relation
  between two children also implicitly relates their parents. Off by default;
  turn on if your system-context view is missing edges.
- **Identifier conflicts**: with `!identifiers flat`, two `container "Web"`
  in different systems collide. Either rename or switch to `hierarchical`.
- **Encryption password is per-workspace**, set in Settings. Lose it and the
  workspace contents are unrecoverable (the server only stores the
  ciphertext). Out of band: stash in Vault if it matters.
- **API write workflow**: `structurizr-cli push` overwrites the workspace
  atomically; use `lock` first if multiple writers.
