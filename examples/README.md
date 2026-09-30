# Example manifests

These examples implement the [deployment practices](../docs/practices.md).
They include complete Services, named port references, a PVC where needed, and
illustrative resource requests/limits. They are templates, not deployable demo
applications: this repository supplies no application images.

The seven YAML files were parsed with duplicate-key checks. Their ten Kubernetes
resources passed strict Kubernetes 1.34 schema validation; both OpenShift Routes
were checked for Service/port references and TLS settings. Selectors and
Secret/ConfigMap/PVC references were also checked. This offline validation does
not establish compatibility with the GU cluster's version, policy, or runtime.

## Choose the components

| Component | Apply these files | Application contract |
| --- | --- | --- |
| API | `config.yaml`, a populated `secrets.local.yaml`, `api.yaml` | Listen on `0.0.0.0:8080`; implement `/healthz` and `/readyz`; read the documented environment variables |
| Public API access | `api-route.yaml`, after the API | Supply an actual hostname, DNS, certificate, and application authentication |
| Static website | `web.yaml` | Serve the built site on 8080 with `/` returning HTTP 200 and SPA fallback if needed |
| PocketBase | `pocketbase.yaml` | Custom PocketBase image serving on 8080 and storing data under `/pb/pb_data`; baseline `/api/health` endpoint |
| Worker | `worker.yaml`, plus the API's ConfigMap/Secret | Entrypoint runs a continuous worker; reads `API_URL` and `UPSTREAM_API_KEY`; no inbound server required |

API, website, and PocketBase are separate choices. The generic API does not
automatically connect to PocketBase. The worker example points to the generic
API. Adapt those relationships to your application.

## Prepare the files

Copy the selected manifests into the application's `deploy/` directory, or edit
a working copy here. Before applying:

1. Replace every `example.invalid/...:REPLACE_ME` image with your versioned image.
2. Rename `example-*` resources and update their selectors and references together.
3. Set hostnames in Routes and public URLs in `config.yaml`. Arrange DNS and TLS
   for public Routes; enable the commented ACME annotation only if supported.
4. Match ports, environment variable names, health paths, and writable paths to
   the actual image. Build browser configuration into static assets when needed.
5. Choose resource sizes, PVC capacity, and a storage class for your namespace.
   The PVC example omits `storageClassName` to use the cluster default; specify
   one if no suitable default exists.

From this repository's root, create the local Secret file and replace both
`REPLACE_ME` values with real credentials using your editor:

```sh
umask 077
cp examples/secrets.example.yaml examples/secrets.local.yaml
```

`*.local.yaml` and `.secrets/` are ignored by this repository. When copying the
examples elsewhere, copy the corresponding ignore rules too. Also exclude them
from each Docker build context:

```dockerignore
.env
.env.*
**/*.local.yaml
**/*.local.yml
.secrets/
```

Only the placeholder template belongs in Git. Do not apply it unchanged, and do
not bulk-apply the `examples/` directory: it contains optional components and the
placeholder Secret as well as your local copy.

## Validate and apply

The commands below are for an operator with access to the intended cluster.
They have not been run against a cluster as part of creating these examples.
Set the namespace explicitly:

```sh
PROJECT=YOUR_PROJECT
oc project "$PROJECT"
oc -n "$PROJECT" apply --dry-run=server -f examples/config.yaml
oc -n "$PROJECT" apply --dry-run=server -f examples/secrets.local.yaml
oc -n "$PROJECT" apply --dry-run=server -f examples/api.yaml
```

A server dry run checks API/admission acceptance; it does not pull images, bind
storage, verify referenced Secret keys, or test health endpoints. Then apply the
prepared API in dependency order:

```sh
oc -n "$PROJECT" apply -f examples/config.yaml
oc -n "$PROJECT" apply -f examples/secrets.local.yaml
oc -n "$PROJECT" apply -f examples/api.yaml
oc -n "$PROJECT" rollout status deployment/example-api --timeout=180s
```

Apply each additional component only when needed:

```sh
oc -n "$PROJECT" apply -f examples/api-route.yaml
oc -n "$PROJECT" apply -f examples/web.yaml
oc -n "$PROJECT" rollout status deployment/example-web --timeout=180s
```

For PocketBase, apply `examples/pocketbase.yaml`, check that `example-pocketbase-data`
binds, and watch `deployment/example-pocketbase`. For the worker, apply
`examples/worker.yaml` and inspect its logs after rollout. It has no probe by
default: rollout success alone does not establish that it is processing jobs.

PocketBase has an internal Service but no public Route in this example. Use
`oc -n "$PROJECT" port-forward service/example-pocketbase 8090:8080` for local
administration, or deliberately add a Route with the appropriate authentication.

## Credentials supplied as a file

Some applications need a JSON credential or an Nginx password file instead of an
environment variable. Store the file in a Secret and mount it into the consuming
container.

For a JSON file stored locally at `.secrets/service-account.json`, create a
separate Secret without putting its contents on the command line:

```sh
oc -n "$PROJECT" create secret generic example-credentials \
  --from-file=service-account.json=.secrets/service-account.json
```

This command creates a new Secret; updating an existing one requires the team's
chosen secret update workflow. Add these fields to the consuming Deployment's
existing pod spec/container rather than replacing its other settings:

```yaml
# Under spec.template.spec:
volumes:
  - name: credentials
    secret:
      secretName: example-credentials
      items:
        - key: service-account.json
          path: service-account.json
# Under the consuming container:
volumeMounts:
  - name: credentials
    mountPath: /var/run/app-credentials
    readOnly: true
env:
  - name: GOOGLE_APPLICATION_CREDENTIALS
    value: /var/run/app-credentials/service-account.json
```

Use the environment variable expected by your application's library. Directory
mounts can receive updated Secret files, but the process still needs to reload
them. Avoid `subPath` if you expect automatic file updates. See
[Kubernetes Secret volumes](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod).

For Nginx basic auth, use a Secret key named `htpasswd`, mount its directory, and
point `auth_basic_user_file` at the mounted file. Reload or restart Nginx as needed
after rotation. A private `.htpasswd` file should not be copied into the image.
