# Project Context — SSP-Exchange GitOps Config Repo (read this first)

This is the **config repo** (GitOps desired state) for a self-built **SSP/Exchange simulator** running
on a learning **AWS EKS** cluster. Argo CD (in-cluster) watches this repo and reconciles its contents
into the cluster. **You do not deploy from here by hand — you change git, Argo CD makes it so.**

Read the behavioral contract at the bottom. It overrides the default instinct to be maximally helpful
by doing things *for* the human — this is a learning project; understanding is the goal, not speed.

---

## The five repos (know which one you're in)

| Repo | Owns | Notes |
|------|------|-------|
| **App repo** — `AlexanderMinko/ssp-ming-exchange` | Spring Boot app code, `Dockerfile`, CI workflow | On push, GitHub Actions builds the image (OIDC, no static keys) and pushes to **ECR** |
| **Infra repo** — the Terraform repo (`infra/` + `platform/`) | The EKS cluster, platform add-ons, and the Argo CD install itself | Two Terraform state layers; Terraform installs Argo CD, Argo CD manages workloads (not itself) |
| **Bidders repo** — `AlexanderMinko/ssp-ming-bidders` | Dummy DSP bidders the exchange calls in-cluster | Its chart lives in THIS repo: `charts/bidders/` |
| **Dashboards repo** — `ssp-ming-monitoring` (no remote yet) | Grafana dashboard JSON + a tiny chart (ConfigMaps for the sidecar) | Synced by `argocd/apps/grafana-dashboards.yaml` into `monitoring`; the chart lives THERE, not here |
| **This repo** (config) | Helm charts + per-env values + Argo CD `Application` manifests | The **only** thing Argo CD reads to decide what runs |

**Deploy flow = push + pull:**
```
push:  git push (app repo) → Actions builds image → ECR
pull:  image tag updated in THIS repo → Argo CD syncs → cluster
```
Actions never touches the cluster; Argo CD never touches app source. That separation is the point.

---

## Repo layout (this repo)

```
charts/
  exchange/            # the SSP/Exchange app: Deployment, Service, HPA, ConfigMap,
                       #   ExternalSecret, HTTPRoute + TargetGroupConfiguration, NOTES, _helpers
  mongodb/             # single-node MongoDB StatefulSet + headless Service + ExternalSecret
  bidders/             # dummy DSP bidders: Deployment (2 fixed replicas, no HPA), Service `bidders`,
                       #   ConfigMap, ExternalSecret, ServiceMonitor. In-cluster only — no HTTPRoute. Own DB `ssp-bidders`
                       #   on the shared mongodb; image from ECR repo eks-dev/bidders
  envs/
    dev/  exchange.yaml  mongodb.yaml     # per-env DELTAS only (layered over chart values.yaml)
    prod/ exchange.yaml  mongodb.yaml
argocd/                # Argo CD Application / App-of-Apps manifests (what to sync, from where)
```

- **Two INDEPENDENT charts, not a subchart dependency.** app↔DB is a *runtime* coupling (the app
  retries until Mongo answers), not a *deployment* one. Two releases into the **same** namespace.
- **Argo CD loads each chart's `values.yaml` ONLY** (`valueFiles: [values.yaml]`). CI's `bump-config` job writes
  `exchange.image.tag` there. The `envs/` overlays are NOT wired into Argo — manual `helm` runs only.
