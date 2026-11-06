# Contributing to service-provider-ocm

For the general contribution process (fork-and-pull workflow, commit requirements, DCO,
code of conduct, and more), see the
[OCM Contributing Guide](https://ocm.software/community/contributing/).

This document covers repository-specific setup and development workflows.

## Prerequisites

- **Go 1.27+**
- **[Task](https://taskfile.dev/) v3.x** - runs all build, test, and lint commands
- **Docker** - required for image builds and E2E tests
- **[Flux CLI](https://fluxcd.io/flux/installation/)** - required for E2E tests (`test/e2e/main_test.go` runs `flux install`)
- **kubectl** - cluster interaction
- **[Kind](https://kind.sigs.k8s.io/)** - local cluster provisioning for E2E tests
- **[ocpctl](https://github.com/openmcp-project/ocpctl)** - local openMCP environment (see below)

## Getting Started

```bash
# Clone with submodules (hack/common is the shared build toolchain)
git clone --recurse-submodules https://github.com/open-component-model/service-provider-ocm.git
cd service-provider-ocm

# If already cloned without submodules
git submodule update --init --recursive
```

## Project Structure

```text
.
├── api/v1alpha1/          # CRD type definitions (OCM, ProviderConfig)
├── api/crds/              # Generated CRD manifests
├── cmd/                   # Controller entrypoint
├── internal/controller/   # Reconciler implementations
├── test/e2e/              # End-to-end tests
├── hack/common/           # Shared build toolchain (git submodule -> openmcp-project/build)
├── Taskfile.yaml          # Project-specific task definitions
├── VERSION                # Release version (semver or semver-dev)
└── .github/workflows/     # CI/CD pipelines
```

## Common Tasks

```bash
# Generate code (deepcopy, CRDs, formatting)
task generate

# Run linters and validation
task validate

# Run unit tests
task test

# Build container image for local platform
task build:img:build

# Build image and run E2E tests
task test-e2e
```

## Development Workflow

For the general fork-and-pull workflow, commit signing requirements, and DCO sign-off, refer to
the [OCM Contributing Guide](https://ocm.software/community/contributing/). The steps below cover
the repo-specific workflow after you have your local branch ready.

1. Run `task generate` after modifying API types in `api/v1alpha1/`
2. Run `task validate` to ensure lint and vet pass
3. Run `task test` for unit tests
4. Run `task test-e2e` for full end-to-end validation (see below)
5. Submit a pull request - CI runs the same checks automatically

### Pull Request Requirements

Use the [pull request template](.github/pull_request_template.md). CI ([Validate Pull Request Content](.github/workflows/validate-pr-content.yaml), backed by [validate-pr-content.lib.yaml](https://github.com/openmcp-project/build/blob/main/.github/workflows/validate-pr-content.lib.yaml)) rejects PRs whose description lacks the `**What this PR does / why we need it**:` section. Fill in the **Release note** block as well; use `NONE` if no release note is needed.

~~~markdown
**What this PR does / why we need it**:

**Release note**:
```<category> <target_group>

```
~~~

Allowed values:
- `category`: `breaking|feature|bugfix|doc|other`
- `target_group`: `user|operator|developer|dependency`

For additional requirements (conventional commits, DCO, commit signing, squash merging), see
the [OCM Contributing Guide](https://ocm.software/community/contributing/).

### E2E Tests

The E2E tests use [sigs.k8s.io/e2e-framework](https://github.com/kubernetes-sigs/e2e-framework)
and:

1. Build a container image for the controller
2. Bootstrap a local kind cluster with the openMCP operator and cluster-provider-kind (via [openmcp-testing](https://github.com/openmcp-project/openmcp-testing)), then install Flux
3. Deploy the service provider and verify reconciliation

## Local Development with openMCP

To test the service provider in a full openMCP environment, use [ocpctl](https://github.com/openmcp-project/ocpctl). It creates a kind platform cluster named `<env>-platform` and deploys the openMCP operator, cluster-provider-kind, Flux and a default set of service providers (including a released `ocm` image) — see its [defaults](https://github.com/openmcp-project/ocpctl/blob/main/pkg/config/environment-defaults.yaml). A config file overrides entries by name; other default service providers cannot be removed and keep running.

### 1. Install ocpctl

```bash
go install github.com/openmcp-project/ocpctl@latest
```

### 2. Create the environment

```bash
ocpctl env apply ocm-dev
```

This creates the `ocm-dev-platform` kind cluster (needed for image loading in the next step).

### 3. Build and load the image

In this repo:

```bash
VERSION_OVERRIDE=local task build:img:build
# produces ghcr.io/open-component-model/images/service-provider-ocm:local-linux-<arch>  (arch: amd64 or arm64)
kind load docker-image ghcr.io/open-component-model/images/service-provider-ocm:local-linux-<arch> --name ocm-dev-platform
```

(`VERSION_OVERRIDE` is honored by `hack/common/get-version.sh`; `-linux-<arch>` suffix matches our `Taskfile.yaml` `build:img:build-test` tag pattern.)

### 4. Point the `ocm` service provider at the local image

```yaml
# env.yaml
apiVersion: ocpctl.open-control-plane.io/v1alpha1
kind: Environment
spec:
  serviceProviders:
    - name: ocm
      image: ghcr.io/open-component-model/images/service-provider-ocm:local-linux-<arch>
```

```bash
ocpctl env apply ocm-dev --config env.yaml
```

### 5. Access clusters

```bash
# List the openMCP Cluster names for the environment
ocpctl clusters list

# Merge a kubeconfig into $KUBECONFIG / ~/.kube/config
ocpctl clusters kubeconfig export --environment ocm-dev --name <cluster-name>

# Or write a standalone kubeconfig file
ocpctl clusters kubeconfig get --environment ocm-dev --name <cluster-name> > <cluster-name>.kubeconfig
```

### 6. Create the ProviderConfig

Apply on the platform cluster:

```bash
cat <<EOF > provider-config.yaml
apiVersion: ocm.services.open-control-plane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: ocm
spec:
  pollInterval: 1m
  versions:
    - version: v0.22.1
      chartVersion: "0.22.1"
      chartURL: oci://ghcr.io/open-component-model/kubernetes/controller/chart
EOF
kubectl apply -f provider-config.yaml
```

### 7. Create a ControlPlane

Apply on the onboarding cluster:

```yaml
apiVersion: core.open-control-plane.io/v2alpha1
kind: ControlPlane
metadata:
  name: test-mcp
  namespace: default
spec:
  iam: {}
```

Wait until `kubectl get controlplanes -n default` shows phase `Ready`.

### 8. Create the OCM instance

Apply on the onboarding cluster:

```bash
cat <<EOF > ocm-instance.yaml
apiVersion: ocm.services.open-control-plane.io/v1alpha1
kind: OCM
metadata:
  name: test-mcp        # must match the ControlPlane name (kubectl get controlplanes)
  namespace: default
spec:
  version: v0.22.1
EOF
kubectl apply -f ocm-instance.yaml
```

### 9. Verify

On the MCP cluster (pick it from `ocpctl clusters list`):

```bash
kubectl get pods -n ocm-k8s-toolkit-system
```

### 10. Tear down

```bash
ocpctl env delete ocm-dev
```

## Further Reading

- [Service Provider Development Guide](https://open-control-plane.io/developers/serviceprovider/service-providers) -
  design, development, testing, and deployment of openMCP service providers
- [General Controller Guidelines](https://open-control-plane.io/developers/general) -
  operation annotations, status reporting, event filtering
- [openMCP Documentation](https://open-control-plane.io/) -
  full platform documentation
- [openmcp-project/build](https://github.com/openmcp-project/build) -
  shared build toolchain (the `hack/common` submodule)
- [ocpctl](https://github.com/openmcp-project/ocpctl) -
  local development environment for the openMCP platform
