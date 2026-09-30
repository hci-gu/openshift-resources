# OpenShift deployment practices

Use this guide when preparing container images and manifests for an OpenShift
application. The [example manifests](../examples/README.md) provide starting
points for an API, static website, PocketBase backend, and background worker.

## APIs, Services, and Routes

Use an `apps/v1` Deployment to manage application pods. Keep pod labels,
Deployment selectors, and Service selectors consistent, for example with
`app: example-api`. Name the container and Service port `http` and point the Route
to that Service port. The app must bind to `0.0.0.0`, not only localhost.

Use cluster Service DNS for backend-to-backend calls. A browser needs a public
hostname or an application proxy; configure CORS when it calls an API on a
separate origin. Expose only the components that need public access through
`route.openshift.io/v1` Routes.

Edge TLS termination encrypts the client-to-router connection; the hop to the
backend is HTTP. Set `insecureEdgeTerminationPolicy: Redirect` to redirect HTTP
requests to HTTPS. TLS does not authenticate application users. Arrange DNS and
a certificate that covers the chosen hostname. The annotation
`kubernetes.io/tls-acme: "true"` requires a supporting controller on the target
cluster; the examples leave it commented out. See
[Red Hat's route documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/ingress_and_load_balancing/index).

Include every resource needed for a fresh installation. A Deployment alone does
not create a Service or Route, and commented-out YAML does not create resources.
Removing or commenting out a resource in a file also does not delete an existing
object when applying the remaining manifest.

## Websites and container images

For a static frontend, build the assets in a Node stage and serve them with Nginx
on port 8080. Use `try_files $uri $uri/ /index.html` for SPA deep links. A
server-rendered website needs its application runtime rather than just a static
Nginx image.

Use a multi-stage build, lockfile-based dependency installation, and versioned
application images. Choose an image compatible with OpenShift's assigned user
ID. An unprivileged Nginx image can simplify this; check that the process can
write to its cache, temporary, and PID directories. Make only necessary writable
directories accessible to the root group. Avoid hard-coding `runAsUser` values
from a particular namespace, since allowed UID ranges can differ. Image
permissions do not by themselves establish the permissions of a mounted volume.
See [Red Hat's image guidance](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/images/creating-images).

Vite's `VITE_*` values used by frontend code are public and replaced during the
build. A Secret injected into the serving Nginx pod cannot make a bundled value
private or change it after the build. Keep service credentials in the backend;
use public build variables or an explicitly implemented runtime-config endpoint
for browser configuration. See [Vite's environment documentation](https://vite.dev/guide/env-and-mode).

Use long-lived immutable caching only for assets with versioned/content-hashed
URLs. Keep `index.html` uncached or revalidated so clients can discover a new
release's assets.

For basic authentication, mount the Nginx password file from a Secret so
credential changes do not require embedding it into image layers. If basic auth
protects only the website, check the API's authentication separately.

## Configuration and secrets

Use explicit `env[].valueFrom.secretKeyRef` entries for session secrets, SMS
credentials, and database passwords. Use `configMapKeyRef` for non-secret
settings such as service URLs and feature flags. Keep credentials used to access
backend services on the server.

Commit a `secrets.example.yaml` containing key names and placeholders. Keep real
values in a locally ignored file such as `secrets.local.yaml`, or in the team's
secret management system, and apply them to the same namespace as the consuming
pod. `stringData` is convenient plaintext input; `data` is base64-encoded, not
encrypted. Do not commit either with real credentials. Limit who can read or
write Secrets. See [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

Keep secrets out of the Docker build context as well as Git. `.gitignore` does
not control `COPY . .`; add local secret files, `.env` files, and credential
directories to `.dockerignore` in each relevant build context. Excluding `.env*`
alone does not exclude credentials stored in YAML files.

Environment-based settings need a container restart to take effect. Mounted
Secret files are updated eventually, but the application must reload them;
`subPath` mounts do not receive those updates. The [examples guide](../examples/README.md)
shows both environment and file-based credentials. See
[Secret consumption and updates](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/).

## Persistent data and rollout strategy

Store database files and uploaded content on persistent storage when they must
survive pod replacement. Define or document each claim's capacity, storage class,
mount path, and backup/restore procedure. Keep independently managed data, such
as database storage and uploaded files, on separate volumes when appropriate.

The [PocketBase example](../examples/pocketbase.yaml) mounts a PVC at
`/pb/pb_data`. It uses one replica and `strategy.type: Recreate` to avoid
overlapping old/new processes during a Deployment upgrade. This causes downtime.
It is not database replication or a general guarantee against every possible
concurrent pod; see the
[Deployment strategy semantics](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#recreate-deployment).

`ReadWriteOnce` means writable from one node, not necessarily one pod. Select
access mode and storage backend deliberately. A PVC is persistence, not a backup;
do not delete it as part of a routine application rollout. The PocketBase example
includes a 1 GiB claim as an editable starting point. See
[persistent volume access modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes).

## Health checks and resources

Choose checks that match the application. Startup probes allow initialization;
readiness controls whether traffic reaches a pod; liveness should identify a
stuck process that restarting can repair. Avoid coupling liveness to an unrelated
remote dependency, which can cause restart loops during an outage. See
[Kubernetes probe behavior](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

The API example assumes `/healthz` and `/readyz`; implement or change those paths.
The PocketBase example's `/api/health` is only a baseline. Use an
application-specific readiness check when serving database-backed operations:
an HTTP health response alone may not establish that required data is accessible.
Allow enough startup time for migrations and other initialization.

The examples include illustrative resource requests and limits. Tune them to
measured load, startup memory, and namespace quotas; they are starting values,
not established GU sizing standards.

## Test environments and workers

Keep test data and test-only authentication separate from production. Use
separate Secrets for each environment and keep test-login flags out of production
manifests. Separate namespaces simplify resource naming and access control. If
sharing a namespace, update every name, selector, Service/Route target, PVC
reference, and Secret reference together. A `-test` Deployment name alone does
not isolate its credentials or data.

A continuously running worker fits a Deployment; finite or scheduled work may
instead need a Job or CronJob. Add a Service only if the process accepts inbound
traffic, and a Route only if it needs public HTTP(S) access. The
[worker example](../examples/worker.yaml) expects an image whose entrypoint runs
the worker and uses a cluster-local API URL.

## Deployment automation

Use a dedicated ServiceAccount for deployment automation. Define the required
namespace-scoped permissions in a Role and grant them with a RoleBinding; a Role
alone grants nobody access.

Tailor permissions to the workflow. It may need to manage Deployments, Services,
Routes, and ConfigMaps, and read pods and logs for rollout verification. Grant
Secret access and delete permissions only where the workflow requires them.
Document image publication, manifest updates, rollout verification, and recovery
steps so the release process is repeatable.
