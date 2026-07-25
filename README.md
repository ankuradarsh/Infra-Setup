# Project Observability — ELK Stack on Kubernetes

This chart deploys a fully automated, zero-touch ELK (Elasticsearch, Logstash, Kibana)
observability stack into the `observability` namespace, managed by ArgoCD.  
No manual token creation, no hardcoded passwords — everything is driven from Vault via
External Secrets Operator.

---

## Directory Structure

```
02-Project-Observability/
├── Chart.yaml                          ← Umbrella Helm chart definition
├── charts/
│   ├── 01-Filebeat/                    ← Log collector (runs on every node)
│   │   └── templates/
│   │       ├── cm.yaml                 ← Filebeat configuration
│   │       ├── daemonset.yaml          ← DaemonSet: one Filebeat pod per node
│   │       └── sa.yaml                 ← ServiceAccount for Filebeat
│   │
│   ├── 02-Logstash/                    ← Log processor & router
│   │   └── templates/
│   │       ├── cm.yaml                 ← Logstash pipeline configuration
│   │       ├── deployment.yaml         ← Logstash Deployment
│   │       └── svc.yaml                ← ClusterIP Service on port 5044
│   │
│   ├── 03-Elasticsearch/               ← Search & storage engine
│   │   └── templates/
│   │       ├── externalsecret.yaml     ← Pulls elastic password from Vault
│   │       ├── service.yaml            ← NodePort Service on port 9200
│   │       └── sts.yaml                ← StatefulSet with password-sync sidecar
│   │
│   └── 04-Kibana/                      ← Visualization UI
│       └── templates/
│           ├── cm.yaml                 ← kibana.yml base configuration
│           ├── configmap-helm-script.yaml  ← Node.js token management script
│           ├── deployment.yaml         ← Kibana Deployment
│           ├── external-secret.yaml    ← (placeholder — secret owned by ES chart)
│           ├── post-delete-job.yaml    ← Cleans up ES token on helm delete
│           ├── pre-install-job.yaml    ← Creates ES service account token
│           ├── rbac.yaml               ← SA + Role + RoleBinding for token job
│           └── svc.yaml                ← NodePort Service on port 5601
```

---

