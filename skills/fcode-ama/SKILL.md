---
name: fcode-ama
description: Factorial Code platform user journey and support — from requesting access through app creation, OAuth setup, demo companies and the Test marketplace to releases and marketplace publication. Use when answering "how do I…" questions about using the platform, guiding a user through any journey stage, or acting as a Factorial Code support agent.
license: MIT
metadata:
  category: factorial-code
---

# Factorial Code — platform guide (AMA)

Factorial Code (fcode) is Factorial's platform for building apps and
automations against the Factorial Public API. This skill covers **using the
platform** — the journey from requesting access to a published marketplace
app — and how to behave when supporting users through it. For *building* the
app itself (processes, modules, CLI, forms), route to the owning skills
(see the routing table below).

The full stage-by-stage walkthrough is in `references/journey.md`; common
problems and fixes are in `references/troubleshooting.md`.

## Who you're helping

Three user roles use the platform. Establish which one you're talking to
early — several flows diverge by role:

| Role | Who | Key differences |
|---|---|---|
| **Factorial internal** | Developers at Factorial | Create their own OAuth apps in the **Factorial Backoffice**; create demo companies directly (using the Demo generator); Slack `#factorial-code-users` available |
| **Partner** | External companies building apps/integrations with Factorial | OAuth app preconfigured by an administrator; demo companies by request |
| **Individual Contributor** | Someone building their own app or automation (may be a customer, but not always — never call this role "end customer") | Same as Partner |

All roles talk to Factorial through the **Public API**, and all belong to a
**development team**: every app a team creates is shared by all its members.
When someone requests access and an administrator sees their organization
already has a team on the platform, they are added to that existing team.

## Support-agent behavior rules

- **Answer only from documented flows** — this skill, its references, the
  other `fcode-*` skills, and the public docs. Never invent UI elements,
  URLs, commands, or policies. If it isn't covered, say so and escalate.
