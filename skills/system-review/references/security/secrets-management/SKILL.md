---
name: secrets-management
description: How to store, distribute, rotate, and revoke secrets (DB passwords, API keys, TLS keys, cloud creds) without baking them into images, leaking them in env vars, or paging the team at 3am because a key rotated and half the fleet is now 401-ing. Reach for this when you see "credentials checked into git", "API key in CloudWatch logs", "rotation broke prod", "secret expired and pods can't start", "shared root password between five services", or "we need to revoke a leaked token but don't know who's using it".
---

# Secrets Management

## Why This Exists

**Problem.** Applications need credentials — database passwords, API keys, TLS private keys, OAuth client secrets, cloud IAM creds. The naive answers all fail at scale:

- **Env vars baked into images** → secret is in every layer, every registry, every CI log, and `docker history` reveals it.
- **Config files checked into git** → one `git log -p` away from disclosure; rotation means a deploy.
- **Shared "service account" with a static password** → no audit trail, no per-caller revocation, blast radius = everyone.
- **Manual rotation every 90 days** → never actually happens, or happens once and breaks prod because nobody knew who was using the key.

**Key insight.** Secrets are not configuration. They have a **lifecycle** (issue → distribute → use → audit → rotate → revoke) and a **threat model** (who can read them at rest, in transit, in memory, in logs). A real secrets management system separates four concerns: **storage** (encrypted at rest with a KMS root), **distribution** (authenticated fetch, not push), **lifecycle** (short TTLs, automatic rotation, revocation), and **audit** (every read logged, every issuance traceable).

The strongest pattern is **dynamic secrets**: the secret doesn't exist until the workload asks for it, has a short TTL (minutes to hours), and is bound to that workload's identity. If it leaks, it expires before the attacker can pivot. Vault popularized this; cloud providers have caught up partially (RDS IAM auth, IAM Roles for Service Accounts, Workload Identity).

### Reach for this when

- You're choosing where to store secrets for a new service (Vault vs AWS Secrets Manager vs SSM Parameter Store vs Kubernetes Secrets vs sealed-secrets vs External Secrets Operator).
- You need to **rotate** a credential without a deploy, or **revoke** a leaked one in seconds.
- A secret leaked in logs / a Slack message / a public S3 bucket and you need to scope the blast radius.
- You're hitting `Secret not found` or `403 AccessDenied` because pods restart faster than secrets sync.
- You want to remove **long-lived static credentials** entirely (replace with workload identity + short-lived tokens).
- An auditor asks "who read the prod DB password in the last 90 days?" and you can't answer.

### Don't reach for this when

- You just need **non-sensitive config** (feature flags, region names, log levels) — use a config service or env vars.
- The "secret" is a public key or certificate fingerprint — that's distribution, not secrecy.
- You're encrypting **data** at rest, not credentials — see `../encryption-at-rest/` (envelope encryption, KMS, application-layer crypto).
- The threat model is "protect against a co-located process on the same host" — that's a sandboxing / TEE problem, not a secrets store problem.

---

## Diagrams

### Static secret vs dynamic secret lifecycle

```mermaid
sequenceDiagram
  participant App
  participant SM as Secrets Manager
  participant DB
  Note over App,DB: STATIC (anti-pattern at scale)
  App->>SM: GetSecret("prod-db-password")
  SM-->>App: "hunter2" (TTL: forever)
  App->>DB: connect("admin", "hunter2")
  Note over App,DB: leak = full compromise until manual rotation

  Note over App,DB: DYNAMIC (Vault DB engine, RDS IAM, etc.)
  App->>SM: IssueCredential(role="orders-readonly")
  SM->>DB: CREATE USER v-token-abc123 WITH PASSWORD ... VALID UNTIL '+1h'
  SM-->>App: {user: v-token-abc123, pass: ..., ttl: 1h}
  App->>DB: connect(v-token-abc123, ...)
  Note over App,DB: leak = attacker has 1h max, all actions audit-logged per-user
```

