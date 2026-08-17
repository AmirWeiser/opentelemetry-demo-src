# opentelemetry-demo-src

Application source for the [OpenTelemetry demo](https://opentelemetry.io/docs/demo/) — 19 microservices spanning 8+ languages — plus the CI/CD pipeline that builds, tests, and ships them. The application itself is upstream reference code; **the pipeline is the point of this repo.**

This is one of three repositories that make up the full project:

| Repo | Role |
|---|---|
| [opentelemetry-aws-infra](https://github.com/AmirWeiser/opentelemetry-aws-infra) | The infrastructure the app runs on |
| [opentelemetry-demo-gitops](https://github.com/AmirWeiser/opentelemetry-demo-gitops) | The Helm chart ArgoCD deploys |
| **opentelemetry-demo-src** (this repo) | Application source + CI pipeline |

## The services

| Service | Language/runtime |
|---|---|
| accounting | C# (.NET) |
| ad | Java (Gradle) |
| cart | C# (.NET) |
| checkout | Go |
| currency | C++ |
| email | Ruby |
| flagd-ui | TypeScript (Next.js) |
| fraud-detection | Java |
| frontend | TypeScript (Next.js) |
| frontend-proxy | Envoy |
| image-provider | Nginx |
| kafka | Java (custom KRaft-mode build) |
| load-generator | Python (Locust) |
| payment | JavaScript (Node.js) |
| product-catalog | Go |
| quote | PHP |
| recommendation | Python |
| shipping | Rust |

All services share a common gRPC contract defined in `pb/demo.proto`.

## CI/CD pipeline

`.github/workflows/ci-cd.yaml` is a single generic pipeline that handles all 18 buildable services without per-service duplication:

```
push to main
     |
detect-changes  --- dorny/paths-filter: which src/<service>/ dirs changed?
     |               (pb/** changing, or a manual "build_all" dispatch, rebuilds everything)
     v
   build         --- matrix over only the changed services
     |               reads .github/services.json for dockerfile/context/build-args
     |               runs the service's test command if one is defined
     |               docker build+push to ghcr.io/amirweiser/<service>:<sha>
     v
update-gitops    --- only runs if every matrix build succeeded
                     bumps that service's image.tag in the gitops repo's values.yaml
                     commits + pushes -> ArgoCD picks it up from there
```

**Why matrix + path-filter instead of one job per service:** adding a 19th language to this pipeline is a `services.json` entry and a path-filter line, not a new job definition. The matrix build step is entirely generic — it doesn't know or care what language `matrix.service` is, it just reads that service's Dockerfile path, build context, and optional build-args out of `.github/services.json`.

**`.github/services.json`** is the per-service build config — Dockerfile path, build context, build-args, and (where the folder name doesn't match the Helm values key, e.g. `fraud-detection` → `fraudDetection`) an explicit `valuesKey`. This is the single source of truth the pipeline reads from; adding a service means adding an entry here, nothing else.

**The `update-gitops` job is deliberately gated on `success()`.** A custom `if:` condition on a job replaces GitHub Actions' implicit "did everything upstream succeed" check — without spelling out `success()` explicitly, this job would still run (and bump image tags in the gitops repo) even if some matrix builds failed, pointing the cluster at images that were never pushed.

**Manual `build_all` dispatch** exists for cases the path-filter can't reason about well — e.g. bootstrapping every service's first image, or a base-image bump that doesn't show up as a `src/**` diff.

## Repo layout

```
.github/
├── workflows/ci-cd.yaml   # the pipeline described above
└── services.json           # per-service build config - single source of truth for CI
pb/
└── demo.proto               # shared gRPC contract, vendored (read-only) from the upstream reference
src/
└── <service>/                # one directory per service, its own language toolchain and Dockerfile
```

## Where this fits in the bigger picture

This repo never touches Kubernetes manifests directly. A successful build's only side effect is a commit to [opentelemetry-demo-gitops](https://github.com/AmirWeiser/opentelemetry-demo-gitops) bumping an image tag — from there, ArgoCD's auto-sync takes over. Promotion is fully automated end-to-end: merge to `main` here, and (assuming the build passes) the running cluster is updated with no manual step in between.
