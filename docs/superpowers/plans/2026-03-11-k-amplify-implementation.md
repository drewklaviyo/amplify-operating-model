# K Amplify Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build K Amplify — an internal website for Amplify's partners and leadership with live Linear roadmap, intake form, demo archive, changelog, digest history, and operating model.

**Architecture:** Next.js App Router on Vercel with serverless API routes proxying the Linear GraphQL API. Static pages for operating model and metrics placeholder. Dark theme matching existing site. ISR caching with time-based revalidation.

**Tech Stack:** Next.js 15 (App Router), React, TypeScript, Linear SDK (`@linear/sdk`), Vercel, Tailwind CSS (utility classes matching the existing dark theme CSS variables)

**Spec:** `docs/superpowers/specs/2026-03-11-k-amplify-design.md`

---

## File Structure

```
k-amplify/
├── app/
│   ├── layout.tsx              # Root layout: nav, dark theme, 🔥 logo
│   ├── page.tsx                # Home dashboard (5 org cards + activity feed)
│   ├── roadmap/
│   │   └── page.tsx            # Roadmap with org filter
│   ├── demos/
│   │   └── page.tsx            # Demo archive with Loom embeds
│   ├── shipped/
│   │   └── page.tsx            # Changelog feed
│   ├── digests/
│   │   └── page.tsx            # Monthly digest archive
│   ├── intake/
│   │   └── page.tsx            # Intake form
│   ├── how-we-work/
│   │   └── page.tsx            # Operating model (static HTML)
│   ├── metrics/
│   │   └── page.tsx            # Coming soon placeholder
│   └── api/
│       ├── roadmap/
│       │   └── route.ts        # GET: goals, projects, milestones, health
│       ├── shipped/
│       │   └── route.ts        # GET: completed projects
│       ├── demos/
│       │   └── route.ts        # GET: Loom links from updates
│       ├── digests/
│       │   └── route.ts        # GET: goal-level status updates
│       └── intake/
│           └── route.ts        # POST: create Linear issue
├── lib/
│   ├── linear.ts               # Linear SDK client singleton
│   ├── types.ts                # Shared TypeScript types
│   ├── config.ts               # Team/org mappings, constants
│   └── loom.ts                 # Loom URL extraction utility
├── components/
│   ├── nav.tsx                 # Site navigation
│   ├── org-filter.tsx          # Org filter bar component
│   ├── health-badge.tsx        # On Track / At Risk / Off Track badge
│   ├── project-card.tsx        # Project card for roadmap
│   ├── demo-card.tsx           # Demo entry with Loom embed
│   ├── shipped-entry.tsx       # Changelog entry
│   ├── digest-entry.tsx        # Expandable digest entry
│   └── org-card.tsx            # Home dashboard org card
├── public/
│   └── how-we-work.html        # Existing operating model HTML (iframe source)
├── .env.local                  # LINEAR_API_KEY (not committed)
├── next.config.ts              # Next.js config
├── tailwind.config.ts          # Tailwind with dark theme tokens
├── tsconfig.json
├── package.json
└── vercel.json
```

---

## Chunk 1: Project Scaffolding + Static Pages

### Task 1: Initialize Next.js Project

**Files:**
- Create: `k-amplify/package.json`
- Create: `k-amplify/tsconfig.json`
- Create: `k-amplify/next.config.ts`
- Create: `k-amplify/tailwind.config.ts`
- Create: `k-amplify/vercel.json`
- Create: `k-amplify/.env.local`
- Create: `k-amplify/.gitignore`

- [ ] **Step 1: Create the project directory**

```bash
mkdir -p k-amplify && cd k-amplify
```

- [ ] **Step 2: Initialize Next.js with TypeScript and Tailwind**

```bash
npx create-next-app@latest . --typescript --tailwind --app --eslint --src-dir=false --import-alias="@/*" --use-npm
```

- [ ] **Step 3: Install Linear SDK**

```bash
npm install @linear/sdk
```

- [ ] **Step 4: Create vercel.json**

```json
{
  "framework": "nextjs"
}
```

- [ ] **Step 5: Create .env.local with placeholder**

```
LINEAR_API_KEY=lin_api_REPLACE_WITH_REAL_KEY
```

- [ ] **Step 6: Update tailwind.config.ts with dark theme tokens**

```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./components/**/*.{ts,tsx}",
    "./app/**/*.{ts,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        bg: "#0a0a0f",
        surface: "#12121a",
        "surface-2": "#1a1a26",
        border: "#2a2a3a",
        text: "#e4e4ef",
        "text-secondary": "#8888a0",
        accent: "#6c5ce7",
        "accent-light": "#a29bfe",
        green: "#00b894",
        orange: "#fdcb6e",
        red: "#e17055",
        blue: "#74b9ff",
      },
    },
  },
  plugins: [],
};
export default config;
```

- [ ] **Step 7: Update app/globals.css**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  background: #0a0a0f;
  color: #e4e4ef;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  -webkit-font-smoothing: antialiased;
}
```

- [ ] **Step 8: Verify dev server starts**

```bash
npm run dev
```

Expected: Server starts at localhost:3000 with no errors.

- [ ] **Step 9: Commit**

```bash
git add -A && git commit -m "feat: scaffold Next.js project with Tailwind dark theme"
```

---

### Task 2: Shared Types and Config

**Files:**
- Create: `k-amplify/lib/types.ts`
- Create: `k-amplify/lib/config.ts`

- [ ] **Step 1: Create shared types**

```typescript
// lib/types.ts

export type OrgSlug = "sales" | "support" | "cs" | "rnd" | "marketing";

export type HealthStatus = "onTrack" | "atRisk" | "offTrack" | "none";

export interface OrgConfig {
  slug: OrgSlug;
  label: string;
  teamNames: string[];
  goalName: string;
  pmOwner: string;
}

export interface ProjectSummary {
  id: string;
  name: string;
  health: HealthStatus;
  teamName: string;
  orgSlug: OrgSlug;
  latestUpdate: string | null;
  latestUpdateDate: string | null;
  milestones: MilestoneSummary[];
  completedAt: string | null;
  updatedAt: string;
}

export interface MilestoneSummary {
  id: string;
  name: string;
  targetDate: string | null;
  sortOrder: number;
}

export interface GoalSummary {
  id: string;
  name: string;
  health: HealthStatus;
  orgSlug: OrgSlug;
  pmOwner: string;
  latestUpdate: string | null;
  latestUpdateDate: string | null;
  projects: ProjectSummary[];
}

export interface DemoEntry {
  id: string;
  loomUrl: string;
  title: string;
  orgSlug: OrgSlug;
  projectName: string;
  date: string;
}

export interface DigestEntry {
  id: string;
  orgSlug: OrgSlug;
  goalName: string;
  health: HealthStatus;
  content: string;
  date: string;
  month: string; // "March 2026"
}

export interface ShippedEntry {
  id: string;
  projectName: string;
  orgSlug: OrgSlug;
  completedAt: string;
  lastUpdate: string;
  loomUrl: string | null;
}

export interface IntakeRequest {
  name: string;
  team: OrgSlug;
  requestType: string;
  urgency: string;
  description: string;
}

export interface ActivityItem {
  type: "shipped" | "demo" | "update";
  title: string;
  orgSlug: OrgSlug;
  date: string;
  detail: string;
}
```

- [ ] **Step 2: Create org config mapping**

```typescript
// lib/config.ts

