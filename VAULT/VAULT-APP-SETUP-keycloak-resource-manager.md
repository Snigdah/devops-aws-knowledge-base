# Wiring a microservice to Vault — keycloak-resource-manager

Store one microservice's secrets in Vault and give the app a safe, narrow login
(AppRole) to read them. **Both UI and CLI** are shown for every step, with a note
on **which one to prefer** for that step.

---

## Mental model — Vault's building blocks

Think of Vault as a bank:

1. **Secrets engine** — the *type of storage box* Vault uses. We use **KV v2**
   (key/value, versioned). Must be enabled once before you can store anything.
2. **Secret** — the valuable thing in the box (DB password, client secret).
3. **Policy** — a *permission slip*: "holder may open box #7, read-only." Belongs
   to no one yet; just a written rule.
4. **Auth method (AppRole)** — gives the app its own *ID card + PIN*
   (`role_id` + `secret_id`) so it logs in by itself — no human, no root token.
5. **Role** — the glue: "any app that logs in with THIS ID card gets THAT policy."

### How the app reads a secret at runtime
```
App logs in with role_id + secret_id
   -> Vault checks the role -> attaches the policy -> returns a short-lived token
   -> App uses that token to read secret/keycloak-resource-manager
   -> App gets db.password etc. -> connects to Oracle
```

### Why not put the root token in the app?
The root token is the master key to everything and never expires. If it leaks,
the whole Vault is gone. AppRole gives a **narrow, expiring badge** that can only
read its own one box. Root token = human setup only.

---

## UI vs CLI — the rule of thumb

Both call the same Vault API underneath — identical result.

- **UI** — best for *learning*, *seeing what exists*, and *one-off* actions.
- **CLI** — best for *production*: scriptable, repeatable across 100+ services,
  and the standard your TL expects.

| Step | Action | UI? | CLI? | Prefer for this step |
|---|---|---|---|---|
| 0 | Enable KV v2 engine | yes | yes | **Either** (one-time; UI is clear) |
| 1 | Store the secret | yes | yes | **Either** (UI great for first learn; CLI for bulk) |
| 2 | Create the policy | yes | yes | **CLI** (easy to version-control the .hcl) |
| 3 | Enable AppRole | yes | yes | **Either** |
| 4 | Create the role | limited | yes | **CLI** (UI support for AppRole roles is unreliable) |
| 5 | Get role_id / secret_id | limited | yes | **CLI** |

> Bottom line: learn the first few in the UI if it helps you *see* it; but the
> whole thing should ultimately be a CLI script, because you will repeat it for
> every microservice.

---

## STEP 0 — Enable the KV v2 secrets engine (one-time)

Vault ships with only `cubbyhole/` (a per-token scratch space, not for shared
app secrets). You must enable a KV v2 engine at path `secret/` before storing
anything. Do this **once** for the whole Vault; every service then lives under it.

**Prefer: Either.** It's a one-time action.

### UI
1. Left sidebar -> **Secrets Engines** (a.k.a. Secrets).
2. Top right -> **Enable new engine +**.
3. Choose **KV** -> **Next**.
4. **Path:** `secret`
5. **Version:** `2`
6. **Enable Engine**.

Now `secret/` appears next to `cubbyhole/`.

### CLI
```bash
docker exec vault vault secrets enable -path=secret kv-v2
docker exec vault vault secrets list      # should show secret/ and cubbyhole/
```

---

## STEP 1 — Store the secret

**Prefer: Either.** UI is great to learn; CLI is faster and scriptable.

### UI
1. **Secrets Engines** -> click **`secret/`**.
2. **Create secret +**.
3. **Path for this secret:** `keycloak-resource-manager`
4. Add these 5 key/value rows (click **Add** for each):
   - `spring.datasource.username` = `keycloak`
   - `spring.datasource.password` = `keycloak`
   - `keycloak.credentials.secret` = `cbRgr3bdODKkLKWockyARvITSx6TbAE4`
   - `keycloak-admin.username` = `admin`
   - `keycloak-admin.password` = `admin`
5. **Save**.

