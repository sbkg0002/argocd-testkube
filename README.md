# Testkube — Ingress Health Tests

Two shell-script based [Testkube Test Workflows](https://docs.testkube.io/articles/test-workflows)
that check the health of an ingress:

| Workflow                | What it checks                                                        | Image                  |
|-------------------------|-----------------------------------------------------------------------|------------------------|
| `ingress-reachable`     | The external FQDN responds over HTTP(S) with an acceptable status     | `curlimages/curl`      |
| `ingress-cert-valid`    | The TLS cert verifies against the trust store, matches the host, and isn't expiring | `alpine/openssl` |
| `ingress-health-suite`  | Runs both of the above for one FQDN                                   | —                      |

## Prerequisites

- A Kubernetes cluster with [Testkube installed](https://docs.testkube.io/articles/install/overview)
  (the `testkube` namespace and Test Workflow CRDs must exist).
- The `testkube` CLI, or just `kubectl`.

## Install the workflows

```sh
kubectl apply -f workflows/
```

## Run

Override the `fqdn` config at run time — no need to edit the YAML:

```sh
# Reachability only
testkube run testworkflow ingress-reachable --config fqdn=app.example.com -f

# Certificate only (fail if it expires within 30 days)
testkube run testworkflow ingress-cert-valid \
  --config fqdn=app.example.com \
  --config minDaysValid=30 -f

# Both at once
testkube run testworkflow ingress-health-suite --config fqdn=app.example.com -f
```

`-f` streams the logs and makes the CLI exit non-zero if the test fails.

## Configurable parameters

### `ingress-reachable`
| Config           | Default                              | Notes                                  |
|------------------|--------------------------------------|----------------------------------------|
| `fqdn`           | `example.com`                        | Host to request                        |
| `scheme`         | `https`                              | `http` or `https`                      |
| `expectedStatus` | `200 201 204 301 302 401 403`        | Space-separated set of OK status codes |
| `timeout`        | `10`                                 | Connect/total timeout in seconds       |

### `ingress-cert-valid`
| Config         | Default | Notes                                            |
|----------------|---------|--------------------------------------------------|
| `fqdn`         | `example.com` | Host whose cert is validated (sent as SNI) |
| `port`         | `443`   | TLS port                                         |
| `minDaysValid` | `14`    | Fail if the cert expires within this many days   |
| `timeout`      | `10`    | Connect timeout in seconds                       |

## Scheduling (optional)

To run these continuously as health probes, add a cron trigger:

```sh
testkube create testworkflowexecution \
  --testworkflow ingress-health-suite \
  --schedule "*/15 * * * *" \
  --config fqdn=app.example.com
```

## Run on deploy: everything via ArgoCD

`demo/` runs the whole thing through ArgoCD with an **app-of-apps** — the
TestWorkflow definitions *and* the `helm-guestbook` deploy *and* the smoke-test
[resource hook](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/)
are all GitOps-managed. The test fires every time `helm-guestbook` is deployed
or updated.

| File                                          | Purpose                                                                  |
|-----------------------------------------------|--------------------------------------------------------------------------|
| `demo/root-app.yaml`                          | Root app-of-apps — the single thing you apply by hand to bootstrap       |
| `demo/applications/testkube-workflows.yaml`   | App that installs the TestWorkflow definitions (`workflows/`), wave 0    |
| `demo/applications/helm-guestbook.yaml`       | App for `helm-guestbook` + the PostSync hook, wave 1                     |
| `demo/hooks/hello-service-up-postsync.yaml`   | PostSync **Job** hook — runs the workflow and **fails the sync** if it fails |
| `demo/hello-service-up-postsync.exec.yaml`    | Non-gating `TestWorkflowExecution` alternative (reference, not synced)    |

How it works:

- The root app deploys the two child Applications. `argocd.argoproj.io/sync-wave`
  orders them: `testkube-workflows` (wave 0) becomes Healthy before
  `helm-guestbook` (wave 1) syncs, so `hello-service-up` already exists when the
  hook runs it.
- An ArgoCD hook is just a resource carrying `argocd.argoproj.io/hook: PostSync`
  that lives in the Application's manifest set, applied after the app's own
  resources are healthy. The upstream `helm-guestbook` chart can't be edited, so
  the hook is attached via a **second source** on that Application.
- The hook is a **Job** that runs `testkube run testworkflow hello-service-up -f`.
  ArgoCD natively tracks Job health and waits for it, so a non-zero exit (failed
  test) fails the PostSync hook and marks the whole **sync Failed**.
  `generateName` gives a fresh run per sync; `hook-delete-policy: HookSucceeded`
  cleans up passing runs and keeps failed ones for inspection.

Bootstrap (one command — everything else is reconciled by ArgoCD):

```sh
kubectl apply -f demo/root-app.yaml
```

> The Applications point at `https://github.com/sbkg0002/argocd-testkube.git`
> (repo root = this directory). Testkube itself (CRDs + controllers) must already
> be installed in the cluster.

### Talking to Testkube from the hook Job

The Job uses the `kubeshop/testkube-cli` image and reaches the self-hosted
Testkube API server in-cluster:

```yaml
env:
  - name: TESTKUBE_API_URI
    value: http://testkube-api-server.testkube.svc.cluster.local:8088
```

Adjust the service name / namespace / port to match your install if you changed
the Helm release name or namespace.

### Don't want the deploy to fail on a failed test?

Use the non-gating `demo/hello-service-up-postsync.exec.yaml` instead — a
declarative `TestWorkflowExecution` that triggers the workflow with no CLI or
token, but is fire-and-forget (ArgoCD has no health check for that CRD, so the
sync won't wait for or fail on the result). Move it into `hooks/` and remove the
Job.