### External Secrets Operator (Kubernetes pull model)

```mermaid
flowchart TD
  A[Workload Pod] -->|reads| B[Kubernetes Secret]
  C[ExternalSecret CR] -->|reconciles| B
  D[ExternalSecrets Operator] -->|watches| C
  D -->|fetch with IRSA/Workload Identity| E[(AWS Secrets Manager / Vault / GCP Secret Manager)]
  E -->|encrypted at rest| F[(KMS Root Key)]
  D -->|on rotation event| B
  B -.->|projected volume / envFrom| A

  style E fill:#fef3c7
  style F fill:#fee2e2
```

### Sidecar injection (push model, Vault Agent)

```mermaid
sequenceDiagram
  participant K as Kubelet
  participant I as Vault Agent Injector
  participant P as Pod (app + sidecar)
  participant V as Vault
  K->>I: admission webhook on pod create
  I->>P: mutate pod spec (add init+sidecar containers, shared volume)
  P->>V: auth via K8s ServiceAccount JWT (TokenReview)
  V-->>P: client token (short TTL)
  loop every renewal interval
    P->>V: read secret/data/orders/db
    V-->>P: {username, password, lease_duration}
    P->>P: render template to /vault/secrets/db.env
    P->>P: signal app (SIGHUP) or app re-reads file
  end
```

---

## Core patterns

### 1. Threat model first, tooling second

Before picking a tool, write down:

| Question | Why it matters |
|---|---|
| **Who/what reads this secret?** (human, CI job, pod, lambda) | Determines auth method (OIDC, IAM, K8s SA, mTLS) |
| **What's the blast radius if it leaks?** | Drives TTL aggressiveness and rotation cadence |
| **Can it be made dynamic?** | Static DB passwords → Vault DB engine or RDS IAM auth |
| **Who needs to audit reads?** | Static file storage gives you nothing; SM/Vault give you per-read logs |
| **What's the recovery RPO if the secrets store is down?** | Drives caching strategy and offline-tolerance |

If the answer to "can it be dynamic" is yes, **make it dynamic**. Static secrets are a debt you keep paying.

### 2. Static secret storage: AWS Secrets Manager (canonical)

```python
# Python — fetch on startup, cache, refresh on rotation event.
# Real-world pattern: NEVER fetch per-request (rate limits, latency, $).
import boto3
import json
import threading
import time

_cache = {}
_lock = threading.Lock()
_sm = boto3.client("secretsmanager")

def get_secret(name: str, ttl_seconds: int = 300) -> dict:
    """Fetch secret with in-process cache. Background thread refreshes
    before TTL expires so the hot path never blocks on Secrets Manager."""
    now = time.time()
    with _lock:
        entry = _cache.get(name)
        if entry and entry["expires"] > now:
            return entry["value"]

    # Cache miss or expired — fetch (this *can* fail; have a fallback)
    resp = _sm.get_secret_value(SecretId=name)
    value = json.loads(resp["SecretString"])

    with _lock:
        _cache[name] = {"value": value, "expires": now + ttl_seconds}
    return value


# Anti-pattern (DO NOT DO):
# os.environ["DB_PASSWORD"] = get_secret("prod-db")["password"]
# Reasons:
#   1. Env vars are inherited by child processes (subprocess.Popen, exec)
#   2. /proc/<pid>/environ is readable by same-uid processes
#   3. Crash dumps and APM tools (Datadog, NewRelic) often capture env
#   4. Rotation requires process restart
```

**Rotation is the hard part.** Secrets Manager supports automatic rotation via Lambda. The Lambda must implement the four phases:

