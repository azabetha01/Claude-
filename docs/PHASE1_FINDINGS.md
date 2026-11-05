# Phase 1: Discovery Findings

Project: 4EverFootball Wi-Fi Management System (`wifi.4everfootball.com`)
Status: Phase 1 done. Waiting for approval and answers to the blocking questions in section 6 before Phase 2.

Related drafts: [`ARCHITECTURE.md`](../ARCHITECTURE.md), [`ROUTER_INTEGRATION.md`](../ROUTER_INTEGRATION.md).

---

## 1. Repository inspection

| Item | Expected (from brief) | Actually found |
|---|---|---|
| Existing repo `/home/vagrant/4everfootball` | pnpm monorepo, `apps/mobile` | **Not reachable.** This cloud session doesn't have that path. The repo isn't one of the GitHub repositories this session can access, and no local computer folder is connected. |
| Repo in this session | n/a | `azabetha01/Claude-`: one commit containing only `README.md` and `LICENSE` (BSD-3). No code, no package manifest. |
| Repo visibility | n/a | **Public.** This repo will hold security-sensitive code (auth, router control), so make it private before real infrastructure details are committed. |
| Node.js | 20.20.2 (existing env) | The sandbox has 22.23.4. **Node 20 reached end-of-life in April 2026**, so the new app should target Node 24 LTS (Node 22 also works). |
| pnpm | used by monorepo | 10.28.0 available |
| PostgreSQL | required | 16.15 client available. Docker is also available, so tests can run against a real Postgres. |

Because I couldn't inspect the existing 4EverFootball repo, I make **no assumptions about its backend, database or auth**, as the brief asked. Nothing in it can be modified from here, so there's no risk to existing production workflows.

## 2. Monorepo vs separate repository

**Recommendation: a separate repository** (this one, made private, or a new private `4everfootball-wifi`), structured internally as its own small pnpm workspace.

Reasons:
- **Blast radius.** The app controls physical network access. Keeping it out of the main monorepo keeps its secrets, CI permissions and deploy credentials away from the mobile app's pipeline, and the reverse.
- **Independent deployment** is a hard requirement. A separate repo gets that without adding conditional pipelines to the main monorepo.
- **Different release cadence and audience.** Only a few admins use it. The mobile app has public users and app-store releases.
- **Nothing to share yet.** Until I see the main repo, there's no known shared code (auth, UI kit) worth coupling to. If useful shared code turns up later, it can be published as a package.

If you'd rather keep it inside the monorepo, it would live as `apps/wifi-api`, `apps/wifi-web` and `apps/wifi-agent` with its own deploy workflow. To do that, I need the monorepo added to this session (see question Q2).

## 3. Recommended technical stack

| Concern | Choice | Why |
|---|---|---|
| Runtime | Node 24 LTS, TypeScript (strict) | Supported LTS |
| HTTP | Fastify | Fast, built-in schema validation, mature plugins (helmet, rate-limit, cookie, CSRF) |
| Validation and OpenAPI | Zod schemas used to generate OpenAPI 3.1 | One source of truth for request validation and API docs |
| Database access | Kysely (typed SQL builder) with plain SQL migrations | Explicit SQL, parameterized queries, no heavy ORM magic |
| Background jobs | pg-boss (job queue in Postgres) | No Redis to run, and job state is transactional with app data |
| Password hashing | Argon2id | Current OWASP recommendation |
| Sessions | Server-side sessions: opaque random token in an `HttpOnly; Secure; SameSite=Strict` cookie, only a SHA-256 hash stored in `admin_sessions` | Instant revocation, no JWTs in the browser |
| CSRF | Synchronizer token required on every state-changing request | Defense in depth alongside SameSite cookies |
| Logging | pino (structured JSON) with secret redaction | |
| Frontend | React + Vite + TanStack Query/Table, served from the same origin | Same origin means a strict CORS policy (no cross-origin allowed) |
| Router agent | Separate small Node service, the same TS codebase, running on the router's LAN | See section 4 |
| Tests | Vitest + Testcontainers (real Postgres) + `SimulatedRouterAdapter` | |

## 4. Router connectivity: options evaluated

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **1. LAN agent with outbound authenticated connection** | No inbound ports on the router network. Router credentials never leave the LAN. Works behind NAT/CGNAT. The agent exposes only a fixed allowlist of typed commands. | One small always-on device is needed on the LAN (a Raspberry Pi, a mini-PC or the router itself on OpenWrt). | **Recommended default** |
| 2. Site-to-site VPN (WireGuard) | Backend talks to the router's API directly. Simple adapters. | Router credentials live on the public server. The VPN tunnel widens the attack surface into the LAN. Needs firewall rules that allow only the router's management port. | Good fallback, especially when the router itself runs WireGuard (OpenWrt, MikroTik) |
| 3. Vendor cloud or controller API (UniFi, Omada, Meraki) | Official, often supports multiple sites. | Depends on the vendor, write support is uneven, and there are cloud availability and rate limits. | Use when the hardware is managed this way. It plugs in as just another adapter |