import { OrgConfig, OrgSlug } from "./types";

export const ORG_CONFIGS: OrgConfig[] = [
  {
    slug: "sales",
    label: "Sales",
    teamNames: ["Amplify Sales", "Amplify Demos"],
    goalName: "Increase Seller PPR by 30%",
    pmOwner: "Jay Chiruvolu",
  },
  {
    slug: "support",
    label: "Support",
    teamNames: ["Amplify Support"],
    goalName: "Resolve 55% via automated resolution",
    pmOwner: "Jeremy Blanchard",
  },
  {
    slug: "cs",
    label: "CS/Services",
    teamNames: ["Amplify Success & Services"],
    goalName: "Increase Success & Services time w/ customers by 30%",
    pmOwner: "Tyler Beck",
  },
  {
    slug: "rnd",
    label: "R&D",
    teamNames: ["Amplify R&D"],
    goalName: "Increase R&D efficiency by 13%",
    pmOwner: "Christina Valente",
  },
  {
    slug: "marketing",
    label: "Marketing",
    teamNames: ["Amplify Marketing"],
    goalName: "Improve G&A, Marketing, Ops efficiency by 15%",
    pmOwner: "Richard Ng",
  },
];

export const ORG_BY_SLUG: Record<OrgSlug, OrgConfig> = Object.fromEntries(
  ORG_CONFIGS.map((c) => [c.slug, c])
) as Record<OrgSlug, OrgConfig>;

export const ALL_TEAM_NAMES = ORG_CONFIGS.flatMap((c) => c.teamNames);

export function orgSlugForTeamName(teamName: string): OrgSlug | null {
  const config = ORG_CONFIGS.find((c) => c.teamNames.includes(teamName));
  return config?.slug ?? null;
}

export const INTAKE_REQUEST_TYPES = [
  "New capability",
  "Enhancement to existing",
  "Bug/issue",
  "Data request",
  "Escalation",
] as const;

export const INTAKE_URGENCIES = [
  "Blocking us now",
  "Need this quarter",
  "Future idea",
] as const;
```

- [ ] **Step 3: Commit**

```bash
git add lib/types.ts lib/config.ts && git commit -m "feat: add shared types and org config mapping"
```

---

### Task 3: Linear Client + Loom Extraction

**Files:**
- Create: `k-amplify/lib/linear.ts`
- Create: `k-amplify/lib/loom.ts`

- [ ] **Step 1: Create Linear SDK client singleton**

```typescript
// lib/linear.ts

import { LinearClient } from "@linear/sdk";

let client: LinearClient | null = null;

export function getLinearClient(): LinearClient {
  if (!client) {
    const apiKey = process.env.LINEAR_API_KEY;
    if (!apiKey) {
      throw new Error("LINEAR_API_KEY environment variable is not set");
    }
    client = new LinearClient({ apiKey });
  }
  return client;
}
```

- [ ] **Step 2: Create Loom URL extraction utility**

```typescript
// lib/loom.ts

const LOOM_REGEX = /https?:\/\/(www\.)?loom\.com\/share\/[a-zA-Z0-9]+/g;

export function extractLoomUrls(text: string): string[] {
  if (!text) return [];
  const matches = text.match(LOOM_REGEX);
  return matches ? [...new Set(matches)] : [];
}
```

- [ ] **Step 3: Commit**

```bash
git add lib/linear.ts lib/loom.ts && git commit -m "feat: add Linear client and Loom URL extraction"
```

---

### Task 4: Navigation Component

**Files:**
- Create: `k-amplify/components/nav.tsx`
- Modify: `k-amplify/app/layout.tsx`

- [ ] **Step 1: Create nav component**

```tsx
// components/nav.tsx

"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";

const links = [
  { href: "/", label: "Home" },
  { href: "/roadmap", label: "Roadmap" },
  { href: "/demos", label: "Demos" },
  { href: "/shipped", label: "Shipped" },
  { href: "/digests", label: "Digests" },
  { href: "/intake", label: "Intake" },
  { href: "/how-we-work", label: "How We Work" },
];

export function Nav() {
  const pathname = usePathname();

  return (
    <nav className="fixed top-0 left-0 right-0 z-50 bg-bg/85 backdrop-blur-xl border-b border-border px-8">
      <div className="max-w-[900px] mx-auto flex items-center h-[52px] gap-8">
        <Link href="/" className="font-bold text-[0.92rem] tracking-tight flex items-center gap-2 whitespace-nowrap">
          🔥 K Amplify
        </Link>
        <div className="flex gap-0.5">
          {links.map((link) => (
            <Link
              key={link.href}
              href={link.href}
              className={`text-[0.78rem] font-medium px-2.5 py-1.5 rounded-md transition-all ${
                pathname === link.href
                  ? "text-text bg-surface-2"
                  : "text-text-secondary hover:text-text hover:bg-surface-2"
              }`}
            >
              {link.label}
            </Link>
          ))}
        </div>
      </div>
    </nav>
  );
}
```

- [ ] **Step 2: Update root layout**

```tsx
// app/layout.tsx

import type { Metadata } from "next";
import { Nav } from "@/components/nav";
import "./globals.css";

export const metadata: Metadata = {
  title: "K Amplify",
  description: "Amplify partner portal and operating hub",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Nav />
        <main className="max-w-[900px] mx-auto px-8 pt-[76px] pb-20">
          {children}
        </main>
      </body>
    </html>
  );
}
```

- [ ] **Step 3: Verify nav renders in browser**

```bash
npm run dev
```

Expected: Dark page with nav bar showing 🔥 K Amplify and all nav links.

- [ ] **Step 4: Commit**

```bash
git add components/nav.tsx app/layout.tsx && git commit -m "feat: add site navigation with dark theme"
```

---

### Task 5: Shared UI Components

**Files:**
- Create: `k-amplify/components/health-badge.tsx`
- Create: `k-amplify/components/org-filter.tsx`

- [ ] **Step 1: Create health badge component**

```tsx
// components/health-badge.tsx

import { HealthStatus } from "@/lib/types";

const statusConfig: Record<HealthStatus, { label: string; className: string }> = {
  onTrack: { label: "On Track", className: "text-green bg-green/10 border-green/25" },
  atRisk: { label: "At Risk", className: "text-orange bg-orange/10 border-orange/25" },
  offTrack: { label: "Off Track", className: "text-red bg-red/10 border-red/25" },
  none: { label: "No Status", className: "text-text-secondary bg-surface-2 border-border" },
};

export function HealthBadge({ status }: { status: HealthStatus }) {
  const config = statusConfig[status];
  return (
    <span className={`text-[0.68rem] font-semibold px-2 py-0.5 rounded border ${config.className}`}>
      {config.label}
    </span>
  );
}
```

- [ ] **Step 2: Create org filter component**

```tsx
// components/org-filter.tsx

"use client";

import { OrgSlug } from "@/lib/types";
import { ORG_CONFIGS } from "@/lib/config";

interface OrgFilterProps {
  selected: OrgSlug | "all";
  onChange: (slug: OrgSlug | "all") => void;
}