## How It All Fits Together

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Kubernetes Cluster                           │
  │                                                                 │
  │  ┌──────────┐    logs     ┌──────────┐   index   ┌───────────┐ │
  │  │ Filebeat │────────────▶│ Logstash │──────────▶│   Elastic │ │
  │  │(DaemonSet│  port 5044  │(Deployment│  port 9200│  search   │ │
  │  │ every    │             │          │           │(StatefulSet│ │
  │  │ node)    │             └──────────┘           └─────┬─────┘ │
  │  └──────────┘                                          │       │
  │                                                        │query  │
  │  ┌──────────┐              ┌──────────────┐            │       │
  │  │  Kibana  │──SA token───▶│  ES Security │◀───────────┘       │
  │  │   UI     │              │   (xpack)    │                    │
  │  │port 5601 │              └──────────────┘                    │
  │  └──────────┘                                                  │
  │                                                                 │
  │  ┌──────────────────┐                                          │
  │  │  Vault (external)│──ExternalSecret──▶ K8s Secret            │
  │  │  elastisearch/   │                  elasticsearch-          │
  │  │  kibana-token    │                  credentials             │
  │  └──────────────────┘                                          │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Component-by-Component Explanation

---

### 01-Filebeat

#### `cm.yaml` — Filebeat Configuration
**What it is:** A ConfigMap that holds the `filebeat.yml` configuration file.  
**What it does:**
- Reads container log files from `/var/log/containers/*.log` on the host node
- Decodes JSON-formatted container logs automatically (`json.keys_under_root`)
- Enriches every log event with Kubernetes metadata (pod name, namespace, container
  name, node name) via the `add_kubernetes_metadata` processor
- Removes noisy low-value fields (`agent`, `ecs`, `input`, `host`) to keep Kibana clean
- Tags all events with `cluster: k8s-prod` for future multi-cluster support
- Forwards all logs to Logstash at `logstash.observability.svc.cluster.local:5044`

#### `daemonset.yaml` — Filebeat DaemonSet
**What it is:** A DaemonSet that ensures one Filebeat pod runs on every node in the cluster.  
**What it does:**
- Mounts the host's `/var/log` directory (read-only) so it can read all container logs
- Mounts the `filebeat-config` ConfigMap as `/usr/share/filebeat/filebeat.yml`
- Uses an `emptyDir` volume for Filebeat's own state/registry data (tracks which
  log lines have already been shipped to avoid re-sending on restart)
- Runs as the `filebeat` ServiceAccount (defined in `sa.yaml`) which has
  the necessary RBAC to query the Kubernetes API for metadata enrichment

#### `sa.yaml` — ServiceAccount
**What it is:** A ServiceAccount named `filebeat`.  
**Why it exists:** The `add_kubernetes_metadata` processor needs to call the Kubernetes
API to look up pod/node metadata. The pod must run under a ServiceAccount that has
permission to `get`/`list` pods and nodes. The RBAC ClusterRole binding for this
is assumed to exist at the cluster level.

---

### 02-Logstash

#### `cm.yaml` — Logstash Pipeline Configuration
**What it is:** A ConfigMap holding `logstash.conf` — the full Logstash pipeline definition.  
**What it does:**

**Input section:**
- Listens on port `5044` (Beats protocol) for incoming events from Filebeat
- Sets a 300-second inactivity timeout to handle quiet containers

**Filter section:**
- If the event has a `kubernetes` field (enriched by Filebeat), extracts
  `namespace`, `pod`, `container`, `node` as flat top-level fields for easier
  Kibana filtering
- Attempts to parse any `message` field that looks like JSON, storing the result
  under a `json` key (`skip_on_invalid_json: true` means non-JSON logs pass through)
- Removes the redundant fields `agent`, `ecs`, `input`, `log`, `host`
- Lowercases the `level` field for consistent filtering in Kibana

**Output section:**
- Routes `level == error` events → `logs-error-YYYY.MM.dd` index (separate for alerting)
- Routes events with a `namespace` field → `logs-k8s-{namespace}-YYYY.MM.dd` (one
  index per namespace per day for efficient querying)
- Everything else → `logs-misc-YYYY.MM.dd` (catch-all)
- All outputs authenticate to Elasticsearch using `${ELASTIC_PASSWORD}` — an env var
  injected at runtime from the `elasticsearch-credentials` secret (never hardcoded)
- Also prints to stdout via `rubydebug` codec (useful for debugging pipeline issues
  in early development — can be removed in production)

#### `deployment.yaml` — Logstash Deployment
**What it is:** A single-replica Deployment for the Logstash process.  
**What it does:**
- Mounts the `logstash-config` ConfigMap at the pipeline path
- Injects `ELASTIC_PASSWORD` from the `elasticsearch-credentials` secret so the
  pipeline config can reference `${ELASTIC_PASSWORD}` without hardcoding credentials
- Exposes port 5044 for Filebeat to connect to

#### `svc.yaml` — Logstash Service
**What it is:** A ClusterIP (internal) Service.  
**Why it exists:** Filebeat references Logstash by DNS name
(`logstash.observability.svc.cluster.local:5044`). This Service provides that stable
DNS name, routing traffic to whichever Logstash pod is running.

---

### 03-Elasticsearch

#### `externalsecret.yaml` — The Master Password Secret
**What it is:** An `ExternalSecret` resource (processed by External Secrets Operator).  
**What it does:**
- Connects to the `vault-secretstore` (HashiCorp Vault) SecretStore
- Reads the property `kibana-token` from the Vault path `elastisearch`
- Creates and owns a Kubernetes `Secret` named **`elasticsearch-credentials`**
  with a single key `password`
- Refreshes every 1 hour — if the password rotates in Vault, the secret updates

**Why it lives here (not in the Kibana chart):**  
Elasticsearch needs this secret to exist when the pod starts (to set `ELASTIC_PASSWORD`).
This is independent of whether Kibana is installed. Keeping it in the ES chart ensures
it is always present as long as Elasticsearch is deployed.

#### `service.yaml` — Elasticsearch Service
**What it is:** A NodePort Service exposing Elasticsearch.  
**What it does:**
- Provides in-cluster DNS `elasticsearch.observability.svc.cluster.local:9200`
  used by Logstash, Kibana, and the token-creation Job
- Also exposes `nodePort: 30092` for external access (e.g. direct API calls from
  outside the cluster during development)

#### `sts.yaml` — Elasticsearch StatefulSet
**What it is:** A StatefulSet running a single Elasticsearch 9.1.7 node.  
**Why a StatefulSet (not a Deployment):** Elasticsearch stores its data on disk.
StatefulSets provide stable pod names (`elasticsearch-0`) and stable PVC bindings,
ensuring the same pod always re-attaches to the same data volume.

**Container 1 — `elasticsearch` (main node):**
- Runs as `single-node` (no cluster formation needed)
- `xpack.security.http.ssl.enabled: "false"` — **critical**: Elasticsearch 9
  auto-generates TLS certificates on first boot and enables HTTPS by default.
  All other components in this stack use plain `http://`. Setting this to false
  disables that behaviour so everything communicates over HTTP.
- `xpack.security.transport.ssl.enabled: "false"` — same reasoning for the
  internal transport layer
- `ELASTIC_PASSWORD` is set from the `elasticsearch-credentials` secret.
  On a **fresh PVC** (first ever boot), Elasticsearch reads this env var and
  sets the `elastic` user's password. On subsequent restarts the env var alone
  is NOT enough (ES ignores it after first boot) — which is why the sidecar exists.
- Mounts a 10Gi `ReadWriteOnce` PVC (`elastic-data`) for index data persistence

**Container 2 — `password-sync` sidecar (`curlimages/curl`):**  
**Why it exists:** `ELASTIC_PASSWORD` is only applied by Elasticsearch on the very
first boot when the data directory is empty. After that, the password lives inside
ES's internal security index on the PVC. If the Vault secret ever rotates or if the
cluster is rebuilt with a different password, ES and the secret drift out of sync,
causing 401 errors everywhere.

**What it does:**
1. Polls `/_cluster/health` with the current password until ES responds (ES takes
   10–30 seconds to fully initialize after the port opens)
2. Calls `PUT /_security/user/elastic/_password` to force-reset the password
   to match whatever is currently in the `elasticsearch-credentials` secret
3. If auth fails (401 — password already mismatched), attempts an unauthenticated
   reset via `POST` (ES 8+ allows this from localhost as a recovery mechanism)
4. After the reset, runs `sleep infinity` so the sidecar container keeps running
   (Kubernetes would restart the pod if the container exited)

This guarantees that on **every pod start**, the running ES password matches the K8s
secret, regardless of what happened previously.

---

### 04-Kibana

#### `cm.yaml` — Kibana Base Configuration
**What it is:** A ConfigMap holding `kibana.yml`.  
**What it does:** Provides the minimal Kibana configuration:
- `server.host: "0.0.0.0"` — listen on all interfaces inside the container
- `elasticsearch.hosts` — points to the ES service DNS

The service account token (authentication) is intentionally NOT set here — it is
injected as an environment variable (`ELASTICSEARCH_SERVICEACCOUNTTOKEN`) in the
Deployment so it comes from the K8s secret and is never baked into the config file.

#### `configmap-helm-script.yaml` — Token Management Script
**What it is:** A ConfigMap holding a Node.js script `manage-es-token.js`.  
**Why Node.js:** The `node:20-alpine` image is small, has no extra dependencies,
and the built-in `http`/`https` modules are enough to call both the Elasticsearch
REST API and the Kubernetes API.

**What the script does (two modes):**

**`create` mode** (called by `pre-install-job.yaml`):
1. Calls `DELETE` on the ES service token API to remove any pre-existing token
   (idempotent — 404 is ignored)
2. Calls `POST` to create a fresh ES service account token for `elastic/kibana`
3. Checks if the K8s secret `kibana-es-serviceaccount-token` already exists via
   the Kubernetes API
4. If it exists → `PATCH` (strategic merge) to update the token value
5. If it doesn't → `POST` to create it fresh
6. The secret key is `password` to match what the Kibana Deployment reads

**`clean` mode** (called by `post-delete-job.yaml`):
1. Deletes the ES service account token from Elasticsearch
2. Deletes the `kibana-es-serviceaccount-token` K8s secret

**Technical notes:**
- Reads the in-cluster service account token and CA cert from the standard
  projected volume path to authenticate against the Kubernetes API
- Uses a shared `https.Agent({ ca: k8sCa })` so all K8s API calls trust the
  cluster's own CA certificate
- The service account token file is read with `'utf8'` encoding and `.trim()`
  to strip the trailing newline Kubernetes adds — without this, the `Authorization`
  header would be malformed

#### `rbac.yaml` — RBAC for the Token Job
**What it is:** Three resources — ServiceAccount, Role, RoleBinding — all in one file.  
**Why it exists:** The token management script runs inside a Kubernetes pod and
needs to call the Kubernetes API to create/read/update/delete the
`kibana-es-serviceaccount-token` secret. Kubernetes denies all API calls by default.

- **ServiceAccount `kibana-token-sa`:** The identity the job pod runs as
- **Role `kibana-token-role`:** Grants `get`, `create`, `patch`, `delete` on
  `secrets` within the `observability` namespace only (least-privilege)
- **RoleBinding `kibana-token-binding`:** Binds the Role to the ServiceAccount

#### `pre-install-job.yaml` — Token Creation Job
**What it is:** A Kubernetes Job that creates the Kibana service account token.  
**Why a Job (not an initContainer on Kibana):** The token needs to be created
*before* Kibana starts but *after* Elasticsearch is ready. A Job with readiness
gates handles this better than coupling it to the Kibana pod lifecycle.

**Why no Helm hook annotations:** This Job is a plain manifest (no
`helm.sh/hook`). On a fresh install ArgoCD deploys all resources in parallel.
If this were a `pre-install` hook, ArgoCD would wait for the Job to complete
before deploying Elasticsearch — a deadlock since the Job waits for ES.
As a plain Job, it runs in parallel with ES startup and its initContainers
handle the dependency ordering.

**`backoffLimit: 10` + `restartPolicy: OnFailure`:** Generous retry budget
because ES may take a while to start and the job may fail a few times before
succeeding.

**initContainer 1 — `wait-for-credentials` (`bitnami/kubectl`):**
- Polls `kubectl get secret elasticsearch-credentials` every 5 seconds
- Ensures the External Secrets Operator has synced the secret from Vault before
  the main container tries to use it as an env var
- Without this, the pod would fail to start with `CreateContainerConfigError`

**initContainer 2 — `wait-for-elasticsearch` (`curlimages/curl`):**
- Step A: `nc -z` TCP check — waits until port 9200 accepts connections.
  `nc` (not `wget` or `curl`) is used because it succeeds on any TCP connection
  regardless of HTTP response code. Elasticsearch 9 with security enabled returns
  `401` on unauthenticated HTTP — `curl`/`wget` would treat 401 as failure.
- Step B: Authenticated `curl` to `/_security/user/elastic` — waits until the
  ES security subsystem is fully initialized. The TCP port opens ~10 seconds
  before the security API is ready. Without this second check the script would
  get an `ECONNRESET` when hitting the security endpoint.

**Main container — `create-token` (`node:20-alpine`):**
Runs `manage-es-token.js create` which creates the ES service account token
and stores it in the `kibana-es-serviceaccount-token` K8s secret.

#### `deployment.yaml` — Kibana Deployment
**What it is:** A single-replica Deployment for the Kibana UI.  
**What it does:**
- Connects to Elasticsearch via `ELASTICSEARCH_HOSTS` (plain HTTP)
- Authenticates using `ELASTICSEARCH_SERVICEACCOUNTTOKEN` — a service account
  token is more secure than username/password because it is scoped specifically
  to the `elastic/kibana` service and can be rotated independently
- The token value is read from the `kibana-es-serviceaccount-token` secret (key:
  `password`) which was written by the `create-kibana-token` Job
- Mounts `kibana-config` ConfigMap for `kibana.yml`

#### `svc.yaml` — Kibana Service
**What it is:** A NodePort Service exposing Kibana.  
**What it does:** Exposes port `5601` on `nodePort: 30201` so you can reach
the Kibana UI at `http://<any-node-ip>:30201` from outside the cluster.

#### `post-delete-job.yaml` — Token Cleanup Job
**What it is:** A Helm `post-delete` hook Job.  
**What it does:** Runs `manage-es-token.js clean` when the Kibana Helm release
is deleted. This:
1. Deletes the ES service account token from Elasticsearch's security index
2. Deletes the `kibana-es-serviceaccount-token` K8s secret

**Why it matters:** Without cleanup, stale service account tokens accumulate
in Elasticsearch. The next install creates a fresh token with the same name
after first deleting the old one — but the K8s secret cleanup ensures no
stale credentials persist in the cluster.

#### `external-secret.yaml` — (Placeholder)
**What it is:** A comment-only file explaining that the `elasticsearch-credentials`
secret is owned by the Elasticsearch chart (`03-Elasticsearch/templates/externalsecret.yaml`),
not by the Kibana chart. Kept as a file so the intent is documented and the
location is not confusing.

---

## Secret Flow

```
HashiCorp Vault
└── path: elastisearch
    └── property: kibana-token  (the elastic user password)
            │
            ▼
  ExternalSecret (ES chart)
            │  creates & owns
            ▼
  K8s Secret: elasticsearch-credentials
  └── key: password
            │
            ├──▶ Elasticsearch StatefulSet  (ELASTIC_PASSWORD env var → sets elastic user password on first boot)
            │
            ├──▶ password-sync sidecar      (resets elastic password via REST API on every pod restart)
            │
            ├──▶ Logstash Deployment        (ELASTIC_PASSWORD env var → used in pipeline output blocks)
            │
            └──▶ create-kibana-token Job    (authenticates to ES to create service account token)
                            │  writes
                            ▼
              K8s Secret: kibana-es-serviceaccount-token
              └── key: password  (the ES service account token value)
                            │
                            └──▶ Kibana Deployment (ELASTICSEARCH_SERVICEACCOUNTTOKEN env var)
```

---

## Access Points

| Service | Internal DNS | External |
|---|---|---|
| Elasticsearch | `elasticsearch.observability.svc.cluster.local:9200` | `<node-ip>:30092` |
| Logstash (Beats) | `logstash.observability.svc.cluster.local:5044` | — |
| Kibana UI | `kibana.observability.svc.cluster.local:5601` | `<node-ip>:30201` |

---

## Elasticsearch Index Naming Convention

| Index pattern | Contents |
|---|---|
| `logs-error-YYYY.MM.dd` | Events where `level == error` |
| `logs-k8s-{namespace}-YYYY.MM.dd` | All logs from a specific namespace |
| `logs-misc-YYYY.MM.dd` | Logs without a namespace field (catch-all) |

---

## Troubleshooting Reference

| Symptom | Likely cause | Where to look |
|---|---|---|
| Kibana shows "Unable to connect to Elasticsearch" | `kibana-es-serviceaccount-token` secret missing or stale | `kubectl logs -l job-name=create-kibana-token -n observability` |
| `401 Unauthorized` on any component | Password in `elasticsearch-credentials` doesn't match ES | `kubectl logs elasticsearch-0 -c password-sync -n observability` |
| ES pod crashes with `LockObtainFailedException` | Stale `node.lock` on PVC from a previous crash | Delete the PVC: `kubectl delete pvc elastic-data-elasticsearch-0 -n observability` |
| ES pod crashes with `plaintext http traffic on an https channel` | PVC was initialized without the SSL-disabled env vars | Delete the PVC and ensure `xpack.security.http.ssl.enabled: false` is in sts.yaml |
| Token Job stuck in `Init:0/2` | `elasticsearch-credentials` secret not yet synced from Vault | Check ESO: `kubectl get externalsecret -n observability` |
| Token Job stuck in `Init:1/2` | ES port open but security API not ready yet | Normal — wait ~30s. Check: `kubectl logs <job-pod> -c wait-for-elasticsearch -n observability` |
