# x402-escrow-schema

A specification for `scheme: "escrow"` — a payment scheme for [x402](https://github.com/x402-foundation/x402) HTTP servers that adds non-custodial on-chain escrow between buyer and seller.

---

## Why escrow is a necessary addition to x402

x402's built-in `exact` scheme is **settle-and-done**: a facilitator moves funds directly to the seller the moment the buyer signs. This is ideal for low-value, high-trust, commodity API calls where the resource is delivered atomically in the HTTP 200 response. However, it fails in three common real-world scenarios:

**1. High-value transactions.** When a buyer is paying a meaningful amount — for a premium report, a license, a physical good, or any service where the value is high enough to care about — they need a confirmation signal before funds are irrevocably transferred. A single-round-trip settle provides no recourse if the seller delivers nothing.

**2. Non-trusted or pseudonymous sellers.** x402 puts the buyer in the position of trusting whoever runs the server. Established SaaS APIs have reputation and legal accountability; first-time sellers, autonomous agents, and pseudonymous sellers do not. Without escrow, buyers face a binary choice: trust or skip.

**3. Delivery that is separate from payment.** Physical goods, generated content, gated access credentials, and asynchronous services all have a provable gap between "funds sent" and "resource received". The `exact` scheme has no model for this gap. A buyer who pays and receives nothing has no on-chain footprint to reference in a dispute.

The `escrow` scheme closes all three gaps:

- **Funds are held in a neutral on-chain escrow contract** at commit time. The seller cannot access them until delivery is confirmed.
- **A dispute window** gives the buyer time to signal delivery failure before funds are auto-released.
- **A registered dispute resolver** can split funds and slash a seller bond if delivery provably fails — enforced by the contract, not by trust.
- **No trusted intermediary** holds funds at any point. The escrow contract is the custodian.

---

## What the `escrow` scheme adds to x402

The `escrow` scheme extends x402's multi-scheme `accepts[]` design. It is **not a fork** of x402; it is a new scheme value that x402 was designed to host alongside `exact` and any future schemes.

Key additions beyond `exact`:

| Feature | `exact` | `escrow` |
|---|---|---|
| Fund custody | Facilitator (brief) → seller | Escrow contract until delivery |
| Seller signs offer | No | Yes — off-chain `OfferCommitment` |
| Delivery negotiation | No | Yes — pluggable transport registry |
| Dispute resolution | No | Yes — on-chain, resolver-enforced |
| Post-payment actions | None | `nextActions` envelope on every response |
| Censorship resistance | N/A | Multi-channel fallback (server / facilitator / on-chain / MCP) |

---

## How it works (one-paragraph summary)

The server publishes a 402 response carrying a signed `OfferCommitment` and the address of an on-chain escrow contract. The buyer signs an meta-transaction authorizing the escrow contract to lock their funds and record the exchange, then attaches it to the retry request. A facilitator (or the buyer directly) submits the meta-transaction; funds move into escrow; the server verifies the on-chain state and returns the resource (or a delivery receipt). Every server response carries a `nextActions` envelope listing legal next steps and every channel through which the buyer can invoke them — so the seller can never strand the buyer by going offline.

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
