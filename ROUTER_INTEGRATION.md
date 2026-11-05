# Router Integration (Phase 1 draft)

Status: interface proposed. **No vendor adapter will be built until the actual router model and firmware are confirmed** (Phase 1 question Q1).

## RouterAdapter contract (draft)

```ts
type OpResult<T> =
  | { ok: true; data: T; verified: boolean }
  | { ok: false; code: 'UNSUPPORTED_OPERATION' | 'ROUTER_UNREACHABLE' | 'AUTH_FAILED'
        | 'REJECTED_BY_ROUTER' | 'VERIFICATION_FAILED' | 'TIMEOUT'; message: string };

interface RouterCapabilities {
  listClients: boolean; clientDetails: boolean; connectionHistory: boolean;
  pushEvents: boolean;      // true = real router events, false = poll-derived only
  macBlock: boolean; macAllowlist: boolean; kickClient: boolean;
}

interface RouterAdapter {
  readonly capabilities: RouterCapabilities;
  connect(): Promise<OpResult<void>>;
  disconnect(): Promise<void>;
  getRouterInfo(): Promise<OpResult<RouterInfo>>;
  getRouterStatus(): Promise<OpResult<RouterStatus>>;
  getConnectedClients(): Promise<OpResult<ClientInfo[]>>;
  getClientDetails(mac: Mac): Promise<OpResult<ClientInfo>>;
  getConnectionHistory(range: DateRange): Promise<OpResult<RouterEvent[]>>;
  addMacAddress(mac: Mac, meta: MacMeta): Promise<OpResult<void>>;   // router-side allowlist/registry, if supported
  removeMacAddress(mac: Mac): Promise<OpResult<void>>;
  blockMacAddress(mac: Mac, idempotencyKey: string): Promise<OpResult<void>>;
  unblockMacAddress(mac: Mac, idempotencyKey: string): Promise<OpResult<void>>;
  isMacBlocked(mac: Mac): Promise<OpResult<boolean>>;
  synchronizeRouterState(): Promise<OpResult<RouterSnapshot>>;
}
```

Rules:
- Every write is **idempotent**. Blocking an already-blocked MAC returns success, with verification.
- Every write is followed by a **read-back verification**. `verified: false` is never reported as success.
- Unsupported operations return `UNSUPPORTED_OPERATION`. They never return a fake success.

## Known management interfaces by platform

To be confirmed against official documentation for the specific firmware version once it's known.

| Platform | Interface | Clients | Block/unblock | Real events |
|---|---|---|---|---|
| OpenWrt (21.02+) | ubus over rpcd JSON-RPC (HTTPS), or SSH with a restricted key; UCI config | `hostapd.*` `get_clients`, DHCP leases | `wireless.*.macfilter=deny` + `maclist`, plus `hostapd del_client` to kick | Yes: hostapd ubus association/disassociation events |
| MikroTik RouterOS 7 | Official REST API (`/rest/...`) over HTTPS | WiFi registration table, DHCP leases | WiFi access-list entries | Partial (log, or polling) |
| UniFi Network | Controller API (local or official integration API) | Yes | Yes (block station) | Yes (controller events) |
| TP-Link Omada | Official Omada Open API (controller) | Yes | Yes (block client) | Yes (controller logs) |
| Typical stock consumer router | Usually none documented | Varies | Varies | Rare |

**Consumer routers without a documented API are not supported for production use.** The recommendation is to put a supported access point in front of them or replace them.

## Event provenance

`wifi_events.source` is one of:
- `ROUTER`: reported by the router itself (push event or router log).
- `POLL_OBSERVED`: inferred by the app by comparing successive client snapshots. The timestamp is the observation time, not the true connect time.
- `APPLICATION`: an admin action or a system event (for example, router connection lost).

History from before monitoring started is never fabricated.

## MAC address caveats

- A MAC with the locally administered bit (`first_octet & 0x02`) set is flagged as likely **randomized**. The same physical device may show up with different MACs on different SSIDs, or after the OS rotates its address.
- MAC filtering can be bypassed by spoofing. Treat it as access management, not as a security boundary. Strong alternatives are WPA3/WPA2-Enterprise (802.1X) or per-device PSKs where the hardware supports them.

## Testing

- `SimulatedRouterAdapter` is used for unit and integration tests and for local development. It's selectable only when `NODE_ENV !== 'production'`, and the app refuses to start in production with it configured.
- A separate `test:hardware` suite runs against an authorized physical router. It's opt-in through environment variables and never runs in CI by default.