**Agent security design (option 1):**
- The agent dials out to `wss://wifi.4everfootball.com/agent/v1` using **mutual TLS**. Each agent has its own client certificate and is enrolled with a one-time code a Super Admin generates. The backend can revoke it.
- The protocol is a **closed set of typed RPCs**, the same ones as `RouterAdapter` (`blockMac(mac)`, `getConnectedClients()` and so on). Arguments are validated on both sides. There is **no shell or exec command of any kind**.
- Every command carries an **idempotency key** and a server signature. The agent rejects replays and expired commands.
- Router admin credentials are stored only on the agent host (file permissions 0600 or the OS keyring). They are never sent to the backend or the frontend.
- The router's admin UI stays LAN-only. **No port forwarding.**

## 5. Router integration requirements

Detailed in [`ROUTER_INTEGRATION.md`](../ROUTER_INTEGRATION.md). Summary:

- **The router model and firmware are unknown, and that is the main blocker for Phase 3.** Support depends heavily on the vendor:
  - **Good programmatic support:** OpenWrt (ubus/rpcd JSON-RPC, hostapd events), MikroTik RouterOS 7 (REST API), UniFi and Omada controllers.
  - **Typically no documented API:** most consumer ISP or retail routers (stock TP-Link Archer, Netgear, ISP-supplied boxes). Scraping their web UI is fragile and breaks on firmware updates. It isn't suitable for production, so I'd recommend replacing such a router or putting an OpenWrt or MikroTik access point in front of it.
- **Real vs observed events.** Some platforms push association and disassociation events (hostapd on OpenWrt, UniFi events). Others only expose a current client table, so "connected" and "disconnected" events have to be derived by polling. Every event will carry a `source` of `ROUTER`, `POLL_OBSERVED` or `APPLICATION`, and history from before the app started will never be invented.
- **Unsupported operations** return a typed `UNSUPPORTED_OPERATION` result. Each adapter declares a capability map that the UI reads, so it never shows controls the hardware can't honor.
- **MAC limitations, which the UI and docs will state explicitly:**
  - Modern phones and laptops (iOS 14+, Android 10+, Windows 10+) use **randomized, per-network MAC addresses**. The app will flag "locally administered" MACs (bit `0x02` of the first octet) as likely randomized.
  - MAC blocking is **not strong security**. A determined user can spoof an allowed MAC. For real access control, recommend WPA2/WPA3-Enterprise or per-device keys (PPSK) where the hardware supports them.

## 6. Questions I need answered before Phase 2 and 3

**Blocking:**
1. **Q1 – Router hardware.** What are the router's exact make, model, hardware version and firmware version? Is it the main ISP gateway or a separate access point? Is there a controller (UniFi or Omada)?
2. **Q2 – Existing repo.** Do you want me to inspect `4everfootball`? If so, push it to GitHub and add it to this session. Otherwise, do you confirm the standalone-repo approach?
3. **Q3 – LAN agent host.** Is there an always-on device on the router's network that can run the agent (Raspberry Pi, mini-PC, NAS), or should the agent run on the router itself (OpenWrt only)?

**Needed by Phase 6, but useful now:**

4. **Q4 – Hosting.** Where will `wifi.4everfootball.com` run (VPS provider, cloud, existing server)? Who manages DNS for `4everfootball.com`?
5. **Q5 – Repo visibility.** Can this repo be made private?
6. **Q6 – Bootstrap output.** Printing one-time credentials to stdout means they also land in container or platform logs (Docker, systemd journal, PaaS log drains). I propose printing them **only when stdout is an interactive TTY**. Otherwise, the app writes them once to a `0600` file and logs only the file path. Is that acceptable?

## 7. Proposed additions to the entity list

The brief's tables are all kept. To meet the "don't claim success before the router confirms" and multi-site requirements, I'd add:
- `router_operations`: one row per requested router change, moving through `PENDING → SENT → CONFIRMED | FAILED | UNSUPPORTED`, with an idempotency key. The status in `mac_block_rules` is updated only from a `CONFIRMED` operation.
- `router_agents`: enrolled agents, certificate fingerprints and last-seen times.
- `networks` (sites): owns routers, for future multi-network and commercial expansion.

## 8. Phase 2 plan (for approval)

1. Scaffold the workspace: `apps/api`, `apps/web` (empty shell), `apps/agent` (empty shell), `packages/shared`.
2. Add SQL migrations for the full schema, including audit-log immutability (a separate DB role with no UPDATE/DELETE grants, a trigger guard, and a hash chain).
3. Implement Super Admin bootstrap (under a Postgres advisory lock so it's race-safe), the forced password change, and the CLI recovery command.
4. Implement auth: login, logout, sessions, throttling, CSRF, RBAC middleware, and the user, group and permission APIs.
5. Add the OpenAPI spec, plus `README.md`, `DATABASE.md`, `API.md`, `SECURITY.md` and `.env.example`.
6. Write tests: bootstrap, repeated startup, login, RBAC, group membership, unauthorized access.

Phase 2 doesn't depend on the router answer (Q1). Phase 3 does.
