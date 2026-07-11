# OCP DevSecOps Pipeline (Local, Apple Silicon)

A production-shaped DevSecOps pipeline running on a **single-node OpenShift cluster on a Mac** (Apple Silicon, 24 GB RAM), using OpenShift Local (CRC), Tekton Pipelines, Trivy, and Argo CD.

The pipeline does: **clone → security scan (gate) → build → push → deploy**, with GitOps-driven continuous delivery via Argo CD watching this repo.

---

## Architecture

```
Developer commit ─▶ Tekton Pipeline
                      ├─ git-clone        (fetch source)
                      ├─ trivy-scan       (SECURITY GATE — fails on CRITICAL CVEs)
                      ├─ prepare-dockerfile (write UBI9 Node.js Dockerfile)
                      ├─ buildah          (build + push image, tagged by commit SHA)
                      └─ deploy           (apply manifests)
                                │
                                ▼
                    Internal image registry
                                │
                                ▼
                    Argo CD  ◀── watches GitHub repo (gitops/ folder)
                                │  selfHeal + prune = cluster always matches git
                                ▼
                    OpenShift Deployment / Service / Route
```

**Lean-vs-production substitutions** (this runs on a laptop, so heavy tools are swapped for lightweight equivalents that teach the same concepts):

| Concept              | Production tool          | Used here                    |
|----------------------|--------------------------|------------------------------|
| Image/dep scanning   | Clair / RHACS            | Trivy (Tekton task)          |
| CD / GitOps          | Argo CD (full HA)        | OpenShift GitOps (Argo CD)   |
| Registry             | Quay (HA)                | Internal registry (Quay later)|
| Admission policy     | Gatekeeper / RHACS       | Kyverno (Phase 4)            |
| Image signing        | Cosign / Tekton Chains   | (Phase 4)                    |

---

## Prerequisites

- macOS on **Apple Silicon** (built/tested on M-series, 24 GB RAM)
- A free **Red Hat account** (for the OpenShift Local pull secret)
- **OpenShift Local (CRC)** — https://console.redhat.com/openshift/create/local
- **Homebrew**, **git**, and a **GitHub account** with a Personal Access Token (`repo` scope)
- `tkn` (Tekton CLI): `brew install tektoncd-cli`

> **RAM note:** On 24 GB, quit Docker Desktop and other heavy apps before running builds. The empty cluster idles around 9–10 GB; the Buildah step is the hungriest.

---

## Repository Layout

```
ocp-devsecops/
├── tasks/                     # Vendored Tekton catalog tasks (hub-independent)
│   ├── git-clone.yaml
│   ├── buildah.yaml
│   ├── trivy-scanner.yaml
│   └── openshift-client.yaml
├── pipelines/
│   ├── operator-subscription.yaml   # OpenShift Pipelines operator
│   ├── workspace-pvc.yaml           # Shared workspace PVC
│   ├── pipeline.yaml                # The build-scan-deploy pipeline
│   └── gitops-operator.yaml         # OpenShift GitOps (Argo CD) operator
├── app/
│   └── .trivyignore                 # Documented accepted-risk register
├── gitops/                    # Argo CD source of truth
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── route.yaml
│   └── application.yaml             # Argo CD Application
└── README.md
```

---

## Step-by-Step Setup

### 1. Start the OpenShift Local cluster

Tune resources for 24 GB before starting:

```bash
crc config set memory 14336        # 14 GB to the VM, ~10 GB left for macOS
crc config set cpus 6
crc config set disk-size 80
crc config set pull-secret-file ~/Downloads/pull-secret.txt

crc setup      # one-time; downloads the cluster bundle (~6 GB)
crc start      # ~5–15 min; prints the kubeadmin password — SAVE IT
```

Wire up the CLI and log in:

```bash
eval $(crc oc-env)
oc login -u developer https://api.crc.testing:6443
```

### 2. Install OpenShift Pipelines (Tekton)

Operator installs need cluster-admin:

