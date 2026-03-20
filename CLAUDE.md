# Provider Upjet GCP

Fork of [crossplane-contrib/provider-upjet-gcp](https://github.com/crossplane-contrib/provider-upjet-gcp) — an Upjet-based Crossplane provider for Google Cloud Platform.

## Repository Structure

- `config/` — Provider resource configuration (external names, references, custom behavior)
  - `config/namespaced/compute/config.go` — Namespaced compute resource configurators
  - `config/cluster/compute/config.go` — Cluster-scoped compute resource configurators
  - `config/externalname.go` — External name mappings for all resources
- `cmd/provider/` — Provider subpackage entrypoints (one per GCP service: compute, sql, iam, etc.)
- `apis/` — Generated CRD types (namespaced and cluster-scoped)
- `internal/controller/` — Generated controllers
- `internal/clients/gcp.go` — Terraform provider setup and credential handling
- `cluster/images/provider-gcp/` — Docker image and xpkg build configuration
- `.github/workflows/publish-compute-provider.yaml` — Builds and publishes just the compute provider to `ghcr.io/myprizepicks/provider-gcp-compute`

## Remotes

- `origin` — `git@github.com:myprizepicks/provider-upjet-gcp.git` (our fork)
- `upstream` — `git@github.com:crossplane-contrib/provider-upjet-gcp.git`

## Building

### Full build (all subpackages)
```bash
make build
```

### Single subpackage (e.g., compute)
```bash
# Just compile the binary
go build ./cmd/provider/compute/

# Build binary + Docker image + xpkg
make build-provider.compute
```

### Publish compute provider to GHCR
Trigger via GitHub Actions:
```bash
gh workflow run publish-compute-provider.yaml \
  --ref <branch> \
  -R myprizepicks/provider-upjet-gcp \
  -f version=v2.4.0-fix.1
```
The workflow builds the binary, creates an xpkg, loads it into Docker, retags it, and pushes to `ghcr.io/myprizepicks/provider-gcp-compute:<version>`.

**Important**: The xpkg metadata embeds `XPKG_REG_ORGS` in the `dependsOn` field. When building with `XPKG_REG_ORGS=ghcr.io/myprizepicks`, the package will declare a dependency on `ghcr.io/myprizepicks/provider-family-gcp` which doesn't exist. Set `skipDependencyResolution: true` on the Provider resource when deploying to a cluster that has the family provider installed from a different registry.

## Custom Patches

### NetworkFirewallPolicyAssociation 400-as-not-found fix
**Files**: `config/namespaced/compute/config.go`, `config/cluster/compute/config.go`

The GCP Compute API's `getAssociation` endpoint returns HTTP 400 instead of 404 when a firewall policy association doesn't exist. The Terraform provider's `HandleNotFoundError` only checks for 404, so upjet treats the 400 as a reconciliation error and never proceeds to Create.

The fix wraps the Terraform `Read` function for both `google_compute_network_firewall_policy_association` and `google_compute_region_network_firewall_policy_association` to intercept 400 errors using `transport_tpg.IsGoogleApiErrorWithCode(err, 400)`, set the resource ID to empty, and return nil — signaling to upjet that the resource doesn't exist.

## Deploying to a Cluster

When deploying a custom-built compute provider image:

1. **Package pull secret**: The Provider needs `packagePullSecrets` referencing a secret named `ghcr` to pull from `ghcr.io/myprizepicks/`.
2. **Skip dependency resolution**: Set `skipDependencyResolution: true` since the xpkg's `dependsOn` points to our GHCR org for the family provider.
3. **RBAC**: The provider revision creates a new service account with a hash derived from the image digest (e.g., `provider-gcp-compute-e726e95d4b60`). Any custom ClusterRoleBindings that reference the SA by name need to be updated with the new hash.

## Deployment Config (app-ops)

The provider is deployed via kustomize overlays in `app-ops`:
- Base: `tooling/crossplane/providers/base/deployments/gcp/compute.yaml`
- Overlays: `tooling/crossplane/providers/{latest,preprod,prod}/kustomization.yaml`
- RBAC: `tooling/crossplane/providers/latest/rbac.yaml` — ClusterRoleBinding granting providers access to `ProviderConfig` resources. Contains hardcoded SA hashes that must be updated when provider images change.

PRs to app-ops should target the `multiverse` branch (not `main`).
