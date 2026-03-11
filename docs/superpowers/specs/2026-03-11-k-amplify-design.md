# K Amplify — Design Spec

**Date:** 2026-03-11
**Owner:** Drew Kull
**Status:** Approved

## Overview

K Amplify is a unified internal website serving Amplify's partner orgs (Sales, Support, CS/Services, R&D, Marketing) and product/engineering leadership. It provides live roadmap visibility from Linear, a structured intake form, demo archives, a changelog, partner digest history, and the Amplify operating model — all in one place.

The site uses the campfire emoji (🔥) as the K Amplify logo.

## Audiences

Both audiences are first-class with different lenses into the same data:

- **Partners** (Sales VP, CS Ops lead, Support Eng lead, etc.) — filter to their org, see what Amplify is building for them, submit requests, catch up on demos
- **Leadership** (Drew, PMs, eng leads) — see the cross-cutting view across all 5 orgs, track health, spot cross-org tradeoffs

## Site Map

### / — Home Dashboard
- 5 org cards in a grid, each showing:
  - Org name, PM owner, goal
  - Health indicator (On Track / At Risk / Off Track) — live from Linear goal status
  - Next milestone with target date
  - Latest shipped item (most recent completed project)
  - Click to drill into that org's filtered view
- Recent activity feed: last 5-10 items across all orgs (shipped projects, new demos, status changes)
- **Data:** `/api/roadmap` — Linear goals + health + child projects/milestones. Cached 5 min.

### /roadmap
- Filter bar: All | Sales | Support | CS/Services | R&D | Marketing
- Grouped by org (All) or flat list (filtered)
- Each project card: name, health indicator, latest status update (truncated, expandable), milestone timeline, team
- Sorted: At Risk/Off Track float to top, then most recently updated
- Click to expand full status history and milestone details
- **Data:** `/api/roadmap` — active projects under Working AI First initiative. Cached 5 min.

### /demos
- Filter bar: All | Sales | Support | CS/Services | R&D | Marketing
- Reverse chronological, grouped by week
- Each entry: date, title (from Linear), Loom embed (inline playable), org tag, linked project
- Loom links extracted from Linear project updates and issue comments
- If a Loom isn't in Linear, it doesn't show — reinforces the habit
- **Data:** `/api/demos` — scans project updates and issue comments for Loom URLs. Cached 15 min.

### /shipped
- Filter bar: All | Sales | Support | CS/Services | R&D | Marketing
- Reverse chronological, grouped by month
- Auto-generated: any project that moves to Done/Completed appears automatically
- Each entry: project name, completion date, org tag, last status update as description, Loom link if exists
- No curation needed — Friday status updates already contain metric impact
- **Data:** `/api/shipped` — completed projects with most recent status update. Cached 15 min.

### /digests
- Filter bar: All | Sales | Support | CS/Services | R&D | Marketing
- Reverse chronological, grouped by month
- Each entry: month + org header, goal health indicator, full digest content from monthly goal status update
- Expandable/collapsible (collapsed by default)
- Replaces digging through Slack history; Slack post remains the notification mechanism
- **Data:** `/api/digests` — goal-level status updates from Linear. Cached 30 min.

### /intake
- Structured form with routing to Linear team triage
- Fields:
  1. Your name (text)
  2. Team (dropdown: Amplify Sales, Amplify Support, Amplify CS/Services, Amplify R&D, Amplify Marketing)
  3. Request type (dropdown: New capability, Enhancement to existing, Bug/issue, Data request, Escalation)
  4. Urgency (radio: Blocking us now, Need this quarter, Future idea)
  5. Description (textarea)
- Submit creates a Linear issue in the selected team's triage
  - Title: auto-generated from first line of description
  - Labels: request type + urgency
  - Assignee: none (PM triages at Monday PM Sync)
- Confirmation: "Request submitted to [team name]. Your PM will respond within one sprint."
- **Data:** `/api/intake` — POST, creates Linear issue. No caching.

### /how-we-work
- Existing operating model page (Amplify PM Operating Rhythm) integrated as a route
- Static content: daily, weekly, monthly (including Discovery Hours), quarterly cadences, rules, FAQ
- Same dark theme, gains K Amplify nav header
- No Linear API needed

### /metrics (Coming Soon)
- Static placeholder page
- Preview mockup cards (grayed out): hours saved, agent autonomy rate, business metrics, coverage
- No backend work — future implementation will pull from Linear + internal metrics pipelines

## Linear Data Model Mapping

### Hierarchy
- **Initiative** = "Working AI First" (top-level Linear Initiative, Drew owns)
- **Goals** = Linear Sub-Initiatives under Working AI First. One per partner org. These have native health status (On Track / At Risk / Off Track) set via Linear's project update mechanism.
- **Projects** = Linear Projects within each team. Each project belongs to a team and connects to a goal.
- **Milestones** = Linear Milestones on projects. At least one per project per quarter.
- **Issues** = Linear Issues within projects, assigned to cycles.

