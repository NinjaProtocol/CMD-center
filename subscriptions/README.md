# Command Center — Subscriptions Module

A **pure client-side** billing, payment, and domain-expiration tracker that runs entirely in the browser. No API routes, no server calls, no secrets — all data lives in `localStorage`. This module plugs into the Command Center shell alongside Email and Deployments.

Command Center is designed as a **modular platform**. The base application ships with a shell, navigation, and dashboard overview. Each module is an independent feature set that plugs into the shell. This document provides the Subscriptions module. Additional modules (Email, Deployments, Automation, and more) are released as separate instruction sets — same architecture, same shell, no rework required.

---

## Quick Setup

1. Download the `/subscriptions/` folder from this repo
2. Give this folder as context to an AI agent and ask it to follow the instructions and build it
3. The agent will build the full application — no API tokens, no environment variables, no external setup
4. Run the app

This module is entirely self-contained. Unlike Deployments or Email, there are no `.env.local` variables or API keys to configure.

---

## Table of Contents

1. [How This Works](#how-this-works)
2. [Architecture Overview](#architecture-overview)
3. [Step 1: Scaffold the Application](#step-1-scaffold-the-application)
4. [Step 2: Build the Base Shell](#step-2-build-the-base-shell)
5. [Step 3: Build the Subscriptions Module](#step-3-build-the-subscriptions-module)
6. [Step 4: Run](#step-4-run)
7. [Appendix A: Complete Type Definitions](#appendix-a-complete-type-definitions)
8. [Appendix B: localStorage Key Reference](#appendix-b-localstorage-key-reference)
9. [Appendix C: Implementation Notes & Edge Cases](#appendix-c-implementation-notes--edge-cases)

---

## How This Works

This document is a **complete specification** — it contains every detail needed to build the module from zero. You do not need to write any code yourself. The intended workflow is:

1. **You** hand this document to an AI coding agent (Copilot, Cursor, Windsurf, Cline, etc.)
2. **The agent** reads these instructions and builds the full application
3. **You** run the app — no configuration step needed

The application is a Next.js web app that runs locally on your machine. This module stores all billing, payment, deposit, and domain data in the browser's `localStorage`. Data persists across page refreshes and browser restarts. There are no external APIs, no server-side state, and no authentication requirements.

### What You Get

- A full **Command Center shell** with navigation sidebar, topbar, and modular view system
- A **Command dashboard** (home page) with a Subscriptions panel showing upcoming bills
- A complete **Subscriptions module** with:
  - Recurring bill tracker with add, edit, delete (soft + permanent), pause/resume, and mark-paid
  - Card balance tracking with deposits and payment history
  - Transaction ledger combining payments and deposits
  - Financial stats row: card balance, due amounts at 7/14/30/60 day horizons, monthly and yearly run rates
  - Domain expiration sidebar with renewal tracking
  - Inline editing with domain sync
  - Vendor autocomplete from historical entries
  - Multi-filter system: status (all/dueSoon/overdue), dynamic categories, free-text search
  - Rate calculation toggles: autopay-only (AP) and include/exclude domains (DM)

### Modular Design

The shell is built so that future modules plug in without changing existing code. Each module is:
- A **navigation entry** in the sidebar
- A **component** that renders in the main area when its nav item is active
- An optional **topbar section** for module-specific controls
- A **card on the dashboard** for at-a-glance status

Today, only the Subscriptions module is functional. The navigation sidebar already includes placeholder entries for Email, Deployments, Automation, and Settings — these show a "module queued" message when clicked. As new module instruction sets are released, you follow the same workflow: hand the instructions to your agent, and it adds the module into your existing shell.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                        Browser (React)                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ CommandCenterShell                                   │    │
│  │  ├── Sidebar (navigation: Command, Subscriptions, …) │    │
│  │  ├── Topbar (view label + module-specific controls)  │    │
│  │  └── Main Area                                       │    │
│  │      ├── CommandPage (dashboard overview)             │    │
│  │      │   └── Subscriptions card (upcoming bills)      │    │
│  │      ├── SubscriptionsTracker (full module)           │    │
│  │      └── [Future modules plug in here]                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│                    localStorage                              │
│      ┌─────────────────────────────────────────────┐         │
│      │  command-center.billing-tracker   (bills)   │         │
│      │  command-center.billing-deleted   (trash)   │         │
│      │  command-center.payment-history  (payments) │         │
│      │  command-center.card-deposits   (deposits)  │         │
│      │  command-center.domain-tracker   (domains)  │         │
│      └─────────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

**Key principle**: This module has zero server-side dependencies. All data is stored in browser `localStorage`, persisted via `useEffect` hooks that write JSON on every state change. No API routes are needed for the Subscriptions module.

### File Structure

```
src/
├── app/
│   ├── globals.css                      # All styles and CSS variables
│   ├── layout.tsx                       # Root layout with fonts + ToastProvider
│   ├── page.tsx                         # Renders CommandCenterShell
│   └── subscriptions/
│       └── page.tsx                     # Standalone subscriptions page (wraps SubscriptionsTracker)
├── components/
│   ├── toast-provider.tsx               # Toast notification system
│   ├── command-center-shell.tsx         # App shell (sidebar, topbar, routing)
│   ├── command-page.tsx                 # Dashboard overview page
│   └── subscriptions-tracker.tsx        # Full Subscriptions module UI (~740 lines)
└── lib/
    ├── dashboard-data.ts                # Navigation items and shared constants
    └── subscription-data.ts             # Types, constants, utility functions
```

---

## Step 1: Scaffold the Application

### Technology Stack

| Technology | Purpose |
|---|---|
| Next.js (latest) | App Router, server-side rendering |
| React (latest) | UI components |
| TypeScript | Type safety |
| Tailwind CSS | Base styling reset |

The agent should scaffold a new Next.js application with TypeScript, App Router, Tailwind CSS, and ESLint. Use npm as the package manager.

### Fonts

The application uses **IBM Plex Mono** (weights: 400, 500, 600) and **IBM Plex Sans** (weights: 400, 500, 600, 700) via `next/font/google`. The CSS variables are `--font-plex-mono` and `--font-plex-sans`. Apply both font variables to the `<body>` element.

### Root Layout

The root layout wraps all content in a `ToastProvider` component (described below). The page title is "Command Center".

### Root Page

The root page renders a single component: `<CommandCenterShell />`.

### Standalone Subscriptions Page

Create `src/app/subscriptions/page.tsx` — a standalone page that renders `<SubscriptionsTracker />` in a full-height main element with `overflow: hidden`. Set the page metadata title to `"Subscriptions | Command Center"`.

This standalone page allows navigating directly to `/subscriptions` outside of the shell context.

---

## Step 2: Build the Base Shell

The base shell is the foundation that all modules share. It provides navigation, layout, and the dashboard overview.

### 2.1 Styling System (globals.css)

All styling uses CSS custom properties and global utility classes. No component-level CSS modules — components use these classes plus inline styles.

#### CSS Variables

```css
:root {
  --bg: #0a0a0a;
  --fg: #c8c8c8;
  --fg-bright: #e0e0e0;
  --border: #1e1e1e;
  --surface: #111111;
  --surface-raised: rgba(17, 17, 17, 0.96);
  --muted: #808080;
  --muted-dim: #585858;
  --accent: #5af0a0;
  --accent-dim: rgba(90, 240, 160, 0.15);
  --warn: #e0a040;
  --warn-dim: rgba(224, 160, 64, 0.15);
  --danger: #f05050;
  --log-info: #6cb6ff;
  --log-success: #5af0a0;
  --log-warn: #e0a040;
  --log-error: #f05050;
  --log-sync: #c49bff;
  --log-dim: #808080;
}
```

Additionally, include Tailwind's theme integration:
```css
@import "tailwindcss";

@theme inline {
  --color-background: var(--bg);
  --color-foreground: var(--fg);
  --font-sans: var(--font-plex-sans);
  --font-mono: var(--font-plex-mono);
}
```

#### Base Styles

```css
* { box-sizing: border-box; margin: 0; padding: 0; }

html, body {
  height: 100%;
  overflow: hidden;
  background: var(--bg);
  color: var(--fg);
  font-family: var(--font-plex-mono), monospace;
  font-size: 14px;
  line-height: 1.55;
}

a { color: var(--accent); text-decoration: none; }
```

#### Component Classes

These are the core CSS classes used by the shell and all modules. Implement each one:

| Class | Purpose |
|---|---|
| `.cc-shell` | Full-screen flex container — `display: flex; height: 100vh; overflow: hidden; background: var(--bg)` |
| `.cc-sidebar` | Left navigation bar — fixed width 48px, flex column, border-right, background `var(--bg)` |
| `.cc-nav-item` | Sidebar nav button — 48×44px centered, monospace, `var(--muted)` text, hover/active states |
| `.cc-nav-item.active` | Active nav — `var(--accent)` text, accent-dim background |
| `.cc-main` | Right content area — `flex: 1; display: flex; flex-direction: column; overflow: hidden` |
| `.cc-topbar` | Top header bar — 36px height, flex row, border-bottom, background `var(--bg)` |
| `.cc-body` | Content below topbar — `flex: 1; display: flex; overflow: hidden` |
| `.cc-panel` | Panel container — flex column, border, background `var(--surface)` |
| `.cc-panel-header` | Panel header — flex row, space-between, border-bottom, padding 6px 12px, uppercase, 11px font |
| `.cc-panel-body` | Panel body — `flex: 1; overflow: auto` |
| `.cc-row` | Table/list row — padding 6px 12px, border-bottom, hover background `var(--accent-dim)` opacity 0.3 |
| `.cc-btn` | Button — 12px font, padding 3px 10px, border 1px `var(--border)`, `var(--fg)` text, rounded 3px |
| `.cc-btn:hover` | Button hover — border `var(--muted)`, brighter text |
| `.cc-btn.active` | Active button — `var(--accent)` border and text, `var(--accent-dim)` background |
| `.cc-btn.primary` | Primary button — `var(--accent)` border and text |
| `.cc-input` | Input field — 12px font, padding 4px 8px, `var(--bg)` background, `var(--border)` border, `var(--fg)` text |
| `.cc-label` | Label — 10px font, uppercase, `var(--muted)` color, letter-spacing 0.08em |
| `.cc-stat` | Stat cell — padding 6px 12px, border-right 1px `var(--border)` |
| `.cc-stat-label` | Stat label — 10px uppercase, `var(--muted)` color, letter-spacing 0.08em |
| `.cc-stat-value` | Stat value — 14px bold, `var(--fg-bright)` color |

Also provide these utility tone classes used in the domains sidebar:
```css
.tone-warn { color: var(--warn); }
.tone-ok { color: var(--accent); }
.tone-muted { color: var(--muted); }
```

### 2.2 Toast Provider (toast-provider.tsx)

A context-based toast notification system used throughout the app. This is a `"use client"` component.

**API**: `showToast({ title, description, variant })` where variant is `"default"` | `"error"`. Toasts are rendered as fixed-position notifications at `bottom: 24px, right: 24px`, stacking upward. Each toast auto-dismisses after 3 seconds. Style with `var(--surface)` background, `var(--border)` border, and `var(--danger)` left border for error variants.

### 2.3 Navigation Data (dashboard-data.ts)

Define a `NavItem` type and export a `navItems` array:

```ts
export type NavItem = {
  id: string;
  label: string;
  shortLabel: string;
  status: string;
  description: string;
};

export const navItems: NavItem[] = [
  {
    id: "command",
    label: "Command",
    shortLabel: "00",
    status: "live shell",
    description: "Dashboard overview with module summaries."
  },
  {
    id: "subscriptions",
    label: "Subscriptions",
    shortLabel: "02",
    status: "live shell",
    description: "Recurring bills, due payments, and domain expiration tracking."
  },
  // Include placeholder entries for other modules:
  // id: "email", shortLabel: "01", status: "queued"
  // id: "deployments", shortLabel: "03", status: "queued"
  // id: "automation", shortLabel: "04", status: "queued"
  // id: "settings", shortLabel: "05", status: "queued"
];
```

Also export a `subscriptionPalette` array of command palette actions:
```ts
export const subscriptionPalette = ["add bill", "track domain", "mark paid", "review renewals"];
```

### 2.4 Shell Component (command-center-shell.tsx)

The shell is a `"use client"` component that provides the application frame. For the Subscriptions module, it needs to:

#### Imports
- Import `SubscriptionsTracker` from `@/components/subscriptions-tracker`
- Import `CommandPage` from `@/components/command-page`
- Import `navItems` from `@/lib/dashboard-data`
- Import `billingStorageKey` and `BillingEntry` from `@/lib/subscription-data`

#### Constants
```ts
const commandView = "command";
const subscriptionsView = "subscriptions";
const DASHBOARD_BILL_LIMIT = 5;
```

#### Dashboard Bills Helper Functions

These are defined **outside** the component, at module scope:

```ts
function toDashboardBills(entries: BillingEntry[]) {
  return [...entries]
    .filter((entry) => entry.status === "active")
    .sort((a, b) => a.dueDate.localeCompare(b.dueDate))
    .slice(0, DASHBOARD_BILL_LIMIT);
}

function loadDashboardBills() {
  if (typeof window === "undefined") return [];
  const stored = window.localStorage.getItem(billingStorageKey);
  if (!stored) return [];
  try {
    return toDashboardBills(JSON.parse(stored) as BillingEntry[]);
  } catch {
    window.localStorage.removeItem(billingStorageKey);
    return [];
  }
}
```

#### State
- `view` — string controlling which module is active (default: `"command"`)
- `dashboardBills` — `BillingEntry[]` initialized from `loadDashboardBills`

#### View Routing

The main area renders based on `activeNav.id`:

```tsx
{activeNav.id === commandView ? (
  <CommandPage
    onNavigate={setView}
    dashboardBills={dashboardBills}
  />
) : activeNav.id === subscriptionsView ? (
  <div style={{ flex: 1, overflow: "auto", padding: 0 }}>
    <SubscriptionsTracker onEntriesChange={(entries) => {
      setDashboardBills(toDashboardBills(entries));
    }} />
  </div>
) : (
  <div style={{ flex: 1, display: "flex", alignItems: "center", justifyContent: "center", color: "var(--muted)" }}>
    {activeNav.label} module queued.
  </div>
)}
```

Key details:
- The `SubscriptionsTracker` receives an `onEntriesChange` callback that updates the shell's `dashboardBills` state whenever the entries change inside the tracker
- `toDashboardBills` filters to active-only, sorts by due date ascending, and caps at `DASHBOARD_BILL_LIMIT` (5)
- `loadDashboardBills` reads directly from `localStorage` on initial render for the dashboard preview

#### Sidebar

The sidebar renders `navItems` as buttons. Each button shows the `shortLabel` (the number). The active view gets the `.active` class. Clicking a nav item sets `view` to that item's `id`.

#### Topbar

The topbar shows the active module label and status. For example: the left side shows the navigation label, the right side shows `status` text.

### 2.5 Command Page / Dashboard (command-page.tsx)

The dashboard overview page. For the Subscriptions module, it needs a subscriptions panel.

#### Props

```ts
type CommandPageProps = {
  onNavigate: (view: string) => void;
  dashboardBills?: BillingEntry[];
};
```

#### Helper Functions (inside command-page.tsx)

```ts
function daysUntil(v: string) {
  const today = new Date().toISOString().slice(0, 10);
  const t = new Date(`${today}T00:00:00`);
  const d = new Date(`${v}T00:00:00`);
  return Math.round((d.getTime() - t.getTime()) / 86400000);
}

function dueLabel(v: string) {
  const d = daysUntil(v);
  if (d < 0) return `${Math.abs(d)}d overdue`;
  if (d === 0) return "due today";
  return `in ${d}d`;
}

function dueTone(v: string) {
  const d = daysUntil(v);
  if (d < 0) return "var(--danger)";
  if (d <= 7) return "var(--warn)";
  return "var(--muted)";
}

function fmtUsd(v: number) {
  return new Intl.NumberFormat("en-US", {
    style: "currency", currency: "USD", maximumFractionDigits: 2
  }).format(v);
}
```

#### Subscriptions Panel

Render a `cc-panel` with `minHeight: 200`:

**Header**: `💳 subscriptions` label. Right side: count of shown bills (`{dashboardBills.length} shown`, only if > 0) and an "open" button that calls `onNavigate("subscriptions")`.

**Body**: If no bills, show "No stored bills" in muted text. Otherwise, render each bill as a `cc-row` with:
- Left side: bill name in `var(--fg-bright)`, vendor + cadence below in `var(--muted)` (format: `{vendor} · {cadence}`)
- Right side: amount formatted as USD, due status below colored by `dueTone`

```tsx
<div className="cc-panel" style={{ minHeight: 200 }}>
  <div className="cc-panel-header">
    <span>💳 subscriptions</span>
    <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
      {dashboardBills.length > 0 && (
        <span style={{ fontSize: 11, color: "var(--muted)" }}>{dashboardBills.length} shown</span>
      )}
      <button className="cc-btn" style={{ fontSize: 11, padding: "1px 8px" }}
        onClick={() => onNavigate("subscriptions")}>open</button>
    </div>
  </div>
  <div className="cc-panel-body" style={{ padding: 0 }}>
    {dashboardBills.length === 0 ? (
      <div style={{ padding: 12, color: "var(--muted)", fontSize: 13 }}>No stored bills</div>
    ) : (
      dashboardBills.map((bill) => (
        <div key={bill.id} className="cc-row"
          style={{ padding: "6px 12px", cursor: "default", display: "flex",
            justifyContent: "space-between", alignItems: "center" }}>
          <div style={{ flex: 1, minWidth: 0 }}>
            <div style={{ fontSize: 13, color: "var(--fg-bright)", overflow: "hidden",
              textOverflow: "ellipsis", whiteSpace: "nowrap" }}>{bill.name}</div>
            <div style={{ fontSize: 11, color: "var(--muted)" }}>
              {bill.vendor} · {bill.cadence}
            </div>
          </div>
          <div style={{ textAlign: "right", flexShrink: 0, marginLeft: 12 }}>
            <div style={{ fontSize: 13, color: "var(--fg)" }}>{fmtUsd(bill.amount)}</div>
            <div style={{ fontSize: 11, color: dueTone(bill.dueDate) }}>{dueLabel(bill.dueDate)}</div>
          </div>
        </div>
      ))
    )}
  </div>
</div>
```

The dashboard can also include placeholder panels for other modules (Email, Deployments, Automation) showing "module queued" or empty state.

---

## Step 3: Build the Subscriptions Module

The Subscriptions module consists of two files: the data/types file and the main component.

### 3.1 Subscription Data (subscription-data.ts)

This file contains all type definitions, constants, and utility functions.

#### Types

```ts
export type BillingCadence = "weekly" | "monthly" | "quarterly" | "yearly";

export type BillingEntry = {
  id: string;           // crypto.randomUUID()
  name: string;         // subscription name
  vendor: string;       // company / provider
  category: string;     // from billingCategoryOptions
  amount: number;       // dollar amount per cadence period
  cadence: BillingCadence;
  dueDate: string;      // ISO date string "YYYY-MM-DD"
  autopay: boolean;     // whether auto-pay is enabled
  status: "active" | "paused";
  note: string;         // free-text note
  createdAt: string;    // ISO date string
  lastPaidDate?: string; // ISO date string, set when markPaid is called
};

export type PaymentRecord = {
  id: string;           // crypto.randomUUID()
  billId: string;       // references BillingEntry.id
  billName: string;     // snapshot of name at time of payment
  vendor: string;       // snapshot of vendor
  amount: number;       // amount paid
  paidDate: string;     // ISO date
  previousDueDate: string; // due date before advancing
  newDueDate: string;   // due date after advancing
};

export type DepositRecord = {
  id: string;           // crypto.randomUUID()
  amount: number;       // deposit amount
  date: string;         // ISO date
  note: string;         // description
};

export type DomainEntry = {
  id: string;           // shares id with BillingEntry when category is "Domains"
  domain: string;       // domain name (maps from BillingEntry.name)
  registrar: string;    // registrar (maps from BillingEntry.vendor)
  expirationDate: string; // ISO date (maps from BillingEntry.dueDate)
  autoRenew: boolean;   // maps from BillingEntry.autopay
  note: string;         // maps from BillingEntry.note
  status: "active" | "watch";
};
```

#### Constants

```ts
export const billingStorageKey = "command-center.billing-tracker";

export const billingCategoryOptions = [
  "Infrastructure", "Domains", "Email", "Software", "Marketing",
  "Payroll", "Utilities", "Insurance", "Taxes", "Other",
] as const;

export const billingCadenceOptions: Array<{ value: BillingCadence; label: string }> = [
  { value: "weekly", label: "Weekly" },
  { value: "monthly", label: "Monthly" },
  { value: "quarterly", label: "Quarterly" },
  { value: "yearly", label: "Yearly" },
];
```

#### Legacy Seed Filtering

The module originally shipped with seed data that was later removed. To ensure any lingering seed entries are filtered out on load, define these lists and filtering functions:

```ts
export const legacySeedBillingEntryIds = [
  "privateemail-mailboxes",
  "cloudflare-domains",
  "linear-product",
  "aws-backups",
] as const;

export const legacySeedDomainIds = [
  "domain-livepumps-fun",
  "domain-pumpstreams-live",
  "domain-lazywars-com",
  "domain-dlmm-lol",
] as const;
```

The seed creation functions now return empty arrays:
```ts
export function createSeedBillingEntries(): BillingEntry[] { return []; }
export function createSeedDomainEntries(): DomainEntry[] { return []; }
```

#### Utility Functions

```ts
// Private: creates deadline at 23:59:59.999 on the due date
function dueDateDeadline(dueDate: string) {
  const [year, month, day] = dueDate.split("-").map(Number);
  return new Date(year, (month || 1) - 1, day || 1, 23, 59, 59, 999);
}

// Hours remaining until end of due date (can be negative if past)
export function hoursUntilDueDate(dueDate: string, now = new Date()) {
  return (dueDateDeadline(dueDate).getTime() - now.getTime()) / 3_600_000;
}

// Whether a bill is due within the specified hours window (only future/current)
export function isBillDueWithinHours(entry: BillingEntry, hours: number, now = new Date()) {
  const hoursUntil = hoursUntilDueDate(entry.dueDate, now);
  return hoursUntil >= 0 && hoursUntil <= hours;
}
```

### 3.2 Subscriptions Tracker Component (subscriptions-tracker.tsx)

This is the main module component. It is a `"use client"` component (~740 lines).

#### Imports

```ts
import { useEffect, useRef, useState } from "react";
import { useToast } from "@/components/toast-provider";
import {
  billingStorageKey,
  billingCadenceOptions,
  billingCategoryOptions,
  createSeedBillingEntries,
  createSeedDomainEntries,
  legacySeedBillingEntryIds,
  legacySeedDomainIds,
  type BillingCadence,
  type BillingEntry,
  type DepositRecord,
  type DomainEntry,
  type PaymentRecord,
} from "@/lib/subscription-data";
```

#### Component Signature

```ts
export function SubscriptionsTracker({
  onEntriesChange
}: {
  onEntriesChange?: (entries: BillingEntry[]) => void
} = {}) {
```

The `onEntriesChange` callback is called whenever the `entries` state changes, allowing the shell to update the dashboard's billing preview.

#### Local Types

```ts
type TrackerFilter = "all" | "dueSoon" | "overdue";
type TxLogFilter = "all" | "payments" | "deposits";

type BillingFormState = {
  name: string;
  vendor: string;
  category: string;
  amount: string;       // string because it's a form input
  cadence: BillingCadence;
  dueDate: string;
  autopay: boolean;
  note: string;
};
```

#### Additional Storage Keys

```ts
const deletedBillingStorageKey = "command-center.billing-deleted";
const paymentHistoryStorageKey = "command-center.payment-history";
const depositStorageKey = "command-center.card-deposits";
const domainStorageKey = "command-center.domain-tracker";
```

Note: `billingStorageKey` is imported from `subscription-data.ts`. The tracker defines 4 additional keys for its other data stores.

#### Helper Functions (module-scope, outside the component)

**`fmtUsd(v: number)`** — Format number as USD currency using `Intl.NumberFormat("en-US", { style: "currency", currency: "USD", maximumFractionDigits: 2 })`.

**`fmtDate(v: string)`** — Parse ISO date string and format as short date: `new Date(\`${v}T00:00:00\`).toLocaleDateString(undefined, { month: "short", day: "numeric", year: "numeric" })`.

**`todayIso()`** — Returns today's date as `"YYYY-MM-DD"`: `new Date().toISOString().slice(0, 10)`.

**`daysUntil(v: string)`** — Calculates integer days from today to the given ISO date. Negative means overdue. Uses midnight-to-midnight comparison.

**`monthlyRate(amount: number, cadence: BillingCadence)`** — Normalizes any cadence amount to a monthly equivalent:
- weekly: `amount * 4.333`
- monthly: `amount`
- quarterly: `amount / 3`
- yearly: `amount / 12`

**`statusLabel(v: string)`** — Returns a due status string:
- Negative days: `"{abs(d)}d overdue"`
- Zero: `"due today"`
- Positive: `"{d}d"`

**`statusTone(v: string)`** — Returns a CSS class name for color:
- Negative days: `"tone-warn"` (overdue)
- ≤ 14 days: `"tone-ok"` (due soon, green accent)
- > 14 days: `"tone-muted"` (far out)

**`advanceDue(cur: string, cadence: BillingCadence)`** — Advances a due date forward by one cadence period. If the result is still in the past, keeps bumping until it's in the future. Returns ISO date string.

**`emptyBillForm()`** — Returns a fresh `BillingFormState` with empty fields, defaulting `category` to the first option (`"Infrastructure"`), `cadence` to `"monthly"`, and `dueDate` to 7 days from now.

**`filterLegacy(entries: BillingEntry[])`** — Filters out any entries whose `id` is in `legacySeedBillingEntryIds`.

**`filterLegacyDomains(domains: DomainEntry[])`** — Filters out any domains whose `id` is in `legacySeedDomainIds`.

#### Load Functions

Each localStorage key has a load function that:
1. Returns empty data on server (`typeof window === "undefined"`)
2. Returns empty data if the key doesn't exist
3. Tries to parse JSON; on failure, removes the corrupt key and returns empty data
4. For billing entries and domains, also applies legacy filtering

```ts
function loadBilling(): BillingEntry[]      // reads billingStorageKey, filters legacy
function loadDeletedBilling(): BillingEntry[] // reads deletedBillingStorageKey
function loadPaymentHistory(): PaymentRecord[] // reads paymentHistoryStorageKey
function loadDeposits(): DepositRecord[]     // reads depositStorageKey
function loadDomains(): DomainEntry[]        // reads domainStorageKey, filters legacy
```

#### State

| State Variable | Type | Initial Value | Purpose |
|---|---|---|---|
| `entries` | `BillingEntry[]` | `loadBilling()` | All active billing entries |
| `domains` | `DomainEntry[]` | `loadDomains()` | Domain expiration tracking |
| `filter` | `TrackerFilter` | `"all"` | Status filter for billing table |
| `catFilter` | `string` | `"all"` | Category filter |
| `search` | `string` | `""` | Free-text search |
| `form` | `BillingFormState` | `emptyBillForm()` | Add-bill form state |
| `showBillForm` | `boolean` | `false` | Whether add-bill form is visible |
| `domainsOpen` | `boolean` | `true` | Whether domain sidebar is expanded |
| `renewingId` | `string \| null` | `null` | Domain currently being renewed |
| `renewDate` | `string` | `""` | Date input for domain renewal |
| `deletedBills` | `BillingEntry[]` | `loadDeletedBilling()` | Soft-deleted bills |
| `payments` | `PaymentRecord[]` | `loadPaymentHistory()` | Payment history |
| `deposits` | `DepositRecord[]` | `loadDeposits()` | Card deposits |
| `showDeleted` | `boolean` | `false` | Show deleted bills view |
| `showHistory` | `boolean` | `false` | Show transaction ledger view |
| `rateFilterAP` | `boolean` | `false` | Rate calc: autopay-only toggle |
| `rateFilterDM` | `boolean` | `false` | Rate calc: include domains toggle |
| `txFilter` | `TxLogFilter` | `"all"` | Ledger filter |
| `showDepositForm` | `boolean` | `false` | Whether deposit form is visible |
| `depositAmount` | `string` | `""` | Deposit form amount |
| `depositNote` | `string` | `""` | Deposit form note |
| `editingId` | `string \| null` | `null` | Bill currently being inline-edited |
| `editForm` | `BillingFormState` | `emptyBillForm()` | Edit form state |
| `vendorIdx` | `number` | `-1` | Vendor autocomplete highlight index |
| `vendorOpen` | `boolean` | `false` | Vendor autocomplete dropdown open |

Also: `billRef = useRef<HTMLDivElement | null>(null)` for the add-bill form container.

#### Persistence (useEffect hooks)

Five `useEffect` hooks write state to `localStorage` on every change:

```ts
useEffect(() => { window.localStorage.setItem(billingStorageKey, JSON.stringify(entries)); }, [entries]);
useEffect(() => { window.localStorage.setItem(domainStorageKey, JSON.stringify(domains)); }, [domains]);
useEffect(() => { window.localStorage.setItem(deletedBillingStorageKey, JSON.stringify(deletedBills)); }, [deletedBills]);
useEffect(() => { window.localStorage.setItem(paymentHistoryStorageKey, JSON.stringify(payments)); }, [payments]);
useEffect(() => { window.localStorage.setItem(depositStorageKey, JSON.stringify(deposits)); }, [deposits]);
```

Plus a sixth `useEffect` to notify the shell:
```ts
useEffect(() => { onEntriesChange?.(entries); }, [entries, onEntriesChange]);
```

#### Computed Values

These are derived values computed on every render (not memoized — they are fast enough for the expected data size):

**`active`** — `entries.filter(e => e.status === "active")`

**`knownVendors`** — Unique vendor names from `entries` + `deletedBills` + `payments`, sorted case-insensitively. Used for vendor autocomplete.

**`vendorMatches`** — `knownVendors` filtered to those containing the current `form.vendor` input (case-insensitive), excluding exact matches. Only computed when `form.vendor.trim()` is non-empty.

**`usedCategories`** — Unique categories from current `entries`, sorted alphabetically. Rendered as dynamic filter buttons.

**`sorted`** — `entries` sorted by `dueDate` ascending.

**`filtered`** — `sorted` filtered by: `catFilter` (if not "all"), `search` (match across name/vendor/category/note, case-insensitive), and `filter` (dueSoon = active + 0–14 days; overdue = active + negative days; all = no status filter).

**`filteredDeleted`** — `deletedBills` optionally filtered by `catFilter`.

**`sortedDomains`** — `domains` sorted by `expirationDate` ascending.

**`activeDomains`** — `sortedDomains` filtered to `status === "active"`.

**Due-window calculations** (all for active entries only):
- `due7bills` / `due7amt` — bills due within 0–7 days + their sum
- `due14bills` / `due14amt` — 0–14 days
- `due30bills` / `due30amt` — 0–30 days
- `due60bills` / `due60amt` — 0–60 days

**Rate calculations**:
- `ratePool` — active entries, optionally filtered by `rateFilterAP` (only autopay) and `rateFilterDM` (include Domains; domains are excluded by default when `rateFilterDM` is false)
- `runRate` — sum of `monthlyRate(amount, cadence)` for all entries in `ratePool`
- `yearRate` — `runRate * 12`

**Balance calculations**:
- `totalDeposits` — sum of all deposit amounts
- `totalPayments` — sum of all payment amounts
- `cardBalance` — `totalDeposits - totalPayments`

**`billCategoryMap`** — `Map<string, string>` mapping `billId → category` for both active entries and deleted bills. Used for category filtering in the transaction ledger.

#### Mutation Functions

**`onFormField(field, value)`** — Updates the add-bill form. Special case: when `category` is set to `"Domains"`, automatically set `cadence` to `"yearly"`.

**`onEditField(field, value)`** — Updates the inline edit form. No special cases.

**`addBill(event)`** — Form submit handler:
1. Validate: name, vendor, and amount are required; amount must be > 0
2. Create `BillingEntry` with `crypto.randomUUID()` as id, `todayIso()` as `createdAt`
3. Prepend to `entries`
4. **Domain sync**: If category is `"Domains"`, also create a `DomainEntry` with the same `id`, mapping `name → domain`, `vendor → registrar`, `dueDate → expirationDate`, `autopay → autoRenew`
5. Reset form, clear filters, hide form
6. Show success toast

**`addDeposit(event)`** — Form submit handler:
1. Validate: amount must be > 0
2. Create `DepositRecord` with `crypto.randomUUID()` id, `todayIso()` date
3. Prepend to `deposits`
4. Clear form, hide form
5. Show success toast

**`markPaid(id)`** — Mark a bill as paid:
1. Find the bill; must be active
2. Advance due date with `advanceDue(bill.dueDate, bill.cadence)`
3. Create `PaymentRecord` with previous and new due dates
4. Prepend record to `payments`
5. Update the bill's `dueDate` and `lastPaidDate`
6. **Domain sync**: If category is `"Domains"`, also update the domain's `expirationDate`
7. Show success toast with new due date

**`toggleStatus(id)`** — Toggle between "active" and "paused". Shows appropriate toast.

**`delBill(id)`** — Soft delete:
1. Find the bill
2. Add to `deletedBills` with status cast to `"deleted"` (note: this is a cast — the type doesn't include "deleted" but the deleted list accepts it)
3. Remove from `entries`
4. **Domain sync**: If category is `"Domains"`, also remove from `domains`
5. Show success toast

**`restoreBill(id)`** — Restore from deleted:
1. Find in `deletedBills`
2. Prepend to `entries` with status set to `"active"`
3. Remove from `deletedBills`
4. **Domain sync**: If category is `"Domains"`, recreate the `DomainEntry`
5. Show success toast

**`renewDomain(id)`** — Update domain expiration:
1. Validate: `renewDate` must be set
2. Update domain's `expirationDate` and set status to `"active"`
3. Clear renewal UI state
4. Show success toast

**`delDomain(id)`** — Remove domain from sidebar (does not delete the billing entry).

**`permDelete(id)`** — Permanently remove from `deletedBills`. Show success toast.

**`startEdit(bill)`** — Enter inline edit mode: set `editingId` to the bill's id and populate `editForm` from the bill's fields.

**`saveEdit(event)`** — Save inline edit:
1. Validate: name, vendor, amount required; amount > 0
2. Update the entry in `entries`
3. **Domain sync logic**:
   - If category is now `"Domains"` and domain exists → update it
   - If category is now `"Domains"` and domain doesn't exist → create it
   - If category was `"Domains"` and is now something else → remove from `domains`
4. Clear `editingId`
5. Show success toast

**`cancelEdit()`** — Exit edit mode: set `editingId` to null.

#### UI Layout

The component renders a horizontal split:

```
┌──────────────────────────────────────────┬──────────────┐
│             Main Billing Area            │   Domain     │
│             (flex: 1)                    │   Sidebar    │
│                                          │  (280px /    │
│  ┌─ Stats Row ────────────────────────┐  │   36px       │
│  │ BAL | 7D | 14D | 30D | 60D |$/MO… │  │ collapsible) │
│  ├─ Filter Bar ───────────────────────┤  │              │
│  │ all|dueSoon|overdue|cats|search|+  │  │  domain1     │
│  ├─ Inline Form (hidden by default) ──┤  │  domain2     │
│  ├─ Table / Ledger ───────────────────┤  │  domain3     │
│  │  status|name|vendor|amt|due|cad|act│  │  …           │
│  │  … rows …                          │  │              │
│  ├─ Footer (sticky) ──────────────────┤  │              │
│  │  N ROWS | $total                   │  │              │
│  └────────────────────────────────────┘  │              │
└──────────────────────────────────────────┴──────────────┘
```

The main billing area has `borderRight: "1px solid var(--border)"`. The domain sidebar has `transition: "width 0.15s ease, min-width 0.15s ease"` for the collapse/expand animation.

##### Stats Row

A horizontal row of stat cells with `borderBottom: "1px solid var(--border)"`. Each cell uses the `cc-stat` class with `flex: 1`.

| Label | Value | Color Rule |
|---|---|---|
| BALANCE | `fmtUsd(cardBalance)` | Green (`var(--accent)`) if ≥ 0, warn (`var(--warn)`) if negative |
| DUE 7D | `fmtUsd(due7amt) (count)` | Red (`#e04040`) if count > 0 |
| DUE 14D | `fmtUsd(due14amt) (count)` | Warn (`var(--warn)`) if count > 0 |
| DUE 30D | `fmtUsd(due30amt) (count)` | Default |
| DUE 60D | `fmtUsd(due60amt) (count)` | Default |
| $/MO | `fmtUsd(runRate)` | Default |
| $/YEAR | `fmtUsd(yearRate)` | Default |

After the 7 stat cells, render two toggle buttons in a vertical column:
- **AP** button — toggles `rateFilterAP`. When active, the $/MO and $/YEAR only count autopay entries. Uses `cc-btn active` class when enabled.
- **DM** button — toggles `rateFilterDM`. When active, domains are included in rate calculations. Style: `fontSize: 10, padding: "2px 8px", minWidth: 36`.

##### Filter Bar

A flex row with `gap: 6, padding: "5px 12px", borderBottom`.

Contents (in order):
1. **Status filter buttons**: `all`, `dueSoon`, `overdue` — each is a `cc-btn` with `.active` when selected
2. **Separator**: A 1px-wide vertical line (only shown if `usedCategories.length > 0`)
3. **Category buttons**: Dynamically generated from `usedCategories`. Each is a `cc-btn` at `fontSize: 10, textTransform: "uppercase"`. Clicking toggles `catFilter` between that category and `"all"`. Active when `catFilter === cat`.
4. **Search input**: `cc-input` with `marginLeft: "auto", maxWidth: 180`, placeholder `"search..."`
5. **"+ add" button**: `cc-btn primary`, toggles `showBillForm`. Clicking also closes the deposit form.
6. **"+ deposit" button**: `cc-btn` with `color: "var(--accent)"`, toggles `showDepositForm`. Clicking also closes the add form.
7. **"deleted" button**: `cc-btn`, toggles `showDeleted`. Shows count badge in warn color when `deletedBills.length > 0`. Toggling on hides ledger.
8. **"ledger" button**: `cc-btn`, toggles `showHistory`. Shows count badge in accent color when `payments.length + deposits.length > 0`. Toggling on hides deleted.

##### Inline Add Bill Form

Only rendered when `showBillForm` is true. A form with `padding: "6px 8px"`, `borderBottom`, `background: "var(--surface)"`.

Fields in a flex row with `flexWrap: "wrap", gap: 4, alignItems: "end"`:
- **name**: `cc-input`, width 120
- **vendor**: `cc-input`, width 100, with autocomplete dropdown. Keyboard navigation: ArrowDown/ArrowUp to highlight, Enter to select. The dropdown renders a list of `vendorMatches` in a positioned div below the input, showing each vendor with highlight styling on the selected index.
- **amount**: `cc-input`, type number, min 0, step 0.01, width 80
- **due**: `cc-input`, type date, width 130, cursor pointer. Calls `showPicker?.()` on click.
- **cadence**: `cc-input` select, width 100, height 30. Options from `billingCadenceOptions`.
- **category**: `cc-input` select, width 110, height 30. Options from `billingCategoryOptions`.
- **autopay**: Checkbox label, 12px font, muted color
- **save** / **cancel** buttons at `marginLeft: "auto"`

Special behavior: when category is set to `"Domains"`, cadence automatically changes to `"yearly"`.

##### Inline Deposit Form

Only rendered when `showDepositForm` is true. Similar styling to add-bill form.

Fields:
- **amount**: `cc-input`, type number, width 100
- **note**: `cc-input`, width 200, placeholder "e.g. card funding"
- **save** / **cancel** buttons

##### Main Table (3 views)

The table area occupies `flex: 1, overflow: "auto"` and renders one of three views based on toggle state:

###### View 1: Billing Table (default — neither `showHistory` nor `showDeleted`)

**Header row**: 7-column grid with `gridTemplateColumns: "80px 1fr 1fr 100px 110px 90px 160px"`. Columns: status, name, vendor, amount, due, cadence, actions. Sticky at `top: 0`, `background: "var(--surface)"`, `zIndex: 1`.

**Empty state**: "No billing rows. Click "+ add" to add." in muted text.

**Each row**: Same grid layout. When `editingId === entry.id`, renders inline edit form instead of display row.

Display row contents:
- **Status**: Badge with `statusLabel(dueDate)`. Background: `var(--warn-dim)` if overdue, `var(--accent-dim)` if ≤14 days, transparent otherwise. Text color: `var(--warn)` if overdue, `var(--accent)` if ≤14 days, `var(--muted)` otherwise.
- **Name**: White text, bold, with optional badges:
  - `AUTO` badge (green accent on accent-dim background) if `autopay` is true
  - `EXPIRING` badge (red on red-dim background) if not autopay, active, and due within 0–6 days
- **Vendor**: `var(--warn)` color, opacity 0.85
- **Amount**: `var(--accent)` color, bold, formatted as USD
- **Due**: White text, 12px, formatted with `fmtDate`
- **Cadence**: `var(--fg)` color, 12px
- **Actions**: 4 buttons — `paid` (primary, disabled if not active), `edit`, `pause`/`resume` (text changes by status), `del`

Inline edit row: Same 7 columns but with input fields replacing display text. Fields: editing label, name input, vendor input, amount number input, date input (with showPicker), cadence select, and actions column with autopay checkbox + save/cancel buttons.

**Footer row**: Sticky at `bottom: 0`, same grid. Shows row count and sum of filtered amounts.

###### View 2: Deleted Bills (`showDeleted` is true)

Same 7-column grid as the billing table. Header and rows styled identically.

Row differences:
- Status shows `"deleted"` in warn color
- Amount and dates in muted color
- Actions: `restore` button and `del` button (permanent delete, warn color)

Footer: row count and sum of deleted amounts.

Empty state: "No deleted bills" + optional category suffix.

###### View 3: Transaction Ledger (`showHistory` is true)

A combined view of payments and deposits.

**Header**: Accent-colored "transaction log" label with count ("X of Y"). Right side: three filter buttons (`all`, `payments`, `deposits` — `TxLogFilter`).

**Entry format**: 6-column grid with `gridTemplateColumns: "80px 1fr 1fr 100px 110px 1fr"`.

Transactions are merged from `payments` (as type "payment") and `deposits` (as type "deposit"), sorted by date descending.

Filtering: by `txFilter` type, and by `catFilter` (payments only — looked up via `billCategoryMap`).

Payment rows:
- Type label: `"payment"` in warn color
- Bill name, vendor (warn color, opacity 0.85)
- Amount: `−{fmtUsd(amount)}` in warn color
- Date in white
- Due date transition: `"{prev} → {new}"` in muted

Deposit rows:
- Type label: `"deposit"` in accent color
- Note (or "Card deposit" fallback), empty vendor column
- Amount: `+{fmtUsd(amount)}` in accent color
- Date in white

**Footer**: Sticky, shows row count and net total (deposits positive, payments negative).

Empty state: "No transactions/payments/deposits yet."

##### Domain Sidebar

A collapsible sidebar at the right side of the layout.

**Collapsed state**: 36px width, shows only the toggle button (`«`).

**Expanded state**: 280px width, shows:
- **Header**: `"domains"` label + active count (`"{activeDomains.length} tracked"` in accent color) + toggle button (`»`)
- **Domain list**: Each domain is a `cc-row` with column layout:
  - Top row: domain name (white, bold, 13px) + status label (using `statusLabel` + `statusTone`)
  - Info row: expiration date (accent color) + separator dot + registrar (warn color, opacity 0.85)
  - Action row: Either renewal UI (date input + save/cancel) or two buttons (renew + del)

**Renew flow**: Clicking "renew" sets `renewingId` to the domain id and pre-fills `renewDate` to one year from now. The inline date picker + save/cancel buttons replace the renew/del buttons. Saving calls `renewDomain`, which updates the expiration date and resets the renewal UI.

---

## Step 4: Run

No configuration is needed. Start the development server:

```bash
npm run dev
```

Open `http://localhost:3000`. The dashboard will show the Subscriptions panel (initially empty). Click the `02` sidebar button or the "open" button on the panel to enter the Subscriptions module.

Start by clicking "+ add" to create your first bill, or "+ deposit" to record a card deposit.

---

## Appendix A: Complete Type Definitions

All types are defined in `src/lib/subscription-data.ts`. See [Section 3.1](#31-subscription-data-subscription-datats) for the complete type definitions.

Summary of all types:

| Type | Fields | Purpose |
|---|---|---|
| `BillingCadence` | `"weekly" \| "monthly" \| "quarterly" \| "yearly"` | Union of billing frequencies |
| `BillingEntry` | id, name, vendor, category, amount, cadence, dueDate, autopay, status, note, createdAt, lastPaidDate? | Core subscription record |
| `PaymentRecord` | id, billId, billName, vendor, amount, paidDate, previousDueDate, newDueDate | Payment history entry |
| `DepositRecord` | id, amount, date, note | Card deposit record |
| `DomainEntry` | id, domain, registrar, expirationDate, autoRenew, note, status | Domain tracker record |
| `BillingFormState` | name, vendor, category, amount (string), cadence, dueDate, autopay, note | Form state (local type) |
| `TrackerFilter` | `"all" \| "dueSoon" \| "overdue"` | Status filter options (local type) |
| `TxLogFilter` | `"all" \| "payments" \| "deposits"` | Ledger filter options (local type) |

---

## Appendix B: localStorage Key Reference

| Key | State Variable | Type | Description |
|---|---|---|---|
| `command-center.billing-tracker` | `entries` | `BillingEntry[]` | Active billing entries |
| `command-center.billing-deleted` | `deletedBills` | `BillingEntry[]` | Soft-deleted billing entries |
| `command-center.payment-history` | `payments` | `PaymentRecord[]` | Payment transaction history |
| `command-center.card-deposits` | `deposits` | `DepositRecord[]` | Card deposit records |
| `command-center.domain-tracker` | `domains` | `DomainEntry[]` | Domain expiration entries |

All keys store JSON arrays. All are read on component mount with error handling: if parsing fails, the corrupt key is removed and the component starts with empty data.

The billing key (`command-center.billing-tracker`) is the only key shared between the shell and the component — the shell reads it to populate the dashboard bills preview.

---

## Appendix C: Implementation Notes & Edge Cases

### Domain–Billing Sync

Billing entries with `category === "Domains"` are automatically mirrored in the domain sidebar. The sync is maintained across all mutations:

| Operation | Billing Effect | Domain Effect |
|---|---|---|
| Add bill (category=Domains) | Creates entry | Creates matching domain |
| Mark paid (category=Domains) | Advances due date | Updates expiration date |
| Delete bill (category=Domains) | Soft-deletes entry | Removes domain |
| Restore bill (category=Domains) | Restores entry | Recreates domain |
| Edit: change to Domains | Updates entry | Creates domain if none, or updates existing |
| Edit: change from Domains | Updates entry | Removes domain |

The billing entry and domain share the same `id`. Field mappings:
- `BillingEntry.name` → `DomainEntry.domain`
- `BillingEntry.vendor` → `DomainEntry.registrar`
- `BillingEntry.dueDate` → `DomainEntry.expirationDate`
- `BillingEntry.autopay` → `DomainEntry.autoRenew`
- `BillingEntry.note` → `DomainEntry.note`

### Category Auto-Cadence

When the add-bill form's category is set to `"Domains"`, the cadence automatically switches to `"yearly"`. This does not apply to the edit form.

### Due Date Advancement

The `advanceDue` function bumps the due date forward by the cadence period. If the result is still in the past (e.g., a bill was overdue for multiple periods), it keeps bumping until the date is in the future. This ensures "mark paid" always produces a future due date.

### Rate Filter Toggles

The $/MO and $/YEAR stats are computed from `ratePool`, which defaults to all active entries **excluding** domains. The toggle buttons modify this pool:
- **AP** (autopay): When active, only includes entries where `autopay === true`
- **DM** (domains): When active, includes entries where `category === "Domains"` (they are excluded by default)

Both toggles can be active simultaneously.

### Vendor Autocomplete

The vendor field in the add-bill form provides autocomplete suggestions from historical vendors. The `knownVendors` list combines vendors from active entries, deleted entries, and payment records (sorted case-insensitively). The dropdown supports keyboard navigation (ArrowUp/ArrowDown/Enter) and mouse selection. The dropdown closes on blur with a 150ms delay to allow click events to fire.

### Inline Editing

Clicking "edit" on a billing row replaces the display row with an inline edit form in the same 7-column grid. The edit form includes all fields. Pressing "save" validates and updates the entry, including domain sync logic. Pressing "cancel" exits edit mode without changes. Only one row can be in edit mode at a time.

### View Toggle Exclusivity

The `showDeleted` and `showHistory` toggles are mutually exclusive: enabling one disables the other. Both being false shows the main billing table.

### Toast Usage

All mutations provide feedback via the toast system:
- Success: default variant with title and description
- Validation failure: error variant with title and description
- Not-found errors: error variant for defensive checks

### Form Toggle Exclusivity

The "+add" and "+deposit" buttons are mutually exclusive: clicking one closes the other's form.

### Ledger Category Filtering

When the `catFilter` is active and viewing the transaction ledger, payment entries are filtered by looking up their `billId` in the `billCategoryMap`. Deposits are never category-filtered (they don't belong to a category). The map includes both active and deleted bill IDs so that payments for deleted bills still show their correct category.
