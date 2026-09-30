# OpenShift resources

Practical OpenShift deployment guidance for APIs, websites, PocketBase,
background workers, configuration, and secrets, including GU cluster access
commands and reusable YAML examples.

- [Deployment practices](docs/practices.md): networking, container images,
  secrets, storage, health checks, and environment separation.
- [Example manifests and setup](examples/README.md): adaptable YAML with explicit
  dependencies and placeholder values.

## The common deployment shape

```text
Browser -- HTTPS --> OpenShift Route -- HTTP --> Service --> Pods
                                                            |-- ConfigMap
                                                            |-- Secret
                                                            |-- PVC, if stateful

Worker ---------------- cluster Service DNS -----------------> API
```

Keep application manifests in a `deploy/` directory, with separate files for
each component. A Deployment manages the application's pods, a Service makes
them reachable inside the cluster, and a Route provides an external HTTP(S)
endpoint. A worker that only makes outbound requests does not need a Service
or Route.

| Need | Starting point |
| --- | --- |
| HTTP API with runtime configuration | [api.yaml](examples/api.yaml), [config.yaml](examples/config.yaml), [secrets.example.yaml](examples/secrets.example.yaml) |
| Public API endpoint | Add [api-route.yaml](examples/api-route.yaml) |
| Static website served on port 8080 | [web.yaml](examples/web.yaml) |
| PocketBase with persistent data | [pocketbase.yaml](examples/pocketbase.yaml) |
| Long-running background worker | [worker.yaml](examples/worker.yaml) |

The examples use `example-*` names and deliberately nonexistent image paths and
hosts. Adapt them before applying. Resource sizes and health endpoints are
starting values, not measurements from production.

## GU access and image registry

Log in to the GU cluster and select the OpenShift project for your application.
Replace `YOUR_PROJECT` with its namespace name.

```sh
oc login --username=gu-x-account --server=https://api.k8s.gu.se:6443
oc project YOUR_PROJECT
oc whoami
oc project -q
```

Log Docker into the GU registry using the current OpenShift token through stdin:

```sh
oc whoami -t | docker login registry.k8s.gu.se --username unused --password-stdin
```

Use `registry.k8s.gu.se/<project>/<image>:<version>` for images in the GU registry.
Desktop Docker login authenticates pushes; the workload's service account also
needs permission to pull its image.

## Build and release

Build and push a versioned image, update the manifest, then apply it. Choose a
build platform that matches the target nodes; the example uses `linux/amd64`,
including when building from an Apple Silicon Mac.

From the application repository, using its actual Dockerfile and build context:

```sh
PROJECT=YOUR_PROJECT
IMAGE_VERSION=0.1.0
docker buildx build --platform linux/amd64 \
  --tag "registry.k8s.gu.se/${PROJECT}/my-api:${IMAGE_VERSION}" \
  --push ./api
```

Update `image:` in the application manifest to that version. Prefer a new version
or commit tag for each release; reusing a tag and setting `imagePullPolicy: Always`
does not by itself change the Deployment's pod template or trigger a rollout.
See [Deployment updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).

After preparing the example files as described in [their guide](examples/README.md):

```sh
oc -n "$PROJECT" apply -f examples/config.yaml
oc -n "$PROJECT" apply -f examples/secrets.local.yaml
oc -n "$PROJECT" apply -f examples/api.yaml
oc -n "$PROJECT" rollout status deployment/example-api --timeout=180s
# Add public access only if this API needs it and DNS/TLS are configured.
oc -n "$PROJECT" apply -f examples/api-route.yaml
```

For a static Vite website, supply public build-time variables when building the
frontend image. Changing an environment variable on an already-built Nginx
container does not rewrite its JavaScript bundle. See the
[website practices](docs/practices.md#websites-and-container-images).

## Inspect a deployment

These commands use the API example's names; substitute your application names.

```sh
oc -n "$PROJECT" get deployments,pods,services,routes,pvc
oc -n "$PROJECT" describe deployment/example-api
oc -n "$PROJECT" logs deployment/example-api --tail=100
oc -n "$PROJECT" get events --sort-by=.metadata.creationTimestamp
oc -n "$PROJECT" get endpointslices -l kubernetes.io/service-name=example-api
oc -n "$PROJECT" port-forward service/example-api 8080:8080
```

While port forwarding, check `http://localhost:8080/readyz` in another terminal.
For a public deployment, also check its HTTPS hostname and HTTP-to-HTTPS redirect.
If pods fail to start, inspect a failing pod with `oc describe pod/<pod-name>`;
image pull errors, missing Secret keys, probe failures, and unbound PVCs each need
a different fix.

Secret and ConfigMap values injected as environment variables are read at
container startup. After changing those values, restart the consuming Deployment
and watch the rollout:

```sh
oc -n "$PROJECT" rollout restart deployment/example-api
oc -n "$PROJECT" rollout status deployment/example-api --timeout=180s
```

See the Kubernetes guidance for
[Secret updates](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
and [ConfigMap updates](https://kubernetes.io/docs/concepts/configuration/configmap/#mounted-configmaps-are-updated-automatically).
For a code rollback, restore the previous image in the manifest and apply it.
Database migrations and data changes need their own recovery procedure.
