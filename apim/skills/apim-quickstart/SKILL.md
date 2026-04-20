---
name: apim-quickstart
description: Deploy and start WSO2 API Manager 4.6.0. Supports local zip, Docker, Docker Compose, and remote VM (SSH). Use when the user wants to quickly spin up an APIM instance for development, testing, or evaluation.
allowed-tools: Read Write Edit Bash Glob Grep
---

Deploy WSO2 API Manager 4.6.0 and print the portal URLs. Ask the user which deployment mode they want if not specified.

## Deployment Modes

Ask: **"How do you want to deploy APIM?"**

| Mode | When to use |
|------|-------------|
| **local-zip** | User has downloaded the APIM zip, wants to run on bare metal |
| **local-docker** | Quick single-container setup on localhost |
| **docker-compose** | Part of a multi-service stack, or standalone compose |
| **remote-vm** | Deploy to a remote machine via SSH |

---

## Mode A: Local Zip

### Prerequisites
1. Detect and export `JAVA_HOME` for JDK 21. **`java -version` is unreliable on macOS** — it shows whichever JDK is first on `PATH`, which may not be 21 even if 21 is installed. Branch on OS:

   ```bash
   if [ "$(uname)" = "Darwin" ]; then
     # macOS: use the canonical Apple-supplied probe
     JAVA_HOME=$(/usr/libexec/java_home -v 21 2>/dev/null) || { echo "JDK 21 not installed"; exit 1; }
   else
     # Linux: derive from java on PATH (must already be JDK 21)
     java -version 2>&1 | grep -q 'version "21' || { echo "JDK 21 not on PATH"; exit 1; }
     JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
   fi
   export JAVA_HOME
   ```

   If JDK 21 is missing, stop and tell the user to install it.

2. Check `unzip` is available:
   ```bash
   which unzip
   ```
   If missing, tell the user to install it (`sudo apt install -y unzip` on Debian/Ubuntu). On macOS, `unzip` is preinstalled and this check should always pass.

3. Check ports 9443, 8243, 8280 are free. Note: `lsof` exits with status 1 when nothing matches, which is the *good* case — wrap the check explicitly:
   ```bash
   PORTS_IN_USE=$(lsof -i :9443 -i :8243 -i :8280 -sTCP:LISTEN 2>/dev/null)
   if [ -n "$PORTS_IN_USE" ]; then
     echo "ERROR: ports in use:"
     echo "$PORTS_IN_USE"
     exit 1
   fi
   ```
   See **Error Handling** for recovery options when a port is occupied.

### Deploy
1. Ask the user for the path to the APIM zip. Accept both naming conventions:
   - Open source: `wso2am-4.6.0.zip` — `https://github.com/wso2/product-apim/releases`
   - WSO2 with updates: `wso2am-4.6.0.<N>.zip` (e.g., `wso2am-4.6.0.8.zip`) — `https://wso2.com/api-manager/`

   When resolving the path, glob `wso2am-4.6.0*.zip` rather than assuming the exact filename.