```python
# Lambda rotation handler (skeleton)
def lambda_handler(event, context):
    step = event["Step"]  # createSecret | setSecret | testSecret | finishSecret
    secret_id = event["SecretId"]
    token = event["ClientRequestToken"]

    if step == "createSecret":
        # Generate the NEW credential, store as AWSPENDING stage
        new_password = generate_password()
        sm.put_secret_value(SecretId=secret_id, ClientRequestToken=token,
                            SecretString=json.dumps({"password": new_password}),
                            VersionStages=["AWSPENDING"])
    elif step == "setSecret":
        # Apply NEW credential to the actual system (DB, API, etc.)
        # CRITICAL: keep the OLD one working too — readers haven't refreshed yet
        rotate_db_password(new_password)
    elif step == "testSecret":
        # Verify the new credential works end-to-end
        connect_with(new_password)  # raises on failure
    elif step == "finishSecret":
        # Promote AWSPENDING → AWSCURRENT, keep AWSPREVIOUS for grace period
        sm.update_secret_version_stage(SecretId=secret_id, VersionStage="AWSCURRENT",
                                       MoveToVersionId=token,
                                       RemoveFromVersionId=current_version)
```

The four-phase protocol exists because **you cannot atomically swap a secret across N readers**. There must be a window where both old and new work.

### 3. Dynamic secrets with HashiCorp Vault

The signature feature: secrets are issued on demand with a short lease, and Vault revokes them when the lease expires.

```hcl
# Vault — enable database secrets engine, configure Postgres role
vault secrets enable database

vault write database/config/orders-pg \
  plugin_name=postgresql-database-plugin \
  allowed_roles="orders-readonly,orders-readwrite" \
  connection_url="postgresql://{{username}}:{{password}}@pg.internal:5432/orders" \
  username="vault_admin" \
  password="$(cat /run/secrets/vault_admin_pw)"

vault write database/roles/orders-readonly \
  db_name=orders-pg \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
                       GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

```python
# App fetches a fresh credential — Vault creates a NEW Postgres user each time
import hvac
client = hvac.Client(url="https://vault.internal:8200")
client.auth.kubernetes.login(role="orders-app", jwt=read_sa_token())

cred = client.secrets.database.generate_credentials(name="orders-readonly")
# cred = {"username": "v-kube-orders-XYZ", "password": "...", "lease_id": "...", "lease_duration": 3600}
# Connect, do work. When lease expires (or app calls revoke), Vault DROPs the user.
```

**Why this is qualitatively better than static secrets:**

- Per-pod, per-invocation identity → audit logs show *which workload* read the DB.
- Leaked credential is useless after the lease expires (default 1h, max 24h).
- No rotation problem: the credential is born ephemeral, dies, next call gets a fresh one.
- Revocation is `vault lease revoke <lease_id>` — Vault drops the DB user immediately.

### 4. Kubernetes: don't use raw Secrets in git

`Kind: Secret` in Kubernetes is **base64-encoded, not encrypted**. If you commit it to git, you've committed plaintext. Three real options:

#### Option A: Sealed Secrets (Bitnami)

Encrypts secrets with a per-cluster public key. The private key lives in the controller; only that controller can decrypt. Safe to commit the encrypted form to git.

```bash
# Encrypt locally with cluster's pub key
kubeseal --format yaml --cert pub-cert.pem < db-secret.yaml > db-sealed.yaml
# Commit db-sealed.yaml. The controller decrypts in-cluster.
```

**Limitation.** Tied to one cluster's key. Disaster recovery means backing up the key. Rotation is manual. No dynamic secrets, no audit trail beyond Kubernetes events.

#### Option B: External Secrets Operator (ESO)

A controller that reads from a real secrets manager (Vault, AWS SM, GCP SM, Azure Key Vault, 1Password) and materializes Kubernetes Secrets. Workloads use them like normal. ESO refreshes on a schedule.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-sm
  namespace: orders
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: orders-sa  # IRSA — no static AWS keys in cluster
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: orders-db
  namespace: orders
spec:
  refreshInterval: 1h           # poll SM hourly; rotation propagates here
  secretStoreRef:
    name: aws-sm
    kind: SecretStore
  target:
    name: orders-db-secret      # the K8s Secret that ESO creates
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: prod/orders/db
        property: password
```

