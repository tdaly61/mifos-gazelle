# Local Development Tools for Payment Hub EE

## Overview

`src/utils/localdev/` contains tools for iterating on Payment Hub EE Java components without rebuilding Docker images or pushing to a registry.

**The Problem:** Normal testing of Java changes requires: edit → build JAR → build Docker image → push to registry → redeploy. This is slow.

**The Solution:** `localdev.py` mounts your local project directory into the running pod via a `hostPath` volume. You rebuild the JAR locally and restart the pod — no Docker build required.

All PHEE components are **operator-managed**: deployed by the PaymentHub Kubernetes operator from `PaymentHubDeployment` CRs in `src/deployer/operators/paymenthub/config/cr/`. `localdev.py` patches live Kubernetes Deployments directly via `kubectl get/apply` (k8s-direct-mode).

> **Naming note:** the services now live in `openMF/paymenthub-ee-*` repos (some consolidated: `paymenthub-ee-data-pipeline` holds importer-rdbms, importer-es and zeebe-ops). Always verify Deployment names with `kubectl get deployments -n paymenthub`.

### Files

| File | Purpose |
|------|---------|
| `localdev.py` | Main script — patches Deployments and manages repo checkouts |
| `localdev.ini` | Configuration: which components to patch, your fork owner and branches |
| `fork-phee-repos.sh` | Forks the openMF `paymenthub-ee-*` repos to your account and sets up local clones (see [Working From Forks](#working-from-forks)) |

---

## Quick Start

> **Prerequisite:** `localdev.py` patches a component's *existing* Deployment — it can't patch what isn't there yet. PaymentHub must already be deployed to the cluster first:
> ```bash
> cd ~/mifos-gazelle
> ./run.sh -m deploy -a paymenthub
> ```
> Do this once per cluster (not before every `--setup`). If you see `namespaces "paymenthub" not found` or `deployments.apps "<name>" not found`, this step hasn't been done yet.

```bash
cd ~/mifos-gazelle/src/utils/localdev

# 1. Check current state
./localdev.py --status

# 2. Clone the repo / switch to your branch (branch_or_tag in localdev.ini)
./localdev.py --update --component importer-rdbms

# 3. Build the JAR (Java 21)
cd ~/paymenthub-ee-data-pipeline/importer-rdbms
./gradlew bootJar

# 4. Patch the deployment to run your JAR
cd ~/mifos-gazelle/src/utils/localdev
./localdev.py --setup --component importer-rdbms
# after each rebuild: kubectl delete pod -n paymenthub -l app=paymenthub-ee-importer-rdbms

# 5. When done, restore the original deployment
cd ~/mifos-gazelle/src/utils/localdev
./localdev.py --restore --component importer-rdbms
```

> **Java version:** the `paymenthub-ee-*` repos target Java 21 (`eclipse-temurin:21` at runtime). Check `java -version`, or set `JAVA_HOME` to a JDK 21, before building.

---

## How k8s-direct-mode Works

For components with `k8s_deploy_name` in `localdev.ini`, the patcher:

1. Runs `kubectl get deployment <name> -n <namespace> -o json` to fetch the live Deployment
2. Saves the original as a JSON backup at `~/.localdev-backups/<component>-deployment.json`
3. Modifies the Deployment spec:
   - Overrides the container image with the JDK image (e.g., `eclipse-temurin:21`)
   - Adds `command: ["java", "-jar", "<jarpath>"]`
   - Adds a `volumeMount` at `/app` in the container
   - Adds a `hostPath` volume pointing to your local project directory
4. Applies the patched manifest via `kubectl apply -f -`
5. Waits for rollout to complete

`--restore` re-applies the original JSON backup and deletes the backup file.

> **Operator reconciliation:** The in-cluster PaymentHub operator reconciles the 19 `PaymentHubDeployment`-managed components periodically and will revert k8s-direct patches to *those*. Scale it down while developing one of them:
> ```bash
> kubectl scale deployment paymenthub-ee-operator -n paymenthub --replicas=0
> # ... develop ...
> kubectl scale deployment paymenthub-ee-operator -n paymenthub --replicas=1
> ```
> This does **not** apply when patching the operator itself (`paymenthub-operator` in the table below) — its own Deployment is applied directly by `paymenthub.sh`, not reconciled by anything, so nothing reverts a patch to it.

---

## Command Reference

```bash
# Show status of all components (repo branch + patch state)
./localdev.py --status

# Complete setup: checkout repo + patch deployment
./localdev.py --setup
./localdev.py --setup --component importer-rdbms   # single component

# Clone missing repositories (components with checkout_enabled = true)
./localdev.py --checkout
./localdev.py --checkout --component channel

# Fetch, switch to branch_or_tag (created from upstream/<base_branch> if new),
# fast-forward. Never pushes, merges or rebases; won't switch over uncommitted changes.
./localdev.py --update
./localdev.py --update --component channel

# Preview what would be changed without modifying anything
./localdev.py --dry-run
./localdev.py --dry-run --component importer-rdbms

# Patch deployments only (no checkout)
./localdev.py
./localdev.py --component importer-rdbms

# Restore all deployments from backups
./localdev.py --restore
./localdev.py --restore --component importer-rdbms

# Use a custom config file
./localdev.py --config /path/to/custom.ini
```

---

## Build Loop

```bash
cd ~/paymenthub-ee-data-pipeline/importer-rdbms
# ... edit Java files ...
./gradlew bootJar          # Java 21
kubectl delete pod -n paymenthub -l app=paymenthub-ee-importer-rdbms
kubectl logs -f -n paymenthub -l app=paymenthub-ee-importer-rdbms
```

---

## Working From Forks

openMF branch protection blocks new branches on the `paymenthub-ee-*` repos, so
work happens in personal forks. One-off setup:

```bash
gh auth login                      # once; add `-s workflow` so fork syncs can include CI changes
src/utils/localdev/fork-phee-repos.sh            # prompts for the fork owner (default: your gh login)
src/utils/localdev/fork-phee-repos.sh -h         # options: owner/org, clone dir, repo subset, ssh/https
```

Each clone then has `origin` = your fork and `upstream` = openMF (HTTPS, push
URL set to `DISABLED`). localdev.py builds the same remotes from `[general]`
(`fork_owner`, `upstream_owner`, `git_protocol`); `LOCALDEV_FORK_OWNER` overrides
`fork_owner` without editing the ini.

Per ticket:

1. Set `branch_or_tag` in the component's section to your branch name
2. `./localdev.py --update --component <name>` — creates the branch from `upstream/dev` if it's new
3. Build (`./gradlew bootJar`), then `./localdev.py --setup --component <name>`
4. Commit; push to your fork and open the PR against openMF `dev`
5. `./localdev.py --restore --component <name>`

---

## localdev.ini Configuration

### Component Modes

| Fields present | Mode | Effect |
|---------------|------|--------|
| `k8s_deploy_name` | k8s-direct | Patches live Kubernetes Deployment + hostPath mount |
| `checkout_enabled` only | checkout-only | Clones/updates repo; no deployment patching |

The operator itself (`paymenthub-operator`) uses plain k8s-direct mode too — its Deployment is applied directly by `paymenthub.sh`, not by itself, so there's no reconciler to fight the patch.

### Configuration Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `[general].gazelle-home` | Yes | Root path to your mifos-gazelle clone |
| `[general].fork_owner` | No | GitHub user/org holding your forks: `auto` (default, your `gh` login), a name, or `none` to clone upstream directly |
| `[general].upstream_owner` | No | Upstream owner (default: `openMF`) |
| `[general].git_protocol` | No | `ssh` (default) or `https`, for `origin` |
| `checkout_to_dir` | No | Directory to clone into (default: `$HOME`; set in `[general]` or per section) |
| `base_branch` | No | Upstream branch new work branches are cut from (default: `dev`; `[general]` or per section) |
| `k8s_deploy_name` | k8s-direct | Kubernetes Deployment name (`kubectl get deployments -n paymenthub`) |
| `k8s_namespace` | k8s-direct | Namespace of the Deployment (default: `paymenthub`) |
| `app_type` | No | `springboot` (default) or `webapp` |
| `image` | springboot | JDK container image, e.g. `eclipse-temurin:21` |
| `hostpath` | No | Local path mounted at `/app` (default: `<checkout_to_dir>/<repo>[/<module>]`) |
| `jarpath` | No | Path to JAR inside the container. Default: the boot jar auto-detected in `<hostpath>/build/libs` (`*-plain.jar` ignored, newest wins) |
| `checkout_enabled` | No | `true` to enable automatic repo clone/update |
| `repo` | If checkout | Repo name, e.g. `paymenthub-ee-connector-channel`; URLs come from `[general]` |
| `reponame` | — | Full git URL instead of `repo`, for repos that aren't forked (no `upstream` remote) |
| `module` | No | Subdirectory of a multi-module repo that holds the service |
| `branch_or_tag` | No | Your working branch or a tag (default: `base_branch`) |

Sections sharing one repo (`importer-rdbms`/`zeebe-ops`/`importer-es`,
`mock-payment`/`integration-test`) share one working tree, so give them the same
`branch_or_tag`.

### Variable Expansion

- Environment variables: `$HOME`, `$USER`, etc.
- Custom variables: `${gazelle-home}` references the `[general]` section

### Adding k8s-direct patching to a checkout-only component

```ini
[my-component]
k8s_deploy_name = paymenthub-ee-my-component   # verify with: kubectl get deployments -n paymenthub
k8s_namespace   = paymenthub
app_type        = springboot
image           = eclipse-temurin:21
checkout_enabled = true
repo            = paymenthub-ee-my-component
branch_or_tag   = dev
```

---

## Configured Components

| Section | Repo (`module`) | k8s Deployment name |
|---------|-----------------|---------------------|
| `paymenthub-operator` | `paymenthub-ee-k8s-operators` (`paymenthub-operator`) | `paymenthub-ee-operator` |
| `channel` | `paymenthub-ee-connector-channel` | `paymenthub-ee-connector-channel` |
| `bulk-processor` | `paymenthub-ee-bulk-processor` | `paymenthub-ee-bulk-processor` |
| `connector-bulk` | `paymenthub-ee-connector-bulk` | `paymenthub-ee-connector-bulk` |
| `mock-payment` | `paymenthub-ee-e2e-tests` (`mock-payment-schema`) | `paymenthub-ee-connector-mock-payment-schema` |
| `operations-app` | `paymenthub-ee-bff` | `paymenthub-ee-bff` |
| `operationsui-react` | `paymenthub-ee-operationsui-react` (`dist`, webapp) | `paymenthub-ee-operationsui-react` |
| `importer-rdbms` | `paymenthub-ee-data-pipeline` (`importer-rdbms`) | `paymenthub-ee-importer-rdbms` |
| `zeebe-ops` | `paymenthub-ee-data-pipeline` (`zeebe-ops`) | `paymenthub-ee-zeebe-ops` |
| `importer-es` | `paymenthub-ee-data-pipeline` (`importer-es`) | `paymenthub-ee-importer-es` (CR disabled by default) |
| `identity-account-mapper` | `paymenthub-ee-account-mapper` | `paymenthub-ee-account-mapper` |
| `ams-mifos` | `paymenthub-ee-connector-ams-mifosx` | `paymenthub-ee-connector-ams-mifosx` |
| `connector-mojaloop` | `paymenthub-ee-connector-mojaloop` | `paymenthub-ee-connector-mojaloop` |
| `notifications` | `paymenthub-ee-notifications` | `paymenthub-ee-notifications` |
| `vouchers` | `paymenthub-ee-vouchers` | `paymenthub-ee-vouchers` |
| `bill-pay` | `paymenthub-ee-p2g` (`p2g`) | `paymenthub-ee-p2g` |
| `connector-gsma` | `paymenthub-ee-connector-mm-gsma` | `paymenthub-ee-connector-mm-gsma` (CR disabled by default) |
| `integration-test` | `paymenthub-ee-e2e-tests` (`integration-test`) | — checkout only (test suite) |
| `auth`, `core`, `dpg-core`, `connector-ams-paygops`, `connector-crm`, `connector-mpesa`, `connector-slcb`, `operationsui-angular` | `paymenthub-ee-<section>` | — checkout only |
| `connector-mccbs` | `ph-ee-connector-mccbs` (`reponame`, not forked) | — separate operator |

---

## Running the Operator From a Local Build

The operator (`paymenthub-operator`) is set up in `localdev.ini` exactly like any other k8s-direct component — its own Deployment is applied directly by `paymenthub.sh`, not reconciled by anything, so there's no need to scale anything down first:

```bash
cd ~/mifos-gazelle/src/utils/localdev
./localdev.py --setup --component paymenthub-operator   # checkout + patch

cd ~/paymenthub-ee-k8s-operators/paymenthub-operator
./gradlew bootJar
kubectl delete pod -n paymenthub -l app=paymenthub-ee-operator   # pick up the new JAR

kubectl logs -f -n paymenthub -l app=paymenthub-ee-operator

# When done
cd ~/mifos-gazelle/src/utils/localdev
./localdev.py --restore --component paymenthub-operator
```

---

## Troubleshooting

### `namespaces "paymenthub" not found` / `deployments.apps "<name>" not found`
PaymentHub hasn't been deployed to this cluster yet — `--setup`/`--dry-run` patch an *existing* Deployment, they don't create one. Deploy it first (see the Quick Start prerequisite above):
```bash
cd ~/mifos-gazelle
./run.sh -m deploy -a paymenthub
```

### Changes not taking effect
```bash
# Verify the JAR was built
ls -lh ~/your-project/build/libs/

# Check hostPath is mounted in the running pod
kubectl get pod -n paymenthub -l app=your-component -o yaml | grep -A5 hostPath

# Check startup logs
kubectl logs -n paymenthub -l app=your-component
```

### Build fails with Java version error
```bash
# paymenthub-ee-* repos target Java 21
java -version
./gradlew bootJar
```

### Pod in CrashLoopBackOff after patching
```bash
kubectl describe pod -n paymenthub -l app=your-component
kubectl logs -n paymenthub -l app=your-component

# Verify the JAR exists inside the container
kubectl exec -n paymenthub deployment/your-component -- ls -la /app/build/libs/
```

Common causes:
- A forced `jarpath` that no longer matches the built jar — remove it to use auto-detection
- JAR not yet built — run `./gradlew bootJar` (localdev refuses to patch when `build/libs` has no boot jar)
- `hostpath` directory does not exist on the node

### Patch reverted by operator
The in-cluster PaymentHub operator reconciles Deployments and will overwrite patches. Scale it down:
```bash
kubectl scale deployment paymenthub-ee-operator -n paymenthub --replicas=0
```

### Restore fails
```bash
# Check backup exists
ls ~/.localdev-backups/

# Manual restore
kubectl apply -f ~/.localdev-backups/<component>-deployment.json
rm ~/.localdev-backups/<component>-deployment.json
```

### Configuration changes not taking effect
When using hostPath mounts, the Spring Boot JAR contains the compiled YAML configuration. Editing `application.yaml` alone is not sufficient — rebuild the JAR:
```bash
./gradlew bootJar
kubectl delete pod -n paymenthub -l app=your-component
```
ConfigMap changes do NOT take effect when using hostPath mounts — the in-JAR config takes precedence.

### Permission denied on hostPath
```bash
chmod 755 ~/your-project
chmod 644 ~/your-project/build/libs/*.jar
```

---

## Notes

- **Single-node only:** `hostPath` is node-specific. This tooling is designed for single-node k3s local development.
- **After operator reconcile:** k8s-direct patches may be overwritten. Scale down the operator or re-run `./localdev.py --component <name>` (run `--restore` first if backup already exists).
- **Remote debugging:** Add JVM debug flags to the command in the patched deployment, then `kubectl port-forward deployment/your-component 5005:5005 -n paymenthub`.