### CLI
```bash
docker exec vault vault kv put secret/keycloak-resource-manager \
  spring.datasource.username="keycloak" \
  spring.datasource.password="keycloak" \
  keycloak.credentials.secret="cbRgr3bdODKkLKWockyARvITSx6TbAE4" \
  keycloak-admin.username="admin" \
  keycloak-admin.password="admin"

docker exec vault vault kv get secret/keycloak-resource-manager
```

---

## STEP 2 — Create the policy (permission slip)

**Prefer: CLI** — the policy is a file you can commit to git and reuse.

> KV v2 note: the read path uses `secret/data/<name>` (the `/data/` is mandatory
> for KV v2). Omitting it causes "permission denied".

### CLI
```bash
docker exec -i vault sh -c 'cat > /tmp/krm-policy.hcl <<EOF
path "secret/data/keycloak-resource-manager" {
  capabilities = ["read"]
}
EOF
vault policy write keycloak-resource-manager-policy /tmp/krm-policy.hcl'

docker exec vault vault policy read keycloak-resource-manager-policy
```

### UI
1. Left sidebar -> **Policies** -> **ACL Policies**.
2. **Create ACL policy +**.
3. **Name:** `keycloak-resource-manager-policy`
4. **Policy:**
   ```hcl
   path "secret/data/keycloak-resource-manager" {
     capabilities = ["read"]
   }
   ```
5. **Create policy**.

---

## STEP 3 — Enable the AppRole auth method

**Prefer: Either.** One-time.

### UI
1. Left sidebar -> **Access** -> **Authentication Methods**.
2. **Enable new method +** -> **AppRole** -> **Next** -> **Enable Method**.

### CLI
```bash
docker exec vault vault auth enable approle
docker exec vault vault auth list
```

---

## STEP 4 — Create the role

**Prefer: CLI.** The Vault UI's support for creating AppRole roles is limited /
version-dependent, so CLI is the reliable path.

```bash
docker exec vault vault write auth/approle/role/keycloak-resource-manager \
  token_policies="keycloak-resource-manager-policy" \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=0 \
  secret_id_num_uses=0
```

- `token_policies` — which permission slip the app receives on login.
- `token_ttl` / `token_max_ttl` — how long the app's session token lives.
- `secret_id_ttl=0`, `secret_id_num_uses=0` — the app's PIN doesn't expire and is
  reusable (simplest first setup; tighten later in real production).

---

## STEP 5 — Get the app's badge (role_id + secret_id)

**Prefer: CLI.**

```bash
# role_id = the app's stable "username" (not highly secret)
docker exec vault vault read auth/approle/role/keycloak-resource-manager/role-id

# secret_id = the app's "password" (SECRET; generate fresh, store safely)
docker exec vault vault write -f auth/approle/role/keycloak-resource-manager/secret-id
```

Save both `role_id` and `secret_id`. Spring Boot uses them to authenticate to
Vault (Part C).

---

## What was built
```
secret/ (KV v2 engine)                     enabled once        [STEP 0]
  └─ secret/keycloak-resource-manager       (5 secrets)         [STEP 1]
        ^ read-only
keycloak-resource-manager-policy            (permission slip)   [STEP 2]
        ^ attached by
approle role "keycloak-resource-manager"    (glue)              [STEP 4]
        ^ logs in with
role_id + secret_id                         (app badge)         [STEP 5]
        ^ used by
Spring Boot app                             (Part C, next)
```

---

## Verify everything

```bash
docker exec vault vault secrets list
docker exec vault vault kv get secret/keycloak-resource-manager
docker exec vault vault policy read keycloak-resource-manager-policy
docker exec vault vault auth list
docker exec vault vault read auth/approle/role/keycloak-resource-manager
sudo tail -n 10 ~/docker-file/vault/logs/audit.log
```

---

## Next (Part C) — Spring Boot side
1. Add the Spring Cloud Vault dependency.
2. Configure the Vault URI + AppRole (`role_id`/`secret_id`) + a Java truststore
   for the self-signed cert.
3. Point the app at `secret/keycloak-resource-manager`.
4. Remove the plaintext secrets from `application-*.yml`.
5. Start the app, confirm it reads the secrets from Vault, and check the audit log.
