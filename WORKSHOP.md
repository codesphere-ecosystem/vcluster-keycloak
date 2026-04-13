# Workshop: Cloud Native Deployments with vCluster on Codesphere

This hands-on workshop walks you through deploying a real-world application — [Keycloak](https://www.keycloak.org/) — into a virtual Kubernetes cluster managed by Codesphere. You will progressively build up from provisioning the cluster to running a production-ready setup with a platform-managed database.

**Prerequisites:**
- Access to a Codesphere team
- Basic familiarity with Kubernetes concepts (`kubectl`, namespaces, pods, services)
- Basic familiarity with Helm

**Documentation references:**
- [Codesphere Runtimes Overview](https://docs.codesphere.com/core-concepts/runtimes)
- [Cloud Native Deployments](https://docs.codesphere.com/core-concepts/runtimes#cloud-native-deployments)
- [Configuring a Landscape](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/configuring-a-landscape)
- [Configuring Your CI Pipeline](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/configuring-ci-pipeline)

---

## Section 1: Deploy a vCluster in Your Team

In this section you will provision a virtual Kubernetes cluster (vCluster) as a Managed Service in your team. This gives you a full `kubectl`-capable environment without managing underlying infrastructure.

### Steps

1. Navigate to the **Managed Services** tab in your Codesphere team.
2. Click on **Service Catalog** and find the **vCluster** service provider.
3. Click **Start Setup**.
4. Configure the service:
   - **Name:** Leave the default or give it a meaningful name (e.g., `k8s`).
   - **Resources:** The vCluster does not use predefined plans — instead you configure custom resource limits. Use the following defaults for this workshop:
     - **CPU:** `50` (5 vCPU — units are in tenths)
     - **Memory:** `10240` MiB (10 GiB)
5. Click **Create Service** and wait for the status to change to `Synchronized`.

### What just happened?

Codesphere provisioned a virtual Kubernetes cluster for your team. This cluster is:
- Isolated to your team — each team can only have **one vCluster** (it is a singleton per team). If your team already has a vCluster provisioned, you can skip this section and use the existing one.
- The provisioned cluster is fully compatible with standard `kubectl` commands
- Managed by the platform — no node maintenance, no control plane management

The resource values you configured (CPU, memory, storage) are **upper limits**, not reserved allocations. The cluster will use resources up to those limits. You can update these limits at any time through the Managed Service settings if your workloads need more (or less) capacity.

> **Key concept:** Managed Services in Codesphere handle provisioning, lifecycle, and teardown for you. See [Deploying Services](https://docs.codesphere.com/managed-services/deploying-services) for details.

---

## Section 2: Check Out the vCluster

Now let's deploy something into the cluster. We will use a pre-built repository that deploys Keycloak via a Helm chart.

### 2.1 Create a Workspace

1. Create a **new Workspace** in your team.
2. Set the **Git URL** to: `https://github.com/codesphere-ecosystem/vcluster-keycloak`
3. Wait for the workspace to initialize.

### 2.2 Deploy the Landscape

Before looking at any configuration files, let's just run it:

1. Switch to the **CI & Deploy** workspace mode (top center dropdown).
2. Open the **Execution Manager**.
3. Make sure the **Default** CI profile is selected.
4. Set the required secret in **Vault**: add a secret named `KEYCLOAK_ADMIN_PASSWORD` with a password of your choice. See [Secret Management](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/secret-management).
5. Click **Deploy Landscape** and wait for it to complete.

### 2.3 Discover What Was Deployed

Open a **Terminal** in your workspace. You now have `kubectl` access to the vCluster. Let's explore:

**List namespaces:**
```bash
kubectl get namespaces
```
You should see a `keycloak` namespace was created.

**List all resources in the keycloak namespace:**
```bash
kubectl get all -n keycloak
```

> **Exercise:** Answer the following:
> - What Kubernetes resources were created? (Pods, Services, Deployments, StatefulSets?)
> - What image is running in the pod?
> - What port is the service listening on?

**Check the Helm release:**
```bash
helm list -n keycloak
```

**Inspect the Helm values that were applied:**
```bash
helm get values keycloak -n keycloak
```

> **Exercise:** Inspect the Helm values that were applied:
> ```bash
> helm get values keycloak -n keycloak
> ```
> How are the admin password and hostname passed to the Keycloak pod?

### 2.4 Understand the CI Configuration

Now open `ci.yml` in the editor. Let's walk through it:

```yaml
schemaVersion: v0.2

prepare:
  steps: []

test:
  steps: []

run:
  keycloak-deployer:
    plan: 201
    replicas: 1
    network:
      ports:
        - port: 3000
          isPublic: false
      paths: []
    env:
      KEYCLOAK_ADMIN_PASSWORD: ${{ vault.KEYCLOAK_ADMIN_PASSWORD }}
    steps:
      - name: Create namespace
        command: kubectl create namespace keycloak --dry-run=client -o yaml | kubectl
          apply -f -
      - name: Deploy Keycloak to vCluster
        command: helm upgrade --install keycloak
          oci://ghcr.io/codecentric/helm-charts/keycloakx --namespace keycloak
          --create-namespace --values values.yaml --set-string
          extraEnvFromValues.adminPassword=$KEYCLOAK_ADMIN_PASSWORD --set-string
          extraEnvFromValues.hostname=$WORKSPACE_DEV_DOMAIN --wait --timeout 10m
  keycloak-headless:
    network:
      paths:
        - path: /
          stripPath: false
          target: http://keycloak-keycloakx-http-x-keycloak-x-k8s.rg-${{ team.id }}.svc.cluster.local:80
```

#### Key concepts explained

| Concept | Explanation |
|---|---|
| `schemaVersion: v0.2` | The Codesphere CI schema version. See [Configuring Your CI Pipeline](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/configuring-ci-pipeline). |
| `keycloak-deployer` | A Reactive service that runs the deployment steps. Because it runs in the `run` stage, it has access to vault secrets and Codesphere template variables via its `env` block. |
| `env` block | Maps Codesphere template variables (like `${{ vault.* }}`) to environment variables available inside the service's step commands. Template variables cannot be used directly in commands. |
| `${{ vault.KEYCLOAK_ADMIN_PASSWORD }}` | References a secret stored in the [Codesphere Vault](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/secret-management). Resolved at deploy time and injected as the `$KEYCLOAK_ADMIN_PASSWORD` environment variable. |
| `$WORKSPACE_DEV_DOMAIN` | A built-in Codesphere [environment variable](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/environment-variables) containing the workspace's development domain. |
| `--set-string` | Passes values directly to the Helm chart at deploy time. Environment variables are resolved by the shell before being passed to Helm. |
| `helm upgrade --install` | Installs the chart if it doesn't exist, or upgrades it if it does — an idempotency pattern. |
| `keycloak-headless` | A headless service that routes external traffic to the vCluster service. |

Now open `values.yaml` and examine the Helm values:

```yaml
command:
  - "/opt/keycloak/bin/kc.sh"
  - "start-dev"
  - "--http-port=8080"
  - "--hostname-strict=false"

extraEnvFromValues:
  adminPassword: ""
  hostname: ""

extraEnv: |
  - name: KC_BOOTSTRAP_ADMIN_PASSWORD
    value: {{ .Values.extraEnvFromValues.adminPassword | quote }}
  - name: KC_BOOTSTRAP_ADMIN_USERNAME
    value: admin
  - name: KC_HOSTNAME
    value: {{ .Values.extraEnvFromValues.hostname | quote }}
```

> **Discussion points:**
> - The `extraEnv` field is processed through Helm's `tpl` function by the keycloakx chart, so template expressions like `{{ .Values.* }}` are evaluated.
> - `extraEnvFromValues` defines default (empty) values that get overridden by `--set-string` in the CI command. This keeps the values file self-contained while allowing secrets to be injected at deploy time.
> - The admin password comes from `$KEYCLOAK_ADMIN_PASSWORD` and the hostname from `$WORKSPACE_DEV_DOMAIN` — both available as environment variables in the Reactive service's shell. The vault secret is mapped to the env var in the service's `env` block.
> - `start-dev` runs Keycloak in development mode with an embedded H2 database — no external database needed for a quick start.

Also note the **security context** in `values.yaml`:

```yaml
podSecurityContext:
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 1000
  capabilities:
    drop:
      - ALL
```

> **Why is this needed?** The vCluster enforces Kubernetes [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) at the `restricted` level. Pods must explicitly declare a non-root user, drop all capabilities, and set a seccomp profile.

---

## Section 3: Expose the Service

The application is running inside the vCluster, but it's not accessible from outside yet. Let's fix that.

### 3.1 Look at the Run Section

Open `ci.yml` and look at the `run:` section. Alongside the `keycloak-deployer` Reactive, you'll see:

```yaml
  keycloak-headless:
    network:
      paths:
        - path: /
          stripPath: false
          target: http://keycloak-keycloakx-http-x-keycloak-x-k8s.rg-${{ team.id }}.svc.cluster.local:80
```

This defines a **headless service** — a Codesphere routing entry that forwards incoming HTTP traffic to a target URL inside the platform network.

#### How vCluster service discovery works

Services inside the vCluster are accessible from the Codesphere platform network using a deterministic hostname pattern:

```
http://{service-name}-x-{namespace}-x-k8s.rg-{team-id}.svc.cluster.local:{port}
```

| Part | Value | Description |
|---|---|---|
| `{service-name}` | `keycloak-keycloakx-http` | The Kubernetes Service created by the Helm chart |
| `{namespace}` | `keycloak` | The namespace the chart was deployed into |
| `k8s` | fixed | Identifies the vCluster runtime |
| `{team-id}` | `${{ team.id }}` | Your Codesphere team ID (templated at deploy time) |

> **Exercise:** Run this command to find the service name:
> ```bash
> kubectl get svc -n keycloak
> ```
> Does the service name match what's in the `target` URL?

### 3.2 Access the Deployment

Since the landscape was already deployed in Section 2.2, the headless service is already active.

1. In the **CI & Deploy** view, click **Open Deployment** to access the workspace's public URL.

You should see the Keycloak login page.

> **Exercise:** Log in with:
> - **Username:** `admin`
> - **Password:** the value you set for `KEYCLOAK_ADMIN_PASSWORD` in the vault

### 3.3 How Exposure Works

```
Internet → Codesphere Router → keycloak-headless service → vCluster Service → Keycloak Pod
```

The `keycloak-headless` service in the `run:` section does **not** run a process. Instead, the `network.paths` configuration tells the Codesphere workspace router to forward requests matching `path: /` to the `target` URL. The `keycloak-deployer` Reactive handles the actual deployment into the vCluster, while the headless service handles the routing bridge.

> **Key takeaway:** Unlike Reactives or Managed Containers where Codesphere manages the process directly, Cloud Native Deployments require manual networking integration. You define the routing bridge between the platform router and your vCluster services. See [Platform Integration](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/configuring-a-landscape#platform-integration).

---

## Section 4: Production Readiness with a Managed Database

The development setup uses Keycloak's embedded H2 database — great for getting started, but not suitable for production. Data would be lost on every pod restart. Let's add a proper PostgreSQL database using a Codesphere Managed Service.

### 4.1 Why a Platform Managed Service?

You could deploy PostgreSQL as another Helm chart into the vCluster. However, Codesphere offers **Managed Services** that provide significant advantages:

| | Self-managed in vCluster | Platform Managed Service |
|---|---|---|
| **Provisioning** | Manual Helm chart + config | One declaration in `ci.yml` |
| **Backups** | You configure and maintain | Platform-managed |
| **Updates** | You upgrade the chart | Platform rolls out improvements |
| **Monitoring** | Manual setup | Platform-integrated |
| **Lifecycle** | Manual cleanup | Automatically destroyed with the landscape |
| **Quality** | Varies by chart | Consistent, platform-validated |

> **Best practice:** Always prefer Platform Managed Services over self-managed databases. You benefit from platform-wide improvements to reliability, backup strategies, and security — without any extra work on your side. See [Managed Services in Landscapes](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/configuring-a-landscape#managed-services-in-landscapes).

### 4.2 Walk Through `ci.prod.yml`

Codesphere supports [CI Profiles](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/using-ci-profiles) — different configurations for different environments. The file `ci.prod.yml` defines a production profile.

Let's examine the key differences from the default `ci.yml`:

#### The Managed PostgreSQL Service

```yaml
run:
  postgres:
    provider:
      name: postgres
      version: v1
    plan:
      id: 0
    config:
      version: "17.6"
      userName: keycloak
      databaseName: keycloak
    secrets:
      userPassword: ${{ vault.KEYCLOAK_DB_PASSWORD }}
      superuserPassword: ${{ vault.KEYCLOAK_DB_PASSWORD }}
```

This declares a PostgreSQL Managed Service **as part of the landscape**. Let's break it down:

| Field | Purpose |
|---|---|
| `postgres:` | The name of the service within the landscape. Used in hostname resolution. |
| `provider.name` / `provider.version` | Which Managed Service provider to use. |
| `plan.id` | The resource plan (CPU, memory, storage). |
| `config.version` | PostgreSQL version. |
| `config.userName` / `config.databaseName` | The database and user to create automatically. |
| `secrets.userPassword` / `secrets.superuserPassword` | Passwords sourced from the [Codesphere Vault](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/secret-management). |

> **Important:** The service name (`postgres`) determines the hostname where the database is reachable. The hostname follows the pattern:
> ```
> ms-{providerName}-{providerVersion}-{teamId}-{serviceName}
> ```
> See [Service Discovery & Environment Variables](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/configuring-a-landscape#service-discovery--environment-variables).

#### Helm Command with Inline Values

```yaml
      - name: Deploy Keycloak to vCluster
        command: helm upgrade --install keycloak
          oci://ghcr.io/codecentric/helm-charts/keycloakx --namespace keycloak
          --create-namespace --values values.yaml --values values.prod.yaml
          --set-string extraEnvFromValues.adminPassword=$KEYCLOAK_ADMIN_PASSWORD
          --set-string extraEnvFromValues.hostname=$WORKSPACE_DEV_DOMAIN
          --set database.hostname=ms-postgres-v1-$TEAM_ID-postgres
          --set-string database.password=$KEYCLOAK_DB_PASSWORD --wait --timeout 10m
```

Notice the important details:
1. **Two values files** are layered: `values.yaml` (base) + `values.prod.yaml` (production overrides). Helm merges them in order.
2. **`--set database.hostname=...`** passes the database hostname via the command line. The `$TEAM_ID` variable is mapped from `${{ team.id }}` in the service's `env` block.
3. **`--set-string database.password=...`** passes the database password from the `$KEYCLOAK_DB_PASSWORD` environment variable, which is mapped from the vault.

#### Production Helm Values (`values.prod.yaml`)

```yaml
database:
  vendor: postgres
  port: 5432
  database: keycloak
  username: keycloak

dbchecker:
  enabled: true
```

This overrides the base configuration to:
- Connect to PostgreSQL instead of using the embedded H2
- The database password is passed directly via `--set-string database.password=...` in the Helm command
- Enable the database health checker — the Keycloak pod will wait for PostgreSQL to be ready before starting

### 4.3 Deploy the Production Profile

1. Add the required vault secrets:
   - `KEYCLOAK_ADMIN_PASSWORD` — the Keycloak admin password
   - `KEYCLOAK_DB_PASSWORD` — the PostgreSQL password
2. In the **CI & Deploy** view, select the **prod** CI Profile from the dropdown.
3. **Deploy the Landscape**.
4. Once deployed, click **Open Deployment** and log in.

> **Exercise:** After deployment, verify the managed service:
> - Go to the **Managed Services** tab in your team. You should see a new PostgreSQL instance that was provisioned as part of the landscape.
> - In the terminal, verify that Keycloak is connected to PostgreSQL:
>   ```bash
>   kubectl logs -n keycloak -l app.kubernetes.io/name=keycloakx | grep -i "database"
>   ```

### 4.4 Lifecycle

When you **tear down** the landscape, the Managed PostgreSQL service is automatically destroyed along with it — no orphaned databases, no leftover costs. This makes landscapes fully self-contained and reproducible.

If you re-deploy, a fresh PostgreSQL instance is provisioned automatically. See [Lifecycle Management](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/configuring-a-landscape#lifecycle-management).

---

## Summary

| Section | What you learned |
|---|---|
| **1. Deploy vCluster** | Provisioning a virtual Kubernetes cluster as a Managed Service |
| **2. Check Out vCluster** | Deploying Helm charts via a Reactive service in the `run` stage, `kubectl` access, vault secrets integration |
| **3. Expose a Service** | Bridging vCluster services to the Codesphere router via headless service networking |
| **4. Production Readiness** | CI Profiles, Platform Managed Services, layered Helm values, template variables |

**Further reading:**
- [Runtimes Overview](https://docs.codesphere.com/core-concepts/runtimes) — compare Reactives, Managed Containers, and Cloud Native Deployments
- [Environment Variables](https://docs.codesphere.com/workspace-toolkit/ci-and-deploy/environment-variables) — all available template variables (`${{ team.id }}`, `${{ vault.* }}`, etc.)
- [Managed Services Overview](https://docs.codesphere.com/managed-services/overview) — available service providers and capabilities
- [Connecting to Services](https://docs.codesphere.com/managed-services/connecting-to-services) — hostname patterns and connection methods
