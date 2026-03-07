# Command Center — Deployments Module

An open-source, local-first command center dashboard with a modular architecture. This release includes the **Deployments** module for full Vercel project management — deployments, environment variables, and domains — all from a single local dashboard.

Command Center is designed as a **modular platform**. The base application ships with a shell, navigation, and dashboard overview. Each module is an independent feature set that plugs into the shell. This document provides the Deployments module. Additional modules (Email, Subscriptions, Automation, and more) will be released as separate instruction sets over time — same architecture, same shell, no rework required.

---

## Quick Setup

1. Go to https://vercel.com/account/settings/tokens and create a token
2. Copy the token ID and save it (this is how your app accesses Vercel)
3. Download the `/deployments/` folder from this repo
4. Give this folder as context to an AI agent and ask it to follow the instructions and build it
5. Follow the instructions the agent gives you

---

## Table of Contents

1. [How This Works](#how-this-works)
2. [Architecture Overview](#architecture-overview)
3. [Step 1: Scaffold the Application](#step-1-scaffold-the-application)
4. [Step 2: Build the Base Shell](#step-2-build-the-base-shell)
5. [Step 3: Build the Deployments Module](#step-3-build-the-deployments-module)
6. [Step 4: Configure and Run](#step-4-configure-and-run)
7. [Appendix A: Complete Type Definitions](#appendix-a-complete-type-definitions)
8. [Appendix B: Vercel API Endpoint Reference](#appendix-b-vercel-api-endpoint-reference)
9. [Appendix C: Implementation Notes & Edge Cases](#appendix-c-implementation-notes--edge-cases)

---

## How This Works

This document is a **complete specification** — it contains every detail needed to build the application from zero. You do not need to write any code yourself. The intended workflow is:

1. **You** set up a Vercel account and create an API token
2. **You** hand this document to an AI coding agent (Copilot, Cursor, Windsurf, Cline, etc.)
3. **The agent** reads these instructions and builds the full application
4. **You** add your Vercel token and run the app

The application is a Next.js web app that runs locally on your machine. It talks to Vercel's API through your personal token to manage your projects, deployments, environment variables, and domains — all from one dashboard.

### What You Get

- A full **Command Center shell** with navigation sidebar, topbar, and modular view system
- A **Command dashboard** (home page) with overview panels — the Deployments panel is live, others are placeholders for future modules
- A complete **Deployments module** with:
  - Project list with search and bulk delete
  - One-click deploy, redeploy, promote, and rollback
  - Environment variable management with a reusable global template store
  - Domain management (assign, redirect, branch-pinning)
  - Build log viewing
  - Analytics status detection

### Modular Design

The shell is built so that future modules plug in without changing existing code. Each module is:
- A **navigation entry** in the sidebar
- A **component** that renders in the main area when its nav item is active
- An optional **topbar section** for module-specific controls
- A **card on the dashboard** for at-a-glance status

Today, only the Deployments module is functional. The navigation sidebar already includes placeholder entries for Email, Subscriptions, Automation, and Settings — these show a "module queued" message when clicked. As new module instruction sets are released, you follow the same workflow: hand the instructions to your agent, and it adds the module into your existing shell.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                        Browser (React)                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ CommandCenterShell                                   │    │
│  │  ├── Sidebar (navigation: Command, Deployments, ...) │    │
│  │  ├── Topbar (view label + module-specific controls)  │    │
│  │  └── Main Area                                       │    │
│  │      ├── CommandPage (dashboard overview)             │    │
│  │      │   └── Deployments card (recent deploys)        │    │
│  │      ├── DeploymentDashboard (full module)            │    │
│  │      └── [Future modules plug in here]                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────┬───────────────────────────────────────────┘
                   │ fetch() to local Next.js API routes
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                  Next.js API Routes (Server)                 │
│                                                              │
│   /api/vercel/projects/action   → create, delete, rename     │
│   /api/vercel/env               → list, create, update, del  │
│   /api/vercel/deploy            → trigger new deployment     │
│   /api/vercel/deployments       → list, redeploy, promote,   │
│                                   build logs                 │
│   /api/vercel/domains           → list, add, remove          │
│   /api/vercel/globals           → local env template CRUD    │
│                                                              │
│   [Future modules add their own routes here]                 │
└──────────────┬───────────────┬───────────────────────────────┘
               │               │
               ▼               ▼
    ┌──────────────┐   ┌────────────────┐
    │ Vercel REST  │   │  Local File    │
    │     API      │   │  .env-store/   │
    │ api.vercel.  │   │  globals.json  │
    │     com      │   │                │
    └──────────────┘   └────────────────┘
```

**Key principle**: The browser never talks to external APIs directly. All API calls go through Next.js server routes which hold authentication tokens server-side. This keeps your secrets safe.

---