```bash
oc login -u kubeadmin -p <kubeadmin-password> https://api.crc.testing:6443
oc apply -f pipelines/operator-subscription.yaml

# wait for the controllers
oc get pods -n openshift-pipelines -w
oc get csv -n openshift-operators | grep pipelines   # expect: Succeeded
```

### 3. Create the project and workspace

```bash
oc login -u developer https://api.crc.testing:6443
oc new-project devsecops-demo

# the operator auto-creates a 'pipeline' service account with registry push rights
oc get sa pipeline -n devsecops-demo

oc apply -f pipelines/workspace-pvc.yaml
oc get pvc devsecops-workspace          # expect: Bound
```

### 4. Vendor the Tekton tasks

> **Why vendored, not `tkn hub install`:** the public Tekton Hub (`hub.tekton.dev`) was **shut down on 8 Jan 2026**. We pull task YAMLs straight from the still-active `tektoncd/catalog` GitHub repo and commit them, so the pipeline never depends on a live external hub.

```bash
curl -sL https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml -o tasks/git-clone.yaml
curl -sL https://raw.githubusercontent.com/tektoncd/catalog/main/task/buildah/0.8/buildah.yaml -o tasks/buildah.yaml
curl -sL https://raw.githubusercontent.com/tektoncd/catalog/main/task/trivy-scanner/0.2/trivy-scanner.yaml -o tasks/trivy-scanner.yaml
curl -sL https://raw.githubusercontent.com/tektoncd/catalog/master/task/openshift-client/0.2/openshift-client.yaml -o tasks/openshift-client.yaml

oc apply -f tasks/
oc get tasks -n devsecops-demo    # git-clone, buildah, trivy-scanner, openshift-client
```

### 5. Grant the build service account privileged SCC

Buildah needs elevated privileges to build images. The default `restricted-v2` SCC denies this, so the build pod fails admission (`PodAdmissionFailed`). Grant `privileged` to the `pipeline` service account **scoped to this namespace only** (cluster-admin required):

```bash
oc login -u kubeadmin -p <kubeadmin-password> https://api.crc.testing:6443
oc adm policy add-scc-to-user privileged -z pipeline -n devsecops-demo
oc login -u developer https://api.crc.testing:6443
```

> **Security note:** `privileged` is a real tradeoff and a spot to tighten in a hardened cluster (rootless Buildah or a narrower custom SCC). Acceptable here for a local, single-namespace learning setup.

### 6. Apply and run the pipeline

```bash
oc apply -f pipelines/pipeline.yaml

tkn pipeline start build-scan-deploy \
  -w name=shared-workspace,claimName=devsecops-workspace \
  -s pipeline \
  --use-param-defaults \
  --showlog
```

Expected flow: `fetch-source` ✅ → `scan-source` ✅ (gate passes) → `prepare-dockerfile` ✅ → `build-image` ✅ (Buildah STEP 1/6…6/6, push with digest) → `deploy` ✅ (`successfully rolled out`).

Expose the app:

```bash
oc expose service/nodejs-ex -n devsecops-demo
oc get route nodejs-ex -n devsecops-demo
# → http://nodejs-ex-devsecops-demo.apps-crc.testing
```

### 7. Install Argo CD (OpenShift GitOps)

```bash
oc login -u kubeadmin -p <kubeadmin-password> https://api.crc.testing:6443
oc apply -f pipelines/gitops-operator.yaml
oc get pods -n openshift-gitops -w      # wait for all Argo components Running
```

### 8. Push this repo to GitHub

Create a **public** repo at https://github.com/new (don't initialize with a README), then:

```bash
git add .
git commit -m "DevSecOps pipeline + GitOps manifests"
git branch -M main
git remote add origin https://github.com/<your-username>/ocp-devsecops.git
git push -u origin main     # use a Personal Access Token as the password
```

### 9. Point Argo CD at the repo

Grant Argo permission to manage the app namespace, then create the Application:

