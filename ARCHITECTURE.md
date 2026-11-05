# Architecture (Phase 1 draft)

Status: proposed, pending approval. See `docs/PHASE1_FINDINGS.md` for the rationale.

## System context

```
 Browser (admin)                          Router site LAN (private)
      │ HTTPS                                ┌──────────────────────────┐
      ▼                                      │  wifi-agent              │
┌───────────────────────────┐   mTLS WSS     │  - typed RPC allowlist   │
│ wifi.4everfootball.com    │◄───────────────┤  - router creds (local)  │
│ reverse proxy (TLS)       │  outbound from │  - vendor adapter        │
│  ├─ /        → web (SPA)  │  agent only    └────────────┬─────────────┘
│  ├─ /api/v1  → api        │                             │ LAN only
│  └─ /agent/v1→ api (agent │                             ▼
│               gateway)    │                       Physical router / AP
└────────────┬──────────────┘
             ▼
        PostgreSQL (app data, sessions, audit, job queue)
```

## Components

| Component | Responsibility |
|---|---|
| `apps/api` | REST API (`/api/v1`), auth, RBAC, MAC/device/log services, agent gateway, background jobs (polling, reconciliation, health checks) |
| `apps/web` | React SPA admin dashboard. It holds no secrets and decides nothing about authorization. |
| `apps/agent` | Runs on the router LAN. Implements `RouterAdapter` for the actual vendor and executes only typed commands. |
| `packages/shared` | `RouterAdapter` interface, capability types, Zod schemas, error codes |

## Backend layering (`apps/api/src`)

```
modules/<feature>/   controller.ts → service.ts → repository.ts, schemas.ts
  auth, users, groups, permissions, routers, devices, mac-addresses, wifi-events, audit, settings
middleware/          authenticate, authorize(permission), csrf, rate-limit, error-handler
router-integration/  RouterAdapter registry, AgentRemoteAdapter, SimulatedRouterAdapter (tests/dev only)
jobs/                poll-clients, reconcile-router-state, router-health
bootstrap/           super-admin provisioning (advisory-locked, once)
db/                  migrations/, kysely client
```

Controllers never talk to routers directly. They call services, and services call `RouterAdapter` through the registry.

## Router operation flow (consistency guarantee)

1. The service validates the request and checks the permission server-side.
2. In one transaction, it inserts a `router_operations` row (`PENDING`, with an idempotency key) and an audit entry (`REQUESTED`).
3. A job sends the command through the adapter. The status moves to `SENT`.
4. The adapter applies the change, then **reads back** router state to verify it (for example, by calling `isMacBlocked`).
5. In one transaction, the status moves to `CONFIRMED`, `FAILED` or `UNSUPPORTED`. The `mac_block_rules` state and the audit entry with the result are updated only on `CONFIRMED`.
6. The reconciliation job periodically compares router state with the DB and flags drift. It never overwrites silently.

The UI shows each operation's live state (pending, confirmed, failed, unsupported).

## Deployment shape (detailed in Phase 6)

- Docker images for `api`/`web` and for `agent`.
- A reverse proxy (Caddy or nginx) with automatic TLS and HSTS.
- Managed or self-hosted PostgreSQL with scheduled `pg_dump` and WAL backups, plus a restore test.
- An independent CI/CD pipeline for this repository.