- **Observability stack = Start.io's** (decided 16/11/2026, replaces LGTM): **VictoriaMetrics** for metrics (`victoria-metrics-k8s-stack`: VM operator, `vmagent`, `vmsingle`, Grafana), **SigNoz on ClickHouse** for logs and traces (SigNoz `otel-agent` DaemonSet tails `/var/log/pods`; apps send traces over OTLP to the SigNoz collector). Nodes = 4 × `t3.large` (3 until 23/11/2026).
- **App metrics = PULL** (decided 13/11/2026; backend VictoriaMetrics since 16/11/2026): apps expose `/actuator/prometheus` (`micrometer-registry-prometheus`); `vmagent` scrapes them via a `ServiceMonitor` in each app chart (the VM operator converts it to a `VMServiceScrape`; locally: a scrape job in `ssp-ming-exchange/observability/prometheus.yml`, read by the Compose VictoriaMetrics single). OTLP metrics push stays in the code but off. Both apps export histogram buckets for `http.server.requests` (since 24/11/2026, `HTTP_SERVER_HISTOGRAM_ENABLED`, default `true`), so p99 comes from `histogram_quantile`. The buckets are Micrometer's generated ladder plus exact `slo` boundaries at 25, 50, 100, 200, 250, 300, 350 and 500 ms (`management.metrics.distribution.slo`; 250/300/350 ms are the auction deadlines), about 80 series per URI, method and status per pod. Why: `up` distinguishes dead from idle, `pod`/`namespace` labels come free, matches the KEDA query (Phase 15, PromQL against VictoriaMetrics).
- **Traces = 10% of requests, whole traces** (decided 21/11/2026 as every request, ssp-backend's design minus its sampling; sampling 0.1 on EKS since 23/11/2026): apps push OTLP/HTTP straight to `signoz-otel-collector.signoz:4318` (`OTLP_ENDPOINT`, no default in `application.yaml`; local value in `application-local.yaml`). `TracingConfig` in each app: a `Sampler` that keeps parent decisions and starts a trace only from a SERVER span (drops `@Scheduled` roots), and an `ObservationPredicate` that skips `/actuator/**`. Exchange → bidders `traceparent` is kept on purpose (unlike ssp-backend, our DSPs are ours). `TRACING_SAMPLING_PROBABILITY` is 0.1 in both charts (23/11/2026: at 1.0 and 500 rps, about 3,500 spans/s, ClickHouse fell behind and the SigNoz collector buffered 5.3 GB until it was evicted); the app default stays 1.0 for the laptop. The sampler is parent-based and the exchange's server span is every trace's root, so the exchange's value decides for the whole trace: 10% of auctions are traced completely, bidders included. The bidders' per-bid line (`BidEngine`'s logger, switched by `BID_EVENTS_LOG_LEVEL`) is off in the chart (`BID_EVENTS_LOG_LEVEL: OFF`); the exchange's `received adRequest` and `auction` lines stay (2 lines per auction since 26/11/2026, the first carrying the whole request, both unsampled). The `k8s-infra` agent parses the JSON lines and lifts MDC `traceId`/`spanId` into the log record (`trace_parser`), so logs link to traces. `TRACING_ENABLED` is `true` in both charts (flipped 22/11/2026, once SigNoz ran on EKS). `management.tracing.export.enabled` stays `true` always: Boot builds the W3C propagator only under it, so with it `false` no `traceparent` is read or sent and each service mints its own trace id (found 21/11/2026). `TRACING_ENABLED` (default `true`) maps to `management.tracing.export.otlp.enabled` and only switches sending spans.
- **Grafana dashboards = git** (decided 22/11/2026): repo `ssp-ming-monitoring` holds `dashboards/<folder>/<uid>.json`, the only source (neither Grafana persists anything). Local Compose Grafana bind-mounts it (file provider, UI edits allowed; `scripts/pull.sh <uid>` writes them back, or they die with the container). EKS: Argo CD app `grafana-dashboards` (helm repo `argocd/apps/`) renders the repo's chart into one ConfigMap per dashboard in `monitoring` (label `grafana_dashboard=1`, annotation `grafana_folder`), loaded read-only by the VM stack's Grafana sidecar (`folderAnnotation` + `foldersFromFilesStructure` set in `platform/06-monitoring.tf`). Datasource uid `VictoriaMetrics` in both Grafanas (the chart's uid; the local one changed from `victoriametrics`). `scripts/check.sh` lints.
- **Every location an app needs must be in its chart's `config:`** — app code has no defaults for hosts/URLs
  (see the umbrella `CLAUDE.md`), so a missing key crashes the pod at startup rather than calling `localhost`.
- **Per-env values layering:** chart `values.yaml` = baseline; `envs/<env>/<chart>.yaml` = deltas via
  `-f`. Merge rules: maps deep-merge, scalars replace, **lists replace wholesale** (an env
  `route.hostnames` overrides the baseline list entirely — it does not append).
- **Naming:** each chart sets `fullnameOverride` (`exchange`, `mongodb`) to pin resource names, so the
  Service/route/secret DNS names stay stable regardless of Argo CD's release name.

---

## Platform facts these charts ASSUME already exist (owned by the infra repo, not here)

Don't recreate these — the charts attach to them:
- **Cluster:** `eks-dev`, region **`eu-central-1`**, Kubernetes **1.34**, AL2023, nodes in private subnets.
- **Namespace:** apps deploy into **`production`** (pre-created by Terraform — do **not** use `--create-namespace`).
- **Ingress:** AWS Load Balancer Controller + **Gateway API in IP mode**. One shared **Gateway** `edge/shared`
  (= one ALB, `allowedRoutes: from: All`). Each app contributes only an **HTTPRoute** + **TargetGroupConfiguration**
  that attach to it — apps never create their own Gateway/ALB. TLS terminates at the ALB via ACM.
- **Storage:** `gp3` StorageClass (default, `reclaimPolicy: Delete`, `WaitForFirstConsumer`, `allowVolumeExpansion`).
- **Secrets:** External Secrets Operator + `ClusterSecretStore` **`aws-secretsmanager`** (auth via **Pod Identity**).
  Source secret `eks-dev/exchange/mongodb` is created **out-of-band** (never in git/Terraform state).
  ESO maps that one source → `mongodb-secret` (raw `MONGO_INITDB_ROOT_*`) and `exchange-secret` (remapped
  to `MONGO_USER`/`MONGO_PASSWORD`).
- **Registry:** ECR repo `eks-dev/exchange` (`140023370575.dkr.ecr.eu-central-1.amazonaws.com/eks-dev/exchange`, renamed from `eks-dev/app` 22/11/2026), **IMMUTABLE** tags.
- **Auth model:** **Pod Identity** everywhere for pod→AWS; **OIDC** only for GitHub Actions→AWS (it's CI, not a pod).

---

## Conventions & hard-won gotchas (violating these has already burned us)

- **NO secrets in git, ever.** Not in values, not in manifests. Secrets live only in AWS Secrets Manager
  and reach pods via ESO. Terraform state is plaintext in S3 — same reason.
- **Image tags must be immutable & unique for GitOps.** A mutable tag like `:main` breaks Argo CD change
  detection (same tag = no diff to sync) and conflicts with the IMMUTABLE ECR repo. Use SHA or semver.
  "What's committed in this repo" must equal "what's running."
- **Argo CD + HPA fight over `replicas`.** If HPA owns replicas, **omit `replicas`** from the Deployment and
  set the Argo CD Application to ignore diffs on that field — otherwise Argo CD reverts the HPA every sync.
- **Service creation is gated by the LBC webhook.** The `mservice.elbv2.k8s.aws` mutating webhook
  (`failurePolicy=Fail`, empty namespaceSelector) blocks ALL Service creation cluster-wide when the LB
  Controller isn't Ready. Anything creating a Service must be ordered after it.
- **Headless Service DNS = Ready endpoints only.** A `0/1` StatefulSet pod publishes no DNS record, so
  clients get "Name or service not known." Debug the pod's readiness, not DNS. (Mongo's `mongosh` readiness
  probe needs `timeoutSeconds` ≥ 5 — the k8s default of 1s is too short for mongosh cold-start.)
- **A ConfigMap change restarts nothing by itself** — `envFrom` is read only at container start. So every pod template
  that consumes a ConfigMap carries `checksum/config` (sha256 of the rendered `configmap.yaml`; exchange and bidders,
  decided 23/11/2026, Helm's "Automatically Roll Deployments" pattern): a `config:` change alters the pod template and
  Argo CD's sync rolls the pods. Add it to any new chart with a ConfigMap. It does NOT cover the ESO-written Secrets
  (Helm never renders their values): after a Secrets Manager change, `kubectl -n production rollout restart deploy/<app>`.
- **JVM = G1, heap = 60% of the memory limit, exit on OOM** (`JDK_JAVA_OPTIONS: "-XX:+UseG1GC -XX:MaxRAMPercentage=60
  -XX:+ExitOnOutOfMemoryError"` in each app chart's `config:`; 75% on 23/11/2026, 60% since 24/11/2026). The JVM default
  of 25% gave 260 MB (exchange) and 195 MB (bidders); the bidders hit `OutOfMemoryError` in the 22/11 EKS run and lived
  on with a dead Tomcat acceptor. 75% was too much: heap + ~120 MiB non-heap + native memory passed the bidders' 768Mi,
  and the kernel OOMKilled both pods mid-run (exit 137, 23/11); the exchange sat at ~96% of its 1Gi. A container
  OOMKill is not a Java `OutOfMemoryError`, so `ExitOnOutOfMemoryError` doesn't fire for it. Watch
  `container_memory_working_set_bytes` against the limit, not only `jvm_*`. With ZGC, go lower still (ssp-backend: 55%). **Always name the collector** (`-XX:+UseG1GC`, since 26/11/2026): below 1,792 MiB of
  container memory the JVM picks the single-threaded Serial GC (`Copy` / `MarkSweepCompact` in `jvm_gc_pause_seconds`),
  and on kind (25/11) its pauses reached 0.7–1.2 s. ZGC is a later experiment: it needs spare heap and CPU these pods lack. `JDK_JAVA_OPTIONS`, not `JAVA_TOOL_OPTIONS`: only the `java` launcher reads it, so `jcmd` in the pod
  doesn't inherit the flag.
- **App resources: CPU request, NO CPU limit; memory request == limit** (decided 23/11/2026, as ssp-backend's
  OPS-9881). A CPU limit only throttles (85–93% of periods during warm-up on 22/11); without one, pods use idle node
  CPU and still get their request under contention. Memory request == limit so the scheduler reserves the real use
  and the kubelet doesn't evict the pod first under memory pressure. No `-XX:ActiveProcessorCount` (ssp-backend pins it
  to the request): on 2-vCPU nodes it would cut the JVM to 1 CPU. Exchange 700m / 1Gi (1000m didn't fit: free CPU per node was fragmented, 150–790m), HPA 2–3 at 70%; bidders 2 × 500m / 768Mi.
  The node group is fixed (4 × `t3.large` since 23/11/2026, about 7.7 allocatable cores, no Karpenter): a pod that doesn't fit stays `Pending`.
- **Probes: slow ≠ dead** (decided 23/11/2026). Both app charts: a startup probe (30 × 5 s = 150 s), readiness
  timeout 2 s × 3 failures (out of rotation, no restart), liveness timeout 5 s × 6 failures × 10 s (restart only after
  ~60 s without an answer). The defaults (timeout 1 s, 3 failures) restarted pods 4 times under load in the 22/11 run.
- **Exchange rollout = remove first, then add** (`strategy.rollingUpdate` `maxSurge: 0`, `maxUnavailable: 1`, decided
  23/11/2026). The default adds a pod before removing one; on a full cluster that pod stayed `Pending`, the old pod was
  never removed, and two versions served traffic. Cost: one fewer pod during a rollout — don't roll during a load run.
- **Exchange never shares a node with ClickHouse** (required `podAntiAffinity` on `clickhouse.altinity.com/chi=signoz-clickhouse`
  in namespace `signoz`, decided 23/11/2026). On 23/11 ClickHouse used 1.2–2 cores next to an exchange pod on a 2-vCPU node.
  The exchange declares it, not ClickHouse: ClickHouse's EBS volume pins it to one AZ. If SigNoz renames its
  ClickHouseInstallation, this selector silently matches nothing — check it on SigNoz chart upgrades.
- **StatefulSet rolling updates deadlock on an already-not-Ready pod** — you must `kubectl delete pod` to
  force recreation from the new spec.
- **YAML style:** block style everywhere in first-party manifests (no inline `{ }` flow maps); leave
  vendored upstream CRDs untouched.
- **IAM is always scoped to specific ARNs, never `Resource: "*"`** (sole exception: `ecr:GetAuthorizationToken`,
  which has no resource scope).

---

## How to work with me (behavioral contract — this overrides "be maximally helpful")

I'm a senior Java backend engineer deliberately closing a platform/DevOps blind spot. Teach, don't just do.

1. **Explain before you generate.** Say what a thing is, what problem it solves, and how it connects to
   what exists — *then* show code. Never lead with a wall of YAML/HCL.
2. **One concept at a time.** Small steps I understand beat big steps I don't. Check I follow before moving on.
3. **Always answer "why," and pre-empt it.** Why this and not the alternative? State trade-offs in a line or two.
4. **Don't do my thinking for me.** When there's a decision, lay out the options and let me choose.
5. **Push back.** If I'm about to do something wrong, say so plainly. I want an honest peer, not a yes-man.
6. **No over-engineering.** This is a learning cluster, not prod. Flag "prod would do X, but for learning Y is fine."
7. **I run the commands, not you.** ⚠️ Do NOT run `terraform` (init/plan/apply/destroy), `helm install/upgrade`,
   `kubectl apply/delete/scale`, or any state-/resource-changing AWS CLI command. Write the code and hand me
   the exact command(s) — running them is how I learn. **Read-only inspection** (`kubectl get`, `aws ... describe`,
   `helm template`, `terraform validate`, version checks) is fine for you to run when verifying.
8. **Poke it.** After each component, suggest 1–3 ways to interrogate/break it from the admin's perspective, and
   end with "you understand X when you can predict Y." A step is done when I can predict its behavior, not when it exists.
9. **Be concise.** Density over volume — short bullets/tables, no filler, no recap paragraphs.

## Security & org rules (start.io)
- Never put secrets, PII, or record-level revenue data in git or into AI tools. If a secret appears, flag it,
  don't echo it, and treat it as needing rotation.
- AI-generated code/manifests need human review before merge or production — never suggest skipping it.
- Don't install skills/plugins/MCP servers without Org Admin/CISO approval.
- Defaults: metric units, DD/MM/YYYY dates.
