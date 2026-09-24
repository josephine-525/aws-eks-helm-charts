# helm-charts

Shared, reusable Helm chart library for the expense-demo platform. Charts here are
generic and carry zero app-specific values — every consuming app (`expenseapp`,
`fraud-detection-demo`, `analytics-demo`, ...) supplies its own `values.yaml` from its
own repo. ArgoCD's multi-source Application feature is what stitches "chart lives here"
and "values live in the app's own repo" together at deploy time — see the `gitops` repo.

## Versioning — lockstep tags, and why that's a scale tradeoff, not a free lunch

This repo uses one repo-wide semver git tag (`v0.1.0`, `v0.2.0`, ...) covering every chart
in it, same manual-tagging convention as `terraform-modules` (plain `git tag`, no
automation). Consumers pin an exact tag via `targetRevision`, and — this is the part that
makes lockstep tolerable — **a consumer is never forced onto a newer tag just because one
exists.** If `common-worker` gets added later and the repo is tagged `v0.2.0`, everything
still pinned to `v0.1.0` for `common-web-service` stays exactly as it was; nothing about
that chart's content changed, so nothing forces a re-pin. `terraform-modules` already proved
this works in practice: `v1.1.0` added `guardduty`/`config` with zero changes to
`vpc`/`db`/`security`/`compute`, and every existing consumer of those stayed on `v1.0.0`
without needing to care.

**The honest tradeoff:** this is a project-scale choice, not a company-scale one. It works
here because there's realistically one team (me) touching a handful of charts. At real
company scale — many charts, many teams, very different change velocity per chart — lockstep
tagging breaks down in a specific way: the tag number stops meaning anything on its own
("v0.7.0" tells you nothing about which chart actually changed without reading the diff),
and a team bumping their own chart has to think about whether the *number* looks alarming to
people using unrelated charts in the same repo, even though nothing forces them to act on it.
The real fix at that scale is **per-chart prefixed tags** (`common-web-service-vX.Y.Z`,
`common-worker-vX.Y.Z`, ...) — fully independent version lines, changelogs, and release
history per chart, at the cost of one more thing to remember when tagging (don't forget the
prefix). Splitting each chart into its own repo (like `terraform-modules` already is,
separate from `expenseinfra`) is the other real option, trading "one clone gets you
everything" for "each chart's version history is completely unambiguous." Neither move is
warranted here yet — flagged so it's a deliberate choice, not a blind spot, if this ever
needs to look like a real platform team's setup.

## Charts

### `charts/common-web-service/` — built
Generic web service: Deployment + Service, with optional ServiceAccount (IRSA),
PodDisruptionBudget, HorizontalPodAutoscaler, Ingress, ConfigMap, ServiceMonitor, sidecar
containers, and scheduling controls (affinity/topologySpreadConstraints/tolerations).
Used by `expenseapp` (backend + frontend) and the two demo apps — proves one chart serves
multiple, independently-versioned workloads via values alone.

## Roadmap (not built — no current workload needs them)

A real platform team's chart library typically splits by workload shape, not just by app.
These are the natural next charts, listed here deliberately instead of scaffolded as empty
directories (an empty chart with no templates fails `helm lint` and just looks unfinished;
this list is the honest version of "we know what's next"):

- **`common-worker/`** — Deployment only, no Service. For background/queue-consumer workloads
  that don't receive traffic (nothing in this project needs one yet).
- **`common-cronjob/`** — `CronJob` for scheduled batch work (nothing scheduled in this
  project yet).
- **`common-statefulset/`** — `StatefulSet` + PVC, for anything needing stable identity or
  persistent storage (e.g. a self-hosted datastore). Note: this cluster currently has no EBS
  CSI driver installed, so a PVC-backed chart wouldn't actually bind until that's added —
  build the CSI driver piece first if this ever becomes real.
