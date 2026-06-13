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

## Run on deploy: ArgoCD PostSync hook

`demo/` wires the `hello-service-up` workflow to the `helm-guestbook` ArgoCD
Application as a [resource hook](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/),
so the smoke test fires every time the app is deployed or updated.

| File                                       | Purpose                                                                 |
|--------------------------------------------|-------------------------------------------------------------------------|
| `demo/application.yaml`                    | The `helm-guestbook` Application, with a second source for the hook     |
| `demo/hooks/hello-service-up-postsync.yaml`| `TestWorkflowExecution` PostSync hook — triggers the workflow (default)  |
| `demo/hello-service-up-postsync.job.yaml`  | Optional Job variant that **fails the sync** if the test fails          |

How it works:

- An ArgoCD hook is just a resource carrying `argocd.argoproj.io/hook: PostSync`
  that lives in the Application's manifest set. ArgoCD applies it after the app's
  own resources are healthy.
- The upstream `helm-guestbook` chart can't be edited, so the hook is attached
  via a **second source** on the Application. Set that source's `repoURL` to
  wherever you push this repo (and make sure ArgoCD can read it).
- Creating/updating a `TestWorkflowExecution` makes Testkube run the workflow.
  `hook-delete-policy: BeforeHookCreation` deletes the prior execution object
  before each sync recreates it, so the test re-runs on every sync.

Apply:

```sh
kubectl apply -f demo/application.yaml
```

### Gating the deploy on the test result

The `TestWorkflowExecution` hook is fire-and-forget: ArgoCD has no health check
for it, so the sync doesn't wait for or fail on the result. If you want a failed
smoke test to mark the deploy failed, use `demo/hello-service-up-postsync.job.yaml`
instead — ArgoCD waits on Jobs natively and `testkube run ... -f` exits non-zero
on failure. Point the second source's `path` at the directory holding that Job
(or move it into `hooks/` and remove the `TestWorkflowExecution`).
