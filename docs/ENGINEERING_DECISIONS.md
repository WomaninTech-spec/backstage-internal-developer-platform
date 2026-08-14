# Engineering decisions

A redacted decision log — real trade-offs made over ~two years running this platform, stripped of
internal names, tickets, and infrastructure detail. Kept here because the reasoning is the
transferable part, not the specific tool names.

## Foundation: build vs. adopt

**Decision**: adopt Backstage (CNCF) as the IDP foundation rather than build a bespoke portal.

**Why**: the catalog data model and plugin architecture matched what a fast-growing, squad-based
engineering org needed — a place for ownership, docs and observability to live per-service — and
avoided the multi-quarter cost of building that primitive layer from scratch. The trade-off,
accepted going in, was inheriting Backstage's UI constraints and plugin conventions rather than
having a free hand.

**What it actually cost**: Backstage's UI kit lagged behind the version of the design system we
wanted to build with, which pushed a lot of "just show this data" tasks into full custom
frontend plugins earlier than planned. In hindsight, that cost bought us a coherent shared
component library (see [Plugins](PLUGINS.md#shared-foundations)) sooner than we'd have built one
otherwise — a cost that mostly paid for itself.

## Engineering metrics: aggregate, don't rebuild

**Decision**: adopt Apache DevLake to aggregate GitHub, Jira and PagerDuty data into DORA and
sprint-health metrics, instead of building a bespoke ETL pipeline.

**Why**: the hard part of engineering metrics is the connectors and the data model, not the
dashboard on top. Paying that cost once, upstream, and pointing our own plugins at the result let
the platform squad spend its time on portal-native surfacing (see Scrum Health and DevLake views
in the plugin catalog) instead of maintaining connector code.

## Security posture: VPN-gate the portal

**Decision**: require VPN connectivity to reach the portal.

**Why**: the catalog aggregates ownership data, deployment history, secrets-adjacent tooling and
incident context across the whole engineering org in one place — exactly the kind of
single-pane-of-glass that's valuable to engineers and equally valuable to an attacker. Network-
level gating was judged the highest-leverage control for the risk, ahead of (and in addition to)
application-level permissioning.

## Ownership model: catalog metadata lives with the code

**Decision**: every service repository owns its own catalog descriptor and technical
documentation source, including public/open-source repositories — the platform team doesn't
maintain a central registry of everyone else's metadata by hand.

**Why**: centrally-maintained metadata about someone else's service rots the moment that service
changes and nobody remembers to tell the platform team. Co-locating the descriptor with the code
means the team best positioned to keep it accurate — the owning team — is the one who can, by
construction, without a cross-team dependency on every metadata change.

## Production incident: connection-pool exhaustion under autoscaling

**What happened**: each plugin manages its own database connection pool. Under load, the
horizontal autoscaler would add pods to handle traffic — and because every added pod opened a
full set of per-plugin pools, the 3rd or 4th pod would hit the database's max-connection ceiling
and crash-loop instead of relieving load, defeating the purpose of autoscaling.

**Response**: raised the database's max-connection ceiling, tuned the autoscaler's CPU threshold
and baseline CPU request so pods scale out less eagerly, and capped the number of pools any
single plugin is allowed to open (with a documented, higher cap for the catalog plugin, which
legitimately needs more). A parallel attempt to consolidate every plugin onto one shared
connection pool by patching the framework's internals was tried and abandoned — it fought the
framework's own plugin-isolation model harder than the payoff justified.

**What it changed going forward**: connection-pool footprint is now a standing constraint every
new backend plugin is reviewed against, not a one-time fix — the platform squad's plugin review
checklist grew a line item directly because of this incident.

## Scope discipline: know when *not* to build the deeper integration

**What happened**: we wanted to embed a metrics dashboard directly in a service page via iframe.
Both the portal and the dashboard tool sit behind an SSO gateway. The blocker turned out to be
structural, not configurable: the identity provider's sign-in flow refuses to complete inside an
iframe for a user who isn't already authenticated, so the embed only ever "worked" for users with
a live session elsewhere — silently broken for everyone else.

**Decision**: rather than building a session-proxying layer to work around an identity provider
that doesn't want to be iframed — a real option, but disproportionate infrastructure investment
for what was, at the time, a nice-to-have — we shipped the lighter integration (a linked list of
dashboards per service, with click tracking to gauge real demand) and made the proxy a
conditional follow-up, gated on the click data actually justifying it.

**Why it's here**: the discipline to *not* build the more impressive-sounding version of an
integration, and instead ship the version that matches actual demonstrated demand, is as much a
platform-leadership call as any architecture decision above.

## Release process: promotion over rebuild-per-environment

**Decision**: moved to a promotion-based release model — build a single artifact once, then
promote that same artifact through environments — replacing an earlier rebuild-per-environment
flow.

**Why**: rebuilding per environment leaves a window where "what's in staging" and "what's in
production" aren't provably the same bits, only probably. Promotion closes that gap and is the
same principle applied elsewhere in this platform (see the config-as-code sync in
[Architecture](ARCHITECTURE.md)): make the thing that's supposed to be true, provably true.