### Team-to-Linear Mapping

| Org Filter | Linear Team Name | PM Owner |
|------------|-----------------|----------|
| Sales | Amplify Sales | Jay Chiruvolu |
| Sales | Amplify Demos | Jay Chiruvolu |
| Support | Amplify Support | Jeremy Blanchard |
| CS/Services | Amplify Success & Services | Tyler Beck |
| R&D | Amplify R&D | Christina Valente |
| Marketing | Amplify Marketing | Richard Ng |

Team IDs will be resolved at build time via the Linear API `teams` query and stored as environment config. The `?team=` filter parameter accepts the org name (e.g., `?team=sales`) and maps to the corresponding Linear team ID(s).

### Goal-to-Org Mapping

| Goal (Sub-Initiative) | Org | Linear Team(s) |
|----------------------|-----|----------------|
| Increase Seller PPR by 30% | Sales | Amplify Sales, Amplify Demos |
| Resolve 55% via automated resolution | Support | Amplify Support |
| Increase Success & Services time w/ customers by 30% | CS/Services | Amplify Success & Services |
| Increase R&D efficiency by 13% | R&D | Amplify R&D |
| Improve G&A, Marketing, Ops efficiency by 15% | Marketing | Amplify Marketing |

For /digests, each goal maps to exactly one org. Monthly digests are the most recent status update on each goal (Linear Sub-Initiative update) within a calendar month.

### Health Status
Health status comes from Linear's native project/goal health field, set by PMs in their Friday project updates and monthly goal updates. Values: On Track, At Risk, Off Track.

### Loom Extraction Rules
- Scan Linear **project updates** (the status update body text) for URLs matching `loom.com/share/*`
- Also scan **issue comments** on issues within Amplify team projects
- Extract: Loom URL, parent project name (as title), team (for org tag), update date
- If multiple Loom links appear in a single update, each becomes a separate demo entry
- Title = parent project name + update date (e.g., "BDR Intelligence Platform — Mar 7")

### Intake Title Generation
- Title = first 100 characters of the description field, truncated at the last word boundary
- If the description contains newlines, use only the first line (up to 100 chars)
- If the first line is blank, use "Untitled request from [name]"

## Architecture

### Tech Stack
- **Frontend:** Next.js on Vercel
- **Backend:** Vercel serverless functions (API routes)
- **Data source:** Linear API (server-side only, API key in Vercel env vars)
- **Auth:** Internal network / VPN access only
- **Theme:** Dark theme matching existing operating model page
- **Hosting:** Vercel (existing account, drewklaviyo GitHub)

### API Endpoints

| Endpoint | Method | Source | Cache | Purpose |
|----------|--------|--------|-------|---------|
| `/api/roadmap` | GET | Linear API | 5 min | Goals, projects, milestones, health status |
| `/api/shipped` | GET | Linear API | 15 min | Completed projects with final status update |
| `/api/demos` | GET | Linear API | 15 min | Loom links from project updates/comments |
| `/api/digests` | GET | Linear API | 30 min | Monthly goal-level status updates |
| `/api/intake` | POST | Linear API | None | Create issue in team triage |

All read endpoints accept an optional `?team=` query parameter for org filtering.

### Caching Strategy
- Use Next.js ISR (Incremental Static Regeneration) with `revalidate` for read endpoints:
  - `/api/roadmap`: `revalidate: 300` (5 min)
  - `/api/shipped`: `revalidate: 900` (15 min)
  - `/api/demos`: `revalidate: 900` (15 min)
  - `/api/digests`: `revalidate: 1800` (30 min)
- Alternatively, use `Cache-Control` headers on API routes for Vercel's edge cache
- No database needed — Linear is the single source of truth

### Auth & Security Notes
- Internal network / VPN access only — no per-user authentication for v1
- The /intake "Your name" field is unvalidated free text. This is an accepted tradeoff: all users are internal Klaviyo employees, and the PM triages every request at Monday PM Sync. Abuse risk is negligible in an internal tool.
- Linear API key stored in Vercel environment variables, never exposed client-side

## Key Design Decisions

1. **Linear is the only data source** — no second system to maintain
2. **Zero PM effort for shipped/demos** — auto-generated from existing Linear workflow
3. **Intake creates Linear issues** — routes to team triage, PM triages at Monday PM Sync
4. **Two audiences, one site** — partners filter to their org, leadership sees cross-cutting view
5. **Team directory links out** — no duplication of internal directory
6. **Operating model is static** — editorial content that changes infrequently, no API needed
7. **Metrics is a placeholder** — signals the vision without blocking launch

## Out of Scope (for v1)

- Authentication beyond network access
- Metrics dashboard (coming soon placeholder only)
- Slack integration / notifications from K Amplify
- Mobile-optimized layout (desktop-first, responsive is fine)
- Search functionality across all sections