2. Extract into the current working directory (the user's project folder, not `/tmp`) and start:
   ```bash
   EXTRACT_DIR="$(pwd)/wso2apim-quickstart"
   unzip <path-to-zip> -d "$EXTRACT_DIR"
   APIM_HOME=$(find "$EXTRACT_DIR" -maxdepth 1 -type d -name 'wso2am-*' | head -1)
   # Re-export JAVA_HOME (use the OS-branched detection from Prerequisites step 1)
   sh "$APIM_HOME/bin/api-manager.sh" &
   ```

   Note: the extracted folder is always `wso2am-4.6.0/` regardless of the zip's update suffix — derive `APIM_HOME` from `find`, never from the zip filename.

3. Wait for readiness (see **Readiness Check** below).

### Cleanup
`JAVA_HOME` must be set for the stop script too — re-export it (using the same OS-branched detection above) before stopping:
```bash
export JAVA_HOME=...   # see Prerequisites step 1
sh "$APIM_HOME/bin/api-manager.sh" --stop
rm -rf "$EXTRACT_DIR"
```

---

## Mode B: Local Docker

### Prerequisites
1. Check Docker is running:
   ```
   docker info > /dev/null 2>&1
   ```
   If it fails, tell the user to start their Docker runtime (Docker Desktop, Colima via `colima start`, OrbStack, etc. — the right command depends on which one they use).

2. **Check VM memory.** APIM needs ~4 GiB. Default Colima/Docker Desktop installs ship with 2 GiB, which causes silent OOM kills on startup (the only signal is `Killed` at the end of `docker logs`). Recommended: 8 GiB / 4 CPU.
   ```bash
   MEM_BYTES=$(docker info --format '{{.MemTotal}}')
   MEM_GIB=$((MEM_BYTES / 1024 / 1024 / 1024))
   if [ "$MEM_GIB" -lt 4 ]; then
     echo "ERROR: Docker VM has ${MEM_GIB} GiB. APIM needs ~4 GiB minimum."
     echo "  Colima:  colima stop && edit ~/.colima/default/colima.yaml (memory: 6+) && colima start"
     echo "  Desktop: Settings -> Resources -> Memory"
     exit 1
   fi
   ```

3. Check for existing containers already publishing the APIM ports (a stale `wso2apim-quickstart` from a prior run, or another APIM stack the user has running):
   ```bash
   CONFLICT=$(docker ps --filter "publish=9443" --filter "publish=8243" --filter "publish=8280" \
     --format '{{.Names}} ({{.Image}})')
   if [ -n "$CONFLICT" ]; then
     echo "Existing container(s) already bound to APIM ports:"
     echo "$CONFLICT"
     # Offer: (a) reuse it, (b) stop+remove it, (c) run on an offset
   fi
   ```

4. Check ports 9443, 8243, 8280 are free at the host level (same wrapped `lsof` as zip mode). On macOS+Colima with `portForwarder: ssh`, the host listener for forwarded ports may show as an `ssh` PID — that doesn't mean APIM is involved; correlate with the `docker ps` check above.

### Docker Image Selection
Ask: **"Do you have WSO2 subscription credentials for updated Docker images?"**

- **Yes** → WSO2 runs a Harbor registry at `registry.wso2.com`. Guide them to login and pull:
  ```bash
  docker login registry.wso2.com
  # Username/password: WSO2 subscription credentials
  docker pull registry.wso2.com/wso2-apim/am:4.6.0
  ```
  Use image `registry.wso2.com/wso2-apim/am:4.6.0` in the run command.
  Available tags: `4.6.0`, `4.6.0.21`, `4.6.0.0`, `latest`. Use `4.6.0` for stability.

- **No** → Use the open-source image from Docker Hub:
  ```
  wso2/wso2am:4.6.0
  ```

### Deploy
Clean up any stale container with the same name first (idempotent re-run):
```bash
docker rm -f wso2apim-quickstart 2>/dev/null || true

docker run -d \
  --name wso2apim-quickstart \
  -p 9443:9443 \
  -p 8243:8243 \
  -p 8280:8280 \
  <IMAGE>
```

Wait for readiness (see **Readiness Check** below).

### Cleanup
```bash
# Pause (keep container + in-memory H2 state for next session):
docker stop wso2apim-quickstart

# Resume:
docker start wso2apim-quickstart

# Remove entirely (discards H2 state):
docker rm -f wso2apim-quickstart
```

---

## Mode C: Docker Compose

### Prerequisites
Run all four Mode B prerequisite checks: (1) Docker runtime is up, (2) Docker VM has ≥ 4 GiB memory, (3) no existing container is publishing 9443/8243/8280, (4) host ports are free. Skipping the memory check is the most common cause of silent OOM kills in compose deployments.

### Deploy

**Standalone (no existing compose file):** Generate a `docker-compose.yml`:

```yaml
services:
  apim:
    image: <IMAGE>
    container_name: wso2apim-quickstart
    ports:
      - "9443:9443"
      - "8243:8243"
      - "8280:8280"
    healthcheck:
      # /services/Version is served only after the Carbon server has fully started.
      # Do NOT use /carbon/admin/login.jsp — it returns 200 well before apps are deployed.
      test: ["CMD", "curl", "-skf", "--max-time", "5", "https://localhost:9443/services/Version"]
      interval: 15s
      timeout: 10s
      retries: 12
      start_period: 60s
    restart: unless-stopped
```

Use the same image selection logic as Mode B (subscription vs open-source).

**Adding to an existing compose file:** Read the user's existing `docker-compose.yml`, add the `apim` service block above, preserve all existing services unchanged. Do NOT add a `version` key — it is obsolete in modern Docker Compose.

Then start:
```bash
docker compose up -d
```

Wait for readiness (see **Readiness Check** below).

### Cleanup
```bash
# Pause (keep container + in-memory H2 state):
docker compose stop

# Resume:
docker compose start

# Remove entirely (discards H2 state, removes network):
docker compose down
```

---

## Mode D: Remote VM

### Gather Info
Ask the user for:
- **Hostname or IP** of the remote machine
- **SSH user** (e.g., `ubuntu`, `ec2-user`, `azureuser`)
- **Deploy method on remote**: zip or Docker
- **Install path** on remote — suggest:
  - `/opt/wso2` — standard for third-party server software (needs sudo)
  - `/home/<user>/wso2` — user home directory (no sudo needed)
  - `/usr/local/wso2` — locally installed software

### Prerequisites (run via SSH)
1. Check SSH connectivity:
   ```bash
   ssh <user>@<host> 'echo ok'
   ```
   If this fails, stop and tell the user to configure SSH access (key-based auth recommended).

2. If zip mode — check JDK 21 on remote:
   ```bash
   ssh <user>@<host> 'java -version'
   ```
   Must show version 21. If missing, tell the user to install it — do NOT install it yourself.

3. If zip mode — check `unzip` on remote:
   ```bash
   ssh <user>@<host> 'which unzip'
   ```
   If missing, tell the user to install it (`sudo apt install -y unzip`).

4. If zip mode — detect and export `JAVA_HOME` on remote (assumes Linux target, which is nearly always the case for cloud VMs):
   ```bash
   ssh <user>@<host> 'export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java)))) && echo JAVA_HOME=$JAVA_HOME'
   ```

5. If Docker mode — check Docker on remote and verify VM memory ≥ 4 GiB (same rationale as Mode B step 2):
   ```bash
   ssh <user>@<host> 'docker info > /dev/null 2>&1 && echo ok'
   ssh <user>@<host> 'MEM_BYTES=$(docker info --format "{{.MemTotal}}") && echo "Docker MemTotal: $((MEM_BYTES / 1024 / 1024 / 1024)) GiB"'
   ```
   If under 4 GiB, stop and tell the user to resize the Docker VM.

6. If Docker mode — check for existing containers publishing the APIM ports on the remote:
   ```bash
   ssh <user>@<host> 'docker ps --filter "publish=9443" --filter "publish=8243" --filter "publish=8280" --format "{{.Names}} ({{.Image}})"'
   ```
   If non-empty, ask the user how to handle the conflict (reuse, remove, or use offset).

7. Check host-level ports on remote. `ss` exits 0 even when the grep pipeline is empty, so check the captured output explicitly:
   ```bash
   PORTS_IN_USE=$(ssh <user>@<host> 'ss -tlnp 2>/dev/null | awk "/:(9443|8243|8280) /"')
   if [ -n "$PORTS_IN_USE" ]; then
     echo "ERROR: ports in use on remote:"
     echo "$PORTS_IN_USE"
     exit 1
   fi
   ```

### Deploy (zip on remote)

**CRITICAL: All `deployment.toml` changes MUST be applied BEFORE the first startup.** APIM initializes its H2 database on first boot using values from `deployment.toml`. If started with `localhost` defaults, the DB will contain wrong URLs for OAuth callbacks, service providers, and gateway endpoints. These cannot be fixed by editing config later — the only recovery is a full re-extract.

```bash
# Copy zip to remote
scp <path-to-zip> <user>@<host>:/tmp/

# Extract to chosen install path. The zip filename may be wso2am-4.6.0.zip (OSS)
# or wso2am-4.6.0.<N>.zip (subscription) — derive ZIP_NAME from the actual file.
ssh <user>@<host> 'ZIP_NAME=$(basename /tmp/wso2am-4.6.0*.zip) && unzip /tmp/$ZIP_NAME -d <INSTALL_PATH>'
```

The extracted directory is always `wso2am-4.6.0/` regardless of update suffix.

**Apply deployment.toml changes** (replace `<IP_OR_DOMAIN>` with the remote host's IP or domain):

```bash
APIM_HOME="<INSTALL_PATH>/wso2am-4.6.0"
ssh <user>@<host> "cat > /tmp/apim-config-patch.sh << 'SCRIPT'
CONF=\"${APIM_HOME}/repository/conf/deployment.toml\"
HOST=\"<IP_OR_DOMAIN>\"

# a) Server hostname
sed -i 's/^hostname = .*/hostname = \"'\"$HOST\"'\"/' \"\$CONF\"

# b) Gateway environment endpoints
sed -i 's|service_url = .*|service_url = \"https://'\"$HOST\"':\${mgt.transport.https.port}/services/\"|' \"\$CONF\"
sed -i 's|ws_endpoint = .*|ws_endpoint = \"ws://'\"$HOST\"':9099\"|' \"\$CONF\"
sed -i 's|wss_endpoint = .*|wss_endpoint = \"wss://'\"$HOST\"':8099\"|' \"\$CONF\"
sed -i 's|http_endpoint = .*|http_endpoint = \"http://'\"$HOST\"':\${http.nio.port}\"|' \"\$CONF\"
sed -i 's|https_endpoint = .*|https_endpoint = \"https://'\"$HOST\"':\${https.nio.port}\"|' \"\$CONF\"
sed -i 's|websub_event_receiver_http_endpoint = .*|websub_event_receiver_http_endpoint = \"http://'\"$HOST\"':9021\"|' \"\$CONF\"
sed -i 's|websub_event_receiver_https_endpoint = .*|websub_event_receiver_https_endpoint = \"https://'\"$HOST\"':8021\"|' \"\$CONF\"

# c) OAuth revoke endpoint (uncomment and set)
sed -i 's|^#.*revoke_endpoint = .*|revoke_endpoint = \"https://'\"$HOST\"':\${https.nio.port}/revoke\"|' \"\$CONF\"

# d) DevPortal URL (uncomment and set)
sed -i 's|^#.*\[apim.devportal\]|[apim.devportal]|' \"\$CONF\"
sed -i 's|^#.*url = .*/devportal.*|url = \"https://'\"$HOST\"':\${mgt.transport.https.port}/devportal\"|' \"\$CONF\"

# e) OAuth endpoints (append — not in default config)
cat >> \"\$CONF\" << EOF

[oauth.endpoints]
oauth2_token_url = \"https://${HOST}:9443/oauth2/token\"
oauth2_revoke_url = \"https://${HOST}:9443/oauth2/revoke\"
oauth2_authz_url = \"https://${HOST}:9443/oauth2/authorize\"
EOF

echo \"deployment.toml configured for \$HOST\"
SCRIPT
chmod +x /tmp/apim-config-patch.sh && bash /tmp/apim-config-patch.sh"
```

**Start APIM** (only after config is applied):
```bash
ssh <user>@<host> "export JAVA_HOME=\$(dirname \$(dirname \$(readlink -f \$(which java)))) && sh ${APIM_HOME}/bin/api-manager.sh &"
```

### Deploy (Docker on remote)
Idempotent re-run: remove any stale container with the same name first.
```bash
ssh <user>@<host> 'docker rm -f wso2apim-quickstart 2>/dev/null || true'
ssh <user>@<host> 'docker run -d --name wso2apim-quickstart -p 9443:9443 -p 8243:8243 -p 8280:8280 <IMAGE>'
```

Wait for readiness using the **remote host** (see below).

### Firewall / Security Group Ports

After deployment, remind the user to open these ports in their cloud provider's firewall, security group, or NSG:

| Port | Purpose |
|------|---------|
| 9443 | Management console, Publisher, DevPortal, Admin, Carbon |
| 8243 | HTTPS Gateway (API traffic) |
| 8280 | HTTP Gateway (API traffic) |

Example for Azure NSG:
```bash
az network nsg rule create --resource-group <rg> --nsg-name <nsg> \
  --name AllowAPIM --priority 110 \
  --destination-port-ranges 9443 8243 8280 \
  --protocol Tcp --access Allow
```

For AWS Security Groups, GCP Firewall Rules, etc., the user should open the same ports in their respective console or CLI.

### Cleanup (remote)
```bash
# Zip — JAVA_HOME must be set for the stop script too
ssh <user>@<host> "export JAVA_HOME=\$(dirname \$(dirname \$(readlink -f \$(which java)))) && sh <INSTALL_PATH>/wso2am-4.6.0/bin/api-manager.sh --stop && rm -rf <INSTALL_PATH>/wso2am-4.6.0"

# Docker — pause/resume/remove
ssh <user>@<host> 'docker stop wso2apim-quickstart'    # pause
ssh <user>@<host> 'docker start wso2apim-quickstart'   # resume
ssh <user>@<host> 'docker rm -f wso2apim-quickstart'   # remove entirely
```

---

## Readiness Check

**Do not use `/carbon/admin/login.jsp` returning 200 as the readiness signal** — Carbon serves login.jsp very early in boot, well before the Publisher/DevPortal apps are deployed. A user who acts on that signal will hit partial 500s from the Publisher API.

The authoritative signal is the log line `WSO2 Carbon started in <N> sec`. Wait for that, then do a short HTTP probe to confirm the port is reachable.

### Docker mode
```bash
MAX_WAIT=180
ELAPSED=0
until docker logs wso2apim-quickstart 2>&1 | grep -q "WSO2 Carbon started in"; do
  sleep 2
  ELAPSED=$((ELAPSED + 2))
  if [ $ELAPSED -ge $MAX_WAIT ]; then
    echo "ERROR: APIM did not start within ${MAX_WAIT}s. Check 'docker logs wso2apim-quickstart'."
    exit 1
  fi
done
curl -skf --max-time 5 "https://localhost:9443/services/Version" >/dev/null \
  && echo "APIM is ready (took ~${ELAPSED}s)."
```

### Zip mode (local or remote)
```bash
LOG="$APIM_HOME/repository/logs/wso2carbon.log"   # remote: prefix with: ssh <user>@<host>
MAX_WAIT=180
ELAPSED=0
until grep -q "WSO2 Carbon started in" "$LOG" 2>/dev/null; do
  sleep 2
  ELAPSED=$((ELAPSED + 2))
  if [ $ELAPSED -ge $MAX_WAIT ]; then
    echo "ERROR: APIM did not start within ${MAX_WAIT}s. Check $LOG."
    exit 1
  fi
done
curl -skf --max-time 5 "https://${APIM_HOST}:9443/services/Version" >/dev/null \
  && echo "APIM is ready (took ~${ELAPSED}s)."
```

For remote VM, run the curl probe from the local machine (not over SSH) — it confirms the remote portals are reachable from the user's network. Always use `--max-time` on curl to avoid hangs.

---

## Output

Once APIM is ready, print:

```
WSO2 API Manager 4.6.0 is running.

Portal URLs:
  Publisher:  https://<HOST>:9443/publisher
  DevPortal:  https://<HOST>:9443/devportal
  Admin:      https://<HOST>:9443/admin
  Carbon:     https://<HOST>:9443/carbon

Credentials: admin / admin

Logs:    docker logs -f wso2apim-quickstart                    (Docker mode)
         tail -f <APIM_HOME>/repository/logs/wso2carbon.log    (zip mode)
Stop:    docker stop wso2apim-quickstart                       (Docker mode)
         sh <APIM_HOME>/bin/api-manager.sh --stop              (zip mode)

Note: You will see a browser security warning for the self-signed certificate — accept it to proceed.

Note: After opening the Publisher/DevPortal in a browser you'll see log lines like
  ERROR - PostAuthenticationInterceptor Authentication failed: Bearer/Basic authentication header is missing
These are benign — the UI makes unauthenticated probes before login. Not a startup failure.
```

Substitute placeholders before printing:
- `<HOST>` → `localhost` for local modes, or the remote hostname/IP for VM mode
- `<APIM_HOME>` → the actual extract path from the deploy step (zip mode only)

---

## Error Handling

| Error | Action |
|-------|--------|
| Port 9443/8243/8280 in use | Diagnose with BOTH `lsof -i :9443 -P` (host-level owner — may show an `ssh` PID for Colima-forwarded ports) AND `docker ps --filter "publish=9443"` (any container already publishing it). Recovery options: (1) kill/remove the offender if it's a stale APIM instance, or (2) set `[server] offset = 1` in `deployment.toml` so APIM uses 9444/8244/8281 instead. |
| Container exits with `Killed` in `docker logs` | Docker VM ran out of memory. Check `docker info` MemTotal; APIM needs ≥ 4 GiB. Increase the runtime's VM memory (Colima: edit `~/.colima/default/colima.yaml`; Docker Desktop: Settings → Resources → Memory). |
| Docker not running | Tell user to start their Docker runtime (Docker Desktop, `colima start`, OrbStack, etc.). |
| JDK not found or wrong version | Tell user to install JDK 21. On macOS, `/usr/libexec/java_home -v 21` is the canonical probe. Do not install JDK yourself. |
| SSH connection fails | Report the SSH error. Suggest checking hostname, user, and key config. |
| APIM fails to start within 180s | Tell user to check logs at `<APIM_HOME>/repository/logs/wso2carbon.log` (zip) or `docker logs wso2apim-quickstart` (Docker). |
| `registry.wso2.com` login fails | Harbor registry requires valid WSO2 subscription credentials. Confirm credentials and retry, or fall back to open-source Docker Hub image (`wso2/wso2am:4.6.0`). |
| `ERROR - PostAuthenticationInterceptor Authentication failed` in logs | Benign — UI makes unauthenticated probes before login. Not a failure. Ignore unless you see it during automated API calls. |

---

## Reference

Official quick start guide (fetch if you need to verify any details):
`https://apim.docs.wso2.com/en/latest/get-started/api-manager-quick-start-guide/`
