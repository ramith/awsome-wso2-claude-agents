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
1. Check JDK 21:
   ```
   java -version
   ```
   Must show version 21. If missing, stop and tell the user to install JDK 21 and set `JAVA_HOME`.

2. Check ports 9443, 8243, 8280 are free:
   ```
   lsof -i :9443 -i :8243 -i :8280
   ```
   If any are in use, report which port is occupied and stop.

### Deploy
1. Ask the user for the path to the APIM zip (e.g., `wso2am-4.6.0.zip`). If they haven't downloaded it, point them to:
   - Open source: `https://github.com/wso2/product-apim/releases`
   - WSO2 with updates: `https://wso2.com/api-manager/`

2. Extract and start:
   ```bash
   unzip <path-to-zip> -d /tmp/wso2apim-quickstart
   APIM_HOME=$(find /tmp/wso2apim-quickstart -maxdepth 1 -type d -name 'wso2am-*' | head -1)
   sh "$APIM_HOME/bin/api-manager.sh" &
   ```

3. Wait for readiness (see **Readiness Check** below).

### Cleanup
```bash
sh "$APIM_HOME/bin/api-manager.sh" --stop
rm -rf /tmp/wso2apim-quickstart
```

---

## Mode B: Local Docker

### Prerequisites
1. Check Docker is running:
   ```
   docker info > /dev/null 2>&1
   ```
   If it fails, tell the user to start Docker.

2. Check ports 9443, 8243, 8280 are free (same as zip mode).

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
```bash
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
docker stop wso2apim-quickstart && docker rm wso2apim-quickstart
```

---

## Mode C: Docker Compose

### Prerequisites
Same as local Docker (Docker running, ports free).

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
      test: ["CMD", "curl", "-sk", "https://localhost:9443/carbon/admin/login.jsp"]
      interval: 15s
      timeout: 10s
      retries: 10
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
docker compose down
```

---

## Mode D: Remote VM

### Gather Info
Ask the user for:
- **Hostname or IP** of the remote machine
- **SSH user** (e.g., `ubuntu`, `ec2-user`)
- **Deploy method on remote**: zip or Docker

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

3. If Docker mode — check Docker on remote:
   ```bash
   ssh <user>@<host> 'docker info > /dev/null 2>&1 && echo ok'
   ```

4. Check ports on remote:
   ```bash
   ssh <user>@<host> 'ss -tlnp | grep -E "9443|8243|8280"'
   ```

### Deploy (zip on remote)
```bash
# Copy zip to remote
scp <path-to-zip> <user>@<host>:/tmp/

# Extract and start
ssh <user>@<host> 'unzip /tmp/wso2am-4.6.0.zip -d /tmp/wso2apim-quickstart && sh /tmp/wso2apim-quickstart/wso2am-4.6.0/bin/api-manager.sh &'
```

### Deploy (Docker on remote)
```bash
ssh <user>@<host> 'docker run -d --name wso2apim-quickstart -p 9443:9443 -p 8243:8243 -p 8280:8280 <IMAGE>'
```

Wait for readiness using the **remote host** (see below).

### Cleanup (remote)
```bash
# Zip
ssh <user>@<host> 'sh /tmp/wso2apim-quickstart/wso2am-4.6.0/bin/api-manager.sh --stop && rm -rf /tmp/wso2apim-quickstart'

# Docker
ssh <user>@<host> 'docker stop wso2apim-quickstart && docker rm wso2apim-quickstart'
```

---

## Readiness Check

Poll until APIM is ready. APIM takes 60–120 seconds to start. Use the correct `<HOST>` based on deployment mode (`localhost` for local modes, remote hostname for VM mode).

```bash
APIM_HOST="<HOST>"
MAX_WAIT=120
ELAPSED=0
while [ $ELAPSED -lt $MAX_WAIT ]; do
  if curl -sk "https://${APIM_HOST}:9443/carbon/admin/login.jsp" -o /dev/null -w '%{http_code}' 2>/dev/null | grep -q '200'; then
    echo "APIM is ready."
    break
  fi
  sleep 10
  ELAPSED=$((ELAPSED + 10))
  echo "Waiting for APIM to start... (${ELAPSED}s / ${MAX_WAIT}s)"
done

if [ $ELAPSED -ge $MAX_WAIT ]; then
  echo "ERROR: APIM did not become ready within ${MAX_WAIT}s. Check logs."
fi
```

For remote VM, run the readiness check from the local machine (not over SSH) — it confirms the remote portals are reachable from the user's network.

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

Note: You will see a browser security warning for the self-signed certificate — accept it to proceed.
```

Replace `<HOST>` with `localhost` for local modes, or the remote hostname/IP for VM mode.

---

## Error Handling

| Error | Action |
|-------|--------|
| Port 9443/8243/8280 in use | Report which port and what process holds it. Do not proceed. |
| Docker not running | Tell user to start Docker Desktop or the Docker daemon. |
| JDK not found or wrong version | Tell user to install JDK 21 and set JAVA_HOME. Do not install it. |
| SSH connection fails | Report the SSH error. Suggest checking hostname, user, and key config. |
| APIM fails to start within 120s | Tell user to check logs at `<APIM_HOME>/repository/logs/wso2carbon.log` (zip) or `docker logs wso2apim-quickstart` (Docker). |
| `registry.wso2.com` login fails | Harbor registry requires valid WSO2 subscription credentials. Confirm credentials and retry, or fall back to open-source Docker Hub image (`wso2/wso2am:4.6.0`). |

---

## Reference

Official quick start guide (fetch if you need to verify any details):
`https://apim.docs.wso2.com/en/latest/get-started/api-manager-quick-start-guide/`
