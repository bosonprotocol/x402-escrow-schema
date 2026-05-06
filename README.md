# x402-escrow-schema

A specification for `scheme: "escrow"` — a payment scheme for [x402](https://github.com/x402-foundation/x402) HTTP servers that adds non-custodial on-chain escrow between buyer and seller.

---

## Why escrow is a necessary addition to x402

x402's built-in `exact` scheme is **settle-and-done**: a facilitator moves funds directly to the seller the moment the buyer signs. This is ideal for low-value, high-trust, commodity API calls where the resource is delivered atomically in the HTTP 200 response. However, it fails in four common real-world scenarios:

**1. High-value transactions.** When a buyer is paying a meaningful amount — for a premium report, a license, a physical good, or any service where the value is high enough to care about — they need a confirmation signal before funds are irrevocably transferred. A single-round-trip settle provides no recourse if the seller delivers nothing.

**2. Non-trusted or pseudonymous sellers.** x402 puts the buyer in the position of trusting whoever runs the server. Established SaaS APIs have reputation and legal accountability; first-time sellers, autonomous agents, and pseudonymous sellers do not. Without escrow, buyers face a binary choice: trust or skip.

**3. Delivery that is separate from payment.** Physical goods, generated content, gated access credentials, and asynchronous services all have a provable gap between "funds sent" and "resource received". The `exact` scheme has no model for this gap. A buyer who pays and receives nothing has no on-chain footprint to reference in a dispute.

**4. Autonomous AI agents as buyers.** An AI agent cannot evaluate seller reputation the way a human can, cannot invoke legal recourse, and may not even detect that it was cheated until long after the fact. Agents need cryptographic payment guarantees — not reputational trust — as their primary protection. `exact` gives an agent no mechanism to verify delivery before funds move.

The `escrow` scheme closes all four gaps.

---

## Core guarantee: the buyer never needs to trust anyone

The facilitator is optional — the buyer is never stranded because, in every non-terminal state, `nextActions` includes at least one buyer-reachable direct `onchain` action. A server that goes offline, a facilitator that stops responding, or a seller that refuses to cooperate cannot strand the buyer. The escrow contract enforces the available outcomes.

This is a structural improvement over authorization-based approaches (such as `authCapture`), where the buyer's only fallback is waiting for an authorization timeout to expire before `reclaim()` becomes available. In the `escrow` scheme, the buyer always has at least one direct on-chain fallback path in non-terminal states, even though some other actions may require a different channel or additional parties' signatures.

---

## What the `escrow` scheme adds to x402

The `escrow` scheme extends x402's multi-scheme `accepts[]` design. It is **not a fork** of x402; it is a new scheme value that x402 was designed to host alongside `exact` and any future schemes. Servers add an `escrow` entry to their `accepts[]` array; clients that don't understand the scheme fail cleanly with a structured `UnsupportedSchemeError` — never an accidental settle.

Key additions beyond `exact`:

| Feature | `exact` | `escrow` |
|---|---|---|
| Fund custody | Facilitator (brief) → seller | Escrow contract until delivery confirmed |
| Facilitator trust required | Yes — funds transit via facilitator | No — facilitator is optional; buyer always has a direct on-chain path |
| Per-session pricing | Fixed at server config time | Seller signs a fresh offer per request — price computed dynamically |
| Seller signs offer | No | Yes — off-chain `OfferCommitment` |
| Delivery negotiation | No | Yes — pluggable transport registry |
| Dispute resolution | No | Yes — on-chain, resolver-enforced |
| Post-payment actions | None | `nextActions` envelope on every response |
| Censorship resistance | N/A | Multi-channel fallback (server / facilitator / on-chain / MCP) |

---

## How it works (one-paragraph summary)

The server signs a fresh `OfferCommitment` for each request — pricing is computed at request time, not pre-registered on-chain — and returns it in the 402 response alongside the escrow contract address. The buyer signs a meta-transaction authorizing the escrow contract to lock their funds and record the exchange, then attaches it to the retry request. A facilitator (or the buyer directly) submits the meta-transaction; funds move into escrow; the server verifies the on-chain state and returns the resource (or a delivery receipt). Every server response carries a `nextActions` envelope listing legal next steps and every channel through which the buyer can invoke them — so the seller can never strand the buyer by going offline.

---

## How the `escrow` scheme addresses x402 design gaps

Issue [#1645](https://github.com/x402-foundation/x402/issues/1645) identified four open design gaps in x402. The `escrow` scheme addresses all four:

| x402 design gap | How `escrow` addresses it |
|---|---|
| No atomic link between settlement and delivery | The COMMITTED state separates payment lock from delivery confirmation. Funds cannot reach the seller until the buyer signals delivery or the dispute window expires. |
| Authorization replay vulnerability | The meta-tx nonce (`usedNonce[from][nonce]`) is consumed on-chain at commit time. The token-auth nonce is enforced by the token contract. Two independent on-chain replay barriers. |
| Payment history is ephemeral (in-memory only) | Every commit, release, and dispute emits on-chain events. The exchange record is a permanent, queryable source of truth — independent of server uptime. |
| Authorization expiry conflicts with long-running services | `maxTimeoutSeconds` governs the token-auth validity window. Dispute and delivery periods are independently configurable per offer, matching the actual delivery timeline of the service. |

---

## Relationship to existing x402 escrow proposals

Several proposals in the x402 community address parts of the escrow problem — `authCapture` (PR #1425), x402r (#864), the `channel` scheme (#946), and others. This schema is not a competing proposal: it defines the **wire format** that any of those implementations, or future ones, could adopt.

The key design decision: `offer.commitment` and the action IDs in `nextActions` are intentionally implementation-defined. A future `authCapture`-based implementation could publish `coinbase-commitOnly` action IDs and remain fully compatible with any client that understands `scheme: "escrow"`. A Boson-based implementation publishes `boson-` prefixed IDs. Clients that encounter an unrecognized prefix skip that action safely — they never misfire.

---

## Compatibility with vanilla x402

A vanilla x402 client receiving `{ accepts: [{ scheme: "escrow", ... }] }` does not match any registered scheme handler and surfaces a structured `UnsupportedSchemeError`. **No accidental settle is possible.** Buyers must opt in by installing an `escrow`-scheme-aware client library.

A server may simultaneously advertise an `exact` and an `escrow` accept entry, letting legacy x402 clients use the direct-pay path while escrow-aware clients prefer the protected path. The choice belongs to the client.

---

## Document map

| # | File | Status |
|---|---|---|
| — | [README.md](./README.md) | this file |
| 00 | [00-overview.md](./00-overview.md) | detailed |
| 01 | [01-escrow-scheme.md](./01-escrow-scheme.md) | detailed — wire format source of truth |
| 02 | [02-flows.md](./02-flows.md) | detailed — sequence diagrams |
| 03 | [03-delivery-transports.md](./03-delivery-transports.md) | detailed — pluggable delivery |
| 04 | [04-state-machine-and-next-actions.md](./04-state-machine-and-next-actions.md) | detailed — self-describing responses |

---

## Implementations

| Implementation | Repo | Notes |
|---|---|---|
| Boson Protocol (`x402b`) | [bosonprotocol/x402b](https://github.com/bosonprotocol/x402b) | Reference implementation using Boson Protocol escrow |

Implementors are encouraged to open a PR adding their entry to this table.

---

## Status

`v0.1` — initial public specification. Wire format is stable; JSON Schemas are in progress.