- **Identify the user's role first** when the answer differs by role.
- **Describe flows by page and section name** (e.g. "the Getting started
  checklist on the App detail page"), not by exact button pixels — labels
  change; flows are stable.
- **Admin actions: outcome only.** When a step is performed by an
  administrator (access approval, demo provisioning, OAuth preconfiguration,
  release review), tell the user what they will observe and what to do next.
  Never describe admin-side mechanics or how credentials are delivered.
- **Answer in the user's language.**
- **Fetch before deep-diving.** When you have web access, read the linked
  docs page before answering a detailed question; the live docs win over
  this skill on details. Without web access, answer from this skill and say
  which docs page has more.
- **Escalate when needed** — an admin action, a stuck request, or an
  uncovered question. Escalation paths are in the last section.

## Platform map

Everything lives under `https://code.factorialhr.com`, split across three
surfaces users often conflate:

- **Docs** (`/docs/...`) — public documentation, plus the landing page with
  the request-access form.
- **App console** (`/dashboard/...`) — apps, teams, demo companies,
  marketplaces, installations. Where the journey below happens.
- **Workspace console** (`/platform/...`) — inside a workspace: processes,
  executions, schedules, variables, versions, API credentials. The app
  console links into it — every such jump is labelled **"Open on platform"**;
  for a `deploy-` (installation) workspace it lands on the executions page.

App console areas (the sidebar groups them into **Build**, **Operate**, and
**Admin** sections):

| Area | What it's for |
|---|---|
| **Apps** | The team's apps; where a new app is created |
| **App detail page** | Per-app home: the Getting started checklist and the **Development / Production / Publication** tabs |
| **App settings** | **Configuration** (marketplace visibility, lifecycle) and **OAuth** (requested scopes plus the development and production credentials) |
| **Demo companies** | Demo Factorial companies for testing installs |
| **Test marketplace** | Simulation of the production marketplace, run against demo companies |

Plus the self-describing **Marketplace**, **Team**, and **Installations** areas.

## The journey at a glance

| # | Stage | Where | What happens |
|---|---|---|---|
| 1 | Request access | Landing page → request-access form | An administrator reviews it and you're notified; joins an existing development team when one matches |
| 2 | Create an App | Apps → create | Name and purpose; language (**JavaScript** or **Python**); **OAuth scopes**; the **integrations framework** opt-in (decision rule in `references/journey.md`) |
| 3 | Getting started | App detail page checklist | Ordered setup steps: build locally, link the Factorial integration (framework apps only), configure OAuth (when scopes were requested), publish a release, add marketplace metadata |
| 4 | Configure OAuth | App settings → OAuth tab | Per-environment client credentials for the OAuth flow (per-role setup in `references/journey.md`) |
| 5 | Build locally | Your machine | Install the CLI, `fcode clone` the dev workspace, code with your own agent + the `fcode-*` skills, test locally, `fcode push` |
| 6 | Demo company | Demo companies page | Internal: create directly. Partner/IC: request one; you're notified by email |
| 7 | Test installs | Test marketplace | Run the real OAuth flow with the demo company's credentials, install, exercise the `appRole` forms and any UI trigger buttons inside the demo company's Factorial |
| 8 | Release | App detail → Production tab | Request a release (semver + notes); after validation it's promoted to the prod workspace — self-service for a Factorial admin of the owning team, by an operator otherwise (`references/journey.md` §8) |
| 9 | Publish listing | App detail → Publication tab | Marketplace listing metadata: a linked DatoCMS record and/or the fallback fields |
| 10 | Production | Marketplace | Visible and installable; each install gets its own isolated workspace |

Per-stage detail, per-role callouts, and lifecycle states: `references/journey.md`.

## Workspaces behind the journey

Naming caution: a Factorial Code **workspace** is also called a "team" in the
workspace console, in the platform API, and in a few CLI surfaces ("team
variables", the `parentTeamSlugs` field) — that sense is unrelated to the
**development team** of humans described above.

Each app maps to workspaces (slugs carry an encoded token, not the raw id):

- `dev-{appId}` — the development workspace you clone and push to.
- `prod-{appId}` — production copy, created when the first release is
  requested. Treat its code as read-only — changes belong in dev and reach it
  through the release flow; only operators and the app's Factorial team
  admins can write to it at all. Its variables are live — shared defaults
  and credentials for all installations are managed there.
- `deploy-{installationId}` — one per company installation, inheriting from
  the app workspace: it holds only that company's variables and executions,
  fully isolated from other companies. A change in the parent workspace is
  automatically available in every installation workspace.
- `base-app` / `base-integration-app` (language-matched variants, `-js`/`-py`)
  — shared bases every app inherits: API clients, webhook/schedule/email/form
  helpers, and (for framework apps) the integration templates.

Released versions are pinned tags and the **`stable` alias** points at the
current one (model in `fcode-core-concepts`). For a marketplace app, ship
through the release flow (stage 8) — don't publish versions or move `stable`
by hand.

## Buttons inside Factorial — UI triggers

Besides forms and webhooks, an installed app can put **its own buttons on
Factorial's pages**. Factorial teams declare *locations* in the product (a page
header, an actions dropdown — e.g. `calendar.header.admin`); an app process
that declares a `uiTrigger` for that location shows up there as a button, with
the app's label and icon, for every company that installed the app. Clicking it
runs the process with the page's context (the record on screen, the company),
or opens the process's form when it has one — so a "Sync to Acme" action can
live next to Factorial's native actions without Factorial shipping any
integration-specific UI.

What to tell users:

- It works only for **installed marketplace apps** — the buttons are read from
  the company's `deploy-` workspace, so a trigger reaches customers through the
  release flow (stage 8) like any other change; in development, the demo company
  shows the dev workspace's triggers (stage 7).
- The **location id comes from Factorial**: there is no catalogue in the
  platform. A developer who wants a button on a page that has no location yet
  needs the owning Factorial team to add one — escalate as a product request.
- Some locations admit **one app at a time**; installing a second app that
  claims such a location fails with a message naming the first.
- Configuration lives on the process page (**Triggered from Factorial UI**) or
  in `metadata.json`; the developer-facing rules are in `fcode-ui-triggers`.

## Routing — where deep questions live

| Question is about | Route to |
|---|---|
| Platform model: processes, modules, variables & inheritance, datastore/storage, versioning & `stable` | `fcode-core-concepts` |
| CLI commands, `metadata.json` / `settings.json` fields, `appRole` field reference, webhook auth, `FACTORIAL_TOKEN` | `fcode-cli` — docs: `/docs/cli/` |
| Writing process/module code | `fcode-javascript` / `fcode-python` — docs: `/docs/processes/` |
| Input-parameter schemas (forms definition) | `fcode-json-schema` |
| Embedding forms on webpages, themes, pre-fill | `fcode-forms` — docs: `/docs/forms/` |
| Buttons inside Factorial (`uiTrigger`, locations, result envelope) | `fcode-ui-triggers` |
| Recommended agent working method, MCP tools | `fcode-agent` — docs: `/docs/mcp-server/` |
| Worked examples (integration, install/uninstall lifecycle) | `fcode-examples` |
| Executions, schedules, webhooks at runtime | docs: `/docs/executions/` |
| Integrations framework | docs: `/docs/building-apps/integrations-framework/` |
| OAuth & API tokens | docs: `/docs/building-apps/oauth/` |
| Workspace hierarchy | docs: `/docs/building-apps/workspaces/` |
| Roles & permissions: who can read/write/promote per workspace | `references/journey.md` §8 — docs: `/docs/building-apps/permissions/` |
| The end-to-end journey (public version) | docs: `/docs/building-apps/developer-journey/` |

Doc paths are relative to `https://code.factorialhr.com`.

## Escalation & admin actions

| Need | What to do |
|---|---|
| Access request pending | It's under review; you'll be notified of the outcome |
| OAuth app for a Partner / Individual Contributor | Preconfigured by an administrator — visible on the App settings' OAuth tab once done |
| Demo company (Partner / IC) | "Request a demo company" on the Demo companies page; you're emailed when it's ready |
| Release review outcome | The release either deploys or you're notified of required changes; fix and request again with a higher version |
| Promote a release to production | A Factorial admin of the owning team does it themselves (the App detail page's "How to promote" dialog has the commands); everyone else's route is an operator — `references/journey.md` §8 |
| Private app installed for a specific company | A production installation created from the App detail page — by an administrator **or by a developer** who provides the Factorial company ID (e.g. how Factorial's Forward Deployed Engineers roll out private apps) |
| Anything not covered | Internal Factorial users: Slack `#factorial-code-users`. Others: the in-platform request flows above |
