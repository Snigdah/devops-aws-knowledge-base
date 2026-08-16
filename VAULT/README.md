# HashiCorp Vault — Production Setup (TLS + Raft)

A single-node, TLS-enabled, production-grade Vault deployment on Docker Compose.
Uses **Raft integrated storage**, a **self-signed TLS certificate with IP SAN**,
and follows HashiCorp's recommended production patterns (sealed at startup,
initialised once, unsealed by an operator on every restart).

The pattern shown here scales cleanly to N Vault nodes for HA and is reused
across all microservices in the MicroCube platform.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Folder Structure](#folder-structure)
4. [Setup — Step by Step](#setup--step-by-step)
   - [Step 1. Create the project folders](#step-1-create-the-project-folders)
   - [Step 2. Generate the TLS certificate](#step-2-generate-the-tls-certificate)
   - [Step 3. Create `config/vault.hcl`](#step-3-create-configvaulthcl)
   - [Step 4. Create `docker-compose.yml`](#step-4-create-docker-composeyml)
   - [Step 5. Start the container](#step-5-start-the-container)
   - [Step 6. Verify the TLS handshake](#step-6-verify-the-tls-handshake)
   - [Step 7. Initialise Vault](#step-7-initialise-vault)
   - [Step 8. Unseal Vault](#step-8-unseal-vault)
   - [Step 9. Log in as root](#step-9-log-in-as-root)

---

## Overview

| Attribute | Value |
|---|---|
| Vault version | `hashicorp/vault:1.20.4` |
| Storage backend | Raft (integrated storage) |
| Transport | HTTPS (TLS 1.3 min), self-signed cert with IP SAN |
| Server IP | `192.168.10.56` |
| API port | `8200` |
| Cluster port | `8201` (used when scaling to HA) |
| Init | `key-shares=5 key-threshold=3` |
| UI | Enabled |
| Audit log | File-based, persisted on host |

---

## Prerequisites

* Linux server with Docker Engine + Docker Compose v2 installed.
* Port `8200` and `8201` free on the host.
* `openssl` on the host (used once to generate the self-signed cert).
* `curl` (used to verify TLS handshake).
* A password manager or secure vault to store the unseal keys and root token.

---

## Folder Structure

```
~/docker-file/vault/
├── docker-compose.yml
├── config/
│   └── vault.hcl
├── certs/
│   ├── vault.crt
│   └── vault.key
├── data/          # Raft encrypted storage (persistent)
└── logs/          # Audit log (persistent)
```

---

## Setup — Step by Step

### Step 1. Create the project folders

```bash
mkdir -p ~/docker-file/vault/{config,certs,data,logs}
cd ~/docker-file/vault
```

### Step 2. Generate the TLS certificate

Self-signed, 4096-bit RSA, SHA-256, valid 10 years, SANs include the server IP,
loopback, `localhost`, and `vault`.

```bash
cd ~/docker-file/vault/certs

sudo openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout vault.key \
  -out    vault.crt \
  -subj   "/C=BD/O=Leads/CN=vault-internal" \
  -addext "subjectAltName=IP:192.168.10.56,IP:127.0.0.1,DNS:localhost,DNS:vault"
```

Verify the SANs were baked in:

```bash
openssl x509 -in vault.crt -noout -subject -issuer -ext subjectAltName -dates
```

Expected output:

```
subject=C = BD, O = Leads, CN = vault-internal
issuer=C = BD, O = Leads, CN = vault-internal
X509v3 Subject Alternative Name:
    IP Address:192.168.10.56, IP Address:127.0.0.1, DNS:localhost, DNS:vault
notBefore=...
notAfter =...
```

Fix permissions so the Vault container (uid `100`) can read both files:

```bash
sudo chown -R root:root ~/docker-file/vault/certs
sudo chmod 644 ~/docker-file/vault/certs/vault.crt
sudo chmod 644 ~/docker-file/vault/certs/vault.key
ls -l ~/docker-file/vault/certs
```

> **Note:** `644` on the private key is acceptable on an internal server.
> In stricter environments, use `640` with a dedicated group, or move the key
> into a Docker secret / KMS.

### Step 3. Create `config/vault.hcl`

```bash
cat > ~/docker-file/vault/config/vault.hcl <<'EOF'
ui = true

storage "raft" {
  path    = "/vault/data"
  node_id = "vault-1"
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_cert_file   = "/vault/certs/vault.crt"
  tls_key_file    = "/vault/certs/vault.key"
  tls_min_version = "tls13"
}

api_addr      = "https://192.168.10.56:8200"
cluster_addr  = "https://192.168.10.56:8201"

disable_mlock = true
EOF
```

### Step 4. Create `docker-compose.yml`

```bash
cat > ~/docker-file/vault/docker-compose.yml <<'EOF'
services:
  vault:
    image: hashicorp/vault:1.20.4
    container_name: vault
    restart: unless-stopped

    ports:
      - "8200:8200"
      - "8201:8201"

    cap_add:
      - IPC_LOCK

    environment:
      VAULT_ADDR:   "https://127.0.0.1:8200"
      VAULT_CACERT: "/vault/certs/vault.crt"

    volumes:
      - ./config:/vault/config
      - ./data:/vault/data
      - ./certs:/vault/certs:ro
      - ./logs:/vault/logs

    command: vault server -config=/vault/config/vault.hcl
EOF
```

> **Important:** the command **must start with `vault`** (not just `server`).
> The image's entrypoint has a shortcut that auto-adds `-config=/vault/config`
> if the first argument is `server`, which makes Vault load its config twice
> and fail with `bind: address already in use`. See
> [Troubleshooting](#troubleshooting).

### Step 5. Start the container

```bash
cd ~/docker-file/vault

sudo chown -R 100:1000 data logs        # container runs as uid 100

docker compose up -d
sleep 5
docker compose ps
docker logs vault | head -30
```

Look for these lines in the log:

```
Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", tls: "enabled")
   Storage: raft
==> Vault server started! Log data will stream in below:
```

The **`tls: "enabled"`** line confirms TLS is active.

### Step 6. Verify the TLS handshake

**With cert verification** — must succeed cleanly with no `-tls-skip-verify`:

```bash
curl --cacert ~/docker-file/vault/certs/vault.crt \
     https://192.168.10.56:8200/v1/sys/health
```

Expected JSON response:

```json
{"initialized":false,"sealed":true,"standby":true,"performance_standby":false,
 "replication_performance_mode":"unknown","replication_dr_mode":"unknown",
 "server_time_utc":..., "version":"1.20.4", ...}
```

Confirm current Vault status through the CLI (uses `VAULT_ADDR` + `VAULT_CACERT`
that we set in the compose file):

```bash
docker exec -it vault vault status
```

Expected key lines:

```
Initialized             false
Sealed                  true
Storage Type            raft
HA Enabled              true
```

### Step 7. Initialise Vault

**Runs exactly once, ever.** Generates the master unseal keys and the first
root token.

```bash
docker exec -it vault vault operator init -key-shares=5 -key-threshold=3
```

Sample output:

```
Unseal Key 1: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
Unseal Key 2: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
Unseal Key 3: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
Unseal Key 4: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
Unseal Key 5: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=

Initial Root Token: hvs.XXXXXXXXXXXXXXXXXXXXXXXX
```

> **CRITICAL:** Copy all 5 unseal keys and the root token into a password
> manager or a secure secret store **right now**. If lost, all secrets in
> this Vault are unrecoverable — there is no back door.
>
> The `5 / 3` scheme means: 5 keys exist, any 3 together can unseal Vault.
> In real production, the 5 keys are distributed to 5 different trusted
> operators, so no single person can unseal alone.

### Step 8. Unseal Vault

Run three times, pasting a **different** unseal key each time:

```bash
docker exec -it vault vault operator unseal
docker exec -it vault vault operator unseal
docker exec -it vault vault operator unseal
```

After the third key, output shows `Sealed  false`. Verify:

```bash
docker exec -it vault vault status
```

Expected:

```
Sealed          false
Storage Type    raft
HA Enabled      true
```
