# Plugin catalog

Backstage's value comes from its plugin architecture: a small core (catalog, permissions,
scaffolder, search) that everything else extends. Over the life of this platform, the squad and I
have built **45+ custom plugins** — frontend, backend, and shared "common" packages — instead of
bending third-party plugins to fit needs they weren't designed for.

Each entry below describes **what the plugin does and why it exists**, not how — no code, no
endpoints beyond what's needed to explain the shape of the integration, no internal config keys.

## Catalog governance & data quality

- **Catalog error admin** *(frontend + backend module)* — surfaces every catalog entity whose
  discovery/ingestion is failing, with a one-click re-trigger. Before this existed, a broken
  `catalog-info.yaml` in some team's repo would silently vanish from the portal with no signal to
  anyone. This turned that into a visible, actionable queue.
- **Domain entity processors** *(backend catalog module)* — extends catalog ingestion with a
  custom entity kind for on-call scopes/services, wiring catalog annotations to PagerDuty and
  environment-scoped log filters at ingestion time rather than at render time.
- **Permission policy module** — the authorization layer deciding who can trigger a catalog
  refresh, edit a secret, or modify a broadcast notification, expressed as a Backstage permission
  policy rather than scattered `if (isAdmin)` checks.

## Golden paths & self-service

- **Scaffolder template actions & field extensions** — custom scaffolder actions (data-shape
  transforms for downstream CI inputs) plus four custom form field types: repository metadata
  with existing-repo search/autofill, monorepo project setup, a service-scaffolding form with
  toggles for tracing/metrics/auth/GraphQL/versioning that live-previews the resulting service
  URL, and a squad/tribe picker sourced straight from the catalog. These back the two production
  "create a new service" golden paths (a shared monorepo project and a backend service), so a new
  service starts with the platform's conventions already applied instead of a blank repo and a
  wiki page nobody reads.
- **Scorecard (frontend + backend)** — computes a configurable, weighted score per catalog entity
  against governance/documentation/observability checks and renders it as a maturity tier
  (Bronze → Platinum), plus a portal-wide leaderboard view. This is the mechanism that turns
  "golden path" from a suggestion into a measured, comparable signal squads can be held to.

## Observability & incident response

- **Incident context (frontend + BFF)** — the highlight of the observability surface: a
  real-time banner on any service page when PagerDuty reports an active incident, with a
  time-aligned Grafana deep link (±15 min padding around the incident window) and a link straight
  to the runbook. Built specifically to remove the two-to-three minutes an on-call engineer used
  to spend manually assembling that context while an incident was live.
- **GCP incident history** — a per-product timeline of Google Cloud platform incidents, filterable
  by date and product, with a link out to Google's own status page — closes the gap between "GCP
  is degraded" and "does that affect us."
- **GCP alerting (frontend + backend module)** — ingests Google Cloud Monitoring alerts as
  Backstage events (via webhook, since GCP doesn't expose a clean polling API for this) so alerts
  become catalog-linked signals instead of another inbox to watch.
- **GCP Cloud Logging viewer** — an embedded, entity-scoped log viewer added directly to a
  service's page, driven off a single project-id annotation, so "show me this service's logs"
  never requires leaving the portal or knowing the underlying GCP project name.
- **GCP Secret Manager UI** — a permission-gated view (and edit path) for a service's secrets,
  so secret rotation/inspection doesn't require direct cloud console access for people who only
  need to manage one service's secrets.
- **Sentry issues** — surfaces a service's open Sentry issues on its overview page.
- **SLO (frontend + backend)** — reads SLO definitions declared as catalog annotations, evaluates
  them live against Prometheus (compliance, status, error budget, and burn rate over a
  configurable window), and renders health as a traffic-light per entity. SLOs are treated as
  configuration owned by the service team, not a separate spreadsheet the platform team
  maintains on their behalf.
- **Healthcheck backend** — a lightweight liveness surface used by the platform's own operational
  tooling.

## Engineering intelligence & delivery metrics

- **GitHub backend** — ingests GitHub deployment and pull-request webhooks into a queryable store
  (with scheduled retention cleanup), powering both the deployment history widget and PR
  visibility/filtering (by owner, reviewer, comment volume) without hitting GitHub's API live on
  every page load.
- **Pull request & Deployment widgets** — the home-page and entity-page components built on top
  of the GitHub backend: a filtered, paginated view of a team's open PRs, and a deployment
  history feed per service.
- **Scrum health (frontend + backend)** — aggregates active-sprint data per squad from Jira into
  a single dashboard: completion rate, open-bug count, days remaining, and a red/yellow/green
  health signal — a standing "state of the sprint" view instead of a manual weekly roll-up.
- **Timesheet/effort reporting (frontend + backend)** — pulls Jira worklog data and pivots it into
  effort-allocation reporting, connecting day-to-day ticket work back to the initiatives it
  belongs to.
- **DevLake data views (frontend + backend)** — surfaces the metrics Apache DevLake aggregates
  from GitHub, Jira and PagerDuty (deployment frequency, lead time, and related DORA signals) as
  native portal graphs, so engineering-health data lives next to the service it's about.

## Notifications & communication

- **Notifications admin (frontend + backend module)** — an admin console on top of Backstage's
  notification system for creating, editing and targeting company-wide broadcasts (by severity
  and topic), plus oversight of the underlying notification feed — the "how do we tell every
  engineer about a maintenance window" tool.
- **On-call component** — renders on-call scope/service entities with their Kubernetes and
  environment-log bindings, the frontend half of the on-call entity model described above.
- **Analytics provider (Segment)** — a Backstage-native analytics API implementation that ships
  navigation and interaction events to Segment, with privacy considerations built in by design
  (IP addresses are not forwarded; a test mode exists so local development never pollutes
  production analytics).

## Shared foundations

- **Shared React component library** — a small internal design-system layer (tab systems, a
  generic paginated data table, and similar) used across the custom plugins above, so 45+ plugins
  built by different contributors over nearly two years still look and behave like one product
  rather than a plugin marketplace grab-bag.
- **Shared GCP backend service** — a common connection/service layer other GCP plugins depend on,
  so each GCP integration doesn't reinvent authentication and client setup.
- **Common permission types** — shared TypeScript types for the permission model, keeping
  frontend and backend permission checks in sync by construction rather than convention.

---

A few earlier plugins were superseded as the platform matured (an early GCP alerting prototype,
an early deployment-tracking plugin, an early PagerDuty integration) and were consolidated into
the modules listed above — normal churn for a platform two years into its life, not dead weight
left lying around.
