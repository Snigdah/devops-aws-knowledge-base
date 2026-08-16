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
   - [Step 10. Enable KV v2 secrets engine](#step-10-enable-kv-v2-secrets-engine)
   - [Step 11. Enable audit logging](#step-11-enable-audit-logging-optional-recommended)
   - [Step 12. End-to-end test](#step-12-end-to-end-test)
   - [Step 13. Browser access](#step-13-browser-access)
5. [Everyday Operations](#everyday-operations)
6. [Troubleshooting](#troubleshooting)
7. [Security Notes](#security-notes)
8. [Next Steps](#next-steps)

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

### Step 9. Log in as root

```bash
docker exec -it vault vault login
```

Paste the root token from Step 7.

### Step 10. Enable KV v2 secrets engine

This is the "box" where application secrets will be stored
(DB credentials, client IDs, client secrets, etc.).

```bash
docker exec -it vault vault secrets enable -path=secret kv-v2
```

### Step 11. Enable audit logging (optional, recommended)

Records every read/write on Vault. Standard in production for compliance.

```bash
docker exec -it vault vault audit enable file file_path=/vault/logs/audit.log
```

Verify:

```bash
docker exec vault vault audit list
sudo ls -l ~/docker-file/vault/logs/audit.log
```

### Step 12. End-to-end test

```bash
docker exec vault vault kv put secret/test message="tls vault ready"
docker exec vault vault kv get secret/test
sudo tail ~/docker-file/vault/logs/audit.log
```

You should see the written value and a corresponding audit log entry.

### Step 13. Browser access

Open:

```
https://192.168.10.56:8200
```

You will see a **certificate warning** (browser doesn't trust the self-signed
CA). Two options:

1. Click "Advanced" → "Proceed" — one-time bypass on this machine.
2. Import `~/docker-file/vault/certs/vault.crt` into your OS trust store to
   remove the warning permanently.

Log in with the **root token**.

---

## Everyday Operations

| Situation | Command |
|---|---|
| Check Vault status | `docker exec vault vault status` |
| Vault sealed after server restart | `docker exec -it vault vault operator unseal` × 3 (different keys) |
| View audit log | `sudo tail -f ~/docker-file/vault/logs/audit.log` |
| Stop Vault | `docker compose down` |
| Start Vault | `docker compose up -d` (then unseal) |
| Reload TLS certs after renewal | `docker exec vault kill -HUP 1` |
| Snapshot backup (raft) | `docker exec vault vault operator raft snapshot save /vault/data/backup-$(date +%F).snap` |
| List secrets engines | `docker exec vault vault secrets list` |
| Write a secret | `docker exec vault vault kv put secret/<path> key=value` |
| Read a secret | `docker exec vault vault kv get secret/<path>` |

---

## Troubleshooting

### The `vault` prefix gotcha (`bind: address already in use`)

**Symptom:** `docker logs vault` shows a restart loop with
`Error initializing listener of type tcp: listen tcp4 0.0.0.0:8200: bind: address already in use`
even though nothing on the host is using port 8200.

**Cause:** The `hashicorp/vault` image's entrypoint has a helper rule: if the
command starts with `server`, it auto-inserts `-config=/vault/config` before
your args. That causes Vault to load configuration **twice** (once from the
folder, once from the specific file), which produces two listener blocks
both trying to bind port 8200.

**Fix:** In `docker-compose.yml`, prefix the command with `vault`:

```yaml
command: vault server -config=/vault/config/vault.hcl   # correct
# NOT:  server -config=/vault/config/vault.hcl          # wrong: triggers the helper
```

### Port already in use on the host

**Symptom:** `docker compose up` fails with
`failed to bind host port 0.0.0.0:8200/tcp: address already in use`.

**Cause:** Another process (often an old native `vault` systemd service) is
already using port 8200.

**Diagnosis:**

```bash
docker ps --filter "publish=8200"
sudo ss -tlnp | grep ':8200'
sudo systemctl status vault
```

**Fix:** Stop and disable the conflicting service, or map Vault to a different
host port (`"8250:8200"` in the `ports:` section).

### CLI reports `http: server gave HTTP response to HTTPS client`

**Cause:** `VAULT_ADDR` is `https://…` but the listener is currently plain HTTP
(or vice-versa).

**Fix:** Match them. This README uses HTTPS everywhere, both in `vault.hcl` and
in the compose file's `VAULT_ADDR`.

### `docker exec … openssl` says `executable file not found`

Expected — the Vault image is minimal and does not include openssl. Run
openssl on the **host** instead:

```bash
openssl x509 -in ~/docker-file/vault/certs/vault.crt -noout -subject -issuer -ext subjectAltName
```

---

## Security Notes

* **Unseal keys and root token are irreplaceable.** Store them offline in a
  password manager or hardware token. Do not commit them to git.
* **`disable_mlock = true`** is set because mlock is unreliable in containers.
  On the host, disable swap entirely (`sudo swapoff -a` + remove swap from
  `/etc/fstab`) so secrets held in memory can never be written to swap.
* **TLS 1.3 minimum** is enforced by `tls_min_version = "tls13"`.
* **Self-signed cert with IP SAN** is standard for internal services. For
  external-facing or multi-tenant deployments, use a proper internal CA (e.g.
  `step-ca`, or your organisation's PKI) so client trust can be managed
  centrally.
* **Raft snapshots** should be taken on a schedule and copied off-host.
* **Audit logs** must be shipped to a central log store (Elasticsearch,
  Loki, etc.) in production — never rely on the local file alone.

---

## Next Steps

1. **Wire the first microservice (FinBook) to Vault** using Spring Cloud
   Vault, so DB credentials and client secrets come from Vault instead of
   `application.yml`.
2. **Apply the same pattern to the remaining microservices** — one folder
   per service under `secret/<service-name>`.
3. **Scale to HA:** add `vault-2` and `vault-3` nodes, join them to the
   Raft cluster. No changes needed to `vault.hcl` beyond a unique `node_id`.
4. **Auto-unseal:** integrate with a cloud KMS or a Transit key so restarts
   do not require a human. This removes the manual unseal step in `docker-compose`
   restart scenarios.
5. **Cert rotation runbook:** document the yearly cert rotation and the
   `docker exec vault kill -HUP 1` reload step.