**Trade-off.** The materialized Kubernetes Secret is still in etcd (encrypt etcd!), still readable by anyone with `secrets:get` RBAC. ESO is a *bridge* to the source of truth, not a replacement for least-privilege RBAC. **Don't** mount it as `envFrom` — see pitfalls.

#### Option C: Vault Agent Injector (sidecar pattern)

The injector mutates pod specs to add an init container + sidecar that authenticate to Vault using the pod's ServiceAccount JWT, fetch secrets, and render them to a shared `emptyDir` volume that only the app container can read.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "orders-app"
        vault.hashicorp.com/agent-inject-secret-db.env: "database/creds/orders-readonly"
        vault.hashicorp.com/agent-inject-template-db.env: |
          {{- with secret "database/creds/orders-readonly" -}}
          DB_USER={{ .Data.username }}
          DB_PASS={{ .Data.password }}
          {{- end }}
        vault.hashicorp.com/agent-inject-perms-db.env: "0440"
    spec:
      serviceAccountName: orders-app
      containers:
        - name: orders
          image: orders:1.2.3
          # No env vars with secrets. App reads /vault/secrets/db.env
          command: ["/bin/sh", "-c", ". /vault/secrets/db.env && exec /app/orders"]
```

**Why sidecar wins for dynamic secrets.** The sidecar handles lease renewal, re-templating on rotation, and signaling the app. The app stays simple — it reads a file. No SDK in the application.

### 5. Workload identity: kill static cloud credentials

The biggest single win in modern secrets management is **eliminating static cloud creds entirely**. No more `AWS_ACCESS_KEY_ID` env vars, no more `gcloud auth activate-service-account` with a JSON key file.

| Platform | Mechanism | What replaces |
|---|---|---|
| AWS + EKS | **IRSA** (IAM Roles for Service Accounts) | Static IAM access keys |
| AWS + EC2/Lambda/ECS | Instance/task role | Static IAM access keys |
| GCP + GKE | **Workload Identity** | Service account JSON keys |
| Azure + AKS | **Workload Identity** (OIDC federation) | Service principal client secrets |
| Generic K8s | OIDC + cloud STS federation | Long-lived cloud creds in Secrets |

```yaml
# IRSA — pod assumes IAM role via OIDC, no static creds anywhere
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-sa
  namespace: orders
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/orders-app
```

The pod's projected SA token is exchanged for AWS STS credentials by the AWS SDK automatically. **Credentials are short-lived (1 hour default), auto-refreshed, never written to disk.** A leaked container image teaches an attacker nothing.

### 6. Do not put secrets in environment variables (the deep version)

| Reason | Detail |
|---|---|
| **Process tree leakage** | `subprocess.Popen` inherits env by default. Crash handlers (Sentry, Bugsnag) often capture env. |
| **`/proc/<pid>/environ`** | Readable by same-uid processes. Container `nobody` user shares uid with other pods on shared nodes (rare, but real with privileged sidecars). |
| **CI/CD echo** | `make` and shell scripts routinely echo env on error. `set -x` is one mistake away. |
| **Image layers** | `ENV DB_PASSWORD=...` in a Dockerfile lives in *every layer* and registry forever. |
| **Logs** | APM agents, error trackers, and log shippers commonly serialize env in stack traces. |
| **No rotation** | Process restart required to pick up new value. |

**Better.** Mount secrets as files via tmpfs (Vault Agent, ESO with `subPath`, Secrets Store CSI Driver). Files can be re-read; permissions are enforced; not inherited; not in crash dumps.

```yaml
# Secrets Store CSI Driver — mounts directly from the secret manager,
# no Kubernetes Secret object created at all (with --enableSecretRotation)
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: orders-aws-sm
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/orders/db"
        objectType: "secretsmanager"
        jmesPath:
          - path: "password"
            objectAlias: "db_password"