```bash
oc adm policy add-role-to-user admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  -n devsecops-demo

oc apply -f gitops/application.yaml    # repoURL points at your gitops/ folder

oc get application nodejs-ex -n openshift-gitops \
  -o jsonpath='{.status.sync.status} / {.status.health.status}{"\n"}'
# → Synced / Healthy
```

Get the Argo CD UI and admin password:

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}'
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d && echo
```

**Verify the GitOps loop** (Argo restores anything you delete):

```bash
oc delete service nodejs-ex -n devsecops-demo
sleep 30
oc get service nodejs-ex -n devsecops-demo    # selfHeal recreated it
```

---

## Gotchas We Hit (and fixes)

These are OpenShift-on-Apple-Silicon specific and trip up most first-timers:

1. **Tekton Hub is dead** (shut down Jan 2026) → vendor task YAMLs from GitHub instead of `tkn hub install`.
2. **`mirror-registry` won't run on macOS** — it's a RHEL/Linux x86 tool. For local Quay, use Project Quay in Docker Desktop (Phase 3).
3. **`missing IMAGE_PATH` / `multiple targets`** — the Trivy task needs `IMAGE_PATH` set, and the target must appear **once** (not in both `IMAGE_PATH` and `ARGS`).
4. **`PodAdmissionFailed`** — Buildah needs the `privileged` SCC granted to the `pipeline` SA.
5. **Buildah storage driver** — set `STORAGE_DRIVER: vfs` on CRC (overlay-on-overlay fails).
6. **No Dockerfile in `nodejs-ex`** — it's a Source-to-Image repo; we write a UBI9 Dockerfile in a `prepare-dockerfile` step.
7. **`oc` crashes with `lfstack.push`** — the vendored `openshift-client` task image is amd64-only and crashes under ARM64 emulation. The deploy step uses a current multi-arch CLI image (`ose-cli`) instead.

---

## Current State

- ✅ OpenShift Local (CRC), OpenShift 4.22, Apple Silicon
- ✅ Tekton pipeline: clone → **Trivy gate** → build → push → deploy (all green)
- ✅ Tasks vendored in git (hub-independent)
- ✅ Argo CD installed, watching GitHub, `Synced / Healthy`
- ✅ Everything under version control

---

## Roadmap

- **Phase 2 (finish):** Close the GitOps loop — replace the pipeline's imperative `oc apply` with "update image SHA in `gitops/deployment.yaml`, commit, push," letting Argo do the deploy. Requires a GitHub PAT stored as a pipeline secret.
- **Phase 3:** Real **Quay** registry — Project Quay in Docker Desktop as the image target (robot accounts, Clair scanning).
- **Phase 4 (hardening):** **Cosign** image signing, **Kyverno** "no unsigned images" admission gate, per-CVE `.trivyignore` wired in properly, restore the HIGH-severity gate.

---

## Known Issues

- **App needs PostgreSQL:** the sample is `nodejs-rest-http-crud`, which expects a Postgres on `:5432`. The pod runs but logs `ECONNREFUSED :5432` until a database is provided. Deploy a Postgres, or swap to a DB-less sample, for a fully functional app.
- **Security gate is CRITICAL-only:** relaxed from `CRITICAL,HIGH` to let the sample's 6 HIGH npm CVEs through by policy. The accepted risks are recorded in `app/.trivyignore`. Re-tighten in Phase 4.

---

## Useful Commands

```bash
# Stop/start the cluster (state persists to disk)
crc stop
crc start

# Re-attach oc to a new shell
eval $(crc oc-env)

# Run the pipeline
tkn pipeline start build-scan-deploy \
  -w name=shared-workspace,claimName=devsecops-workspace \
  -s pipeline --use-param-defaults --showlog

# Clean up old pipeline runs
tkn pipelinerun delete --keep 1 -n devsecops-demo -f

# Force Argo to re-sync
oc annotate application nodejs-ex -n openshift-gitops \
  argocd.argoproj.io/refresh=hard --overwrite
```
