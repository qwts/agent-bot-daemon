# Splitting agent-bot into a daemon and plugins

**Status:** Draft for review. This is a plan only: nothing in it has been built,
and no refactor commits come with it.
**Decision owner:** Chris
**Sources read, pinned:**

| Source | Ref |
| --- | --- |
| `qwts/agent-bot-identity` | `959daf9` (v0.13.0, after #1181 server mode) |
| `qwts/agent-comms` | `09f1a52` (v0.5.4) |
| agent-bot server API, as deployed | `https://agent-bot-gold.vercel.app/docs/api.md`, read 2026-10-27; response key `kid` `KAqTzL7HfHmOiSbT` |
| Org / SOP | `qwts/qwts-agent-org@180f6dd`, `qwts/qwts-agent-sop@ee554a3` |

## 0. Summary

- Today agent-bot is one 75k-line Node monolith (`agent-bot-identity`) plus
  `agent-comms` (12.7k lines). ADR-0645 already gave every file an owning
  module and enforces the boundaries in tests, and that map is what this plan
  is built on.
- The target is four repositories: a **daemon** host plus three **plugins**
  (identity, comms, soul-runtime). Plugins talk to each other only through
  host services and events, never by importing each other.
- The trust-boundary requirement changes more than the repo layout does.
  Today GitHub App keys, the vouch key, the device key, the inbox bearer and
  provider secrets all sit on the agent's machine. They move behind the gate
  (the agent-bot server, or a local signer the agent can't read). After that
  the local owner gate and confinement are cooperative UX, not security.
- soul-runtime becomes a pure core with five adapters (storage, secrets,
  exec, comms, clock). The daemon implements them with the filesystem,
  worktrees and the signer, and the server implements them with encrypted
  per-user storage, a vault and a sandbox. One conformance suite runs against
  both.
- The migration is about 25 small PRs. The first ten are in place in
  `agent-bot-identity` and need no new repository. The `agent-bot` CLI and
  every MCP tool keep working throughout. `take_inbox` and `gh-app-hook` give
  way to the server's notices.

## 1. Where we start

### 1.1 Facts

- **agent-bot-identity** has 286 runtime `.mjs` files outside `tests/`,
  `scripts/` and `tools/`. 47 of them are one-line re-export shims left by
  ADR-0645 step 6, which leaves 239 real modules and about 75.6k lines. On
  top of that come `keyd/` (Rust), `hooks/`, `web/` and the tests.
- **`governance/runtime-modules.json`** assigns each file to one of these
  modules: `identity`, `soul`, `comms`, `harness`, `org`, `shared`, `host`
  or `cli`. `tests/module-boundaries.test.mjs` enforces the import rules
  between them, and the published entries are `identity-host`,
  `identity-soul`, `identity-cli`, `identity-format`, `daemon-client` and
  `identity-presence`. This plan uses that map as the starting inventory.
- **The daemon already composes modules through injected ports.**
  `createIdentityRoutes({...ports})` and `createSoulRoutes({...})` in
  `agent-daemon.mjs` (around L575–589) are the seed of the plugin route
  contract. Everything else is still one 2.4k-line `if`/`switch`.
- **An extraction is already approved.** `docs/identity-extraction-plan.md`
  extracts identity to `qwts/agent-identity` as a vendored, digest-pinned
  copy. The owner answered that plan's four questions on 2026-10-11. This
  plan replaces its end state; see decision D2.
- **Server mode exists, opt-in** (`identity/server-client.mjs`, #1181):
  - RFC 8628 device enrollment;
  - a software P-256 device key in `device-key.pem` (mode 0600);
  - 15-minute DPoP-bound access tokens and Ed25519-signed responses with a
    TOFU pin;
  - `POST /api/daemon/installation-token`.
- **The deployed server (Go on Vercel)** offers:
  - device auth: device grant, enrollment tokens, OIDC;
  - DPoP and signed responses;
  - repo-scoped installation tokens;
  - enrollment tokens;
  - notices: `peek`, `take` (leases), `ack` and `release`;
  - App webhooks at `/apps/webhook/<slug>`;
  - an MCP endpoint for unmanaged browser-OAuth sessions.

  It has **no relay or RPC channel, no vault, no soul storage, no sandbox and
  no owner-approval API yet**. The brief assigns relay and RPC to the
  server, so those are server work this plan depends on (§9, phase 2).
- **The empty repos.** `qwts/agent-bot-daemon` and
  `qwts/agent-bot-soul-runtime` exist but are empty. `qwts/agent-bot-comms`
  doesn't exist.

### 1.2 Size by destination

This is the result of the inventory in Appendix A.

| Destination | Files | Lines |
| --- | ---: | ---: |
| agent-bot-soul-runtime (core + daemon adapters) | 98 | 30,284 |
| agent-bot-daemon (host, CLI, SDK, org) | 63 | 22,393 |
| agent-bot-identity (plugin) | 49 | 14,773 |
| agent-bot-comms (agent-bot side) | 14 | 4,087 |
| agent-bot-comms (today's agent-comms) | 105 | 12,683 |
| Authority moves to the gate (server) | 9 | 2,258 |
| Deprecated | 6 | 1,848 |

## 2. Inventory and dependency map

The full per-module table is **Appendix A**. It gives each module's lines,
current module, target, trust side, filesystem/worktree/keyd/comms coupling,
fan-in and fan-out, cross-repo imports and cycle membership. This section
summarises it.

### 2.1 Mapping rules

| Current ADR-0645 module | Goes to |
| --- | --- |
| `identity` | agent-bot-identity, except key custody, secret stores and App management (authority → gate), and the inbox (deprecated). |
| `soul` | agent-bot-soul-runtime: `core/` if it can become pure, otherwise `adapters/daemon/`. |
| `comms` | agent-bot-comms. |
| `host`, `cli` | agent-bot-daemon. Soul, identity and comms commands become plugin-registered commands. |
| `shared`, `harness` | `@agent-bot/sdk`, a package inside the agent-bot-daemon repo that plugins vendor at a pinned digest. |
| `org` | agent-bot-daemon `org/`. The per-soul SOP selection in `sop.mjs` goes to soul-runtime core. |

### 2.2 What couples to what

These are heuristic scans; Appendix A has the per-file flags.

- **Filesystem.** 128 of the 239 modules import `node:fs`. In ADR-0645's
  soul module, 61 of 94 do. That includes 28 of the 53 core candidates, such
  as `soul-revisions`, `soul-memory`, `agent-population` and
  `skill-library`. Making the core pure is the biggest single job in the
  plan.
- **Worktrees.** The soul ones are `setup-worktree`,
  `claude-worktree-create`, `soul-worktrees`, `soul-join`, `soul-home`,
  `soul-env*` and `confinement`. The identity ones are `agent-binding`,
  `resolve-agent`, `worktree-token`, `git-credential-bot` and
  `signed-commit`. All of these become the daemon `Executor.workspace`
  adapter or identity's binding service.
- **keyd.** The identity ones are `keyd-client`, `owner-presence`,
  `owner-decision-trust`, `soul-credentials`, `identity-apps` and `vouch`.
  The host one is `keyd-supervisor`. The soul ones are
  `soul-credential-migration` and `soul-secrets`.
- **comms imported from soul.** These are `cold-wake` → `comms-relay`,
  `soul-join`/`soul-fork`/`soul-remove` → `comms-membership`, `soul-comms`
  and `soul-remove` → `daemon-status`, `soul-build`/`soul-builder` →
  `reach-contract`, `cold-wake-settings` → `wake-webhook-store`, and
  `owner-action` → `owner-principal`. Each becomes `services.comms` or a
  `comms.*` event.
- **soul to identity goes through one door:** 15 of the 17 crossings import
  `identity/identity-soul.mjs`. That entry becomes the identity plugin's
  `identity` service object, handed to soul-runtime at `init`.
- **Plugins reaching the host.** These are `keyd-supervisor` →
  `daemon-supervisor`, `delegation-grants` and `cli/soul-env-transfer` →
  `agent-jobs`, `telegram-adapter` → `agent-daemon` and `agent-interaction`,
  and `wake-plane` → `agent-jobs` and `daemon-mcp`. All of them become host
  services (`jobs`, `services`, `mcp`).

### 2.3 Wire code duplicated across repos

The SDK `wire/` folder should own these once:

| Code | Copies |
| --- | --- |
| Binding proof | `identity/binding-proof.mjs` = agent-comms `lib/binding-proof.mjs`, byte-identical by ADR-0008 amendment 1 |
| Presence verifier | `identity/presence-verifier.mjs` + `identity-presence.mjs` = agent-comms `lib/vendor/agent-bot-presence/` (snapshot with hashes) |
| Owner-decision journal | `identity/owner-decision-journal-frame.mjs`, `-prepared`, `-bytes` ≈ agent-comms `lib/owner-decision-*` |
| Windows task jobs | `windows-task-job.mjs` in both |
| Secret store | `secret-store.mjs` in both |

### 2.4 Cycles

There are five strongly connected components at file level, found with
Tarjan's algorithm over shim-resolved imports.

| ID | Files | Crosses | How to break it |
| --- | --- | --- | --- |
| **C1** | 14: `identity/soul-credentials` → `soul-app-declarations` → `agent-population` → `soul-mode`/`soul-model` → `cold-wake-settings` → `identity/identity-soul` → `identity/soul-credentials`. The other members are `owner-action`, `identity-app-store`, `ensure-private-key`, `mint-token`, `git-credential-bot`, `credential-reconciler` and `agent-backfill`. | **identity ↔ soul.** This is the only `identity → soul` edge in the map: the `soul-app-declarations` contract. | (1) Identity stops reading soul declarations. Soul-runtime *registers* an `appDeclarations()` provider with the identity service at init (inversion, no import). (2) With server mode the server picks the App from the device's harness scope, so the declaration becomes a request hint, not an input to custody. (3) `ownerGate` moves out of `cold-wake-settings` (where `soul-runtimes` and others import it) into the host `approvals` service. |
| C2 | `agent-space` ↔ `soul-memory` | none | Both become the soul-runtime `SoulStorage.life` port. The path helpers move into the daemon storage adapter. |
| C3 | 9: `sandbox`, `sandbox-export`, `sandbox-remove`, `skill-manifest`, `sop`, `soul-comms`, `soul-env-export`, `soul-profile`, `soul-revisions` | soul ↔ org (`sop`) | Split `sop.mjs`: the org resolver goes to the daemon, per-soul selection to core. A `RevisionStore` port removes the `soul-revisions` ↔ `skill-manifest`/`soul-profile` back-edges. |
| C4 | `soul-env` ↔ `soul-env-migrate` | none | Extract the descriptor reader that both use. |
| C5 | `agent-daemon` ↔ `agent-web` ↔ `daemon-supervisor` | none (host) | Move `daemonStatus`/`daemonStateFile` into a daemon `state` module. |

At repo level, the remaining risks are soul-runtime ↔ comms (soul imports
comms today, and `wake-plane` composes both) and identity ↔ soul through the
census. Both are handled by the rule in §3.3: plugins never import plugins.

## 3. Target architecture

```text
                     ┌──────────────── gate ────────────────┐
                     │ agent-bot server (Go, Vercel)         │
                     │  device registry · policy · approvals │
                     │  App keys · installation tokens       │
                     │  vault · notices · relay/RPC          │
                     │  encrypted soul storage               │
                     │  soul-runtime (Node) + sandbox        │  ← unmanaged souls
                     └───────▲──────────────────────────────┘
                 DPoP + signed responses (TLS)
┌───────────────────────────┴─────────────── agent machine (untrusted) ──────┐
│ agent-bot-daemon (host)                                                     │
│   core: config · storage · log/audit · exec · server link · local API       │
│   plugins (in process, topological order):                                  │
│     1 agent-bot-identity   2 agent-bot-comms   3 agent-bot-soul-runtime     │
│   surfaces: `agent-bot` CLI · `agent-bot mcp` · `reach-mcp` · UI API        │
│        ▲ UDS / named pipe                                    ▲ stdio/ACP    │
│   dudles (GeniusBar UI)                              harness processes       │
└────────────────────────────────────────────────────────────────────────────┘
        │ local socket, peer-UID check
┌───────▼──────── local signer (gate side on the machine) ───────┐
│ agent-bot-signer (successor of keyd)                            │
│ non-exportable device key: Secure Enclave · TPM · Keystore      │
│ or a separate OS user when no hardware is present               │
└─────────────────────────────────────────────────────────────────┘
```

### 3.1 The repositories

| Repo | Owns | Ships |
| --- | --- | --- |
| **agent-bot-daemon** | Plugin host and loader, lifecycle, host services, server link transport (DPoP session, response verification, relay/RPC client), local API (incl. the UI), `agent-bot` CLI facade, the MCP surfaces, the hook runner, readiness/doctor, install/update, org/SOP resolver, `@agent-bot/sdk` | The daemon release: the pinned plugin set plus Node launchers for macOS, Windows and Linux |
| **agent-bot-identity** | Device enrollment, the signer (keyd → `agent-bot-signer`, Rust), the device assertion signer the server link uses, bindings and binding proofs, Agent ID records, soul tokens (vouch), server-minted GitHub tokens per harness, the git credential helper, gh shim, git hooks, owner-presence for local-only effects, audit | Plugin package plus the signer binary |
| **agent-bot-comms** | Today's agent-comms (broker, CLI, worker, principal client, channels, A2A, owner-decision journal), plus agent-bot's comms modules (relay, membership, wake listen/dispatch, task turns, webhook wakes), the notices client (server), and chat channel adapters (Telegram) | Plugin package plus the `agent-comms` CLI |
| **agent-bot-soul-runtime** | Soul package format, a pure runtime core, the adapter interfaces, daemon adapters, server adapters, the conformance suite | `core/` (also runs on the server), `adapters/daemon/`, `adapters/server/` |

### 3.2 The trust boundary, per plugin

The three sides used in these tables:

- **gate** means the agent-bot server.
- **signer** means the local signer process. It runs as a separate OS user
  or behind a hardware key, and the agent can't read it.
- **agent** means the agent machine, including the daemon. Everything there
  is treated as untrusted.

The daemon runs as the same OS user as the souls it hosts (today's agent
accounts), so it is on the *agent* side and holds nothing durable that
matters.

#### Daemon

| Piece | Side | Why |
| --- | --- | --- |
| Plugin host, local API, CLI, MCP surfaces, hook runner | agent | Agents execute through them. They hold no long-lived secret. |
| DPoP ephemeral key and 15-minute access token | agent (memory only) | Short-lived and bound to the device. Today's `server-client` already keeps the ephemeral key off disk. |
| Pinned server response key | agent | Public. Integrity comes from TOFU plus a fingerprint shipped in the signed release, checked out of band. |
| Policy: which repo, scopes, rate limits, approvals | gate | The agent may request but never grant itself access. |
| Relay/RPC broker between daemons, UI and server | gate | Routing across machines needs an authority the agent can't impersonate. |
| Audit receipts | agent copy plus gate copy | The local copy can be tampered with, so the gate copy is authoritative. |

#### Identity

| Piece | Side | Why |
| --- | --- | --- |
| Device private key (today `device-key.pem`, software) | **signer** | It must be non-exportable. The plan uses Secure Enclave, TPM (NCrypt/tpm2), Android Keystore or StrongBox, or a separate OS user as the fallback. |
| JWT-bearer assertion signing for `/oauth/token` | signer | Only the signer can prove "this device". |
| Enrollment approval and device revocation | gate (`/device`, `/devices`) | The owner approves, after checking the thumbprint. |
| GitHub App private keys (today Keychain, pass, file and keyd items) | **gate** | Long-lived. The server already stores them encrypted. |
| Installation token minting | gate | It decides repo and scope. |
| Installation tokens (≤1 h, one repo) | agent (credential helper memory, cache until 5 min before expiry) | GitHub can't bind them to DPoP. See risk R2 for the mitigations. |
| Vouch / soul-token signing key (today `vouch-key.pem` 0600, readable by any process in the account) | **signer** | Holding the file lets you sign keyd grants and soul tokens as any soul. |
| Bindings registry and binding proofs | agent | Local attribution, not authority. The server re-checks the device's harness scope on every mint. |
| Owner presence (Touch ID via keyd) | signer, local-only effects | Fine for pause, stop and "show me". Never enough to grant a credential, an App change or a scope. |
| Owner approvals for policy | gate (passkey/WebAuthn on the owner's device) | A local prompt on an untrusted machine proves nothing to the gate. |
| Owner statement keys (SSH, keyd) | owner device, verified at gate | |
| gh shim, git hooks, credential helper, `resolve-agent` | agent | Cooperative enforcement against mistakes, not a security boundary. |
| Inbox bearer (`INBOX_TOKEN`, one value for the whole fleet, in Proton Pass) | removed | Notices use the device's DPoP token and harness scope. |

#### Comms

| Piece | Side | Why |
| --- | --- | --- |
| Local broker, mailbox, census | agent (owner account or a separate OS user) | Routing state. Message bodies at rest are local; see open question Q7. |
| Principal credentials (human owner, host apps) | owner keystore (dudles' keychain item, ACL'd), never readable by agent accounts | A principal speaks for the human. |
| Notices store, leases, webhook verification | gate | Already server-side. |
| Cross-machine relay, federation, A2A beyond loopback | gate | Needs an authority the agent can't impersonate. |
| Chat channel tokens (e.g. Telegram bot token) | gate | Long-lived. The adapter runs server-side or gets a lease. |

#### Soul runtime

| Piece | Side | Why |
| --- | --- | --- |
| Core logic | both | Pure, and identical on both. |
| Definition and life storage | agent (cache) plus gate (canonical, encrypted per user) | Losing the machine must not lose the soul. |
| Provider secrets (model API keys, `credentials.secrets`) | **gate** vault, leased | Long-lived. See decision D1 for keys that can't be short-lived. |
| Runtime installs | agent | Verified by the catalog digest, which ships in the signed release. |
| Revision approvals, capability grants (computer-use, autopilot) | gate | They are policy. |
| Exec, worktrees, harness processes | agent (daemon) / gate sandbox (server) | |

### 3.3 Rules that keep the split honest

1. Plugins import only `@agent-bot/sdk`, never the daemon and never each
   other. A boundary test in each repo checks this, and the daemon checks the
   pinned set.
2. Every cross-plugin need is a **host service** (named and versioned) or an
   **event**.
3. Soul-runtime `core/` imports none of `node:fs`, `node:child_process`,
   `node:os`, `node:net` or `process.env`. Its test fails on any of them.
4. Zero npm dependencies stays the rule. The SDK and the plugins arrive as
   vendored copies pinned by version and digest
   (`governance/plugins.json`), which reuses the owner's 2026-10-11
   answer 2.

## 4. Plugin contract (v1)

### 4.1 Manifest and entry

```js
// plugin.mjs — exactly one default export
import { definePlugin } from '@agent-bot/sdk';

export default definePlugin({
  name: 'agent-bot-comms',          // unique and stable
  version: '0.6.0',                 // the plugin's semver
  hostApi: 1,                       // host plugin-API major it targets
  requires: { 'agent-bot-identity': '^1.0.0' },
  optional: { 'agent-bot-soul-runtime': '>=0.1.0 <1' },
  services: ['config', 'storage', 'log', 'audit', 'server', 'exec', 'events', 'approvals'],
  serverRoutes: ['/api/daemon/notices*', '/api/daemon/relay*'],   // allowlist for services.server
  async init(ctx) { /* register; return provided services */ },
  async start(ctx) {},
  async stop(ctx, { reason, deadlineMs }) {},
});
```

The host reads `name`, `version`, `hostApi`, `requires`, `optional`,
`services` and `serverRoutes` from a static `plugin.json` next to the entry.
It runs no plugin code until the set has resolved.

### 4.2 What a plugin registers (inside `init`)

| Kind | Call | Shape | Notes |
| --- | --- | --- | --- |
| Tools (MCP) | `ctx.tools.register(tool)` | `{ name, surface: 'agent'\|'reach', description, inputSchema, auth, handler }` | `agent` is `agent-bot mcp`; `reach` is the soul's `reach-mcp`. Names must be unique across plugins. |
| Commands (CLI) | `ctx.commands.register(cmd)` | `{ path: ['soul','runtimes'], usage, parse, run, json: true, auth }` | `agent-bot <path…>`. The help text and the `--json` contract are golden-tested. |
| Routes (local API) | `ctx.routes.register(route)` | `{ method, path: '/v2/<plugin>/…', auth, schema, handler }` | `auth` is one of the existing classes (`owner`, `owner-credential`, `bearer`, `binding`, `principal`) or the new `ui-session`. The daemon enforces the class before the handler runs, as `daemon-route-auth.test.mjs` does today. |
| Legacy routes | `ctx.routes.registerLegacy(route)` | the same, for `/v0`/`/v1` paths | Only for the compatibility window. |
| Events | `ctx.events.publish(type, payload)` / `subscribe(type, fn)` | `type` is `<plugin>.<noun>.<verb>`, e.g. `comms.wake.requested` | In process, at most once. Anything durable goes through storage. Payload schemas are versioned with `schemaVersion`. |
| Harness hooks | `ctx.hooks.register(hook)` | `{ event: 'session-start'\|'pre-tool-use'\|…, run }` | Wired by the hook runner (`agent-hook.mjs` today). |
| Readiness probes | `ctx.readiness.register(probe)` | `{ id, run }` | Splits `readiness.mjs` (3.1k lines). |
| Provided services | `return { provides: { identity: {...} } }` | a plain object of functions | Dependents receive it as `ctx.deps['agent-bot-identity'].identity`. |

### 4.3 Host services (what a plugin gets)

| Service | Interface (abridged) | Daemon implementation |
| --- | --- | --- |
| `config` | `get(key)`, `section()`: read-only, schema-validated, no secrets | `~/.config/agent-bot/config.json`, the plugin's section only |
| `storage` | `kv.get/put/cas`, `blob.read/write`, `log.append/scan`, `lock(name)` | `<state>/plugins/<name>/`, 0700, atomic rename, `file-lock` |
| `secrets` | `lease(name, { scope, purpose }) → { value, expiresAt }`. No `put`. | Fetched from the gate vault through the server link and kept in memory until TTL. Never persisted. |
| `log` / `audit` | `log.info/warn/error(obj)` (redacted), `audit.append(receipt)` | `daemon-log`, `diagnostic-redact`, `audit.jsonl`, forwarded to the gate |
| `server` | `call(method, route, body)`, `subscribe(channel)` | DPoP plus response-signature verification. Calls are limited to the plugin's `serverRoutes`. |
| `exec` | `run({argv, cwd, env, timeoutMs})`, `spawn(…)`, `track(pid)` | `child-env` (minimal env), `process-ownership`, Windows job objects |
| `approvals` | `request({ action, digest, scope }) → { id, status }`, `await(id)` | Asks the gate. Local presence is used only when `scope: 'local'`. |
| `events`, `clock`, `random`, `jobs`, `ui` | as named | `ui.push(event)` feeds the UI event stream. |

### 4.4 Lifecycle and bootstrap order

```text
discover → resolve → load → init (topological order) → start (topological order) → running → stop (reverse order, deadline)
```

- **discover** reads `plugin.json` only.
- **resolve** checks semver and `hostApi`, then requires an acyclic `requires`
  graph and refuses the set otherwise.
- `init` must be idempotent and side-effect free apart from registering. It
  times out at 10 s.
- `stop` gets a deadline. On timeout the host unregisters the plugin and logs.

**Bootstrap, with identity first and hard-required:**

1. **Daemon core:** config, storage, log, a local API serving only `health`
   and `status`, and signer discovery.
2. **identity `init`:** asks the signer for device status. Unenrolled or
   revoked → **safe mode**, where only `status`, `identity enroll` and
   `doctor` work. The server link stays down and no other plugin starts.
3. **Server link `start`:** uses `identity.device.signAssertion` for
   `/oauth/token`, verifies the pinned key, then connects relay/RPC once the
   server offers it.
4. **identity `start`:** binding registry, server minting, soul tokens.
5. **comms:** `init` then `start` (requires identity).
6. **soul-runtime:** `init` then `start` (requires identity, comms
   optional). A soul with `comms: false` never asks for it.
7. The full local API and UI routes open, and `status` reports `ready`.

If a non-identity plugin fails, it is marked `degraded` with a reason, its
dependents don't start, and the rest keep running.

### 4.5 Versioning and compatibility

- **Plugin semver.** A breaking change to a provided service or a registered
  route or tool is a major bump.
- **`hostApi`** is an integer. The daemon supports `hostApi` N and N-1 for
  at least one release.
- **Wire formats are versioned on their own and never change in place:**
  binding proof `v1`, soul token `v1`, keyd grant `v1`, presence `p1`, key
  record `k1`, soul package format `1`/`2`/`3`, and server signature
  `agent-bot-sig-v1`.
- **Each daemon release pins one compatible set** in
  `governance/plugins.json` (`name`, `version`, `sha256`). CI refuses a tree
  that doesn't match the pins. Pins never move backwards, the same rule as
  the extraction plan's §6.

### 4.6 Usable on its own for testing

The SDK ships `createTestHost({ plugins, services })` with in-memory
`storage`, a fake `server` (canned signed responses), a deterministic
`clock`/`random`, and a fake identity provider: a test device key and fake
bindings. Each plugin repo runs:

- its own tests on the test host;
- the SDK's **plugin conformance suite**, which checks that registration is
  idempotent, that `start` and `stop` can be repeated, that `stop` meets its
  deadline, that there is no global state, that nothing is written outside
  `storage`, that tool, command and route names are unique, and that every
  route has an `auth` class.

## 5. Soul package format, draft v3

Formats 1 and 2 are specified in `agent-bot-identity/docs/soul-package.md`
and stay as they are. Format 3 adds the persona, separates *definition* from
*life*, and declares where a soul can run.

### 5.1 Layout

```text
NAME.soul/
  soul.json            required  manifest (formatVersion 3)
  SOUL.md              required  persona: who the agent is (voice, values, role). Stable and owner-approved.
  AGENTS.md            required  standing operating instructions. Grows through reviewed revisions.
  skills/<name>/SKILL.md          optional, Agent Skills layout (unchanged)
  hooks/<event>/<name>            optional (unchanged)
  tools.json  mcp.json  policy.json   optional, opaque to the validator (unchanged)
  bin/  workflows/  sop/  agent-sop.toml   (unchanged; bin/ always needs approval)
  .soul-state/         LIFE — excluded from the revision (as in format 2)
    memory/            notes and facts (Agent Space under ADR-0583)
    history/runs/      turn and revision facts, never prompts or outputs
    threads/  asides/  comms-derived conversation state
    runtime-pins.json  exact installed versions (unchanged)
  worktrees/           working state, excluded (unchanged)
  .claude/ .codex/ …   generated harness output, excluded when it matches the build (unchanged)
```

**Where SOUL.md fits:** the builder renders `SOUL.md` first, then
`AGENTS.md`, into each harness's instruction file (`CLAUDE.md`, `GEMINI.md`,
`.github/copilot-instructions.md`, and so on). `SOUL.md` changes only
through an owner-approved revision, with no self-change policy, while
`AGENTS.md` keeps today's growth paths. A format-2 package is read as if
`SOUL.md` were empty. Open question Q3 asks whether SOUL.md should instead
replace AGENTS.md.

### 5.2 Manifest additions (v3)

All format-2 fields are kept: `name`, `description`, `displaySeed`,
`preferredHarnesses`, `revision`, `parentRevision`, `ignore`, `harness`,
`harnesses`, `comms`, `credentials`, `runtimes`, `appearance`, `skills` and
the template fields. These are new:

| Field | Type | Meaning |
| --- | --- | --- |
| `formatVersion` | `3` | |
| `targets` | `["daemon"]`, `["server"]` or both | Where the soul may run. The server refuses packages that need host-only capabilities. |
| `capabilities` | closed object: `{ exec, network: { domains: [] }, computerUse, comms }` | *Requested* capabilities. The gate decides; the package never grants. |
| `secrets` | array of names (the successor of `credentials.secrets`) | Names only. Values come from `SecretResolver`, never from the package. |
| `env` | `NAME: "value"` strings | Non-secret environment. Secret-looking names or values are refused, as `harness.env` already does. |
| `packages` | `{ npm?: lock ref, uv?: lock ref }` | Pinned package locks, a generalisation of `harnesses.*.install.lock`. |
| `runtime` | `{ min: "0.x" }` | The lowest soul-runtime version that understands this package. |

The revision encoding is the format-2 netstring scheme with the prefix
`agent-bot-soul-package-v3\0`. `SOUL.md` is ordinary content, so it
participates like any other file.

### 5.3 Bundle (transport) versus package (definition)

- **Package**: the hashed definition inventory above.
- **Life bundle**: `soul env export` generalised. It is a manifest
  `{ agentId, revision, parts: [{ name, sha256, bytes }] }` plus parts for
  memory, history, threads, asides and runtime-pins. It **never** carries
  credentials, secrets, sign-in files, runtimes or caches. The server stores
  bundles encrypted per user.

### 5.4 Conformance test plan (both runtimes must pass)

The suite lives in `agent-bot-soul-runtime/conformance/` and runs as
`npm run conformance -- --adapters daemon|server-sim|server-staging`.
`server-sim` is an in-memory implementation of the server adapters for CI.
`server-staging` runs nightly against a staging deployment.

| # | Area | What both runtimes must produce |
| --- | --- | --- |
| 1 | Validation vectors | Identical accept/refuse and identical error codes for the existing `tests/fixtures/soul-package/vectors.json` plus v3 cases: missing SOUL.md, bad `targets`, secret-looking `env`, symlinks, NFC collisions, mode normalisation. |
| 2 | Revision hashes | Byte-identical preimages and digests for the fixed vectors (v1, v2, v3). |
| 3 | Round trip | Unknown files and manifest fields preserved byte for byte through import → revise → export. |
| 4 | Builder | Golden `buildHarnessFiles` output per harness, including the SOUL.md-then-AGENTS.md ordering. |
| 5 | Lifecycle script | spawn from template → prepare/edit/apply revision (parent chain) → memory write/read → one turn on a fake executor → export life bundle. Same revision, same memory digest, same history facts. |
| 6 | Cross-runtime move | Export on daemon, import on server-sim, and the reverse. The Agent ID, revision and life digests are unchanged. |
| 7 | Secrets canary | A declared secret resolved through the adapter appears in no export, log, audit receipt, revision or event payload. |
| 8 | Determinism | With `clock` and `random` injected, two runs produce byte-identical stores. |
| 9 | Refusals | A capability unavailable on a target (e.g. `computerUse` on server) gives the same code, `capability-unavailable`. A stale parent gives `revision-conflict`. |
| 10 | Approvals | `SOUL.md`, `bin/` and capability changes always produce an approval request. Nothing is applied before the decision. |

## 6. Adapter interfaces for soul-runtime

```js
createSoulRuntime({ storage, secrets, exec, comms, clock, random, identity, approvals, log })
```

| Port | Interface (abridged) | Daemon adapter (filesystem / signer / worktrees) | Server adapter (encrypted storage / vault / sandbox) |
| --- | --- | --- | --- |
| `SoulStorage` | `definition.read(soulId, rev?) → entries`; `definition.commit(soulId, entries, { parent }) → rev` (CAS on parent); `registry.list/get/upsert` (census); `life.get/put/append/list(soulId, part, …)`; `lock(soulId, name)` | `<souls root>/<name>.soul` plus `.soul-state/`; census at today's `agent-population` file; `file-lock`; atomic rename. **Source modules:** `soul-revisions`, `soul-memory`, `agent-space`, `agent-population`, `souls-root`, `soul-dir`. Syncs the life bundle to the gate. | Per-user envelope encryption (a per-user DEK wrapped by a KMS KEK). Blob store for entries and bundles; a database for the registry and revision CAS. Decision D3. |
| `SecretResolver` | `lease(soulId, name, { purpose }) → { value, expiresAt }`; `declared(soulId) → names` | `services.secrets.lease` → gate vault through the server link, kept in memory until TTL. Transitional: today's `soul-secrets` (Keychain/pass) behind `legacy-local-secrets`, removed at 1.0. | Vault read, scoped to (user, soul, name, purpose). The value is injected into the sandbox for one turn only. |
| `Executor` | `workspace.ensure(soulId, { repo, ref }) → ws`; `workspace.release(ws)`; `run({ argv, ws, env, timeoutMs })`; `turn({ harness, soulId, ws, prompt, onEvent }) → turnId`; `cancel(turnId)`; `runtimes.ensure(soulId)` | Git worktrees (`soul-worktrees`, `setup-worktree`, `claude-worktree-create`), ACP processes (`acp-engine`, `acp-registry`, `daemon-launch`, `muse-acp`), runtime installer (`soul-runtimes`, `runtime-catalog`, `archive-budget`), confinement hooks (`confinement`), tool homes (`soul-home`, `soul-tool-home*`), turn env (`turn-env`) | One sandbox (microVM) per turn. Ephemeral checkout using a token the server mints for that repo. Runtimes come from a pre-baked image keyed by catalog pin. No host overrides, no computer-use. |
| `CommsPort` | `send`, `inbox`, `ack`, `onWake(fn)`, `notices.take/ack/release`, `membership.join/leave` | `ctx.deps['agent-bot-comms']` service plus `comms.wake.requested` events (`wake-plane`, `cold-wake`, `comms-membership`) | Server relay and notices called directly |
| `IdentityPort` | `agentId(soulId)`, `mintRepoToken(soulId, repo)`, `soulToken(soulId)`, `coAuthorTrailer(soulId)` | `ctx.deps['agent-bot-identity'].identity` (today `identity-soul.mjs`) | The server's own identity: device-free, owner-session scoped |
| `ApprovalPort` | `request({ action, digest, scope })`, `await(id)` | `services.approvals` → gate (today `owner-action`, `owner-gate`, `cold-wake-settings.ownerGate`) | Gate approvals, the same API |
| `Clock`, `Random`, `Log` | `now()`, `bytes(n)`, `log.*` | Host services | The server's equivalents |

**Core today versus core after.** Appendix A marks 53 modules as
`soul-runtime/core`. 28 of them still import `node:fs` today; the
"Coupling" column is the to-do list for making them pure.

## 7. Daemon API for the UI (dudles / GeniusBar)

### 7.1 Transport

| Platform | Transport | Peer check |
| --- | --- | --- |
| macOS, Linux | HTTP/1.1 over a Unix domain socket at `<state>/daemon/ui.sock` (0700 directory, 0600 socket); WebSocket or SSE on the same socket for events | Peer UID must equal the owner UID. On macOS, optionally the audit-token code-signing requirement of the dudles app. |
| Windows | Named pipe `\\.\pipe\agent-bot-<user SID>-ui`, DACL limited to the user SID | `GetNamedPipeClientProcessId` → SID check |
| Browser PWA | Today's loopback `127.0.0.1` `/ui` with a pairing code and HttpOnly cookie (`agent-web.mjs`) | `Origin` plus the `x-agent-bot-ui` header |
| iOS, Android | No local daemon (background limits). dudles reaches a daemon through the **server relay** with the same request shapes, end-to-end signed by both device keys. | Relay authenticates both ends. Depends on the server relay. |

### 7.2 Authentication

- **Pairing.** `agent-bot ui pair`, or a first-run dudles flow, shows a code
  that the daemon checks. dudles generates a key pair in its own keychain or
  keystore, and the daemon records its public key as a `ui-session` client
  with scopes (`read`, `chat`, `control-local`).
- **Each request** carries a proof like the binding proof:
  `v1.<keyId>.<ms>.<nonce>.<sig>` over method, path, body hash and the
  transport name. It is fresh, single use, and refused if made before the
  daemon started (the ADR-0008 amendment 1 pattern).
- **The bearer is not used.** The per-start `daemon.json` bearer is readable
  by every process in the account (`docs/daemon-api.md`, #785), so the UI
  API doesn't accept it.
- **Owner decisions stay at the gate.** A UI session can *show* a pending
  approval and open the gate's approval page (passkey) by deep link. It can
  never approve policy itself. A local `control-local` scope may pause or
  stop a soul without the gate, which keeps today's emergency-brake
  behaviour.

### 7.3 Main calls (`/v2`, JSON, `schemaVersion` on every response)

| Area | Calls |
| --- | --- |
| Daemon | `GET /v2/status` (plugins and versions, safe mode, server link, degraded reasons) · `GET /v2/schema` (generated from route registrations) · `GET /v2/events` (SSE or WebSocket; scoped plugin events) |
| Identity | `GET /v2/identity/device` (enrolled, `key_storage`, thumbprint, harnesses) · `POST /v2/identity/enroll` → `{ user_code, verification_uri_complete, jkt }` · `GET /v2/identity/apps` (from server, no secrets) · `GET /v2/identity/bindings` |
| Souls | `GET /v2/souls` · `GET /v2/souls/{id}` (profile, env, runtimes, readiness) · `POST /v2/souls` (spawn from template) · `POST /v2/souls/{id}/{launch\|stop\|pause\|resume}` · `GET/POST /v2/souls/{id}/revisions` (prepare/propose; applying needs gate approval) · `GET /v2/souls/{id}/memory` · `GET /v2/souls/{id}/history` |
| Sessions | `POST /v2/sessions` · `POST /v2/sessions/{id}/messages` · `GET /v2/invocations/{id}` · `GET /v2/invocations/{id}/events` · `POST /v2/invocations/{id}/cancel` · `GET /v2/invocations/{id}/artifacts` (the `/v1` semantics, moved) |
| Comms | `GET /v2/comms/census` · `POST /v2/comms/send` · `GET /v2/comms/inbox` · `POST /v2/comms/ack` · `GET /v2/comms/channels` · `GET /v2/notices` (peek through the server) |
| Approvals | `GET /v2/approvals` (pending, mirrored from the gate) · `POST /v2/approvals/{id}/open` → gate URL |
| Diagnostics | `GET /v2/diagnostics` · `POST /v2/diagnostics/export` |

`/v0` and `/v1` are still served by legacy registrations until GeniusBar and
dudles have moved. The route-auth test is extended to `/v2` in both
directions.

## 8. What happens to the current surfaces

### 8.1 Kept unchanged

The following keep their names, arguments, exit codes and `--json` shapes,
pinned by golden tests in PR 1:

- the `agent-bot` command and every subcommand in `cli/dispatch.mjs`;
- the `agent-comms` command;
- the installed path `~/.local/bin/agent-bot`;
- the hook path `agent-bot hook <name>`;
- the state paths, service labels and sockets;
- `daemon.json`, bindings, Agent ID records and audit receipts;
- the `QWTS_*` and `AGENT_BOT_*` environment variables.

The root-level re-export shims stay for at least two releases, because
installed hooks, GeniusBar and the Worker load modules by path.

### 8.2 Deprecated

| Old | Replacement | Removed |
| --- | --- | --- |
| MCP `take_inbox`, `POST /v0/inbox/take`, `gh-inbox-query`, `inbox-probe` | Notices through comms: MCP `take_notices` / `ack_notices` / `release_notices` / `peek_notices` → `/api/daemon/notices/*`. Until then `take_inbox` is an alias that does take then ack and warns. | 2 releases after notices ship |
| `identity/gh-app-hook.mjs` Cloudflare Worker, `wrangler.toml`, `INBOX_TOKEN`, `WEBHOOK_SECRETS`, `SUBSCRIBERS` | Server `/apps/webhook/<slug>` (HMAC per App, 72 h dedupe, notices kept 7 days, optional forward) | After a dual-run window with App webhooks repointed |
| Local App key custody: `ensure-private-key`, `identity-app-store`, `soul-credentials`, `credential-reconciler`, keyd App items, `identity migrate-credentials`, `soul-credential-migration` | Server-held keys and server-minted tokens | 1.0 (behind `legacy-local-custody` until then) |
| `identity app create/connect/rotate-key` locally | Server `/apps` onboarding (`onboard_agent_app`, `/apps/import`) | 1.0 |
| `secret*.mjs`, `secret-providers/*` (Proton Pass), `soul secret set` storing locally | Gate vault, leased | 1.0 |
| `vouch-key.pem`, keyd `/v0/keyd/grant`, keyd `credential`/`git_credential` tools | Signer-held key; `/v0/credential` → server mint | Phase 2 |
| `device-key.pem` (software) | Signer hardware key; software devices re-enroll and the old ones are revoked | Phase 2 |
| Owner gate for credential, App or scope changes (Touch ID, admin dialog) | Gate approvals (passkey) | Phase 2 |
| Daemon `/v0/*` routes | `/v2/<plugin>/*` | After GeniusBar/dudles migrate |

## 9. Migration plan

Each step is one PR with one objective, mergeable on its own and reversible.
Steps 1–10 happen **inside agent-bot-identity** behind its existing tests,
so nothing depends on new repos until phase 3. Each PR follows the SOP
(issue first, PR-first, validate before push, bot-signed commits) and uses
`Refs #N` for partial work.

### Phase 1: prepare in place (agent-bot-identity, plus the SDK in agent-bot-daemon)

| # | Repo | PR | Done when |
| --- | --- | --- | --- |
| 1 | identity | **Golden surface tests:** snapshot every `agent-bot` command's usage, `--help` and `--json` schema, every MCP tool list and schema, and the route-auth table | Tests pass on `main` unchanged |
| 2 | identity | **Break C1:** soul registers `appDeclarations` with identity (inversion); `ownerGate` moves from `cold-wake-settings` to a host `approvals` port | The `identity → soul` edge and the `soul-app-declarations` contract are gone from the module map |
| 3 | identity | **Break C2, C4, C5** (`agent-space`/`soul-memory`, `soul-env`/`migrate`, daemon `state` module) | SCC count down to 2 |
| 4 | identity | **Break C3:** split `sop.mjs` (org resolver / soul selection), add `RevisionStore` | No file-level cycles left |
| 5 | identity | **Target column:** `runtime-modules.json` gains `target` (`daemon`, `sdk`, `identity`, `comms`, `soul-runtime/core`, `soul-runtime/daemon-adapter`, `gate`, `deprecated`) and a test that forbids new plugin → plugin edges | Map and test merged, no runtime change |
| 6 | daemon | **SDK v0:** `definePlugin`, host-service interfaces, `createTestHost`, the plugin conformance suite, `wire/` (binding proof, presence verifier, identity-format) | Published as a pinned, digested copy |
| 7 | identity | **Route registry:** the `agent-daemon.mjs` request switch becomes registered routes with auth classes (generalising `createIdentityRoutes`/`createSoulRoutes`). SDK vendored. | `daemon-route-auth.test.mjs` unchanged and green |
| 8 | identity | **MCP and CLI registries:** `agent-mcp`, `daemon-mcp` and `cli/dispatch` read registered tools and commands | Golden tests from PR 1 green |
| 9 | identity | **Server link split:** `server-client.mjs` → daemon-side transport (`ServerSession`, verification, retry) plus identity-side enrollment and assertion signing behind a `signAssertion` port; a notices client is added | Server mode behaves the same; new unit tests |
| 10 | identity | **Notices:** MCP `take_notices`/`ack_notices`/`release_notices`/`peek_notices`; `take_inbox` becomes the compat alias when server mode is on; `doctor` reports inbox deprecation | Both paths tested; receipts unchanged |

### Phase 2: trust boundary (server + identity)

| # | Repo | PR | Done when |
| --- | --- | --- | --- |
| 11 | server | **Device hardening:** `key_storage=hardware` attestation check; per-device policy for read-only scope and hardware-only mint; a self-revoke endpoint (`server-mode.md` "Known limits") | Documented in `/docs/api.md` |
| 12 | server | **Issued-token tracking:** record each installation token minted per device (hash, expiry); on device revoke, revoke its live tokens through `DELETE /installation/token` | Revoke kills live tokens within a minute |
| 13 | identity | **Signer v1:** keyd becomes `agent-bot-signer`. Device key in Secure Enclave (macOS) or TPM (Windows NCrypt, Linux tpm2), with a separate-OS-user fallback. Protocol: `device/public-key`, `device/sign-assertion`, `device/attest`, `soul/sign-token`. | `server enroll` uses the signer; `key_storage=hardware` reported |
| 14 | identity | **Server mode becomes the default;** local custody sits behind `legacy-local-custody`; vouch key moves into the signer; `doctor` flags any `.pem` on disk | A fresh install holds no long-lived secret on disk |
| 15 | server | **Approvals API** (owner passkey) and **vault** (leased secrets) | Endpoints plus docs |
| 16 | identity | **Owner gate split:** policy actions (grants, App, scope, capability) → gate approvals; local presence only for `scope: local` | Route-auth table updated; tests |
| 17 | identity | **Retire gh-app-hook:** repoint App webhooks to the server, dual-run, then delete the Worker, `wrangler.toml` and the inbox secrets | `doctor` has no `inbox.*` checks |
| 18 | server | **Relay/RPC channel** for daemon ↔ UI ↔ server (needed for mobile dudles and cross-machine comms) | Documented route plus `services.server.subscribe` |

### Phase 3: split repositories

| # | Repo | PR | Done when |
| --- | --- | --- | --- |
| 19 | daemon | **Skeleton:** host, loader, server link, local API, CLI facade, hook runner, readiness host, carried with path-filtered history (re-signed as the bot with an origin trailer, the owner's 2026-10-11 answer 1) | Runs with **zero** plugins in safe mode; CI green |
| 20 | daemon | **`/v2` UI API** (§7) with UDS and named pipe transports | Route-auth test covers `/v2` |
| 21 | soul-runtime | **Core plus ports** carried from soul modules; the core purity test; daemon adapters | Conformance suite green on `daemon` and `server-sim` |
| 22 | comms | **Rename `agent-comms` → `agent-bot-comms`** (GitHub redirect; Formula and tap updated), import agent-bot's comms modules, wrap as a plugin, keep the `agent-comms` CLI | Plugin conformance green; `agent-comms` release unchanged for users |
| 23 | identity | **Shed host, soul and comms** from agent-bot-identity, leaving the identity plugin and signer; old root paths become shims into the pinned daemon install | Plugin conformance green |
| 24 | daemon | **First bundled release:** `governance/plugins.json` pins the three plugins; Homebrew, GeniusBar and the Linux bundle switch to the daemon artifact | Golden surface tests from PR 1 green against the release |
| 25 | server | **soul-runtime on server:** Node function using the server adapters for unmanaged souls | Conformance `server-staging` nightly green |

### Phase 4: cleanup

Remove the shims (two releases after step 24), remove `/v0` and `/v1` once
dudles has moved, and remove the legacy local custody and local secrets at
1.0.

## 10. Risks and open questions

### 10.1 Top three decisions for Chris

- **D1. Do we accept the trust boundary's consequences, and what do we do
  with secrets that can't be short-lived?** Moving policy to the gate makes
  keyd's owner presence, `owner-gate`, confinement and the gh shim
  cooperative UX rather than security. Model-provider API keys and harness
  sign-in tokens (Claude/Codex OAuth in tool homes) are long-lived and live
  on the machine today. The options:
  - **(a)** a server-side LLM gateway that issues per-session tokens;
  - **(b)** vault leases where the provider supports scoped or short keys,
    and accepting per-device exposure where it doesn't;
  - **(c)** running those souls only on the server.

  Recommended: (b) now, (a) as the target.
- **D2. Does this four-repo split supersede the approved ADR-0645 extraction
  (`qwts/agent-identity`)?** If it does, three things follow:
  - agent-bot-identity keeps its name and history and sheds everything else,
    instead of a new repo being created;
  - agent-comms is renamed to agent-bot-comms rather than a new repo being
    created;
  - ENG-0128 needs another amendment (qwts-agent-sop#83 recorded the last
    one).

  Recommended: yes, with the history carried by path filter as the owner
  already chose.
- **D3. How do we host the server-side soul runtime?** The questions are
  which storage and KMS back the per-user encrypted store (for example
  Postgres for the index and CAS plus blob storage with envelope encryption,
  with Upstash already in use for notices), and which sandbox runs turns
  (for example Vercel Sandbox microVMs or another provider). Both affect
  cost and the conformance `server-staging` target.

### 10.2 Other open questions

- **Q1. Seeding the empty repos.** `agent-bot-daemon` and
  `agent-bot-soul-runtime` have no `main`, so a PR needs a base. Who seeds
  `main`, and with what (README, LICENSE, CODEOWNERS, branch protection,
  signed-commit rule)?
- **Q2.** Should the daemon run as its own service user (stronger) or as the
  agent user (simpler)? The plan assumes the agent user and keeps only the
  signer separate.
- **Q3.** Should SOUL.md sit beside AGENTS.md (recommended) or replace it?
- **Q4.** Should signed commits be created server-side (the Git Data API run
  by the server with the App), so the agent never holds a write token for
  commits?
- **Q5.** For plugin distribution, vendored digest-pinned copies
  (recommended, zero npm) or private npm packages?
- **Q6.** Mobile: is dudles on iOS/Android a UI-and-relay client only? The
  plan says yes; soul-runtime runs only on the daemon or the server.
- **Q7.** Should comms message bodies at rest be end-to-end encrypted between
  souls and principals?
- **Q8.** The server relay and RPC aren't in the deployed API. Which repo
  owns that design (`true-line/agent-bot`), and on what timeline?

### 10.3 Risks

| ID | Risk | Mitigation |
| --- | --- | --- |
| R1 | Hub files: `agent-daemon.mjs` (2.4k lines), `readiness.mjs` (3.1k), `agent-population.mjs` (1.6k, 65 importers), `soul-runtimes.mjs` (1.5k) | Registries first (PRs 7–8), so the hubs shrink by moving registrations rather than by rewriting |
| R2 | GitHub installation tokens are 1 h bearer tokens and can't be DPoP-bound | One repo and least permissions per mint, server-side revoke on device revoke (PR 12), optional server-side commits (Q4) |
| R3 | No TPM or Secure Enclave on some Linux hosts and containers | Separate-OS-user signer fallback, reported as `key_storage=software`; the server can refuse to mint for software devices per policy |
| R4 | In-process plugins aren't isolated from each other | Accepted, because the trust boundary is the gate, not the plugin. An out-of-process plugin host can come later without changing the contract. |
| R5 | GeniusBar, installed hooks and the Worker load modules by path | Shims for two or more releases; golden surface tests; the `RELEASE_COMMIT` and compatibility checks kept |
| R6 | agent-bot-identity is busy (several merges a day), so long-lived branches would rot | Phase 1 is in place, one PR at a time; each repo split carries history by path filter at a single cut |
| R7 | Windows parity (named pipes, the `comms-windows` native relay, DPAPI) | Windows CI on every plugin; the agent-comms Windows acceptance suite moves with it |
| R8 | Re-enrollment churn when software device keys move to the signer | Dual keys during the transition: re-enroll per device, then revoke the old device on the server |
| R9 | The local owner gate stops being authoritative before the gate approvals API exists | Order matters: PR 16 lands only after PR 15 |

---

## Appendix A. Module inventory and dependency map

Generated from a static scan at the pinned commits. Shims are resolved to
their targets. Coupling flags are heuristic text matches:

- `fs` is `node:fs`;
- `exec` is `node:child_process`;
- `worktree`, `keyd` and `comms` are mentions;
- `net` is `fetch` or `node:http(s)`/`node:net`;
- `os-secret` is Keychain, pass or DPAPI;
- `key-material` is PEM or key generation.

### A.1 agent-bot-identity runtime modules

Static scan of every non-shim runtime `.mjs` (239 files; 47 one-line re-export shims omitted; tests, `scripts/`, `tools/`, `web/` and `keyd/` excluded).
Columns: **Now** is the ADR-0645 module; **Target** is the destination; **Side** is the trust-boundary side (agent machine, signer, or gate); **Coupling** is a heuristic text scan; **In/out** is internal imports / importers; **Cross-repo imports** are the edges that must become a host service, event or SDK import; **Cycle** is the strongly connected component from §2.4.

| Module | Lines | Now | Target | Side | Coupling | Out/In | Cross-repo imports | Cycle | Note |
| --- | ---: | --- | --- | --- | --- | ---: | --- | --- | --- |
| `acp-engine.mjs` | 729 | soul | soul-runtime/daemon-adapter | agent | exec | 3/1 | - |  |  |
| `acp-registry.mjs` | 387 | soul | soul-runtime/daemon-adapter | agent | fs, comms | 0/15 | - |  |  |
| `agent-approvals.mjs` | 154 | cli | daemon | agent | fs, keyd | 3/0 | `approval-action` `identity/identity-cli` |  |  |
| `agent-backfill.mjs` | 143 | soul | soul-runtime/core | agent | fs, worktree | 1/1 | - | C1 |  |
| `agent-bot.mjs` | 52 | cli | daemon | agent | fs, worktree, comms | 3/0 | - |  |  |
| `agent-daemon.mjs` | 2402 | host | daemon | agent | fs, exec, worktree, keyd, comms, net, os-secret | 61/4 | `identity/identity-host` `agent-space` `soul-routes` `soul-history` `soul-memory` `agent-population` `soul-templates` `delegation-grants` `grant-github` `approval-action` `owner-action` `comms-membership` `comms-client` `agent-wake` `soul-mode` `identity-app-souls` `soul-profile` `soul-env` `sandbox` `soul-model` `cold-wake-settings` `daemon-launch` `team-start` `soul-home` `harness-auth` `wake-webhook` `acp-registry` `soul-package` `soul-env-migrate` `soul-runtimes` `soul-secrets` `soul-revisions` `wake-plane` `task-turns` `comms-relay` `soul-asides` `wake-resume` `skill-dream-service` `skill-dream-evidence` `soul-join` `soul-fork` | C5 |  |
| `agent-hook.mjs` | 602 | host | daemon | agent | fs, exec, worktree, comms | 5/1 | `identity/identity-host` `confinement` `agent-population` |  |  |
| `agent-interaction.mjs` | 980 | host | daemon | agent | fs, comms | 5/2 | `identity/identity-host` `agent-population` `soul-asides` |  |  |
| `agent-jobs.mjs` | 1180 | host | daemon | agent | fs | 3/6 | - |  |  |
| `agent-mcp.mjs` | 505 | host | daemon | agent | fs, exec, worktree | 3/0 | `identity/identity-host` |  | `agent-bot mcp` surface; tools registered by plugins. |
| `agent-population.mjs` | 1644 | soul | soul-runtime/core | agent | fs, worktree, comms | 9/65 | - | C1 |  |
| `agent-space-pack.mjs` | 382 | soul | soul-runtime/daemon-adapter | agent | fs, worktree, key-material | 3/1 | `identity/identity-soul` |  |  |
| `agent-space.mjs` | 670 | soul | soul-runtime/daemon-adapter | agent | fs, worktree | 7/10 | - | C2 |  |
| `agent-wake.mjs` | 157 | comms | comms | agent | - | 0/1 | - |  |  |
| `agent-web.mjs` | 667 | host | daemon | agent | fs, exec | 2/1 | `identity/identity-host` | C5 | Current `/ui`; becomes the UI API browser transport. |
| `approval-action.mjs` | 45 | soul | soul-runtime/core | agent | - | 1/2 | - |  |  |
| `archive-budget.mjs` | 351 | soul | soul-runtime/core | agent | fs | 0/1 | - |  |  |
| `bootstrap.mjs` | 827 | cli | daemon | agent | fs, exec, worktree, os-secret | 8/0 | `identity/identity-cli` |  |  |
| `canonical-json.mjs` | 13 | shared | daemon/sdk | agent | - | 0/10 | - |  |  |
| `child-env.mjs` | 59 | shared | daemon/sdk | agent | keyd, comms | 0/8 | - |  |  |
| `claude-worktree-create.mjs` | 351 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree | 11/0 | `identity/identity-soul` |  |  |
| `cli/dispatch.mjs` | 273 | cli | daemon | agent | fs, exec, worktree, keyd, comms, os-secret | 2/1 | - |  |  |
| `cli/identity-apps.mjs` | 20 | cli | identity (command) | agent | comms | 3/0 | `identity-app-souls` `owner-action` |  |  |
| `cli/identity.mjs` | 234 | cli | identity (command) | agent | exec, worktree | 6/0 | `agent-population` `soul-package` |  |  |
| `cli/migrate-credentials.mjs` | 22 | cli | deprecated | gate | comms | 2/0 | `owner-action` |  |  |
| `cli/output.mjs` | 53 | cli | daemon | agent | worktree, keyd, comms, os-secret | 0/1 | - |  |  |
| `cli/owner.mjs` | 33 | cli | identity (command) | agent | keyd | 3/0 | `owner-action` |  |  |
| `cli/parse.mjs` | 72 | cli | daemon | agent | worktree, keyd | 0/1 | - |  |  |
| `cli/principal.mjs` | 18 | cli | identity (command) | agent | comms | 2/0 | `owner-action` |  |  |
| `cli/sandbox-export.mjs` | 26 | cli | soul-runtime (command) | agent | - | 2/0 | - |  |  |
| `cli/skill-references.mjs` | 44 | cli | daemon | agent | worktree, keyd | 0/1 | - |  |  |
| `cli/soul-dream.mjs` | 161 | cli | soul-runtime (command) | agent | fs | 5/1 | `identity/identity-cli` |  |  |
| `cli/soul-env-transfer.mjs` | 28 | cli | soul-runtime (command) | agent | - | 2/1 | `agent-jobs` |  |  |
| `cli/soul-skill.mjs` | 344 | cli | soul-runtime (command) | agent | fs, worktree, keyd | 12/0 | - |  |  |
| `cold-wake-lock.mjs` | 43 | shared | daemon/sdk | agent | fs | 1/3 | - |  |  |
| `cold-wake-settings.mjs` | 203 | soul | soul-runtime/daemon-adapter | agent | fs, comms | 7/14 | `identity/identity-soul` `wake-webhook-store` | C1 | Also exports `ownerGate` used across soul; move to host approvals port. |
| `cold-wake.mjs` | 405 | soul | soul-runtime/daemon-adapter | agent | worktree, comms | 7/1 | `comms-relay` |  |  |
| `comms-client.mjs` | 958 | comms | comms | agent | fs, exec, comms, net, key-material | 2/4 | - |  |  |
| `comms-membership.mjs` | 54 | comms | comms | agent | fs, worktree, comms | 2/4 | - |  |  |
| `comms-relay.mjs` | 73 | comms | comms | agent | exec, worktree, comms | 1/3 | - |  |  |
| `comms-windows.mjs` | 606 | comms | comms | agent | fs, exec, comms | 1/1 | - |  |  |
| `computer-use-activity.mjs` | 26 | host | daemon | agent | - | 0/1 | - |  |  |
| `config-migration.mjs` | 95 | shared | daemon/sdk | agent | fs | 2/2 | - |  |  |
| `config.mjs` | 592 | shared | daemon/sdk | agent | fs, keyd | 1/42 | - |  |  |
| `confinement.mjs` | 467 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree, keyd, os-secret, key-material | 11/1 | `identity/identity-soul` |  |  |
| `daemon-diagnostics.mjs` | 127 | host | daemon | agent | - | 1/1 | - |  |  |
| `daemon-launch.mjs` | 531 | soul | soul-runtime/daemon-adapter | agent | fs, worktree, comms | 9/2 | - |  |  |
| `daemon-log.mjs` | 60 | shared | daemon/sdk | agent | fs | 0/2 | - |  |  |
| `daemon-mcp.mjs` | 881 | host | daemon | agent | fs, worktree, comms | 10/1 | `agent-population` `comms-relay` `soul-threads` `soul-asides` `identity/identity-host` `reach-contract` |  | `reach-mcp` surface; tools registered by plugins. |
| `daemon-status.mjs` | 65 | comms | daemon | agent | comms | 2/3 | `comms-client` |  | Folds comms status; becomes `/v2/status` aggregation. |
| `daemon-supervisor.mjs` | 874 | host | daemon | agent | fs, exec, keyd, comms | 6/4 | - | C5 |  |
| `delegation-grants.mjs` | 272 | host | identity (client) → gate | gate | keyd | 3/1 | `agent-jobs` |  | Grant approval/spend ledger moves to gate. |
| `detect-harness.mjs` | 568 | harness | daemon/sdk | agent | exec, worktree | 2/12 | - |  |  |
| `diagnostic-redact.mjs` | 100 | shared | daemon/sdk | agent | key-material | 0/3 | - |  |  |
| `diagnostics-cli.mjs` | 115 | cli | daemon | agent | fs | 2/0 | `identity/identity-cli` |  |  |
| `diagnostics.mjs` | 224 | shared | daemon/sdk | agent | fs | 2/2 | - |  |  |
| `doctor.mjs` | 131 | cli | daemon | agent | worktree | 2/0 | `identity/identity-cli` |  |  |
| `executor-contract.mjs` | 432 | soul | soul-runtime/core | agent | worktree | 2/3 | - |  |  |
| `file-lock.mjs` | 260 | shared | daemon/sdk | agent | fs | 0/23 | - |  |  |
| `grant-github.mjs` | 142 | host | identity (client) → gate | gate | os-secret | 2/1 | - |  | Grant policy decided by server. |
| `harness-auth.mjs` | 166 | soul | soul-runtime/daemon-adapter | agent | fs, exec | 7/2 | - |  |  |
| `hook-dialects.mjs` | 756 | harness | daemon/sdk | agent | worktree | 0/6 | - |  |  |
| `identity-app-souls.mjs` | 18 | soul | soul-runtime/core | agent | - | 2/2 | - |  |  |
| `identity-presence.mjs` | 4 | identity | daemon/sdk | agent | - | 1/0 | - |  | Entry for the verifier above. |
| `identity/agent-audit.mjs` | 180 | identity | identity | agent | fs | 1/1 | - |  |  |
| `identity/agent-binding.mjs` | 474 | identity | identity | agent | fs, exec, worktree | 3/10 | - |  |  |
| `identity/agent-identity.mjs` | 768 | identity | identity | agent | fs, exec, worktree, comms | 5/10 | - |  |  |
| `identity/agent-principals.mjs` | 657 | identity | identity | agent | fs, comms | 3/2 | - |  |  |
| `identity/binding-proof.mjs` | 61 | identity | daemon/sdk | agent | comms | 0/5 | - |  | Wire module duplicated in agent-comms `lib/binding-proof.mjs`. |
| `identity/cli/mint-output.mjs` | 10 | identity | identity | agent | - | 0/1 | - |  |  |
| `identity/cli/mint-token.mjs` | 82 | identity | identity | agent | - | 7/0 | - |  |  |
| `identity/credential-reconciler.mjs` | 326 | identity | gate | gate | fs, keyd, os-secret, key-material | 5/2 | `identity/mint-token` | C1 | Nothing local to reconcile once keys are server-held. |
| `identity/daemon-client.mjs` | 378 | identity | daemon/sdk | agent | fs, exec, worktree, keyd, comms, os-secret | 3/33 | `identity/agent-binding` |  | Local API client every plugin uses. |
| `identity/ensure-private-key.mjs` | 478 | identity | gate | gate | fs, keyd, os-secret, key-material | 6/2 | `identity/resolve-agent` | C1 | App key custody leaves the machine; legacy path until 1.0. |
| `identity/gh-app-hook.mjs` | 845 | identity | deprecated | gate | net | 0/0 | - |  | Replaced by server `/apps/webhook/<slug>` + notices. |
| `identity/gh-inbox-query.mjs` | 158 | identity | deprecated | gate | fs | 0/0 | - |  | Notices peek filters replace it. |
| `identity/gh-pr-view-json.mjs` | 97 | identity | identity | agent | fs, worktree | 1/0 | `identity/identity-app-store` |  |  |
| `identity/gh-shim.mjs` | 448 | identity | identity | agent | worktree | 1/1 | - |  |  |
| `identity/git-credential-bot.mjs` | 224 | identity | identity | agent | fs, worktree | 9/5 | - | C1 | Holds ≤1h repo token in memory/cache. |
| `identity/git-publish-scan.mjs` | 742 | identity | identity | agent | worktree | 0/1 | - |  |  |
| `identity/identity-app-store.mjs` | 186 | identity | gate | gate | fs, keyd, os-secret, key-material | 4/8 | - | C1 | App metadata/keys held by server. |
| `identity/identity-apps.mjs` | 863 | identity | identity (thin client) → gate | gate | fs, exec, keyd, net, os-secret, key-material | 11/2 | `organization-profile` `identity/soul-credentials` `identity/ensure-private-key` `identity/identity-app-store` |  | Create/connect/rotate via server `/apps`. |
| `identity/identity-cli.mjs` | 28 | identity | identity | agent | keyd | 18/12 | `identity/credential-reconciler` `identity/identity-app-store` `identity/inbox-probe` `identity/secret-store` `identity/secret` `identity/soul-credentials` |  |  |
| `identity/identity-entry.mjs` | 235 | identity | identity | agent | worktree | 2/0 | - |  |  |
| `identity/identity-format.mjs` | 325 | identity | daemon/sdk | agent | fs, worktree, keyd, os-secret | 1/86 | - |  | Wire/format constants; byte-identical copy today. |
| `identity/identity-host.mjs` | 39 | identity | identity | agent | keyd, comms, os-secret | 21/16 | `identity/inbox-take` `identity/secret-providers/pass-cli-credentials` `identity/secret-store` `identity/secret` |  |  |
| `identity/identity-routes.mjs` | 381 | identity | identity | agent | worktree, keyd, comms, os-secret | 10/1 | `identity/identity-app-store` `identity/inbox-take` |  |  |
| `identity/identity-soul.mjs` | 29 | identity | identity | agent | keyd | 14/18 | `identity/credential-reconciler` `identity/identity-app-store` `identity/soul-credentials` | C1 |  |
| `identity/inbox-probe.mjs` | 94 | identity | deprecated | gate | comms, net | 0/1 | - |  | Probe the server instead. |
| `identity/inbox-take.mjs` | 243 | identity | deprecated | gate | exec, worktree, comms, os-secret | 3/2 | `identity/secret-providers/proton-pass` `identity/secret-store` |  | `take_inbox` → notices take/ack via comms. |
| `identity/install-gh-shim.mjs` | 503 | identity | identity | agent | fs, exec | 3/1 | - |  |  |
| `identity/keyd-client.mjs` | 296 | identity | identity | agent | fs, keyd, net, os-secret | 4/7 | - |  | Client of the signer (keyd successor). |
| `identity/mint-token.mjs` | 319 | identity | identity | gate | fs, worktree, keyd, net, key-material | 5/6 | `identity/soul-credentials` `identity/server-client` | C1 | Server-mode mint only; local JWT path is legacy. |
| `identity/owner-approval.mjs` | 110 | identity | identity | gate | exec, worktree, keyd | 3/3 | - |  | macOS admin dialog fallback removed for policy. |
| `identity/owner-decision-action.mjs` | 236 | identity | identity | agent | keyd, comms | 2/4 | - |  |  |
| `identity/owner-decision-bytes.mjs` | 31 | identity | identity | agent | comms | 0/8 | - |  |  |
| `identity/owner-decision-evidence.mjs` | 481 | identity | identity | agent | comms | 3/3 | - |  |  |
| `identity/owner-decision-journal-frame.mjs` | 243 | identity | identity | agent | comms | 2/4 | - |  |  |
| `identity/owner-decision-journal-store.mjs` | 1082 | identity | identity | agent | fs, comms | 8/2 | - |  |  |
| `identity/owner-decision-linked-event.mjs` | 105 | identity | identity | agent | comms | 2/1 | - |  |  |
| `identity/owner-decision-local-outcome.mjs` | 113 | identity | identity | agent | comms | 3/0 | - |  |  |
| `identity/owner-decision-prepared.mjs` | 109 | identity | identity | agent | comms | 5/0 | - |  |  |
| `identity/owner-decision-receipt.mjs` | 131 | identity | identity | agent | keyd, comms | 3/1 | - |  |  |
| `identity/owner-decision-trust.mjs` | 913 | identity | identity | agent | fs, keyd, comms, os-secret, key-material | 4/2 | - |  |  |
| `identity/owner-gate.mjs` | 214 | identity | identity | gate | worktree, keyd, comms, os-secret | 6/6 | - |  | Becomes a client of gate approvals for policy. |
| `identity/owner-presence.mjs` | 589 | identity | identity | signer + gate | fs, exec, worktree, keyd, os-secret | 6/7 | - |  | Local presence for local-only effects; policy approvals at gate. |
| `identity/owner-statement.mjs` | 986 | identity | identity | gate | fs, exec, keyd | 4/4 | - |  | Owner statement keys live on the owner device / passkey. |
| `identity/owner-text.mjs` | 32 | identity | identity | agent | - | 0/2 | - |  |  |
| `identity/presence-verifier.mjs` | 38 | identity | daemon/sdk | agent | keyd | 0/3 | - |  | Pure verifier; vendored into agent-comms. |
| `identity/resolve-agent.mjs` | 238 | identity | identity | agent | fs, exec, worktree | 6/11 | `organization-profile` |  |  |
| `identity/secret-providers/pass-cli-credentials.mjs` | 88 | identity | gate | gate | os-secret | 2/2 | - |  |  |
| `identity/secret-providers/pass-cli.mjs` | 77 | identity | gate | gate | exec, os-secret | 0/2 | - |  | Proton Pass is a long-lived local secret path. |
| `identity/secret-providers/proton-pass.mjs` | 312 | identity | gate | gate | exec, os-secret | 1/2 | - |  |  |
| `identity/secret-store.mjs` | 157 | identity | gate | gate | - | 0/5 | - |  | Vault on server; no local store. |
| `identity/secret.mjs` | 81 | identity | gate | gate | - | 3/2 | - |  | Secrets come from the vault, leased. |
| `identity/server-client.mjs` | 448 | identity | daemon (transport) + identity (enrollment) | signer + agent | fs, exec, net, key-material | 1/1 | - |  | Split: DPoP session/verification → daemon server link; device key → signer. |
| `identity/session-approvals.mjs` | 28 | identity | identity | agent | - | 0/2 | - |  |  |
| `identity/signed-commit.mjs` | 310 | identity | identity | agent | exec, worktree | 4/0 | - |  | Optionally server-side commit creation (open question). |
| `identity/soul-credentials.mjs` | 553 | identity | gate | gate | fs, exec, keyd, os-secret, key-material | 9/7 | `identity/agent-identity` `identity/owner-gate` `soul-app-declarations` `identity/mint-token` | C1 | Local App credential resolution; server mints instead. |
| `identity/soul-genesis.mjs` | 33 | identity | identity | agent | - | 2/1 | - |  |  |
| `identity/uninstalled-identity-hook.mjs` | 900 | identity | identity | agent | fs, exec | 2/1 | - |  |  |
| `identity/vouch.mjs` | 318 | identity | identity | signer + agent | fs, comms, key-material | 1/5 | - |  | Signing key moves from `vouch-key.pem` into the signer. |
| `identity/worktree-token.mjs` | 207 | identity | identity | agent | fs, exec, worktree | 7/0 | - |  |  |
| `install.mjs` | 512 | cli | daemon | agent | fs, exec, worktree | 7/3 | `spaces-cutover` `sandbox` |  |  |
| `keyd-supervisor.mjs` | 116 | host | identity | signer | fs, exec, keyd, os-secret | 2/0 | `daemon-supervisor` |  | Installs/supervises the signer service. |
| `metrics.mjs` | 393 | host | daemon | agent | fs, exec, worktree, comms | 4/1 | `identity/identity-host` |  |  |
| `muse-acp.mjs` | 408 | soul | soul-runtime/daemon-adapter | agent | fs, exec | 0/0 | - |  |  |
| `organization-profile-schema.mjs` | 539 | shared | daemon/sdk | agent | keyd | 0/3 | - |  |  |
| `organization-profile.mjs` | 33 | org | daemon | agent | fs | 1/6 | - |  |  |
| `otlp-export.mjs` | 256 | host | daemon | agent | - | 0/1 | - |  |  |
| `owner-action.mjs` | 268 | soul | soul-runtime/daemon-adapter | gate | keyd, comms | 4/16 | `identity/identity-soul` `owner-principal` | C1 | Becomes the `ApprovalPort` adapter. |
| `owner-decision-computer-use.mjs` | 85 | host | daemon | agent | comms | 4/0 | `identity/identity-host` `agent-population` |  |  |
| `owner-decision-effect-gate.mjs` | 126 | host | daemon | agent | comms | 7/1 | `identity/identity-host` `owner-decision-target` `agent-population` `comms-client` |  |  |
| `owner-decision-identity-port.mjs` | 18 | host | daemon | agent | comms | 1/1 | `identity/identity-host` |  |  |
| `owner-decision-reconcile.mjs` | 84 | host | daemon | agent | fs, comms | 4/0 | `identity/identity-host` `soul-revisions` |  |  |
| `owner-decision-revision-port.mjs` | 77 | host | daemon | agent | fs, comms | 0/2 | - |  |  |
| `owner-decision-target.mjs` | 84 | soul | soul-runtime/core | agent | comms | 2/1 | - |  |  |
| `owner-decision-watch.mjs` | 57 | comms | comms | agent | comms | 0/1 | - |  |  |
| `owner-principal.mjs` | 57 | comms | comms | agent | comms | 2/1 | - |  |  |
| `process-ownership.mjs` | 112 | soul | soul-runtime/daemon-adapter | agent | fs, exec | 0/3 | - |  |  |
| `reach-contract.mjs` | 4 | comms | comms | agent | - | 0/3 | - |  |  |
| `reach-env.mjs` | 9 | shared | daemon/sdk | agent | - | 0/2 | - |  |  |
| `readiness.mjs` | 3119 | cli | daemon | agent | fs, exec, worktree, keyd, comms, os-secret, key-material | 25/2 | `agent-space` `soul-memory` `soul-env` `sandbox` `agent-population` `spaces-cutover` `identity/identity-cli` `soul-builder` `soul-build` `soul-harness-contract` `setup-worktree` |  | 3.1k-line hub; each plugin contributes readiness probes. |
| `runtime-catalog.mjs` | 380 | soul | soul-runtime/core | agent | - | 0/2 | - |  |  |
| `sandbox-export.mjs` | 443 | soul | soul-runtime/daemon-adapter | agent | fs, exec | 6/2 | - | C3 |  |
| `sandbox-remove.mjs` | 186 | soul | soul-runtime/daemon-adapter | agent | comms | 4/1 | - | C3 |  |
| `sandbox.mjs` | 804 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree, comms | 7/5 | `sop` | C3 |  |
| `session-bind.mjs` | 154 | cli | identity (command) | agent | exec, worktree | 3/0 | - |  |  |
| `setup-worktree.mjs` | 535 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree | 7/3 | `identity/identity-soul` |  |  |
| `shell-path.mjs` | 266 | shared | daemon/sdk | agent | fs, exec, comms | 1/4 | - |  |  |
| `skill-dream-evidence.mjs` | 255 | soul | soul-runtime/core | agent | fs | 3/1 | - |  |  |
| `skill-dream-inputs.mjs` | 465 | soul | soul-runtime/core | agent | fs | 3/4 | - |  |  |
| `skill-dream-notices.mjs` | 226 | soul | soul-runtime/core | agent | - | 2/1 | - |  |  |
| `skill-dream-outcomes.mjs` | 193 | soul | soul-runtime/core | agent | - | 2/4 | - |  |  |
| `skill-dream-previews.mjs` | 124 | soul | soul-runtime/core | agent | fs | 2/1 | - |  |  |
| `skill-dream-scheduler.mjs` | 815 | soul | soul-runtime/core | agent | - | 5/3 | - |  |  |
| `skill-dream-service.mjs` | 252 | host | soul-runtime/daemon-adapter | agent | fs, comms | 8/1 | - |  |  |
| `skill-dream-store.mjs` | 189 | soul | soul-runtime/core | agent | fs | 1/1 | - |  |  |
| `skill-github.mjs` | 114 | soul | soul-runtime/core | agent | - | 1/2 | - |  |  |
| `skill-install.mjs` | 260 | soul | soul-runtime/core | agent | fs | 6/2 | - |  |  |
| `skill-learning.mjs` | 280 | soul | soul-runtime/core | agent | fs | 6/3 | - |  |  |
| `skill-library-paths.mjs` | 11 | shared | daemon/sdk | agent | - | 0/2 | - |  |  |
| `skill-library.mjs` | 644 | soul | soul-runtime/core | agent | fs | 9/4 | - |  |  |
| `skill-manifest.mjs` | 132 | soul | soul-runtime/core | agent | - | 3/3 | - | C3 |  |
| `skill-references.mjs` | 16 | soul | soul-runtime/core | agent | - | 0/4 | - |  |  |
| `skill-remote.mjs` | 243 | soul | soul-runtime/core | agent | net | 1/3 | - |  |  |
| `skill-search.mjs` | 77 | soul | soul-runtime/core | agent | - | 5/1 | - |  |  |
| `skill-source-check.mjs` | 130 | soul | soul-runtime/core | agent | fs | 3/1 | - |  |  |
| `skill-source-provenance.mjs` | 62 | soul | soul-runtime/core | agent | - | 2/2 | - |  |  |
| `skill-status.mjs` | 50 | soul | soul-runtime/core | agent | - | 5/1 | - |  |  |
| `skill-update-merge.mjs` | 44 | soul | soul-runtime/core | agent | - | 0/1 | - |  |  |
| `skill-workspace.mjs` | 416 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree | 5/1 | - |  |  |
| `skill.mjs` | 322 | cli | daemon | agent | fs, exec | 2/0 | - |  |  |
| `sop-policy.mjs` | 80 | org | daemon | agent | - | 0/1 | - |  |  |
| `sop.mjs` | 1522 | soul | daemon (org resolver) + soul-runtime/core (selection) | agent | fs, exec, keyd | 10/6 | `agent-population` `identity/identity-soul` `owner-action` `acp-registry` `sandbox` | C3 | In the 9-file soul cycle. |
| `soul-app-declarations.mjs` | 45 | soul | soul-runtime/core | agent | - | 2/1 | - | C1 |  |
| `soul-asides.mjs` | 322 | soul | soul-runtime/core | agent | fs, comms | 7/4 | `identity/identity-soul` |  |  |
| `soul-build.mjs` | 234 | soul | soul-runtime/daemon-adapter | agent | fs | 7/3 | `reach-contract` |  |  |
| `soul-builder.mjs` | 1292 | soul | soul-runtime/core | agent | exec, comms | 4/6 | `reach-contract` |  |  |
| `soul-coauthor.mjs` | 111 | soul | soul-runtime/core | agent | fs, exec, worktree | 4/0 | `identity/identity-soul` |  |  |
| `soul-comms.mjs` | 126 | soul | soul-runtime/core | agent | fs, comms | 7/4 | `daemon-status` | C3 |  |
| `soul-computer-use-effect.mjs` | 45 | soul | soul-runtime/core | agent | comms | 2/0 | - |  |  |
| `soul-computer-use.mjs` | 69 | soul | soul-runtime/daemon-adapter | agent | fs | 5/0 | `identity/identity-soul` |  |  |
| `soul-credential-migration.mjs` | 486 | soul | deprecated | gate | fs, keyd, os-secret, key-material | 10/1 | `agent-population` `organization-profile` `identity/identity-soul` `soul-providers` `soul-runtimes` `soul-package` `soul-revisions` |  | Migrates local keys; obsolete once keys are server-held. |
| `soul-dir.mjs` | 61 | soul | soul-runtime/daemon-adapter | agent | fs | 3/1 | - |  |  |
| `soul-env-clean.mjs` | 294 | soul | soul-runtime/daemon-adapter | agent | fs | 7/0 | - |  |  |
| `soul-env-contract.mjs` | 117 | soul | soul-runtime/core | agent | worktree | 1/4 | - |  |  |
| `soul-env-export.mjs` | 1177 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree | 15/2 | - | C3 |  |
| `soul-env-history.mjs` | 95 | soul | soul-runtime/daemon-adapter | agent | fs | 2/0 | - |  |  |
| `soul-env-migrate.mjs` | 575 | soul | soul-runtime/daemon-adapter | agent | fs, os-secret | 16/3 | - | C4 |  |
| `soul-env.mjs` | 594 | soul | soul-runtime/daemon-adapter | agent | fs, worktree, comms | 18/3 | `identity/identity-soul` | C4 |  |
| `soul-fork.mjs` | 252 | soul | soul-runtime/core | agent | fs, worktree, comms | 14/1 | `comms-membership` |  |  |
| `soul-harness-contract.mjs` | 141 | soul | soul-runtime/core | agent | worktree | 0/5 | - |  |  |
| `soul-history.mjs` | 192 | soul | soul-runtime/core | agent | fs | 1/4 | - |  |  |
| `soul-home.mjs` | 490 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree | 11/6 | - |  |  |
| `soul-join.mjs` | 441 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree, comms | 17/2 | `identity/identity-soul` `comms-membership` |  |  |
| `soul-memory.mjs` | 296 | soul | soul-runtime/core | agent | fs, worktree | 5/10 | - | C2 |  |
| `soul-migration-journal.mjs` | 66 | soul | soul-runtime/core | agent | fs | 0/5 | - |  |  |
| `soul-mode.mjs` | 256 | soul | soul-runtime/core | agent | fs | 5/4 | - | C1 |  |
| `soul-model.mjs` | 263 | soul | soul-runtime/core | agent | fs | 6/4 | - | C1 |  |
| `soul-package.mjs` | 570 | soul | soul-runtime/core | agent | fs, keyd, comms, os-secret | 7/29 | - |  |  |
| `soul-pause.mjs` | 23 | soul | soul-runtime/daemon-adapter | agent | - | 1/0 | - |  |  |
| `soul-profile.mjs` | 315 | soul | soul-runtime/core | agent | fs, worktree, os-secret | 5/4 | `sop` | C3 |  |
| `soul-providers.mjs` | 213 | soul | soul-runtime/core | agent | keyd, os-secret | 1/6 | - |  |  |
| `soul-remove.mjs` | 268 | soul | soul-runtime/daemon-adapter | agent | fs, comms | 9/0 | `comms-membership` `daemon-status` |  |  |
| `soul-revisions.mjs` | 736 | soul | soul-runtime/core | agent | fs, worktree, keyd, comms | 12/14 | - | C3 |  |
| `soul-routes.mjs` | 50 | soul | soul-runtime/core | agent | worktree | 2/1 | - |  |  |
| `soul-runtimes.mjs` | 1473 | soul | soul-runtime/daemon-adapter | agent | fs, exec | 8/7 | - |  |  |
| `soul-secrets.mjs` | 256 | soul | soul-runtime/daemon-adapter | gate | fs, keyd, os-secret | 6/2 | `identity/identity-soul` |  | Becomes `SecretResolver` → vault lease. |
| `soul-stop.mjs` | 45 | soul | soul-runtime/daemon-adapter | agent | - | 4/1 | `identity/identity-soul` |  |  |
| `soul-template-refresh.mjs` | 195 | soul | soul-runtime/core | agent | fs | 7/0 | - |  |  |
| `soul-templates.mjs` | 359 | soul | soul-runtime/core | agent | fs | 10/5 | - |  |  |
| `soul-threads.mjs` | 395 | soul | soul-runtime/core | agent | fs, worktree, comms | 4/5 | - |  |  |
| `soul-tool-home-record.mjs` | 131 | soul | soul-runtime/core | agent | fs | 1/5 | - |  |  |
| `soul-tool-home.mjs` | 214 | soul | soul-runtime/daemon-adapter | agent | fs | 9/0 | `identity/identity-soul` |  |  |
| `soul-tool-homes.mjs` | 257 | soul | soul-runtime/core | agent | os-secret | 0/8 | - |  |  |
| `soul-worktrees.mjs` | 87 | soul | soul-runtime/daemon-adapter | agent | fs, worktree | 2/3 | - |  |  |
| `souls-root.mjs` | 12 | soul | soul-runtime/daemon-adapter | agent | - | 1/6 | - |  |  |
| `spaces-cutover.mjs` | 300 | soul | soul-runtime/daemon-adapter | agent | fs | 3/2 | - |  |  |
| `state-paths.mjs` | 27 | shared | daemon/sdk | agent | - | 0/39 | - |  |  |
| `sync-hooks.mjs` | 271 | cli | daemon | agent | fs, worktree | 4/2 | `identity/identity-cli` |  |  |
| `task-turns.mjs` | 50 | comms | comms | agent | fs, comms | 0/1 | - |  |  |
| `team-start.mjs` | 284 | soul | soul-runtime/core | agent | fs | 6/1 | - |  |  |
| `telegram-adapter.mjs` | 817 | host | comms | gate | fs | 6/0 | `agent-daemon` `agent-interaction` `identity/identity-host` |  | Chat channel adapter; bot token is a long-lived secret. |
| `traceparent.mjs` | 74 | shared | daemon/sdk | agent | comms | 0/3 | - |  |  |
| `turn-env.mjs` | 53 | soul | soul-runtime/daemon-adapter | agent | keyd | 2/3 | - |  |  |
| `turn-registry.mjs` | 86 | soul | soul-runtime/core | agent | - | 2/2 | `identity/identity-soul` |  |  |
| `update.mjs` | 17 | cli | daemon | agent | - | 1/0 | - |  |  |
| `wake-dispatch.mjs` | 307 | comms | comms | agent | comms | 2/1 | - |  |  |
| `wake-listen.mjs` | 783 | comms | comms | agent | fs, exec, worktree, comms, net | 1/0 | `identity/identity-soul` |  |  |
| `wake-plane.mjs` | 339 | host | soul-runtime/daemon-adapter | agent | worktree, keyd, comms | 10/2 | `agent-jobs` `daemon-mcp` `identity/identity-host` `wake-dispatch` |  | Subscribes to `comms.wake` events; launches turns. |
| `wake-resume.mjs` | 379 | soul | soul-runtime/daemon-adapter | agent | fs, exec, worktree, comms | 5/5 | - |  |  |
| `wake-webhook-store.mjs` | 94 | comms | comms | agent | fs | 3/2 | - |  |  |
| `wake-webhook.mjs` | 70 | comms | comms | agent | worktree, comms | 2/1 | - |  |  |
| `windows-account-custody.mjs` | 292 | shared | daemon/sdk | agent | exec, comms | 0/3 | - |  |  |
| `windows-cli-launcher.mjs` | 59 | shared | daemon/sdk | agent | fs | 0/3 | - |  |  |
| `windows-git-shell.mjs` | 36 | shared | daemon/sdk | agent | fs, exec, comms | 0/3 | - |  |  |
| `windows-install-path.mjs` | 36 | shared | daemon/sdk | agent | exec | 0/1 | - |  |  |
| `windows-task-job.mjs` | 75 | host | daemon | agent | comms | 0/1 | - |  |  |

### A.2 agent-comms modules

Every `bin/` and `lib/` module (105 files, 12683 lines). Default target is the comms plugin.

| Module | Lines | Target | Side | Coupling | Out/In | Note |
| --- | ---: | --- | --- | --- | ---: | --- |
| `bin/agent-comms.mjs` | 587 | comms | agent | fs, comms | 17/0 |  |
| `bin/owner-supervised-broker.mjs` | 47 | comms | agent | comms | 3/0 |  |
| `lib/asides-report.mjs` | 73 | comms | agent | comms | 1/2 |  |
| `lib/binding-proof.mjs` | 61 | daemon/sdk | agent | comms | 0/1 | Byte-identical copy of identity `binding-proof.mjs`. |
| `lib/broker.mjs` | 272 | comms | agent | comms | 20/3 |  |
| `lib/broker/a2a-outbound.mjs` | 318 | comms | agent | fs, comms | 2/1 | Cross-machine reach should move to server relay. |
| `lib/broker/a2a.mjs` | 219 | comms | agent | fs, comms, net | 2/2 | Cross-machine reach should move to server relay. |
| `lib/broker/census.mjs` | 96 | comms | agent | fs, comms | 3/1 |  |
| `lib/broker/channel-census.mjs` | 73 | comms | agent | comms | 5/1 |  |
| `lib/broker/channels.mjs` | 258 | comms | agent | comms | 4/1 |  |
| `lib/broker/launch.mjs` | 208 | comms | agent | comms | 4/2 | Launch requests forwarded to daemon/soul-runtime (ADR-0059: agent-bot owns process creation). |
| `lib/broker/launchagent.mjs` | 4 | comms | agent | - | 2/1 | Launch requests forwarded to daemon/soul-runtime (ADR-0059: agent-bot owns process creation). |
| `lib/broker/mailbox.mjs` | 229 | comms | agent | comms | 6/1 |  |
| `lib/broker/message-admission.mjs` | 67 | comms | agent | comms | 4/3 |  |
| `lib/broker/owner-admission.mjs` | 33 | comms | gate | - | 2/1 | Owner decisions: authority belongs at the gate. |
| `lib/broker/owner-decisions.mjs` | 521 | comms | gate | comms | 18/1 | Owner decisions: authority belongs at the gate. |
| `lib/broker/owner-enrollment-store.mjs` | 308 | comms | gate | fs, comms | 5/0 | Owner decisions: authority belongs at the gate. |
| `lib/broker/owner-enrollment.mjs` | 43 | comms | gate | comms | 1/2 | Owner decisions: authority belongs at the gate. |
| `lib/broker/owner-grants.mjs` | 23 | comms | gate | - | 4/1 | Owner decisions: authority belongs at the gate. |
| `lib/broker/owner-ingress.mjs` | 45 | comms | gate | comms | 5/1 | Owner decisions: authority belongs at the gate. |
| `lib/broker/owner-receipt-fixture.mjs` | 52 | comms | gate | comms | 3/1 | Owner decisions: authority belongs at the gate. |
| `lib/broker/pairing.mjs` | 372 | comms | agent | comms | 6/1 | Cross-machine reach should move to server relay. |
| `lib/broker/principal-target.mjs` | 32 | comms | agent | comms | 2/1 |  |
| `lib/broker/server.mjs` | 3 | comms | agent | - | 1/2 |  |
| `lib/broker/shared.mjs` | 44 | comms | agent | comms | 1/14 |  |
| `lib/broker/tasks.mjs` | 136 | comms | agent | comms | 2/2 |  |
| `lib/broker/watch.mjs` | 387 | comms | agent | comms | 5/1 |  |
| `lib/build-metadata.mjs` | 13 | comms | agent | fs, comms | 1/3 |  |
| `lib/channel-state.mjs` | 105 | comms | agent | comms | 1/7 |  |
| `lib/client.mjs` | 351 | comms | agent | fs, exec, worktree, comms, net | 10/4 |  |
| `lib/custody.mjs` | 4 | comms | owner keystore | comms | 2/5 | Principal secret: owner side, never readable by agents. |
| `lib/diagnostic-export-supervisor.mjs` | 41 | comms | agent | - | 2/0 |  |
| `lib/diagnostic-export-writer.mjs` | 18 | comms | agent | comms | 1/0 |  |
| `lib/diagnostic-export.mjs` | 276 | comms | agent | fs, comms | 4/3 |  |
| `lib/diagnostic-fd.mjs` | 20 | comms | agent | - | 0/1 |  |
| `lib/diagnostic-profile.mjs` | 108 | comms | agent | comms | 0/4 |  |
| `lib/diagnostic-writer.mjs` | 20 | comms | agent | fs, comms | 1/0 |  |
| `lib/diagnostics.mjs` | 193 | comms | agent | comms | 3/2 |  |
| `lib/errors.mjs` | 39 | comms | agent | - | 0/29 |  |
| `lib/host-config.mjs` | 56 | comms | agent | fs, comms | 1/8 |  |
| `lib/owner-decision-admission.mjs` | 28 | comms | gate | - | 2/3 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-canonical.mjs` | 81 | comms | gate | - | 0/16 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-challenge.mjs` | 61 | comms | gate | - | 2/2 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-codec.mjs` | 129 | comms | gate | comms | 2/8 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-daemon-evidence.mjs` | 228 | comms | gate | comms | 4/0 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-frame-classifier.mjs` | 71 | comms | gate | - | 1/1 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-identities.mjs` | 9 | comms | gate | - | 0/5 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-issuer.mjs` | 98 | comms | gate | comms | 4/2 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-journal-errors.mjs` | 20 | comms | gate | - | 0/1 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-journal-frame.mjs` | 97 | comms | gate | comms | 0/2 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-journal.mjs` | 712 | comms | gate | fs, comms | 10/1 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-lifecycle-records.mjs` | 176 | comms | gate | comms | 7/1 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-namespace.mjs` | 12 | comms | gate | comms | 1/0 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-prepared.mjs` | 45 | comms | gate | comms | 3/5 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-proof.mjs` | 44 | comms | gate | comms | 3/4 | Owner decisions: authority belongs at the gate. |
| `lib/owner-decision-wire-operations.mjs` | 19 | comms | gate | - | 0/2 | Owner decisions: authority belongs at the gate. |
| `lib/owner-enrollment-codec.mjs` | 116 | comms | gate | comms | 1/2 | Owner decisions: authority belongs at the gate. |
| `lib/owner-enrollment-frame.mjs` | 71 | comms | gate | comms | 2/1 | Owner decisions: authority belongs at the gate. |
| `lib/owner-operation-grants.mjs` | 21 | comms | gate | - | 0/4 | Owner decisions: authority belongs at the gate. |
| `lib/owner-scoped-grants.mjs` | 43 | comms | gate | - | 2/5 | Owner decisions: authority belongs at the gate. |
| `lib/paths.mjs` | 26 | comms | agent | comms | 1/5 |  |
| `lib/platform/account-isolation.mjs` | 401 | comms | agent | fs, exec, comms | 4/10 |  |
| `lib/platform/daemon-requests.mjs` | 6 | comms | agent | comms | 0/1 |  |
| `lib/platform/diagnostic-export-process.mjs` | 11 | comms | agent | exec | 0/2 |  |
| `lib/platform/diagnostic-stderr.mjs` | 70 | comms | agent | exec, comms | 0/1 |  |
| `lib/platform/durable-file.mjs` | 72 | comms | agent | fs, exec | 2/1 |  |
| `lib/platform/identity-validators.mjs` | 18 | comms | agent | - | 0/2 |  |
| `lib/platform/local-channel.mjs` | 597 | comms | agent | fs, exec, comms, net, key-material | 6/4 |  |
| `lib/platform/owner-decision-readiness.mjs` | 26 | comms | gate | fs, comms | 1/1 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-enrollment-guard.mjs` | 31 | comms | gate | fs | 2/1 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-enrollment-source.mjs` | 8 | comms | gate | comms | 0/0 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-journal-lock.mjs` | 39 | comms | gate | fs | 1/4 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-journal-native.mjs` | 100 | comms | gate | fs, exec | 1/2 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-observer-native.mjs` | 73 | comms | gate | fs, exec | 3/1 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-observer-wire.mjs` | 69 | comms | gate | comms | 0/2 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-receipt-capability.mjs` | 6 | comms | gate | - | 0/0 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-receipt-native.mjs` | 145 | comms | gate | fs, exec, comms | 3/0 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-receipt-wire.mjs` | 70 | comms | gate | comms | 1/1 | Owner decisions: authority belongs at the gate. |
| `lib/platform/owner-supervised-session.mjs` | 171 | comms | gate | fs, comms | 1/2 | Owner decisions: authority belongs at the gate. |
| `lib/platform/secret-store.mjs` | 130 | comms | owner keystore | fs, exec, os-secret | 5/2 | Principal secret: owner side, never readable by agents. |
| `lib/platform/service-startup.mjs` | 752 | comms | agent | fs, exec, comms | 6/1 |  |
| `lib/platform/windows-environment.mjs` | 7 | comms | agent | - | 0/5 |  |
| `lib/platform/windows-pipe-client.mjs` | 125 | comms | agent | exec | 3/1 |  |
| `lib/platform/windows-pipe-stage.mjs` | 23 | comms | agent | - | 0/2 |  |
| `lib/platform/windows-relay-artifact.mjs` | 36 | comms | agent | fs, comms | 0/3 |  |
| `lib/platform/windows-task-job.mjs` | 65 | comms | agent | - | 0/1 |  |
| `lib/platform/windows-task-stop.mjs` | 98 | comms | agent | fs, exec, comms | 3/1 |  |
| `lib/principal-client.mjs` | 89 | comms | agent | comms | 6/1 |  |
| `lib/retention.mjs` | 94 | comms | agent | - | 0/8 |  |
| `lib/skill.mjs` | 50 | comms | agent | fs, exec, comms | 1/1 |  |
| `lib/state.mjs` | 552 | comms | agent | fs, comms | 5/2 |  |
| `lib/trace-context.mjs` | 6 | comms | agent | - | 0/2 |  |
| `lib/vendor/agent-bot-presence/identity-presence.mjs` | 4 | daemon/sdk | agent | - | 1/4 | Vendored identity presence verifier snapshot. |
| `lib/vendor/agent-bot-presence/identity/presence-verifier.mjs` | 38 | daemon/sdk | agent | keyd | 0/1 | Vendored identity presence verifier snapshot. |
| `lib/wire.mjs` | 75 | comms | agent | comms | 1/8 |  |
| `lib/worker/adapters.mjs` | 72 | comms | agent | - | 1/1 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/index.mjs` | 304 | comms | agent | comms | 11/1 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/jail.mjs` | 71 | comms | agent | fs, worktree | 0/1 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/metrics.mjs` | 11 | comms | agent | fs | 0/1 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/options.mjs` | 66 | comms | agent | fs, worktree | 3/1 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/prompt.mjs` | 64 | comms | agent | comms | 1/2 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/task-turn.mjs` | 16 | comms | agent | - | 0/2 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/tiers.mjs` | 43 | comms | agent | fs, comms | 1/1 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/turn.mjs` | 94 | comms | agent | fs, exec | 0/1 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
| `lib/worker/watch.mjs` | 23 | comms | agent | exec, comms | 1/1 | R1 worker turns overlap soul-runtime `Executor`; converge later. |
