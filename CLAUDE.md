# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a GitOps repository for deploying Red Hat OpenShift AI (RHOAI) and its dependencies using Kustomize. It provides a standardized, repeatable way to deploy the OpenShift AI stack on OpenShift clusters using either GitOps tools (ArgoCD, Flux) or `kubectl`/`oc` CLI.

## Key Architectural Concepts

### Layered Structure

The repository uses a **layered component-based architecture** with Kustomize:

1. **Components Layer** (`components/`): Base operator definitions that are reusable
   - Contains Kustomize Components (kind: Component)
   - Each operator has its own directory under `components/operators/`
   - Example: `components/operators/cert-manager/` contains namespace.yaml, operatorgroup.yaml, subscription.yaml

2. **Dependencies Layer** (`dependencies/`): Composition layer that references components
   - Contains Kustomize Kustomizations (kind: Kustomization)
   - References components using the `components:` field in kustomization.yaml
   - Can add patches and customizations on top of base components
   - Example: `dependencies/operators/cert-manager/kustomization.yaml` references `../../../components/operators/cert-manager/`

3. **Hierarchical Installation**: Each level includes its subdirectories
   - `dependencies/operators/kustomization.yaml` includes all operator directories
   - `dependencies/kustomization.yaml` includes `operators/`
   - This enables granular or full-stack installation

### Component vs Resource Pattern

**Critical**: Understand the distinction between Kustomize Components and Resources:
- **Components** (`components/operators/*/kustomization.yaml`): Use `kind: Component` and are referenced via the `components:` field
- **Dependencies** (`dependencies/operators/*/kustomization.yaml`): Use `kind: Kustomization` and reference components
- Components cannot be applied directly; they must be included in a Kustomization

### Namespace Handling

**Important**: Do NOT set namespace in kustomization.yaml files. Set namespaces directly as string values in the YAML manifests where needed (namespace.yaml, subscription.yaml, etc.).

## Common Commands

### Validation and Testing

The repository uses multi-layer validation to ensure manifests are correct:

```bash
# Run all validation checks (recommended before committing)
make validate-all

# Run essential validations only (fast, runs in CI)
make validate

# Run specific validation layers
make validate-yaml         # YAML syntax and formatting
make validate-kustomize    # Kustomize build validation
make validate-lint         # Best practices linting
make validate-security     # Security scanning

# Note: Schema validation runs in CI via kind cluster + kubectl dry-run

# Install all validation tools locally
make tools

# Build a specific kustomization (without applying)
kustomize build dependencies/operators/cert-manager
kustomize build dependencies

# Dry-run a specific folder
make dry-run FOLDER=dependencies/operators/cert-manager

# Server-side validation (requires cluster access)
kustomize build dependencies | kubectl apply --dry-run=server -f -
```

**Validation layers explained:**
1. **validate-yaml**: Checks YAML syntax, indentation, line length
2. **validate-kustomize**: Ensures all kustomization builds succeed
3. **validate-lint**: Checks best practices (non-blocking)
4. **validate-security**: Scans for security issues (fails on CRITICAL)
5. **Schema validation**: Runs in CI via kind cluster + kubectl dry-run (most authoritative)

### Installation

```bash
# Install all dependencies
make apply FOLDER=dependencies
# or
kubectl apply -k dependencies

# Install specific operator
make apply FOLDER=dependencies/operators/cert-manager
# or
kubectl apply -k dependencies/operators/cert-manager
```

### Removal

```bash
# Remove specific dependency
make remove FOLDER=dependencies/operators/cert-manager

# Remove all dependencies (uses cleanup script)
make remove-all-dependencies
```

### Tools

```bash
# Install all validation tools
make tools

# Install specific tool
make kustomize
make kube-linter
make trivy

# Clean local binaries
make clean
```

## Adding a New Dependency Operator

Follow this exact sequence:

1. **Create Component Directory**:
   ```bash
   mkdir -p components/operators/your-operator
   ```

