# Command Center — Deployments Module

An open-source, local-first command center dashboard with a modular architecture. This release includes the **Deployments** module for full Vercel project management — deployments, environment variables, and domains — all from a single local dashboard.

Command Center is designed as a **modular platform**. The base application ships with a shell, navigation, and dashboard overview. Each module is an independent feature set that plugs into the shell. This document provides the Deployments module. Additional modules (Email, Subscriptions, Automation, and more) will be released as separate instruction sets over time — same architecture, same shell, no rework required. Feel free to contribute module if you like the design.

---

## Quick Setup

1. Download the `/deployments/` folder from this repo
2. Give this folder as context to an AI agent and ask it to follow the instructions and build it
3. The agent will build the full application, then walk you through final setup:
   - It will ask you to create a Vercel API token (with a link and clear scope instructions)
   - It will offer to create the `.env.local` file and insert the token for you
   - It will ask about optional customizations (GitHub org name, default team)
4. Run the app

You do not need to set anything up beforehand — the agent handles everything after the build.
