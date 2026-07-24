# LeptoStack Cluster Template

A template FluxCD repository for configuring and deploying [LeptoStack](https://github.com/leptostack-fluxcd/leptostack-base) on Kubernetes clusters. Fork or clone this repository to create a dedicated deployment repository per cluster, then bootstrap FluxCD from it.

## Repository Structure

```
.
├── kustomization.yaml       # Root Kustomization — controls which resources are applied
├── leptostack-config.yaml   # Cluster-specific configuration (domain, IPs, SMTP, etc.)
├── cluster-config.yaml      # Cluster infrastructure resources (MetalLB, GitRepository source)
├── infrastructure.yaml      # Infrastructure components (cert-manager, trust-manager, external-secrets, crossplane, operators)
├── leptostack-base.yaml     # Core LeptoStack services and their deployment order
├── leptostack-modules.yaml  # Optional application modules (disabled by default)
└── clusters/
    └── leptostack/
        └── flux-system/     # FluxCD system manifests (auto-generated during bootstrap)
```

## Getting Started

### 1. Create a Cluster Repository

Fork or clone this template into a new repository dedicated to your cluster:

```bash
git clone <this-template-url> my-cluster-fluxcd
cd my-cluster-fluxcd
git remote set-url origin <your-new-repo-url>
```

### 2. Configure Your Cluster

Edit `leptostack-config.yaml` to set your cluster-specific values:

| Variable | Description | Example |
|---|---|---|
| `cluster_name` | Unique name for this cluster | `production` |
| `leptostack_domain` | Base domain for all services | `leptostack.example.com` |
| `leptostack_name` | LeptoStack instance name | `portal` |
| `leptostack_metallb_subnet` | MetalLB IP address range | `10.0.0.100-10.0.0.110` |
| `leptostack_smtp_server` | SMTP server hostname | `smtp.example.com` |
| `leptostack_smtp_port` | SMTP server port | `587` |
| `leptostack_smtp_username` | SMTP username | `noreply@example.com` |
| `leptostack_smtp_password` | SMTP password | |
| `apisix_lb_ip` | APISIX load balancer IP | `10.0.0.100` |

Update the Ingress resource in the same file to match your domain and TLS configuration.

### 3. Enable Modules (Optional)

To deploy application modules on top of the base platform, uncomment the `leptostack-modules.yaml` line in `kustomization.yaml`:

```yaml
resources:
  - leptostack-config.yaml
  - cluster-config.yaml
  - infrastructure.yaml
  - leptostack-base.yaml
  - leptostack-modules.yaml  # Uncomment to enable LeptoStack modules
```

Then define your module sources and Kustomization resources in `leptostack-modules.yaml`.

### 4. Bootstrap FluxCD

Bootstrap FluxCD pointing to your cluster repository:

```bash
flux bootstrap git \
  --url=<your-cluster-repo-url> \
  --branch=main \
  --path=clusters/leptostack
```

Refer to the [FluxCD documentation](https://fluxcd.io/flux/installation/bootstrap/) for detailed bootstrap options.

### 5. Create the LeptoStack Base Secret

After bootstrapping, create a secret in the `flux-system` namespace that grants FluxCD read access to the `leptostack-base` repository. This is required by the `leptostack-base` GitRepository source defined in `cluster-config.yaml`.

**Using a personal access token (HTTPS):**

```bash
flux create secret git leptostack-base \
  --namespace=flux-system \
  --url=https://github.com/leptostack-fluxcd/leptostack-base.git \
  --username=<username> \
  --password=<personal-access-token>
```

**Using an SSH deploy key:**

```bash
flux create secret git leptostack-base \
  --namespace=flux-system \
  --url=ssh://git@github.com/leptostack-fluxcd/leptostack-base.git \
  --private-key-file=<path-to-private-key>
```

> **Note:** The token or key only needs read permissions on the `leptostack-base` repository.

### 6. Apply LeptoStack Resources

Once the secret is in place, trigger a reconciliation to deploy the LeptoStack resources:

```bash
flux reconcile kustomization flux-system --with-source
```

FluxCD will reconcile all Kustomization resources and begin deploying the infrastructure and LeptoStack services in the correct dependency order. You can monitor the progress with:

```bash
flux get kustomizations --watch
```

## Deployment Order

FluxCD manages dependencies between components. The deployment follows this order:

```
Infrastructure (cert-manager → trust-manager, external-secrets, crossplane, operators)
    │
    ▼
  base
    │
    ├──► valkey
    ├──► apisix → apisix-post-deploy
    ├──► centrifugo
    ├──► camel-k
    ├──► seaweedfs
    ├──► openbao → openbao-init → openbao-post-deploy
    │
    ├──► supabase-pre-deploy → supabase
    │         │
    │         ▼
    │    keycloak-pre-deploy → keycloak → keycloak-post-deploy
    │         │
    │         ├──► rabbitmq → flowable-pre-deploy → flowable
    │         └──► superset → superset-post-deploy
    │
    └──► (modules, if enabled)
```

## Key Files

- **`leptostack-config.yaml`** — The primary file to edit. Contains all cluster-specific configuration as a ConfigMap, consumed by all LeptoStack Kustomizations via `postBuild.substituteFrom`.
- **`cluster-config.yaml`** — Defines the MetalLB address pool and the `leptostack-base` GitRepository source. Update the MetalLB subnet to match your network.
- **`infrastructure.yaml`** — Deploys shared infrastructure operators from the `leptostack-base` repository. Generally does not need modification.
- **`leptostack-base.yaml`** — Defines all core LeptoStack service Kustomizations and their dependency chain. Modify only if you need to add or remove base services.
- **`leptostack-modules.yaml`** — Add custom module deployments here.
- **`kustomization.yaml`** — Root Kustomization that ties everything together.