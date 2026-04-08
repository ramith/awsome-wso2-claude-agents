---
name: APIM Quick Start Skill Design
description: Design decisions and flow for the apim/skills/quick-start skill that deploys WSO2 APIM and sets up PizzaShack sample API end-to-end
type: project
---

## Skill Location
`apim/skills/quick-start/SKILL.md`

## Target Version
WSO2 API Manager **4.6.0** (4.7 releasing soon, will update later)

## Flow

```
1. Check prerequisites (JDK 21, Docker, ports 9443/8243/8280)
        │
        ▼
2. Ask: "Zip or Docker?"
   ├─ Docker → Ask "Do you have WSO2 subscription credentials?"
   │    ├─ Yes → pull from WSO2 private registry (with updates)
   │    └─ No  → pull wso2/wso2am:4.6.0 from Docker Hub (open-source)
   └─ Zip → user downloads manually, skill extracts & starts
        │
        ▼
3. Start APIM, wait for readiness (poll health endpoint, ~120s timeout)
        │
        ▼
4. PizzaShack auto-deploys on first start in CREATED state
   - Find via GET /api/am/publisher/v4/apis?query=name:PizzaShackAPI
   - Publish via POST change-lifecycle?action=Publish
        │
        ▼
5. Automate subscription via REST APIs:
   a. Register DCR client → get client_id/secret
   b. Get OAuth2 token via password grant (admin/admin)
   c. Publish PizzaShack
   d. Subscribe PizzaShack to DefaultApplication
   e. Generate consumer key/secret + access token
        │
        ▼
6. Execute test curl to prove it works, then print:
   - "Your APIM is running"
   - Publisher URL (https://localhost:9443/publisher) + DevPortal URL (/devportal)
   - Credentials: admin/admin
   - Ready-to-use curl command with generated token
   - Postman note: disable SSL verification for localhost
        │
        ▼
7. Cleanup: stop server / docker stop & rm
```

## Key Design Decisions

1. **Deployment options**: Docker (preferred) or Zip — ask user first
2. **Docker registry**: Ask for subscription credentials, fall back to open-source Docker Hub image
3. **Sample API**: Use built-in PizzaShack (ships with APIM, auto-deployed on first start)
4. **Automation level**: Full — create+publish+subscribe+generate token all via REST API
5. **Test invocation**: Execute the curl in-skill to prove it works, also print command for reuse
6. **Docker ports**: Minimal — 9443 (portals), 8243 (gateway HTTPS), 8280 (gateway HTTP)
7. **Container name**: `wso2apim-quickstart` for easy cleanup
8. **Skill structure**: Single `SKILL.md` — agent handles branching logic, no helper scripts
9. **Tone**: Automated setup with instructional output at the end ("here's what's running, here are the links")

## Open Items to Verify During Implementation

- Does PizzaShack in 4.6 ship with a working backend, or does it point to an external mock?
- REST API auth flow: DCR → client credentials → password grant — need exact curl sequences for 4.6
- Docker Hub image tag for open-source 4.6 (`wso2/wso2am:4.6.0` — confirm exact tag)
- WSO2 private registry URL and auth mechanism for subscription users

## Error Handling

- Port already in use → detect and inform user
- Docker daemon not running → detect and suggest starting it
- APIM fails to start (bad JDK, insufficient memory ~2GB heap)
- REST API calls fail despite health check passing → retry with backoff
- Self-signed cert → all curl calls use `-k` flag