2. **Create Component Manifests**:
   - Create required YAML files (namespace.yaml, operatorgroup.yaml, subscription.yaml, etc.)
   - Create `kustomization.yaml` with `kind: Component`
   - Set namespace as string in manifests, NOT in kustomization.yaml

3. **Create Dependency Directory**:
   ```bash
   mkdir -p dependencies/operators/your-operator
   ```

4. **Create Dependency Kustomization**:
   - Create `kustomization.yaml` with `kind: Kustomization`
   - Reference the component: `components: ["../../../components/operators/your-operator"]`
   - Add any needed patches
   - If the operator depends on other operators, include them in the `components` list

5. **Update Parent Kustomization**:
   - Add `- your-operator/` to `dependencies/operators/kustomization.yaml` resources list

6. **Document**:
   - Update README.md Dependencies table
   - Include operator purpose, namespace, and what requires it

7. **Add CRDs for Validation** (if needed):
   - If operator uses custom CRDs, add them to `.github/crds/operators/install.sh`
   - This allows kind-based validation in CI
   - See `.github/crds/README.md` for details

8. **Test**:
   - Run `make validate-all` locally
   - Ensure GitHub Actions validation passes
   - Test on real cluster with `make apply FOLDER=dependencies/operators/your-operator`

## Branch Strategy

- **No formal releases**: Users fork/clone and customize
- **Branch per RHOAI version**: e.g., `rhoai-3.0`, `rhoai-3.1`
- **Always develop against appropriate version branch**

## Contributing Standards

### Commit Message Format

Follow conventional commits:
- **Type**: `fix`, `feat`, `docs`, `chore`
- All commits except `chore` require associated Jira issue link
- Format: `type(scope): description`

### Pull Request Process

1. Fork repository and create feature branch off `main`
2. Test changes: `make validate` + cluster validation
3. Open PR against `main` with:
   - Jira issue link
   - Detailed description
   - Testing steps for reviewers
4. Follow Kubernetes review process

### Testing Requirements

Before submitting PR:
1. `make validate-all` must pass locally
2. GitHub Actions validation workflow must pass (automatically runs on PR)
3. For significant changes, validate installation on real OpenShift cluster

**Validation pipeline:**
- **Client-side**: YAML lint, kustomize build, schema validation, security scan
- **kind cluster**: Server-side dry-run with OLM CRDs installed
- **Manual**: Real OpenShift cluster testing (as needed)

## CRD Management for Validation

The repository includes CRD management for validation in kind clusters:

**Location**: `crds/`

**Structure**:
- `olm/install.sh` - Installs OLM CRDs (Subscription, OperatorGroup)
- `operators/install.sh` - Installs operator-specific CRDs
- `openshift/install.sh` - Installs OpenShift-specific CRDs
- `README.md` - Comprehensive CRD management guide

**When to add CRDs:**
- Validation fails with "no matches for kind YourResource"
- Operator uses custom resources requiring schema validation
- Adding OpenShift-specific resources

**See [crds/README.md](crds/README.md) for detailed instructions on adding new operator CRDs.**

## Repository Structure Reference

```
.
├── .github/
│   └── workflows/       # GitHub Actions workflows
│       └── testing.yaml
├── crds/                # CRD installation scripts for validation
│   ├── olm/
│   ├── operators/
│   ├── openshift/
│   └── README.md        # CRD management guide
├── components/          # Reusable base components (kind: Component)
│   └── operators/       # Operator base definitions
│       └── cert-manager/
├── dependencies/        # Composition layer (kind: Kustomization)
│   └── operators/       # Operator deployments that reference components
│       ├── cert-manager/
│       └── kustomization.yaml
├── scripts/             # Helper scripts (e.g., remove-deps.sh)
├── .yamllint.yaml       # YAML linting configuration
├── .kube-linter.yaml    # Best practices linting configuration
└── Makefile             # Validation, build, and deployment targets
```

## Version Requirements

- OpenShift: 4.19 or later
- Kustomize: v5 or later (v5.8.0 in Makefile)
- kubectl or oc CLI
- Cluster admin permissions for installation
