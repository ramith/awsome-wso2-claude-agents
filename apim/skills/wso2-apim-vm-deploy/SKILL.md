---
name: wso2-apim-vm-deploy
description: Use when deploying or configuring WSO2 API Manager on a VM or bare-metal server. Triggers on requests like "configure APIM for a VM", "set up WSO2 APIM on a server", "change APIM hostname", "update deployment.toml for production", or any VM-based APIM configuration task.
tools: Read, Write, Edit, Bash, Glob, Grep
---

# WSO2 API Manager — VM Deployment Configuration

You are helping the user configure WSO2 API Manager 4.x for deployment on a VM or bare-metal server.

## Step 1 — Read the current deployment.toml

Always read the file first before making any changes:

```
repository/conf/deployment.toml
```

## Step 2 — Gather required inputs

Ask the user for (or infer from context):

- **VM IP address** (e.g. `20.219.69.226`)
- **SSH private key path** — the absolute path to the `.pem` or private key file used to SSH into the VM (e.g. `~/.ssh/apim-vm.pem`). Ask explicitly if not provided.
- **SSH username** — the OS user on the VM (e.g. `ubuntu`, `ec2-user`, `azureuser`). Default to `ubuntu` if not specified.
- **Remote deploy path** — the directory on the VM where the pack should be placed (e.g. `/home/ubuntu`). Default to the SSH user's home directory.
- **Custom hostname** — ask the user explicitly before proceeding. If they don't provide one, derive it from the project folder name (e.g. `hilton-poc` → `apim.hilton.com`). Fall back to `apim.wso2.local` if nothing can be inferred. Always confirm the chosen hostname with the user before applying.
- **Admin username and password** (if changing from defaults)

## Step 3 — Update hostname

Change the server hostname in `[server]`:

```toml
[server]
hostname = "<custom-hostname>"
```

## Step 4 — Update gateway environment endpoints

Update all `localhost` references in `[[apim.gateway.environment]]`:

```toml
[[apim.gateway.environment]]
name = "Default"
type = "hybrid"
gateway_type = "Regular"
provider = "wso2"
display_in_api_console = true
description = "This is a hybrid gateway that handles both production and sandbox token traffic."
show_as_token_endpoint_url = true
service_url = "https://<custom-hostname>:${mgt.transport.https.port}/services/"
username= "${admin.username}"
password= "${admin.password}"
ws_endpoint = "ws://<custom-hostname>:9099"
wss_endpoint = "wss://<custom-hostname>:8099"
http_endpoint = "http://<custom-hostname>:${http.nio.port}"
https_endpoint = "https://<custom-hostname>:${https.nio.port}"
websub_event_receiver_http_endpoint = "http://<custom-hostname>:9021"
websub_event_receiver_https_endpoint = "https://<custom-hostname>:8021"
```

## Step 5 — Update event listener notification endpoint

Update the `notification_endpoint` under `[event_listener.properties]`:

```toml
notification_endpoint = "https://<custom-hostname>:${mgt.transport.https.port}/internal/data/v1/notify"
```

## Step 6 — Update Developer Portal URL

Uncomment and update the `[apim.devportal]` section:

```toml
[apim.devportal]
url = "https://<custom-hostname>:${mgt.transport.https.port}/devportal"
```

## Step 7 — Enable analytics and set Moesif key (if provided)

If the user provides a Moesif key, enable analytics and add the properties block immediately after `[apim.analytics]`:

```toml
[apim.analytics]
enable = true
type = "moesif"

[apim.analytics.properties]
moesifKey = "<moesif-key>"
```

If analytics is already present in the file (e.g. `enable = false`), update it in place and append the `[apim.analytics.properties]` block directly below it.

## Step 8 — Update admin credentials (if requested)

Change the `[super_admin]` block:

```toml
[super_admin]
username = "<new-username>"
password = "<new-password>"
create_admin_account = true
```

## Step 9 — Add host mapping to the local machine

Check if an entry for `<custom-hostname>` already exists:

```bash
grep "<custom-hostname>" /etc/hosts
```

If the entry is **not present**, add it using `sudo sh -c` (this form allows sudo to prompt for the password via the terminal):

```bash
sudo sh -c 'echo "<vm-ip>   <custom-hostname>" >> /etc/hosts'
```

Confirm by grepping `/etc/hosts` again and showing the line to the user:

```bash
grep "<custom-hostname>" /etc/hosts
```

For **other client machines** (CI/CD agents, teammates), they must add the same line:
```
<vm-ip>   <custom-hostname>
```

## Step 10 — Provide portal URLs

After configuration, share the portal access URLs:

| Portal            | URL                                               |
|-------------------|---------------------------------------------------|
| Publisher         | `https://<custom-hostname>:9443/publisher`        |
| Developer Portal  | `https://<custom-hostname>:9443/devportal`        |
| Admin             | `https://<custom-hostname>:9443/admin`            |
| Carbon Console    | `https://<custom-hostname>:9443/carbon`           |
| Gateway HTTP      | `http://<custom-hostname>:8280`                   |
| Gateway HTTPS     | `https://<custom-hostname>:8243`                  |

## Step 11 — Update session timeout in settings.json

The portal `settings.json` files contain a `sessionTimeout` block that controls idle warning and logout timing. The default values (160/180 seconds) are too short for most deployments.

**Ask the user first:**

> The default session timeout values will be set to:
> - `idleWarningTimeout`: **580 seconds** (~10 min warning)
> - `idleTimeout`: **600 seconds** (~10 min logout)
>
> Would you like to use these values, or enter custom values?

Wait for the user's response before proceeding. Use their values if provided, otherwise use 580/600.

**Files to update** (update all that exist):

```
repository/deployment/server/webapps/publisher/site/public/conf/settings.json
repository/deployment/server/webapps/admin/site/public/conf/settings.json
repository/deployment/server/webapps/devportal/site/public/theme/settings.json
```

**Find and update** the `sessionTimeout` block in each file:

```json
"sessionTimeout": {
    "enable": true,
    "idleWarningTimeout": <idleWarningTimeout>,
    "idleTimeout": <idleTimeout>
}
```

Use the `Edit` tool to replace the existing values. Read each file first to confirm the current values before editing.

## Step 12 — Verify SSH connectivity

Before transferring files, verify the SSH connection is working:

```bash
ssh -i <private-key-path> -o ConnectTimeout=10 <ssh-user>@<vm-ip> "echo connected"
```

If this fails, stop and report the error to the user. Do not proceed.

## Step 13 — Zip the updated pack

Determine the pack root directory (the folder containing `repository/`, `bin/`, etc.). Then zip it, excluding any previously generated ZIPs and unnecessary files:

```bash
cd <parent-of-pack-dir>
zip -r <pack-name>.zip <pack-dir-name> \
  --exclude "*.zip" \
  --exclude "*/__pycache__/*" \
  --exclude "*/.git/*"
```

Report the zip file size to the user before uploading.

## Step 14 — SCP the zip to the VM

Transfer the zip to the remote deploy path:

```bash
scp -i <private-key-path> <pack-name>.zip <ssh-user>@<vm-ip>:<remote-deploy-path>/
```

Show progress. If the transfer fails, report the error and stop.

## Step 15 — SSH into the VM and extract the pack

SSH in and run the extraction sequence in a single command block:

```bash
ssh -i <private-key-path> <ssh-user>@<vm-ip> bash -s << 'EOF'
  set -e
  cd <remote-deploy-path>
  # Remove any previous extraction to avoid stale files
  rm -rf <pack-dir-name>
  unzip -q <pack-name>.zip
  echo "Extraction complete."
  ls -lh <pack-dir-name>/bin/
EOF
```

## Step 16 — Add hostname entry on the VM

Still within the SSH session (or a new one), add the hostname mapping to the VM's `/etc/hosts`:

```bash
ssh -i <private-key-path> <ssh-user>@<vm-ip> \
  "grep -q '<custom-hostname>' /etc/hosts || echo '127.0.0.1   <custom-hostname>' | sudo tee -a /etc/hosts"
```

## Step 17 — Start the server

Start WSO2 API Manager in the background and tail the log to confirm startup:

```bash
ssh -i <private-key-path> <ssh-user>@<vm-ip> bash -s << 'EOF'
  set -e
  cd <remote-deploy-path>/<pack-dir-name>/bin
  # Kill any existing APIM process
  pkill -f "wso2server" || true
  # Start in background
  nohup ./api-manager.sh start > /tmp/apim-start.log 2>&1
  echo "Server started. Tailing startup log (30s)..."
  timeout 30 tail -f <remote-deploy-path>/<pack-dir-name>/repository/logs/wso2carbon.log || true
EOF
```

After the tail completes, report the portal URLs from Step 10 to the user.

## Notes

- The default self-signed TLS certificate (`wso2carbon.jks`) is issued for `localhost`/`wso2carbon`. Browsers will show a warning for the custom hostname — either import the cert into the browser trust store or generate a new cert with a SAN for the custom hostname.
- Default management port is `9443`, gateway HTTP is `8280`, HTTPS is `8243`. These can be changed via `[transport]` config if needed.
- If the VM has a firewall, ensure ports `9443`, `8280`, `8243`, `9099`, `8099` are open.
