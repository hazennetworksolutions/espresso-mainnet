# 🟤 Espresso Network Mainnet 1 — Validator Node Setup Guide (English)

**A complete guide to running an Espresso Network validator node and applying to the Validator Bootstrap Program**  
_Docker installation, key generation, validator registration, systemd service setup — step by step._

![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat&logo=ubuntu&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-29.x-2496ED?style=flat&logo=docker&logoColor=white)
![Espresso](https://img.shields.io/badge/Espresso-Mainnet%201-6B4226?style=flat)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

[hazennetworksolutions.com](https://www.hazennetworksolutions.com)

---

> **Author:** HazenNetworkSolutions  
> **Network:** Espresso Mainnet 1  
> **Image Tag:** 20260710  
> **Last Updated:** 14 July 2026

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Step 1 — Install Docker](#step-1--install-docker)
- [Step 2 — Create Directory Structure](#step-2--create-directory-structure)
- [Step 3 — Pull Docker Images](#step-3--pull-docker-images)
- [Step 4 — Generate Validator Keys](#step-4--generate-validator-keys)
- [Step 5 — Create Metadata JSON](#step-5--create-metadata-json)
- [Step 6 — Prepare Ethereum Wallet](#step-6--prepare-ethereum-wallet)
- [Step 7 — Get Ethereum RPC Endpoint](#step-7--get-ethereum-rpc-endpoint)
- [Step 8 — Register Validator on Ethereum](#step-8--register-validator-on-ethereum)
- [Step 9 — Create Environment File](#step-9--create-environment-file)
- [Step 10 — Create Systemd Service](#step-10--create-systemd-service)
- [Step 11 — Start the Node](#step-11--start-the-node)
- [Step 12 — Verify Node Health](#step-12--verify-node-health)
- [Step 13 — Apply to Bootstrap Program](#step-13--apply-to-bootstrap-program)
- [Monitoring the Node](#monitoring-the-node)
- [Updating the Node](#updating-the-node)
- [Troubleshooting](#troubleshooting)

---

## Hardware Requirements

| Component | Minimum |
| --------- | ------- |
| Operating System | Ubuntu 22.04+ |
| CPU | 1 Core (any modern CPU works) |
| RAM | 8 GB |
| Storage | Negligible (kilobytes) |
| Network | Stable internet, public IP required |
| Open Ports | 8585/TCP (API), 9000/UDP (LibP2P) |

> ✅ This is a **Regular Node** (non-DA). DA nodes are only required for the DA committee — most operators do not need them.

---

## Step 1 — Install Docker

Install Docker from the official repository (do not use `snap` or `apt install docker.io`):

```bash
apt-get update && apt-get install -y ca-certificates curl gnupg && \
install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
chmod a+r /etc/apt/keyrings/docker.gpg && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
apt-get update && \
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify installation:

```bash
docker --version && systemctl status docker
```

Expected output: `Docker version 29.x.x` and `active (running)`.

---

## Step 2 — Create Directory Structure

```bash
mkdir -p /opt/espresso/keys /opt/espresso/store
```

| Directory | Purpose |
| --------- | ------- |
| `/opt/espresso/keys` | Stores BLS and Schnorr private keys |
| `/opt/espresso/store` | Stores node SQLite database |

---

## Step 3 — Pull Docker Images

Pull the Espresso sequencer image (Mainnet 1):

```bash
docker pull ghcr.io/espressosystems/espresso-sequencer/sequencer:20260710
```

Pull the staking CLI image:

```bash
docker pull ghcr.io/espressosystems/espresso-sequencer/staking-cli:main
```

> 💡 The sequencer image is ~500MB and may take a few minutes to download.
>
> ⚠️ **Note:** The official Espresso docs reference `espresso-network/sequencer` as the image path, but the correct registry path is `espresso-sequencer/sequencer` as used above. Use the commands in this guide.

---

## Step 4 — Generate Validator Keys

Run the key generator to create your BLS (staking) and Schnorr (state) keys:

```bash
docker run -v /opt/espresso/keys:/keys \
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260710 \
  keygen -o /keys
```

The command produces no terminal output — this is normal. The keys are saved silently to the file. Verify the file was created:

```bash
cat /opt/espresso/keys/0.env
```

Expected output:

```
# Seed: <random_hex>
ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY=BLS_VER_KEY~...
ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY=BLS_SIGNING_KEY~...
ESPRESSO_SEQUENCER_PUBLIC_STATE_KEY=SCHNORR_VER_KEY~...
ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY=SCHNORR_SIGNING_KEY~...
```

Next, generate the **x25519 key** (required for the upcoming Cliquenet protocol upgrade) and append it to the same file:

```bash
docker run --rm \
  -v /opt/espresso/keys:/keys \
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260710 \
  keygen --scheme x25519 -o /keys
```

This command overwrites `0.env` with only the x25519 key. You must manually reconstruct the full key file by combining all keys:

```bash
cat > /opt/espresso/keys/0.env << 'EOF'
# Seed: YOUR_ORIGINAL_SEED_HERE
ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY=BLS_VER_KEY~YOUR_BLS_PUBLIC_KEY
ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY=BLS_SIGNING_KEY~YOUR_BLS_PRIVATE_KEY
ESPRESSO_SEQUENCER_PUBLIC_STATE_KEY=SCHNORR_VER_KEY~YOUR_SCHNORR_PUBLIC_KEY
ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY=SCHNORR_SIGNING_KEY~YOUR_SCHNORR_PRIVATE_KEY
ESPRESSO_SEQUENCER_PUBLIC_X25519_KEY=YOUR_X25519_PUBLIC_KEY
ESPRESSO_SEQUENCER_PRIVATE_X25519_KEY=X25519_SK~YOUR_X25519_PRIVATE_KEY
EOF
```

Verify all 3 public keys are present:

```bash
grep "PUBLIC" /opt/espresso/keys/0.env
```

You should see 3 lines: `PUBLIC_STAKING_KEY`, `PUBLIC_STATE_KEY`, `PUBLIC_X25519_KEY`.

> 🔐 **CRITICAL: Back up the following immediately to a secure location (password manager, external drive):**
>
> | Key | Prefix | Description |
> | --- | ------ | ----------- |
> | `ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY` | `BLS_SIGNING_KEY~...` | Signs consensus messages |
> | `ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY` | `SCHNORR_SIGNING_KEY~...` | Signs finalized states |
> | `ESPRESSO_SEQUENCER_PRIVATE_X25519_KEY` | `X25519_SK~...` | Required for Cliquenet protocol upgrade |
> | `Seed` | hex string | Used to regenerate keys |
>
> Download to your local machine:
> ```bash
> scp root@SERVER_IP:/opt/espresso/keys/0.env ./espresso-keys-backup.env
> ```
> 
> ⚠️ **Never share your private keys or seed with anyone.**  
> ⚠️ Each BLS key can only be registered once on the Ethereum mainnet.

---

## Step 5 — Create Metadata JSON

Your validator metadata is hosted publicly and displayed in the Espresso staking dashboard. Nginx must be installed and running on your server.

If nginx is not installed, install it first:

```bash
apt-get install -y nginx && systemctl enable nginx && systemctl start nginx
```

Get your server's public IPv4 address:

```bash
curl -s -4 ifconfig.me
```

Create the metadata file (replace the values with your own information):

```bash
mkdir -p /var/www/html/espresso && cat > /var/www/html/espresso/metadata.json << 'EOF'
{
  "pub_key": "BLS_VER_KEY~YOUR_PUBLIC_STAKING_KEY_HERE",
  "name": "YourValidatorName",
  "description": "Your description here",
  "company_name": "Your Company Name",
  "company_website": "https://yourwebsite.com"
}
EOF
```

Replace `BLS_VER_KEY~YOUR_PUBLIC_STAKING_KEY_HERE` with your `ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY` value from Step 4.

Test that it is accessible:

```bash
curl -s http://YOUR_SERVER_IP/espresso/metadata.json
```

Expected output: the JSON content you just created.

> 💡 The `pub_key` field must match your registered BLS key — this prevents impersonation on the dashboard.

### Adding a Validator Logo (Optional)

You can display your logo on the Espresso staking dashboard by adding an `icon` field to your metadata.

**1. Prepare your logo image** (JPG or PNG, minimum 400x400px recommended) and copy it to the server:

```bash
scp /path/to/logo.jpg root@YOUR_SERVER_IP:/var/www/html/espresso/logo.jpg
```

**2. Install ImageMagick and generate icon sizes:**

```bash
apt-get install -y imagemagick && \
cd /var/www/html/espresso/ && \
convert logo.jpg -resize 14x14 icon-14@1x.png && \
convert logo.jpg -resize 28x28 icon-14@2x.png && \
convert logo.jpg -resize 42x42 icon-14@3x.png && \
convert logo.jpg -resize 24x24 icon-24@1x.png && \
convert logo.jpg -resize 48x48 icon-24@2x.png && \
convert logo.jpg -resize 72x72 icon-24@3x.png && \
chown www-data:www-data icon-*.png && \
chmod 644 icon-*.png
```

**3. Update metadata.json with the icon URLs:**

```bash
cat > /var/www/html/espresso/metadata.json << 'EOF'
{
  "pub_key": "BLS_VER_KEY~YOUR_PUBLIC_STAKING_KEY_HERE",
  "name": "YourValidatorName",
  "description": "Your description here",
  "company_name": "Your Company Name",
  "company_website": "https://yourwebsite.com",
  "client_version": "20260710",
  "icon": {
    "14x14": {
      "@1x": "https://YOUR_DOMAIN/espresso/icon-14@1x.png",
      "@2x": "https://YOUR_DOMAIN/espresso/icon-14@2x.png",
      "@3x": "https://YOUR_DOMAIN/espresso/icon-14@3x.png"
    },
    "24x24": {
      "@1x": "https://YOUR_DOMAIN/espresso/icon-24@1x.png",
      "@2x": "https://YOUR_DOMAIN/espresso/icon-24@2x.png",
      "@3x": "https://YOUR_DOMAIN/espresso/icon-24@3x.png"
    }
  }
}
EOF
```

> ⚠️ **Icon URLs must use HTTPS** — plain `http://` URLs are not rendered by the staking dashboard.  
> 💡 No node restart required — metadata changes take effect immediately.

**4. Preview and validate metadata before publishing:**

```bash
docker run --rm \
  ghcr.io/espressosystems/espresso-sequencer/staking-cli:main \
  staking-cli preview-metadata --metadata-uri http://YOUR_SERVER_IP/espresso/metadata.json
```

---

## Step 6 — Prepare Ethereum Wallet

You need a **dedicated Ethereum wallet** for this validator. Do not reuse existing wallets.

1. Create a new wallet in MetaMask (or any Ethereum wallet)
2. Save the **12-word seed phrase** securely
3. Note the **Ethereum address** (starts with `0x`)
4. Send **0.01–0.02 ETH** (Ethereum Mainnet) to this address for gas fees

> ⚠️ Registration costs approximately 300,000 gas (~$1–2 at current prices).  
> ⚠️ Each Ethereum address can only register **one** validator.  
> ⚠️ Do NOT send ETH on any other chain (not BSC, not Polygon — Ethereum Mainnet only).

---

## Step 7 — Get Ethereum RPC Endpoint

You need an Ethereum Mainnet RPC endpoint for the validator to read from L1.

**Recommended: Infura (free tier — 3M credits/day, more than sufficient)**

1. Go to [infura.io](https://infura.io) and create a free account
2. Create a new project → select **Ethereum**
3. Copy your API key

Your endpoints will be:
- **HTTP:** `https://mainnet.infura.io/v3/YOUR_API_KEY`
- **WebSocket:** `wss://mainnet.infura.io/ws/v3/YOUR_API_KEY`

> 💡 A regular Espresso node uses approximately 15,000–20,000 API requests per day — well within the free tier limit.

> 🔄 **Alternative providers:** If you notice a high missed-vote rate on the staking dashboard, it can be caused by L1 provider latency or intermittent 401/timeout errors. In that case, switching to [ValidationCloud](https://validationcloud.io) or [Tenderly](https://tenderly.co) (both offer generous free tiers) can help. Just replace the `L1_PROVIDER` / `L1_WS_PROVIDER` values in Step 9 with the new endpoints and restart the service — no re-registration needed.

---

## Step 8 — Register Validator on Ethereum

This step registers your validator in the Espresso stake table contract on Ethereum mainnet. You only need to do this **once**.

Run the registration command (replace all placeholder values):

```bash
docker run --rm \
  -e MNEMONIC="your twelve word seed phrase here" \
  -e ACCOUNT_INDEX=0 \
  -e L1_PROVIDER="https://mainnet.infura.io/v3/YOUR_API_KEY" \
  -e STAKE_TABLE_ADDRESS="0xCeF474D372B5b09dEfe2aF187bf17338Dc704451" \
  -e CONSENSUS_PRIVATE_KEY="BLS_SIGNING_KEY~YOUR_PRIVATE_STAKING_KEY" \
  -e STATE_PRIVATE_KEY="SCHNORR_SIGNING_KEY~YOUR_PRIVATE_STATE_KEY" \
  ghcr.io/espressosystems/espresso-sequencer/staking-cli:main \
  staking-cli register-validator \
  --commission 10.00 \
  --metadata-uri "http://YOUR_SERVER_IP/espresso/metadata.json"
```

| Parameter | Description |
| --------- | ----------- |
| `MNEMONIC` | Your 12-word seed phrase (in quotes, space-separated) |
| `ACCOUNT_INDEX` | Derivation index, use `0` for the first account |
| `L1_PROVIDER` | Your Infura HTTP RPC URL |
| `CONSENSUS_PRIVATE_KEY` | Your `BLS_SIGNING_KEY~...` from Step 4 |
| `STATE_PRIVATE_KEY` | Your `SCHNORR_SIGNING_KEY~...` from Step 4 |
| `--commission` | Your commission rate (0.00–100.00) |
| `--metadata-uri` | URL to your metadata.json from Step 5 |

Successful output:

```
Success! transaction hash: 0x...
event: ValidatorRegisteredV2 { account: 0xYOUR_ADDRESS, blsVK: BLS_VER_KEY~..., ... }
```

**After running, clear your terminal history immediately:**

```bash
history -c && history -w
```

> ⚠️ Note your **Ethereum account address** from the output — you will need it for the Bootstrap Program form.

---

## Step 9 — Create Environment File

Create the environment configuration file for the node:

```bash
cat > /opt/espresso/espresso.env << 'EOF'
ESPRESSO_SEQUENCER_CDN_ENDPOINT=cdn.main.net.espresso.network:1737
ESPRESSO_STATE_RELAY_SERVER_URL=https://state-relay.main.net.espresso.network
ESPRESSO_SEQUENCER_GENESIS_FILE=/genesis/mainnet.toml
ESPRESSO_SEQUENCER_EMBEDDED_DB=true
RUST_LOG=warn,libp2p=off
RUST_LOG_FORMAT=json
ESPRESSO_SEQUENCER_STATE_PEERS=https://query.main.net.espresso.network
ESPRESSO_SEQUENCER_CONFIG_PEERS=https://cache.main.net.espresso.network
ESPRESSO_SEQUENCER_L1_PROVIDER=https://mainnet.infura.io/v3/YOUR_API_KEY
ESPRESSO_SEQUENCER_L1_WS_PROVIDER=wss://mainnet.infura.io/ws/v3/YOUR_API_KEY
ESPRESSO_SEQUENCER_API_PORT=8585
ESPRESSO_SEQUENCER_STORAGE_PATH=/store
ESPRESSO_SEQUENCER_KEY_FILE=/keys/0.env
ESPRESSO_SEQUENCER_LIBP2P_BIND_ADDRESS=0.0.0.0:9000
ESPRESSO_SEQUENCER_LIBP2P_ADVERTISE_ADDRESS=YOUR_SERVER_IP:9000
ESPRESSO_NODE_STORAGE_PATH=/store
ESPRESSO_NODE_TELEMETRY_LOGS_ENABLE=true
ESPRESSO_NODE_TELEMETRY_METRICS_ENABLE=true
ESPRESSO_NODE_IDENTITY_COMPANY_NAME=Your Company Name
ESPRESSO_NODE_IDENTITY_NODE_NAME=YourValidatorName
EOF
```

Replace:
- `YOUR_API_KEY` → your Infura API key (in both HTTP and WS lines)
- `YOUR_SERVER_IP` → your server's public IPv4 address
- `ESPRESSO_NODE_IDENTITY_COMPANY_NAME` / `ESPRESSO_NODE_IDENTITY_NODE_NAME` → your operator identity (same values as `company_name` / `name` in `metadata.json` from Step 5)

> 💡 **Since the `20260710` release:** if `ESPRESSO_SEQUENCER_STORAGE_PATH` is set, `ESPRESSO_NODE_STORAGE_PATH` must be set to the same value. Telemetry vars let the Foundation collect your logs/metrics for network health monitoring — enabling them is recommended but optional. Identity vars are used to label your node in Foundation-side dashboards.

Verify the file:

```bash
cat /opt/espresso/espresso.env
```

---

## Step 10 — Create Systemd Service

Create the systemd service file that manages the Docker container:

```bash
cat > /etc/systemd/system/espresso.service << 'EOF'
[Unit]
Description=Espresso Sequencer Node
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
RestartSec=10
ExecStartPre=-/usr/bin/docker stop espresso
ExecStartPre=-/usr/bin/docker rm espresso
ExecStart=/usr/bin/docker run --name espresso \
  --env-file /opt/espresso/espresso.env \
  -v /opt/espresso/keys:/keys \
  -v /opt/espresso/store:/store \
  -p 8585:8585 \
  -p 9000:9000/udp \
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260710 \
  sequencer -- http -- catchup -- status
ExecStop=/usr/bin/docker stop espresso

[Install]
WantedBy=multi-user.target
EOF
```

---

## Step 11 — Start the Node

Enable and start the service:

```bash
systemctl daemon-reload && \
systemctl enable espresso && \
systemctl start espresso
```

Check the service status:

```bash
systemctl status espresso
```

Expected output: `active (running)`.

View live logs:

```bash
journalctl -u espresso -n 50 --no-pager
```

> 💡 On first startup, the node loads network config from peers. This may take 10–30 seconds. You will see:  
> `loaded config` and view numbers incrementing — this is normal and expected.

---

## Step 12 — Verify Node Health

**Healthcheck (all modules should return 200):**

```bash
curl -s http://localhost:8585/healthcheck
```

Expected output:

```json
{"status":"available","modules":{"catchup":{"0":200,"1":200},"state-signature":{"0":200,"1":200},"status":{"0":200,"1":200}}}
```

**Verify your validator appears on the dashboard:**

1. Go to [stake.espresso.network](https://stake.espresso.network)
2. Search for your Ethereum address: `0xYOUR_ADDRESS`
3. Your validator should appear with status `INACTIVE` (this is correct — it becomes `ACTIVE` after receiving delegation)

> ✅ **What is normal:**
> - `WARN` level logs — expected behavior
> - `Vote sending timed out` — node is trying to participate but has no delegation yet
> - `LCV3 signature posted by nodes not on the stake table` — expected until delegation arrives
> - View numbers advancing every ~1 second — this confirms the node is following the chain
>
> ❌ **What is NOT normal:**
> - Service showing `failed` or `inactive (dead)`
> - Healthcheck not returning 200
> - View numbers stuck for more than 5 minutes

---

## Step 13 — Apply to Bootstrap Program

The Espresso Foundation Validator Bootstrap Program offers up to 30 selected validators an initial delegation of **1,000,000 ESP**.

Apply here: [Bootstrap Program Application Form](https://docs.google.com/forms/d/e/1FAIpQLScC5jk-jGE4d4hFRSuQNoP9MrmRB6oy3uRDagS-K5xc-OH64A/viewform)

| Field | Value |
| ----- | ----- |
| Operator name | Your operator/company name |
| Operator website | Your website URL |
| Primary contact name | Your name |
| Primary contact email | Your email (optional) |
| Telegram / Discord handle | Your handle |
| **Ethereum account** | The address used in Step 8 (e.g. `0xCaA51...`) |
| Region | Your server's region (e.g. EU - Finland) |
| Validator experience | Describe your infrastructure experience |

> 💡 The Foundation evaluates operators on uptime, missed slots, responsiveness, and software update speed. Every 6 months, top-performing validators become eligible for the **main delegation program** (larger delegation).

---

## Backup & Recovery

### What Must Be Backed Up

| File / Info | Critical | Notes |
|---|---|---|
| `/opt/espresso/keys/0.env` | 🔴 YES | Contains all 3 private keys — **cannot be recovered if lost** |
| Ethereum wallet mnemonic | 🔴 YES | Required for all L1 operations (rewards, commission, deregister) |
| `/opt/espresso/espresso.env` | 🟡 Recommended | Infura key + server IP — easily recreated but good to have |

### Backing Up Keys

Before running any `keygen` command, always create a backup first:

```bash
cp /opt/espresso/keys/0.env /opt/espresso/keys/0.env.backup
```

Download `0.env` to your local machine:

```bash
scp root@YOUR_SERVER_IP:/opt/espresso/keys/0.env ~/espresso-keys-backup.env
```

Save the contents in a password manager (Bitwarden, 1Password, etc.).

> ⚠️ **Warning:** The `keygen --scheme x25519` command **overwrites** `0.env` entirely. Always back up before running any keygen command.

### Recovery: Moving to a New Server

If you lose access to your server but have your `0.env` backup:

1. Set up a new server following this guide from Step 1
2. Restore your key file:
   ```bash
   scp ~/espresso-keys-backup.env root@NEW_SERVER_IP:/opt/espresso/keys/0.env
   ```
3. Update `ESPRESSO_SEQUENCER_LIBP2P_ADVERTISE_ADDRESS` in `espresso.env` with the new server IP
4. Update `metadata.json` with the new server IP (if using IP-based URL)
5. Start the node — **no re-registration on Ethereum needed**, your validator identity is preserved

---

## Monitoring the Node

**Check service status:**

```bash
systemctl status espresso
```

**View last 50 log lines:**

```bash
journalctl -u espresso -n 50 --no-pager
```

**Healthcheck:**

```bash
curl -s http://localhost:8585/healthcheck
```

**Check connected peers:**

```bash
curl -s http://localhost:8585/status/metrics | grep libp2p_num_connected_peers
```

**Restart the node:**

```bash
systemctl restart espresso
```

**Stop the node:**

```bash
systemctl stop espresso
```

---

## Updating the Node

When a new image version is released (announced in the Espresso Discord `#mainnet-node-operator` channel):

1. Pull the new image:

```bash
docker pull ghcr.io/espressosystems/espresso-sequencer/sequencer:NEW_TAG
```

2. Update the tag in the systemd service file (replace `OLD_TAG` with the currently running tag):

```bash
sed -i 's/sequencer:OLD_TAG/sequencer:NEW_TAG/g' /etc/systemd/system/espresso.service
```

3. If the release notes mention new/renamed environment variables, add them to `/opt/espresso/espresso.env` (old variables usually keep working, but it's best to migrate).

4. Reload and restart:

```bash
systemctl daemon-reload && systemctl restart espresso
```

5. Check startup logs for deprecated env var warnings:

```bash
sleep 15 && journalctl -u espresso --no-hostname -o cat -n 200 | grep -i "warning:"
```

6. Verify the node is running and on the correct version:

```bash
systemctl status espresso && \
curl -s http://localhost:8585/healthcheck && \
curl -s http://localhost:8585/v1/status/metrics | grep consensus_version
```

7. Update `client_version` in your public metadata file:

```bash
sed -i 's/OLD_TAG/NEW_TAG/g' /var/www/html/espresso/metadata.json
```

> 💡 Stay updated via the [Espresso Network GitHub Releases](https://github.com/EspressoSystems/espresso-network/releases) page and the Discord `#mainnet-node-operator` channel.
> 
> ⚠️ Some releases include a DB migration for **query nodes** (nodes running the `query` module). If your `ExecStart` command does not include `query` in the module list, this does not apply to you. Query node operators should check `/database/migration-status` after updating — migrations can take up to several weeks and rolling back the binary during a migration is not supported.

---

## Troubleshooting

**High missed-vote rate on the dashboard:**
- Check `consensus_number_of_timeouts` in `/v1/status/metrics` — if it's climbing fast, something is wrong.
- Check logs for `Event sender queue overflow` (ERROR level) — this indicates the node's internal event queue can't keep up, often caused by CPU/RAM contention from other services on the same server. Stop unrelated services and `systemctl restart espresso` to clear the queue.
- Check logs for `L1 client error` / `401` — your L1 RPC provider may be rate-limited or misconfigured. See the alternative providers note in Step 7.
- `AutoNAT: probe reports this node may not be publicly reachable` right after a restart is usually transient — recheck after a minute. If it persists, verify port `9000/UDP` is open and `ESPRESSO_SEQUENCER_LIBP2P_ADVERTISE_ADDRESS` matches your real public IP.

**Node not signing at all:**
- Verify `consensus_last_voted_view` is close to `consensus_current_view` (within 1–2) via `/v1/status/metrics`.
- Verify your BLS key in `0.env` matches the key used at registration — a `keygen` command run without care can silently overwrite `0.env` (see Backup & Recovery section).

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.  
🌐 [hazennetworksolutions.com](https://www.hazennetworksolutions.com)