```

### 7. Revocation playbook (the one you'll actually need at 2am)

A credential leaked. Slack screenshot, public commit, BugSnag log, whatever. You have **minutes**. The playbook depends on what you have:

```mermaid
flowchart TD
  A[Secret leaked] --> B{Dynamic or static?}
  B -->|Dynamic Vault lease| C[vault lease revoke prefix database/creds/...]
  B -->|Static| D{Cloud credential?}
  D -->|AWS IAM access key| E[aws iam update-access-key --status Inactive]
  D -->|AWS IAM role assumed| F[Attach AWSRevokeOlderSessions deny policy + rotate trust policy]
  D -->|API key in SaaS| G[Hit provider revoke endpoint]
  D -->|DB password| H[ALTER USER ... PASSWORD ...; then update SM/Vault]
  C --> I[Audit logs: who used the lease? scope blast radius]
  E --> I
  F --> I
  G --> I
  H --> I
  I --> J[Post-mortem: how did it leak? add detection / pre-commit hook / log scrubber]
```

The lesson: **build the revoke path before you need it**. Dry-run it quarterly. If revocation requires a human to find the secret in a runbook and manually click buttons in three consoles, you don't have revocation.

### 8. Secret zero / chicken-and-egg

Every secrets system has a "secret zero" — the credential the app uses to authenticate *to the secrets store*. Solving this is the whole game.

| Solution | How |
|---|---|
| **Cloud workload identity** | The platform (AWS/GCP/Azure) signs an OIDC JWT for the workload. Vault/SM trusts that issuer. No secret zero. |
| **Kubernetes ServiceAccount JWT** | Vault validates SA token via TokenReview API. The SA token is provisioned by Kubelet. |
| **TPM / cloud TEE attestation** | Hardware proves "I am a real machine running this signed image". Used by SPIFFE/SPIRE node attestors. |
| **Short-lived bootstrap token** | CI issues a one-shot token good for 5 minutes, used once. Bad on its own; OK as a stopgap. |
| **Stuffed into Dockerfile or env** | **DO NOT.** This is what we're trying to escape. |

[SPIFFE/SPIRE](https://spiffe.io/) generalizes this: every workload gets a verifiable identity (SVID — SPIFFE Verifiable Identity Document) issued via attestation. Your secrets store trusts SPIFFE IDs. No secret zero anywhere in the stack.

---

## Trade-offs

| Benefit | Cost |
|---|---|
| **Dynamic secrets** — short TTL, per-workload identity, automatic revocation | Requires the backend (DB/cloud) to support per-call provisioning. Vault DB engine creates real DB users — can hit `max_connections` / role limits at scale. |
| **Centralized secrets store** (Vault, AWS SM) — one place to audit/rotate | Becomes a critical dependency. Outage cascades to every workload starting/restarting. Plan caching + degraded modes. |
| **Workload identity (IRSA, GKE WI)** — no static cloud creds | Tied to one platform. Multi-cloud means SPIFFE or per-cloud setup. Token exchange adds latency on cold start. |
| **External Secrets Operator** — Git-native, declarative | Materializes K8s Secrets in etcd. RBAC must be tight. Refresh interval = staleness window during rotation. |
| **Sealed Secrets** — works fully offline, GitOps-pure | No dynamic secrets, no real audit, manual rotation. Cluster key compromise = global compromise. |
| **Sidecar injection (Vault Agent)** — app stays SDK-free | Extra container per pod = memory & CPU overhead. Sidecar lifecycle bugs (e.g., Job pods don't terminate). |
| **CSI Secrets Store** — secrets never in K8s API | Filesystem-only access; some tools expect env vars. Per-pod mount = per-pod fetch latency. |
| **Short rotation cadence** (hours instead of days) | Caching, refresh logic, and four-phase rotation must be rock-solid. Bugs cause outages, not security wins. |

---

## Common Pitfalls

- **`envFrom` with rotated secrets.** A `Secret` updated in place does NOT propagate to env vars in running pods. The pod must restart. Teams find this out the hard way after rotation: half the fleet has the old password, half has the new. Use **mounted volumes** (which DO update via kubelet sync, ~60s) or restart on rotation via Reloader.
- **Secrets in `kubectl describe pod`.** Even if you avoid env vars, the pod's `args` and annotations are dumped by `describe`. Don't pass secrets as CLI args.
- **Logs and crash dumps.** Sentry/Datadog/Bugsnag capture local variables and env. Add scrubbers (`SENTRY_DENY_ENV=DB_PASSWORD,*_KEY,*_TOKEN`). Test with intentional crashes.
- **Pre-commit hooks but no server-side scan.** A determined dev can `git commit --no-verify`. Run `gitleaks` / `trufflehog` server-side on every push. Block merges.
- **`docker history` reveals build-time secrets.** `RUN echo $TOKEN > /tmp/x && curl ...` puts the token in the layer's command metadata, even if the file is deleted later. Use BuildKit `--mount=type=secret`.
- **Vault token leak via `~/.vault-token`.** A user logs in, the token sits in `$HOME` with `0600`. Compromise of that user = full Vault access until revoked. Use short TTLs, periodic tokens for daemons.
- **Rotation breaks readers that cache forever.** App fetches secret on startup, never re-reads. Rotation succeeds in SM but old pods keep using old creds. Either use leases (re-fetch when lease expires) or refresh on a timer.
- **"Just in case" wildcard policies.** `path "secret/*" { capabilities = ["read"] }` defeats the point of Vault. Scope policies tightly — per-app, per-environment.
- **Sealed Secrets cluster key never backed up.** Cluster lost = all sealed secrets unrecoverable. Backup the controller's private key to an offline-encrypted location.
- **Static secrets in CI variables.** GitHub Actions / GitLab CI variables are fine for low-blast-radius things, but for prod creds use OIDC federation to AWS/GCP. GitHub now supports this natively — no static cloud keys in CI.
- **Secret stored in URL.** `https://api.com/v1/orders?api_key=abc123`. Logged by every proxy, every CDN, every browser history. Use headers (`Authorization: Bearer ...`) instead.
- **Forgetting `AWSPREVIOUS`.** Secrets Manager rotation creates `AWSCURRENT` and keeps `AWSPREVIOUS` for grace. If a long-running consumer has the old version cached, your DB must accept both during the window. Don't drop the old user on the very first rotation step.
- **Vault sealed at boot.** Vault starts sealed and needs unsealing. Auto-unseal with cloud KMS in production — manual unsealing means an outage during cold start.
- **Treating audit logs as private.** Vault audit logs may contain hashed secrets and request paths. Ship them to a separate, more-restricted SIEM than app logs.

