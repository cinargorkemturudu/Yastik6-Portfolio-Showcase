# Yastık6 – Architectural Blueprint (WIP)

⚠️ **Status: Work in Progress**  
This repository documents an evolving system architecture. Implementation is intentionally incomplete.  
I prefer delaying code over committing to a suboptimal design.

---

## Overview

**Yastık6** is a privacy-first, local-first personal finance application focused on portfolio and lot-based asset tracking.

The core principle is simple:

> The application must function without ever knowing what the user owns.

All sensitive data lives exclusively on the user’s device.  
The backend exists only as an anonymous price provider.

This repository is not a codebase.  
It is an architectural and product-thinking showcase.

---

## Problem

Most finance applications:
- Store user portfolios on servers
- Track ownership, balances, and behavior
- Introduce privacy, trust, and regulatory risks
- Become opaque systems users cannot reason about

---

## Solution

Yastık6 is designed with strict constraints:

- **Local-first**
- **Offline-capable**
- **No user accounts**
- **No authentication**
- **No backend knowledge of user state**
- **Deterministic calculations on-device**

The backend never knows:
- What the user owns
- How much they own
- When they bought or sold
- Their profit or loss

---

## High-Level Architecture

### Frontend (Flutter)
- UI rendering
- User interaction
- Business logic
- Profit/Loss calculations
- State management
- Offline operation

### Local Data
- SQLite database
- Transactions stored as immutable lots
- User settings
- Cached price snapshots

### Backend (.NET Web API)
- Stateless
- Anonymous
- Read-only
- Provides current and historical prices only
- No persistence of user-related data

---

## Data Flow

1. User records a transaction → **Local SQLite**
2. App fetches price data → **Remote API**
3. Calculations happen → **On device**
4. UI updates → **Purely local state**

At no point does the backend infer or store user ownership.

---

## Core Design Decisions

- **Lot-based tracking**
  - Every transaction is independent
  - Enables granular PnL analysis

- **Backend ignorance**
  - Reduces attack surface
  - Simplifies compliance
  - Improves trust

- **Offline-first**
  - App remains usable without network
  - Last known prices are cached

- **Deterministic state**
  - Same inputs always produce same outputs
  - Easier debugging and reasoning

---

## Current State of the Project

- Architecture is actively being refined
- Public code is withheld until structure stabilizes

This is a deliberate choice.


---

## Design Principles

- Privacy by default
- Local-first over cloud-first
- Explicit trade-offs over convenience
- Architecture before implementation
- Product thinking over feature accumulation

---

## Open Questions

- Final state management approach
- Snapshot vs stream-based updates
- Price caching strategy
- Cross-asset normalization
- Long-term extensibility

These are intentionally unresolved.

---

## What This Repository Is

- Architectural blueprint
- Product thinking showcase
- System design narrative
- Hiring-facing documentation

---

## What This Repository Is Not

- A tutorial
- A boilerplate
- A demo app
- An open-source SDK

---

## Motivation

This repository exists to demonstrate how I think about:
- System boundaries
- Trust and privacy
- Trade-offs
- Long-term maintainability
- Product-aligned engineering

Code will follow architecture.  
Not the other way around.