export function OrgFilter({ selected, onChange }: OrgFilterProps) {
  const options: { slug: OrgSlug | "all"; label: string }[] = [
    { slug: "all", label: "All" },
    ...ORG_CONFIGS.map((c) => ({ slug: c.slug, label: c.label })),
  ];

  return (
    <div className="flex gap-1 mb-6">
      {options.map((opt) => (
        <button
          key={opt.slug}
          onClick={() => onChange(opt.slug)}
          className={`text-[0.78rem] font-medium px-3 py-1.5 rounded-md transition-all border ${
            selected === opt.slug
              ? "text-accent-light bg-accent/12 border-accent/25"
              : "text-text-secondary bg-surface border-border hover:text-text hover:bg-surface-2"
          }`}
        >
          {opt.label}
        </button>
      ))}
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add components/health-badge.tsx components/org-filter.tsx && git commit -m "feat: add health badge and org filter components"
```

---

### Task 6: How We Work Page (Static)

**Files:**
- Create: `k-amplify/public/how-we-work.html`
- Create: `k-amplify/app/how-we-work/page.tsx`

- [ ] **Step 1: Copy existing operating model HTML to public directory**

Copy the current `index.html` from the `amplify-operating-model` repo into `k-amplify/public/how-we-work.html`. Remove the existing `<nav>` element from the HTML since K Amplify provides its own nav. Adjust the `<main>` padding-top to `20px` instead of `76px`.

- [ ] **Step 2: Create the How We Work page with iframe**

```tsx
// app/how-we-work/page.tsx

export default function HowWeWorkPage() {
  return (
    <div className="-mx-8 -mt-4">
      <iframe
        src="/how-we-work.html"
        className="w-full border-0"
        style={{ height: "calc(100vh - 52px)" }}
        title="Amplify Operating Rhythm"
      />
    </div>
  );
}
```

- [ ] **Step 3: Verify in browser**

```bash
npm run dev
```

Navigate to `/how-we-work`. Expected: The operating model page renders inside the K Amplify shell with the site nav at the top.

- [ ] **Step 4: Commit**

```bash
git add public/how-we-work.html app/how-we-work/page.tsx && git commit -m "feat: add How We Work page with operating model"
```

---

### Task 7: Metrics Coming Soon Page

**Files:**
- Create: `k-amplify/app/metrics/page.tsx`

- [ ] **Step 1: Create metrics placeholder**

```tsx
// app/metrics/page.tsx

export default function MetricsPage() {
  return (
    <div className="pt-10">
      <div className="text-center py-20">
        <div className="text-5xl mb-4">🔥</div>
        <h1 className="text-2xl font-bold tracking-tight mb-2">
          Metrics Dashboard
        </h1>
        <p className="text-text-secondary text-sm mb-10">Coming soon</p>

        <div className="grid grid-cols-2 gap-4 max-w-lg mx-auto opacity-40">
          {[
            { label: "Hours Saved", value: "—" },
            { label: "Agent Autonomy Rate", value: "—" },
            { label: "Business Metrics", value: "—" },
            { label: "JTBD Coverage", value: "—" },
          ].map((card) => (
            <div
              key={card.label}
              className="bg-surface border border-border rounded-lg p-6"
            >
              <div className="text-2xl font-bold text-accent-light mb-1">
                {card.value}
              </div>
              <div className="text-xs text-text-secondary font-semibold uppercase tracking-wide">
                {card.label}
              </div>
            </div>
          ))}
        </div>

        <p className="text-text-secondary text-xs mt-10">
          Live tracking of hours saved, agent autonomy rates, and business
          metrics across all partner orgs.
        </p>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verify in browser**

Navigate to `/metrics`. Expected: Centered placeholder with grayed-out metric cards.

- [ ] **Step 3: Commit**

```bash
git add app/metrics/page.tsx && git commit -m "feat: add metrics coming soon placeholder"
```

---

## Chunk 2: Linear API Layer

### Task 8: Roadmap API Route

**Files:**
- Create: `k-amplify/app/api/roadmap/route.ts`

- [ ] **Step 1: Create the roadmap API route**

```typescript
// app/api/roadmap/route.ts

import { NextRequest, NextResponse } from "next/server";
import { getLinearClient } from "@/lib/linear";
import { orgSlugForTeamName, ORG_CONFIGS } from "@/lib/config";
import { GoalSummary, ProjectSummary, HealthStatus, MilestoneSummary } from "@/lib/types";

function mapHealth(health: string | undefined | null): HealthStatus {
  if (!health) return "none";
  const h = health.toLowerCase();
  if (h === "ontrack" || h === "on_track") return "onTrack";
  if (h === "atrisk" || h === "at_risk") return "atRisk";
  if (h === "offtrack" || h === "off_track") return "offTrack";
  return "none";
}

export const revalidate = 300; // 5 minutes ISR

export async function GET(request: NextRequest) {
  try {
    const teamFilter = request.nextUrl.searchParams.get("team");
    const client = getLinearClient();

    // Fetch all teams to build ID mapping
    const teamsResponse = await client.teams();
    const amplifyTeams = teamsResponse.nodes.filter((t) =>
      t.name.startsWith("Amplify")
    );
    const teamIdToName: Record<string, string> = {};
    amplifyTeams.forEach((t) => {
      teamIdToName[t.id] = t.name;
    });

    // Fetch projects for Amplify teams
    const projects = await client.projects({
      filter: {
        state: { type: { in: ["planned", "started", "paused"] } },
        accessibleTeams: { name: { startsWithAny: ["Amplify"] } },
      },
      first: 100,
    });

    const projectSummaries: ProjectSummary[] = [];

    for (const project of projects.nodes) {
      const teams = await project.teams();
      const teamName = teams.nodes[0]?.name ?? "Unknown";
      const orgSlug = orgSlugForTeamName(teamName);
      if (!orgSlug) continue;
      if (teamFilter && teamFilter !== orgSlug) continue;

      // Get latest project update
      const updates = await project.projectUpdates({
        first: 1,
        orderBy: "createdAt",
      });
      const latestUpdate = updates.nodes[0];

      // Get milestones
      const milestones = await project.projectMilestones();

      projectSummaries.push({
        id: project.id,
        name: project.name,
        health: mapHealth(latestUpdate?.health as string),
        teamName,
        orgSlug,
        latestUpdate: latestUpdate?.body ?? null,
        latestUpdateDate: latestUpdate?.createdAt?.toISOString() ?? null,
        milestones: milestones.nodes.map((m) => ({
          id: m.id,
          name: m.name,
          targetDate: m.targetDate ?? null,
          sortOrder: m.sortOrder,
        })),
        completedAt: null,
        updatedAt: project.updatedAt.toISOString(),
      });
    }

    // Group by org goal
    const goals: GoalSummary[] = ORG_CONFIGS
      .filter((c) => !teamFilter || c.slug === teamFilter)
      .map((config) => {
        const orgProjects = projectSummaries
          .filter((p) => p.orgSlug === config.slug)
          .sort((a, b) => {
            // At Risk / Off Track first
            const priority: Record<HealthStatus, number> = {
              offTrack: 0,
              atRisk: 1,
              onTrack: 2,
              none: 3,
            };
            return (priority[a.health] - priority[b.health]) ||
              (new Date(b.updatedAt).getTime() - new Date(a.updatedAt).getTime());
          });

        // Derive goal health from worst project health
        const worstHealth = orgProjects.reduce<HealthStatus>((worst, p) => {
          const priority: Record<HealthStatus, number> = {
            offTrack: 0,
            atRisk: 1,
            onTrack: 2,
            none: 3,
          };
          return priority[p.health] < priority[worst] ? p.health : worst;
        }, "none");

        return {
          id: config.slug,
          name: config.goalName,
          health: orgProjects.length > 0 ? worstHealth : "none",
          orgSlug: config.slug,
          pmOwner: config.pmOwner,
          latestUpdate: orgProjects[0]?.latestUpdate ?? null,
          latestUpdateDate: orgProjects[0]?.latestUpdateDate ?? null,
          projects: orgProjects,
        };
      });

    return NextResponse.json({ goals });
  } catch (error) {
    console.error("Roadmap API error:", error);
    return NextResponse.json(
      { error: "Failed to fetch roadmap data" },
      { status: 500 }
    );
  }
}
```

- [ ] **Step 2: Test with curl (requires valid LINEAR_API_KEY in .env.local)**

```bash
npm run dev &
curl http://localhost:3000/api/roadmap | jq '.goals | length'
```

Expected: Returns 5 (one per org).

- [ ] **Step 3: Commit**

```bash
git add app/api/roadmap/route.ts && git commit -m "feat: add roadmap API route with Linear integration"
```

---

### Task 9: Shipped API Route

**Files:**
- Create: `k-amplify/app/api/shipped/route.ts`

- [ ] **Step 1: Create the shipped API route**

```typescript
// app/api/shipped/route.ts

import { NextRequest, NextResponse } from "next/server";
import { getLinearClient } from "@/lib/linear";
import { orgSlugForTeamName } from "@/lib/config";
import { ShippedEntry } from "@/lib/types";
import { extractLoomUrls } from "@/lib/loom";

export const revalidate = 900; // 15 minutes ISR

export async function GET(request: NextRequest) {
  try {
    const teamFilter = request.nextUrl.searchParams.get("team");
    const client = getLinearClient();

    const projects = await client.projects({
      filter: {
        state: { type: { eq: "completed" } },
        accessibleTeams: { name: { startsWithAny: ["Amplify"] } },
      },
      first: 50,
      orderBy: "updatedAt",
    });

    const entries: ShippedEntry[] = [];

    for (const project of projects.nodes) {
      const teams = await project.teams();
      const teamName = teams.nodes[0]?.name ?? "Unknown";
      const orgSlug = orgSlugForTeamName(teamName);
      if (!orgSlug) continue;
      if (teamFilter && teamFilter !== orgSlug) continue;

      const updates = await project.projectUpdates({
        first: 1,
        orderBy: "createdAt",
      });
      const latestUpdate = updates.nodes[0];
      const loomUrls = latestUpdate?.body ? extractLoomUrls(latestUpdate.body) : [];

      entries.push({
        id: project.id,
        projectName: project.name,
        orgSlug,
        completedAt: project.completedAt?.toISOString() ?? project.updatedAt.toISOString(),
        lastUpdate: latestUpdate?.body ?? "",
        loomUrl: loomUrls[0] ?? null,
      });
    }

    // Sort by completion date descending
    entries.sort((a, b) => new Date(b.completedAt).getTime() - new Date(a.completedAt).getTime());

    return NextResponse.json({ entries });
  } catch (error) {
    console.error("Shipped API error:", error);
    return NextResponse.json(
      { error: "Failed to fetch shipped data" },
      { status: 500 }
    );
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add app/api/shipped/route.ts && git commit -m "feat: add shipped API route for changelog"
```

---

### Task 10: Demos API Route

**Files:**
- Create: `k-amplify/app/api/demos/route.ts`

- [ ] **Step 1: Create the demos API route**

```typescript
// app/api/demos/route.ts

import { NextRequest, NextResponse } from "next/server";
import { getLinearClient } from "@/lib/linear";
import { orgSlugForTeamName } from "@/lib/config";
import { DemoEntry } from "@/lib/types";
import { extractLoomUrls } from "@/lib/loom";

export const revalidate = 900; // 15 minutes ISR

export async function GET(request: NextRequest) {
  try {
    const teamFilter = request.nextUrl.searchParams.get("team");
    const client = getLinearClient();

    const projects = await client.projects({
      filter: {
        state: { type: { in: ["planned", "started", "paused", "completed"] } },
        accessibleTeams: { name: { startsWithAny: ["Amplify"] } },
      },
      first: 100,
    });

    const entries: DemoEntry[] = [];

    for (const project of projects.nodes) {
      const teams = await project.teams();
      const teamName = teams.nodes[0]?.name ?? "Unknown";
      const orgSlug = orgSlugForTeamName(teamName);
      if (!orgSlug) continue;
      if (teamFilter && teamFilter !== orgSlug) continue;

      // Scan project updates for Loom links
      const updates = await project.projectUpdates({ first: 20 });
      for (const update of updates.nodes) {
        const loomUrls = extractLoomUrls(update.body ?? "");
        for (const loomUrl of loomUrls) {
          const date = update.createdAt.toISOString().split("T")[0];
          entries.push({
            id: `${update.id}-${loomUrl}`,
            loomUrl,
            title: `${project.name} — ${new Date(date).toLocaleDateString("en-US", { month: "short", day: "numeric" })}`,
            orgSlug,
            projectName: project.name,
            date,
          });
        }
      }
    }

    // Sort by date descending
    entries.sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime());

    return NextResponse.json({ entries });
  } catch (error) {
    console.error("Demos API error:", error);
    return NextResponse.json(
      { error: "Failed to fetch demo data" },
      { status: 500 }
    );
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add app/api/demos/route.ts && git commit -m "feat: add demos API route with Loom extraction"
```

---

### Task 11: Digests API Route

**Files:**
- Create: `k-amplify/app/api/digests/route.ts`

- [ ] **Step 1: Create the digests API route**

```typescript
// app/api/digests/route.ts

import { NextRequest, NextResponse } from "next/server";
import { getLinearClient } from "@/lib/linear";
import { ORG_CONFIGS } from "@/lib/config";
import { DigestEntry, HealthStatus } from "@/lib/types";

function mapHealth(health: string | undefined | null): HealthStatus {
  if (!health) return "none";
  const h = health.toLowerCase();
  if (h === "ontrack" || h === "on_track") return "onTrack";
  if (h === "atrisk" || h === "at_risk") return "atRisk";
  if (h === "offtrack" || h === "off_track") return "offTrack";
  return "none";
}

export const revalidate = 1800; // 30 minutes ISR

export async function GET(request: NextRequest) {
  try {
    const teamFilter = request.nextUrl.searchParams.get("team");
    const client = getLinearClient();

    // Fetch initiatives to find "Working AI First"
    const initiatives = await client.initiatives({ first: 50 });
    const waif = initiatives.nodes.find((i) =>
      i.name.toLowerCase().includes("working ai first")
    );

    if (!waif) {
      return NextResponse.json({ entries: [] });
    }

    const entries: DigestEntry[] = [];

    // Get child initiatives (goals/sub-initiatives)
    const childProjects = await waif.projects({ first: 50 });

    // For each org, find matching goal and get its updates
    for (const config of ORG_CONFIGS) {
      if (teamFilter && teamFilter !== config.slug) continue;

      // Search for the goal by name across initiative projects
      // Linear models goals as sub-initiatives or projects under the initiative
      // We'll fetch project updates that match goal patterns
      const projects = await client.projects({
        filter: {
          accessibleTeams: { name: { in: config.teamNames } },
        },
        first: 50,
      });

      for (const project of projects.nodes) {
        const updates = await project.projectUpdates({ first: 12 });
        for (const update of updates.nodes) {
          const date = update.createdAt;
          const monthStr = date.toLocaleDateString("en-US", {
            month: "long",
            year: "numeric",
          });

          entries.push({
            id: update.id,
            orgSlug: config.slug,
            goalName: config.goalName,
            health: mapHealth(update.health as string),
            content: update.body ?? "",
            date: date.toISOString(),
            month: monthStr,
          });
        }
      }
    }

    // Sort by date descending
    entries.sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime());

    return NextResponse.json({ entries });
  } catch (error) {
    console.error("Digests API error:", error);
    return NextResponse.json(
      { error: "Failed to fetch digest data" },
      { status: 500 }
    );
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add app/api/digests/route.ts && git commit -m "feat: add digests API route for goal-level updates"
```

---

### Task 12: Intake API Route

**Files:**
- Create: `k-amplify/app/api/intake/route.ts`

- [ ] **Step 1: Create the intake API route**

```typescript
// app/api/intake/route.ts

import { NextRequest, NextResponse } from "next/server";
import { getLinearClient } from "@/lib/linear";
import { ORG_CONFIGS } from "@/lib/config";
import { IntakeRequest, OrgSlug } from "@/lib/types";

function generateTitle(description: string, name: string): string {
  if (!description.trim()) return `Untitled request from ${name}`;
  const firstLine = description.split("\n").find((line) => line.trim()) ?? "";
  if (!firstLine) return `Untitled request from ${name}`;
  if (firstLine.length <= 100) return firstLine;
  return firstLine.substring(0, 100).replace(/\s\S*$/, "").trim();
}

export async function POST(request: NextRequest) {
  try {
    const body: IntakeRequest = await request.json();

    if (!body.name || !body.team || !body.requestType || !body.urgency || !body.description) {
      return NextResponse.json(
        { error: "All fields are required" },
        { status: 400 }
      );
    }

    const orgConfig = ORG_CONFIGS.find((c) => c.slug === body.team);
    if (!orgConfig) {
      return NextResponse.json(
        { error: "Invalid team selection" },
        { status: 400 }
      );
    }

    const client = getLinearClient();

    // Find the team ID
    const teams = await client.teams();
    const targetTeam = teams.nodes.find((t) => t.name === orgConfig.teamNames[0]);
    if (!targetTeam) {
      return NextResponse.json(
        { error: "Team not found in Linear" },
        { status: 500 }
      );
    }

    const title = generateTitle(body.description, body.name);

    // Create the issue in the team's triage
    const issue = await client.createIssue({
      teamId: targetTeam.id,
      title,
      description: `**Submitted by:** ${body.name}\n**Request type:** ${body.requestType}\n**Urgency:** ${body.urgency}\n\n---\n\n${body.description}`,
    });

    const created = await issue.issue;

    return NextResponse.json({
      success: true,
      teamName: orgConfig.teamNames[0],
      issueId: created?.id,
    });
  } catch (error) {
    console.error("Intake API error:", error);
    return NextResponse.json(
      { error: "Failed to create intake request" },
      { status: 500 }
    );
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add app/api/intake/route.ts && git commit -m "feat: add intake API route for creating Linear issues"
```

---

## Chunk 3: Frontend Pages

### Task 13: Home Dashboard

**Files:**
- Create: `k-amplify/components/org-card.tsx`
- Create: `k-amplify/app/page.tsx`

- [ ] **Step 1: Create org card component**

```tsx
// components/org-card.tsx

import Link from "next/link";
import { HealthBadge } from "./health-badge";
import { GoalSummary } from "@/lib/types";
import { ORG_BY_SLUG } from "@/lib/config";

export function OrgCard({ goal }: { goal: GoalSummary }) {
  const config = ORG_BY_SLUG[goal.orgSlug];
  const nextMilestone = goal.projects
    .flatMap((p) => p.milestones)
    .filter((m) => m.targetDate && new Date(m.targetDate) > new Date())
    .sort((a, b) => new Date(a.targetDate!).getTime() - new Date(b.targetDate!).getTime())[0];

  return (
    <Link
      href={`/roadmap?team=${goal.orgSlug}`}
      className="block bg-surface border border-border rounded-xl p-5 hover:border-accent/40 transition-all"
    >
      <div className="flex items-center justify-between mb-2">
        <h3 className="font-semibold text-sm">{config.label}</h3>
        <HealthBadge status={goal.health} />
      </div>
      <p className="text-text-secondary text-xs mb-1">{config.pmOwner}</p>
      <p className="text-text-secondary text-[0.78rem] mb-3 line-clamp-2">{goal.name}</p>
      {nextMilestone && (
        <div className="text-xs text-text-secondary border-t border-border pt-2 mt-auto">
          <span className="text-accent-light font-medium">Next:</span>{" "}
          {nextMilestone.name}
          {nextMilestone.targetDate && (
            <span className="ml-1 text-text-secondary">
              ({new Date(nextMilestone.targetDate).toLocaleDateString("en-US", { month: "short", day: "numeric" })})
            </span>
          )}
        </div>
      )}
    </Link>
  );
}
```

- [ ] **Step 2: Create home page**

```tsx
// app/page.tsx

import { OrgCard } from "@/components/org-card";
import { GoalSummary } from "@/lib/types";

async function getRoadmapData(): Promise<GoalSummary[]> {
  const baseUrl = process.env.VERCEL_URL
    ? `https://${process.env.VERCEL_URL}`
    : "http://localhost:3000";
  const res = await fetch(`${baseUrl}/api/roadmap`, {
    next: { revalidate: 300 },
  });
  const data = await res.json();
  return data.goals ?? [];
}

export default async function HomePage() {
  const goals = await getRoadmapData();

  return (
    <div className="pt-10">
      <div className="mb-10">
        <h1 className="text-[2.2rem] font-bold tracking-tight leading-tight mb-2 bg-gradient-to-br from-text to-accent-light bg-clip-text text-transparent">
          🔥 K Amplify
        </h1>
        <p className="text-text-secondary text-sm">
          What Amplify is building across every partner org — live from Linear.
        </p>
      </div>

      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-3 mb-10">
        {goals.map((goal) => (
          <OrgCard key={goal.orgSlug} goal={goal} />
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Verify in browser**

Navigate to `/`. Expected: 5 org cards with health badges and goal names.

- [ ] **Step 4: Commit**

```bash
git add components/org-card.tsx app/page.tsx && git commit -m "feat: add home dashboard with org cards"
```

---

### Task 14: Roadmap Page

**Files:**
- Create: `k-amplify/components/project-card.tsx`
- Create: `k-amplify/app/roadmap/page.tsx`

- [ ] **Step 1: Create project card component**

```tsx
// components/project-card.tsx

"use client";

import { useState } from "react";
import { HealthBadge } from "./health-badge";
import { ProjectSummary } from "@/lib/types";
import { ORG_BY_SLUG } from "@/lib/config";

export function ProjectCard({ project }: { project: ProjectSummary }) {
  const [expanded, setExpanded] = useState(false);
  const config = ORG_BY_SLUG[project.orgSlug];

  return (
    <div className="bg-surface border border-border rounded-xl p-5 mb-3">
      <div className="flex items-center justify-between mb-2">
        <div className="flex items-center gap-2">
          <h3 className="font-semibold text-[0.9rem]">{project.name}</h3>
          <span className="text-[0.68rem] text-text-secondary bg-surface-2 px-2 py-0.5 rounded">
            {config.label}
          </span>
        </div>
        <HealthBadge status={project.health} />
      </div>

      {project.latestUpdate && (
        <div
          className={`text-text-secondary text-[0.82rem] leading-relaxed mb-3 ${
            expanded ? "" : "line-clamp-3"
          }`}
        >
          {project.latestUpdate}
        </div>
      )}

      {project.latestUpdate && project.latestUpdate.length > 200 && (
        <button
          onClick={() => setExpanded(!expanded)}
          className="text-accent-light text-xs font-medium mb-3"
        >
          {expanded ? "Show less" : "Show more"}
        </button>
      )}

      {project.milestones.length > 0 && (
        <div className="flex gap-2 flex-wrap mt-2">
          {project.milestones
            .sort((a, b) => a.sortOrder - b.sortOrder)
            .map((m) => (
              <div
                key={m.id}
                className="text-xs bg-surface-2 border border-border rounded px-2 py-1"
              >
                <span className="font-medium">{m.name}</span>
                {m.targetDate && (
                  <span className="text-text-secondary ml-1">
                    {new Date(m.targetDate).toLocaleDateString("en-US", {
                      month: "short",
                      day: "numeric",
                    })}
                  </span>
                )}
              </div>
            ))}
        </div>
      )}

      {project.latestUpdateDate && (
        <div className="text-[0.7rem] text-text-secondary mt-3">
          Updated{" "}
          {new Date(project.latestUpdateDate).toLocaleDateString("en-US", {
            month: "short",
            day: "numeric",
          })}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Create roadmap page**

```tsx
// app/roadmap/page.tsx

"use client";

import { useEffect, useState } from "react";
import { useSearchParams } from "next/navigation";
import { OrgFilter } from "@/components/org-filter";
import { ProjectCard } from "@/components/project-card";
import { GoalSummary, OrgSlug } from "@/lib/types";

export default function RoadmapPage() {
  const searchParams = useSearchParams();
  const initialTeam = (searchParams.get("team") as OrgSlug | null) ?? "all";
  const [filter, setFilter] = useState<OrgSlug | "all">(initialTeam);
  const [goals, setGoals] = useState<GoalSummary[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const teamParam = filter === "all" ? "" : `?team=${filter}`;
    fetch(`/api/roadmap${teamParam}`)
      .then((r) => r.json())
      .then((data) => {
        setGoals(data.goals ?? []);
        setLoading(false);
      });
  }, [filter]);

  return (
    <div className="pt-10">
      <h1 className="text-xl font-bold tracking-tight mb-1">Roadmap</h1>
      <p className="text-text-secondary text-sm mb-6">
        Active projects across Amplify — live from Linear.
      </p>

      <OrgFilter selected={filter} onChange={setFilter} />

      {loading ? (
        <p className="text-text-secondary text-sm">Loading...</p>
      ) : (
        goals.map((goal) => (
          <div key={goal.orgSlug} className="mb-8">
            {filter === "all" && (
              <h2 className="text-sm font-semibold text-accent-light uppercase tracking-wide mb-3">
                {goal.name}
              </h2>
            )}
            {goal.projects.map((project) => (
              <ProjectCard key={project.id} project={project} />
            ))}
            {goal.projects.length === 0 && (
              <p className="text-text-secondary text-sm">No active projects.</p>
            )}
          </div>
        ))
      )}
    </div>
  );
}
```

- [ ] **Step 3: Verify in browser**

Navigate to `/roadmap`. Expected: Project cards grouped by org with filter bar. Click filter to scope to one org.

- [ ] **Step 4: Commit**

```bash
git add components/project-card.tsx app/roadmap/page.tsx && git commit -m "feat: add roadmap page with project cards and filtering"
```

---

### Task 15: Demos Page

**Files:**
- Create: `k-amplify/components/demo-card.tsx`
- Create: `k-amplify/app/demos/page.tsx`

- [ ] **Step 1: Create demo card component**

```tsx
// components/demo-card.tsx

import { DemoEntry } from "@/lib/types";
import { ORG_BY_SLUG } from "@/lib/config";

export function DemoCard({ demo }: { demo: DemoEntry }) {
  const config = ORG_BY_SLUG[demo.orgSlug];
  // Convert loom.com/share/ID to loom.com/embed/ID for iframe
  const embedUrl = demo.loomUrl.replace("/share/", "/embed/");

  return (
    <div className="bg-surface border border-border rounded-xl overflow-hidden mb-3">
      <div className="relative w-full" style={{ paddingBottom: "56.25%" }}>
        <iframe
          src={embedUrl}
          className="absolute inset-0 w-full h-full border-0"
          allowFullScreen
        />
      </div>
      <div className="p-4">
        <div className="flex items-center justify-between">
          <h3 className="font-semibold text-[0.88rem]">{demo.title}</h3>
          <span className="text-[0.68rem] text-text-secondary bg-surface-2 px-2 py-0.5 rounded">
            {config.label}
          </span>
        </div>
        <p className="text-text-secondary text-xs mt-1">
          {new Date(demo.date).toLocaleDateString("en-US", {
            weekday: "long",
            month: "long",
            day: "numeric",
          })}
        </p>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Create demos page**

```tsx
// app/demos/page.tsx

"use client";

import { useEffect, useState } from "react";
import { OrgFilter } from "@/components/org-filter";
import { DemoCard } from "@/components/demo-card";
import { DemoEntry, OrgSlug } from "@/lib/types";

function groupByWeek(entries: DemoEntry[]): Record<string, DemoEntry[]> {
  const groups: Record<string, DemoEntry[]> = {};
  for (const entry of entries) {
    const date = new Date(entry.date);
    const weekStart = new Date(date);
    weekStart.setDate(date.getDate() - date.getDay());
    const key = weekStart.toLocaleDateString("en-US", { month: "long", day: "numeric", year: "numeric" });
    if (!groups[key]) groups[key] = [];
    groups[key].push(entry);
  }
  return groups;
}

export default function DemosPage() {
  const [filter, setFilter] = useState<OrgSlug | "all">("all");
  const [demos, setDemos] = useState<DemoEntry[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const teamParam = filter === "all" ? "" : `?team=${filter}`;
    fetch(`/api/demos${teamParam}`)
      .then((r) => r.json())
      .then((data) => {
        setDemos(data.entries ?? []);
        setLoading(false);
      });
  }, [filter]);

  const grouped = groupByWeek(demos);

  return (
    <div className="pt-10">
      <h1 className="text-xl font-bold tracking-tight mb-1">Demos</h1>
      <p className="text-text-secondary text-sm mb-6">
        Weekly demo recordings from Thursday sessions.
      </p>

      <OrgFilter selected={filter} onChange={setFilter} />

      {loading ? (
        <p className="text-text-secondary text-sm">Loading...</p>
      ) : demos.length === 0 ? (
        <p className="text-text-secondary text-sm">No demos found. Post Loom links in Linear project updates to see them here.</p>
      ) : (
        Object.entries(grouped).map(([week, entries]) => (
          <div key={week} className="mb-8">
            <h2 className="text-xs font-semibold text-text-secondary uppercase tracking-wide mb-3">
              Week of {week}
            </h2>
            {entries.map((demo) => (
              <DemoCard key={demo.id} demo={demo} />
            ))}
          </div>
        ))
      )}
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add components/demo-card.tsx app/demos/page.tsx && git commit -m "feat: add demos page with Loom embeds"
```

---

### Task 16: Shipped Page

**Files:**
- Create: `k-amplify/components/shipped-entry.tsx`
- Create: `k-amplify/app/shipped/page.tsx`

- [ ] **Step 1: Create shipped entry component**

```tsx
// components/shipped-entry.tsx

"use client";

import { useState } from "react";
import { ShippedEntry } from "@/lib/types";
import { ORG_BY_SLUG } from "@/lib/config";

export function ShippedEntryCard({ entry }: { entry: ShippedEntry }) {
  const [expanded, setExpanded] = useState(false);
  const config = ORG_BY_SLUG[entry.orgSlug];

  return (
    <div className="bg-surface border border-border rounded-xl p-5 mb-2">
      <div className="flex items-center justify-between mb-1">
        <h3 className="font-semibold text-[0.88rem]">{entry.projectName}</h3>
        <span className="text-[0.68rem] text-text-secondary bg-surface-2 px-2 py-0.5 rounded">
          {config.label}
        </span>
      </div>
      <p className="text-text-secondary text-xs mb-2">
        Shipped{" "}
        {new Date(entry.completedAt).toLocaleDateString("en-US", {
          month: "long",
          day: "numeric",
          year: "numeric",
        })}
      </p>
      {entry.lastUpdate && (
        <>
          <div
            className={`text-text-secondary text-[0.82rem] leading-relaxed ${
              expanded ? "" : "line-clamp-3"
            }`}
          >
            {entry.lastUpdate}
          </div>
          {entry.lastUpdate.length > 200 && (
            <button
              onClick={() => setExpanded(!expanded)}
              className="text-accent-light text-xs font-medium mt-1"
            >
              {expanded ? "Show less" : "Show more"}
            </button>
          )}
        </>
      )}
      {entry.loomUrl && (
        <a
          href={entry.loomUrl}
          target="_blank"
          rel="noopener noreferrer"
          className="inline-block mt-2 text-xs text-accent-light hover:underline"
        >
          Watch demo →
        </a>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Create shipped page**

```tsx
// app/shipped/page.tsx

"use client";

import { useEffect, useState } from "react";
import { OrgFilter } from "@/components/org-filter";
import { ShippedEntryCard } from "@/components/shipped-entry";
import { ShippedEntry, OrgSlug } from "@/lib/types";

function groupByMonth(entries: ShippedEntry[]): Record<string, ShippedEntry[]> {
  const groups: Record<string, ShippedEntry[]> = {};
  for (const entry of entries) {
    const key = new Date(entry.completedAt).toLocaleDateString("en-US", {
      month: "long",
      year: "numeric",
    });
    if (!groups[key]) groups[key] = [];
    groups[key].push(entry);
  }
  return groups;
}

export default function ShippedPage() {
  const [filter, setFilter] = useState<OrgSlug | "all">("all");
  const [entries, setEntries] = useState<ShippedEntry[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const teamParam = filter === "all" ? "" : `?team=${filter}`;
    fetch(`/api/shipped${teamParam}`)
      .then((r) => r.json())
      .then((data) => {
        setEntries(data.entries ?? []);
        setLoading(false);
      });
  }, [filter]);

  const grouped = groupByMonth(entries);

  return (
    <div className="pt-10">
      <h1 className="text-xl font-bold tracking-tight mb-1">What's Shipped</h1>
      <p className="text-text-secondary text-sm mb-6">
        Everything Amplify has delivered — auto-generated from Linear.
      </p>

      <OrgFilter selected={filter} onChange={setFilter} />

      {loading ? (
        <p className="text-text-secondary text-sm">Loading...</p>
      ) : entries.length === 0 ? (
        <p className="text-text-secondary text-sm">No completed projects yet.</p>
      ) : (
        Object.entries(grouped).map(([month, items]) => (
          <div key={month} className="mb-8">
            <h2 className="text-xs font-semibold text-text-secondary uppercase tracking-wide mb-3">
              {month}
            </h2>
            {items.map((entry) => (
              <ShippedEntryCard key={entry.id} entry={entry} />
            ))}
          </div>
        ))
      )}
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add components/shipped-entry.tsx app/shipped/page.tsx && git commit -m "feat: add shipped changelog page"
```

---

### Task 17: Digests Page

**Files:**
- Create: `k-amplify/components/digest-entry.tsx`
- Create: `k-amplify/app/digests/page.tsx`

- [ ] **Step 1: Create digest entry component**

```tsx
// components/digest-entry.tsx

"use client";

import { useState } from "react";
import { HealthBadge } from "./health-badge";
import { DigestEntry } from "@/lib/types";
import { ORG_BY_SLUG } from "@/lib/config";

export function DigestEntryCard({ entry }: { entry: DigestEntry }) {
  const [expanded, setExpanded] = useState(false);
  const config = ORG_BY_SLUG[entry.orgSlug];

  return (
    <div className="bg-surface border border-border rounded-xl overflow-hidden mb-2">
      <button
        onClick={() => setExpanded(!expanded)}
        className="w-full flex items-center justify-between p-4 text-left hover:bg-surface-2/50 transition-colors"
      >
        <div className="flex items-center gap-3">
          <span
            className="text-[0.7rem] text-text-secondary transition-transform"
            style={{ transform: expanded ? "rotate(90deg)" : "none" }}
          >
            ▶
          </span>
          <div>
            <span className="font-semibold text-[0.88rem]">
              {entry.month} — {config.label}
            </span>
          </div>
        </div>
        <HealthBadge status={entry.health} />
      </button>
      {expanded && (
        <div className="px-4 pb-4 text-text-secondary text-[0.84rem] leading-relaxed whitespace-pre-wrap border-t border-border pt-3 mx-4">
          {entry.content}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Create digests page**

```tsx
// app/digests/page.tsx

"use client";

import { useEffect, useState } from "react";
import { OrgFilter } from "@/components/org-filter";
import { DigestEntryCard } from "@/components/digest-entry";
import { DigestEntry, OrgSlug } from "@/lib/types";

function groupByMonth(entries: DigestEntry[]): Record<string, DigestEntry[]> {
  const groups: Record<string, DigestEntry[]> = {};
  for (const entry of entries) {
    if (!groups[entry.month]) groups[entry.month] = [];
    groups[entry.month].push(entry);
  }
  return groups;
}

export default function DigestsPage() {
  const [filter, setFilter] = useState<OrgSlug | "all">("all");
  const [entries, setEntries] = useState<DigestEntry[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const teamParam = filter === "all" ? "" : `?team=${filter}`;
    fetch(`/api/digests${teamParam}`)
      .then((r) => r.json())
      .then((data) => {
        setEntries(data.entries ?? []);
        setLoading(false);
      });
  }, [filter]);

  const grouped = groupByMonth(entries);

  return (
    <div className="pt-10">
      <h1 className="text-xl font-bold tracking-tight mb-1">Partner Digests</h1>
      <p className="text-text-secondary text-sm mb-6">
        Monthly goal updates — the archive of what we reported to partners.
      </p>

      <OrgFilter selected={filter} onChange={setFilter} />

      {loading ? (
        <p className="text-text-secondary text-sm">Loading...</p>
      ) : entries.length === 0 ? (
        <p className="text-text-secondary text-sm">No digests found.</p>
      ) : (
        Object.entries(grouped).map(([month, items]) => (
          <div key={month} className="mb-6">
            <h2 className="text-xs font-semibold text-text-secondary uppercase tracking-wide mb-3">
              {month}
            </h2>
            {items.map((entry) => (
              <DigestEntryCard key={entry.id} entry={entry} />
            ))}
          </div>
        ))
      )}
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add components/digest-entry.tsx app/digests/page.tsx && git commit -m "feat: add partner digests archive page"
```

---

### Task 18: Intake Form Page

**Files:**
- Create: `k-amplify/app/intake/page.tsx`

- [ ] **Step 1: Create intake form page**

```tsx
// app/intake/page.tsx

"use client";

import { useState } from "react";
import { ORG_CONFIGS, INTAKE_REQUEST_TYPES, INTAKE_URGENCIES } from "@/lib/config";
import { OrgSlug } from "@/lib/types";

export default function IntakePage() {
  const [name, setName] = useState("");
  const [team, setTeam] = useState<OrgSlug | "">("");
  const [requestType, setRequestType] = useState("");
  const [urgency, setUrgency] = useState("");
  const [description, setDescription] = useState("");
  const [submitting, setSubmitting] = useState(false);
  const [submitted, setSubmitted] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!name || !team || !requestType || !urgency || !description) {
      setError("All fields are required.");
      return;
    }

    setSubmitting(true);
    setError(null);

    try {
      const res = await fetch("/api/intake", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ name, team, requestType, urgency, description }),
      });
      const data = await res.json();
      if (data.success) {
        setSubmitted(data.teamName);
      } else {
        setError(data.error ?? "Something went wrong.");
      }
    } catch {
      setError("Failed to submit. Please try again.");
    } finally {
      setSubmitting(false);
    }
  }

  if (submitted) {
    return (
      <div className="pt-10 text-center py-20">
        <div className="text-4xl mb-4">✓</div>
        <h1 className="text-xl font-bold mb-2">Request Submitted</h1>
        <p className="text-text-secondary text-sm">
          Sent to <strong>{submitted}</strong> triage. Your PM will respond within one sprint.
        </p>
        <button
          onClick={() => {
            setSubmitted(null);
            setName("");
            setTeam("");
            setRequestType("");
            setUrgency("");
            setDescription("");
          }}
          className="mt-6 text-accent-light text-sm font-medium hover:underline"
        >
          Submit another request
        </button>
      </div>
    );
  }

  const inputClass =
    "w-full bg-surface border border-border rounded-lg px-3 py-2 text-sm text-text focus:outline-none focus:border-accent/50 transition-colors";

  return (
    <div className="pt-10 max-w-lg">
      <h1 className="text-xl font-bold tracking-tight mb-1">Submit a Request</h1>
      <p className="text-text-secondary text-sm mb-6">
        Request a new capability, report an issue, or escalate a blocker. Routes to the right team's triage in Linear.
      </p>

      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <label className="block text-xs font-semibold text-text-secondary uppercase tracking-wide mb-1">
            Your name
          </label>
          <input
            type="text"
            value={name}
            onChange={(e) => setName(e.target.value)}
            className={inputClass}
            placeholder="e.g., Sarah from Sales Ops"
          />
        </div>

        <div>
          <label className="block text-xs font-semibold text-text-secondary uppercase tracking-wide mb-1">
            Team
          </label>
          <select
            value={team}
            onChange={(e) => setTeam(e.target.value as OrgSlug)}
            className={inputClass}
          >
            <option value="">Select a team...</option>
            {ORG_CONFIGS.map((c) => (
              <option key={c.slug} value={c.slug}>
                {c.teamNames[0]}
              </option>
            ))}
          </select>
        </div>

        <div>
          <label className="block text-xs font-semibold text-text-secondary uppercase tracking-wide mb-1">
            Request type
          </label>
          <select
            value={requestType}
            onChange={(e) => setRequestType(e.target.value)}
            className={inputClass}
          >
            <option value="">Select type...</option>
            {INTAKE_REQUEST_TYPES.map((t) => (
              <option key={t} value={t}>{t}</option>
            ))}
          </select>
        </div>

        <div>
          <label className="block text-xs font-semibold text-text-secondary uppercase tracking-wide mb-1">
            Urgency
          </label>
          <div className="space-y-2">
            {INTAKE_URGENCIES.map((u) => (
              <label key={u} className="flex items-center gap-2 cursor-pointer">
                <input
                  type="radio"
                  name="urgency"
                  value={u}
                  checked={urgency === u}
                  onChange={(e) => setUrgency(e.target.value)}
                  className="accent-accent"
                />
                <span className="text-sm">{u}</span>
              </label>
            ))}
          </div>
        </div>

        <div>
          <label className="block text-xs font-semibold text-text-secondary uppercase tracking-wide mb-1">
            Description
          </label>
          <textarea
            value={description}
            onChange={(e) => setDescription(e.target.value)}
            className={`${inputClass} min-h-[120px]`}
            placeholder="What do you need and why?"
          />
        </div>

        {error && (
          <p className="text-red text-sm">{error}</p>
        )}

        <button
          type="submit"
          disabled={submitting}
          className="w-full bg-accent hover:bg-accent/80 text-white font-medium text-sm py-2.5 rounded-lg transition-colors disabled:opacity-50"
        >
          {submitting ? "Submitting..." : "Submit Request"}
        </button>
      </form>
    </div>
  );
}
```

- [ ] **Step 2: Verify in browser**

Navigate to `/intake`. Expected: Form with all fields, dropdown routing, urgency radio buttons.

- [ ] **Step 3: Commit**

```bash
git add app/intake/page.tsx && git commit -m "feat: add intake form with Linear issue creation"
```

---

## Chunk 4: Deploy + Finalize

### Task 19: Create GitHub Repo and Deploy to Vercel

- [ ] **Step 1: Initialize git and make initial commit**

```bash
cd k-amplify
git init
git add -A
git commit -m "feat: K Amplify v1 — partner portal with Linear integration"
```

- [ ] **Step 2: Create GitHub repo**

```bash
gh repo create drewklaviyo/k-amplify --public --source=. --push
```

- [ ] **Step 3: Deploy to Vercel and link to existing project**

```bash
vercel link --project amplify-operating-model --yes
vercel env add LINEAR_API_KEY production
```

Enter the Linear API key when prompted.

- [ ] **Step 4: Deploy**

```bash
vercel --prod
```

- [ ] **Step 5: Verify live site**

Open the Vercel URL. Verify:
- Home dashboard loads with 5 org cards
- Roadmap shows projects from Linear
- How We Work shows the operating model
- Intake form submits and creates a Linear issue
- Metrics shows coming soon placeholder

- [ ] **Step 6: Commit any final adjustments**

```bash
git add -A && git commit -m "chore: finalize deployment config"
git push
```

---

## Summary

| Chunk | Tasks | What it produces |
|-------|-------|-----------------|
| 1: Scaffolding + Static | Tasks 1-7 | Running Next.js app with nav, theme, types, config, How We Work, Metrics placeholder |
| 2: API Layer | Tasks 8-12 | All 5 API routes (roadmap, shipped, demos, digests, intake) |
| 3: Frontend Pages | Tasks 13-18 | All dynamic pages (home, roadmap, demos, shipped, digests, intake) |
| 4: Deploy | Task 19 | GitHub repo, Vercel deployment, live site |