---

## Decision Table

| Situation | Use this | Not this |
|---|---|---|
| K8s app, AWS, want to read AWS secrets | **External Secrets Operator + IRSA** or **CSI Secrets Store** | Static AWS keys in `Secret` |
| K8s app, want dynamic DB creds with audit | **Vault DB engine + Vault Agent Injector** | Static creds in any secrets manager |
| Multi-cloud or hybrid, single source of truth | **HashiCorp Vault** | Per-cloud secret managers (you'll re-implement abstraction) |
| Pure cloud, single-cloud, simplicity wins | **AWS Secrets Manager / GCP Secret Manager / Azure Key Vault** | Self-hosted Vault (operational overhead) |
| GitOps-only, air-gapped, no central server | **Sealed Secrets** | ESO (needs network to source of truth) |
| EC2 / Lambda / ECS app needs cloud API access | **Instance/Task IAM role** | `AWS_ACCESS_KEY_ID` env var |
| Service-to-service auth inside cluster | **mTLS via SPIFFE/SPIRE or service mesh (Istio/Linkerd)** | Shared API keys |
| Short-lived script / one-off job | **Cloud workload identity + STS** | Long-lived static key |
| Need to rotate weekly/daily | **Dynamic secrets (Vault) or RDS IAM auth** | Static + manual rotation |
| Customer-supplied secrets (BYOK, third-party API keys) | **Per-tenant secret store namespace + envelope encryption** | Single shared secret per provider |
| Build-time secrets (private package registry, signing key) | **BuildKit `--mount=type=secret`** | `ARG` / `ENV` in Dockerfile |
| Local dev | **Direnv + 1Password CLI / `aws-vault` / `gcloud auth`** | `.env` files in repo |
| CI/CD needs cloud creds | **OIDC federation (GitHub→AWS, GitLab→GCP)** | Long-lived service account keys in CI vars |
| Need offline / disconnected operation | **Sealed Secrets** or **local Vault with replicated unseal** | Cloud-only secret manager |
| Public key / cert distribution (not secret) | **ConfigMap or cert-manager** | Secrets manager (overkill, hides intent) |

---

## References

- HashiCorp — *Vault Concepts: Secrets Engines* — https://developer.hashicorp.com/vault/docs/secrets
- HashiCorp — *Vault Database Secrets Engine* — https://developer.hashicorp.com/vault/docs/secrets/databases
- HashiCorp — *Vault Agent Sidecar Injector for Kubernetes* — https://developer.hashicorp.com/vault/docs/platform/k8s/injector
- HashiCorp — *Auto-unseal with Cloud KMS* — https://developer.hashicorp.com/vault/docs/concepts/seal
- AWS — *AWS Secrets Manager User Guide* — https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html
- AWS — *Rotating Your AWS Secrets Manager Secrets* — https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html
- AWS — *IAM Roles for Service Accounts (IRSA)* — https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
- AWS — *Configuring GitHub Actions OIDC to assume an IAM role* — https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html
- Google Cloud — *Secret Manager overview* — https://cloud.google.com/secret-manager/docs/overview
- Google Cloud — *GKE Workload Identity* — https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity
- Microsoft — *Azure Key Vault concepts* — https://learn.microsoft.com/en-us/azure/key-vault/general/basic-concepts
- Microsoft — *Azure AD Workload Identity for Kubernetes* — https://azure.github.io/azure-workload-identity/docs/
- Kubernetes — *Secrets* (and why they aren't encrypted by default) — https://kubernetes.io/docs/concepts/configuration/secret/
- Kubernetes — *Encrypting Secret Data at Rest* — https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- External Secrets Operator — *Documentation* — https://external-secrets.io/latest/
- Bitnami — *Sealed Secrets* — https://github.com/bitnami-labs/sealed-secrets
- Kubernetes SIG — *Secrets Store CSI Driver* — https://secrets-store-csi-driver.sigs.k8s.io/
- SPIFFE — *SPIFFE Concepts (SVID, Trust Domain, Workload API)* — https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/
- BSRS — *Building Secure and Reliable Systems*, ch. 5 ("Design for Least Privilege") and ch. 9 ("Cryptography and Key Management") — https://sre.google/books/building-secure-reliable-systems/
- Google SRE Workbook — *Managing Load* (relevant for secrets-store dependency planning) — https://sre.google/workbook/table-of-contents/
- NIST SP 800-57 — *Recommendation for Key Management* — https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final
- OWASP — *Secrets Management Cheat Sheet* — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- The Twelve-Factor App — *Config* (and why env-vars-for-everything is now considered too coarse for secrets) — https://12factor.net/config
- Docker — *BuildKit build secrets* — https://docs.docker.com/build/building/secrets/
- DDIA — *Designing Data-Intensive Applications*, ch. 9 ("Consistency and Consensus") for distributed config / leader-based key distribution; ch. 11 for Kafka-style audit log shipping.

---

## See Also

- `../encryption-at-rest/` — KMS, envelope encryption, when to encrypt at the application layer vs storage layer
- `../audit-logging/` — capturing secret access, retention, SIEM forwarding
- `../vulnerability-management/` — signing, SBOMs, build-time secret hygiene, BuildKit
- `../../reliability/graceful-degradation/` — what your app does when the secrets store is down
