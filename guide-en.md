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
> **Image Tag:** 20260407  
> **Last Updated:** May 2026

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
docker pull ghcr.io/espressosystems/espresso-sequencer/sequencer:20260407
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
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260407 \
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

> 🔐 **CRITICAL: Back up the following immediately to a secure location (password manager, external drive):**
>
> | Key | Prefix | Description |
> | --- | ------ | ----------- |
> | `ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY` | `BLS_SIGNING_KEY~...` | Signs consensus messages |
> | `ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY` | `SCHNORR_SIGNING_KEY~...` | Signs finalized states |
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
EOF
```

Replace:
- `YOUR_API_KEY` → your Infura API key (in both HTTP and WS lines)
- `YOUR_SERVER_IP` → your server's public IPv4 address

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
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260407 \
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

When a new image version is released:

1. Pull the new image:

```bash
docker pull ghcr.io/espressosystems/espresso-sequencer/sequencer:NEW_TAG
```

2. Update the tag in the systemd service file:

```bash
nano /etc/systemd/system/espresso.service
```

Replace `20260407` with the new tag.

3. Reload and restart:

```bash
systemctl daemon-reload && systemctl restart espresso
```

4. Verify the node is running:

```bash
systemctl status espresso && curl -s http://localhost:8585/healthcheck
```

> 💡 Stay updated via the [Espresso Network GitHub Releases](https://github.com/EspressoSystems/espresso-network/releases) page.

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.  
🌐 [hazennetworksolutions.com](https://www.hazennetworksolutions.com)
