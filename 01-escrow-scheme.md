# 01 — The `escrow` Scheme

> **Status:** detailed spec (v0.1). This document is the wire-format source of truth.

## 1. Why a first-class scheme (not an extension of `exact`)

An earlier design considered shipping escrow context inside an `extensions["escrow"]` field on top of `scheme: "exact"`. That is unsafe in production:

A non-escrow-aware facilitator that receives such a payload with the standard `ExactEvmPayload` interprets the inner token-authorization signature against the standard exact-scheme settle path. Even with `value: "0"` defanging it consumes the authorization nonce — and **if the buyer's wallet ever signs an authorization with the same parameters via any other channel, that authorization can be replayed** unless we hash-isolate at the EIP-712 level.

A first-class scheme `"escrow"` removes the hazard entirely:

- Non-escrow facilitators see an unfamiliar scheme value and reject with a structured error. **Default behaviour is fail-safe.**
- The buyer's headline signature is a meta-transaction bound to the escrow contract's EIP-712 domain, not reusable elsewhere.
- The token-authorization signature, when present, is the buyer's authorization for *this exact spend* of *this exact token*; if a non-escrow party tried to settle it, they'd just get the authorized spend — not an attack vector.
- The wire format is free to carry escrow-shaped data (OfferCommitment, sellerSig, recipientId, fulfillment options, nextActions) at the top level rather than buried in a generic `info` blob.

The fundamental security property this achieves: **a non-escrow party that receives an escrow `X-PAYMENT` payload can do exactly what the buyer authorized — nothing more.** There is no way to accidentally settle using an escrow payload through an `exact` facilitator, and there is no signature cross-contamination between schemes.

## 2. PaymentRequirements (server → client, in 402 body)

```jsonc
{
  "x402Version": 2,
  "accepts": [
    {
      "scheme": "escrow",
      "network": "eip155:8453",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",   // ERC-20 token
      "amount": "1000000",                                    // atomic units
      "escrowAddress": "0xEscrowContract...",                 // on-chain escrow contract
      "recipientId": "seller-12345",                          // routing-only: sellerId, DID, or address

      "maxTimeoutSeconds": 300,

      "offer": {
        // implementation-defined signed offer (OfferCommitment)
        "commitment": { /* implementation-defined */ },
        "sellerSig":  "0x...",   // EIP-712 over commitment, escrow contract domain
        "creator":    "0xSellerKey..."
      },

      // EVM token-transfer authorization strategies the escrow contract accepts for this token
      "tokenAuthStrategies": ["none", "erc3009", "permit", "permit2"],

      // fulfillment — see 03-fulfillment-channels.md
      // required:true means the buyer MUST pick a channel from options[].
      // inline is listed here, so the buyer may choose same-response delivery;
      // the other options let async buyers (email/xmtp/webhook) receive out-of-band.
      // If only inline delivery is offered, omit this field (or set required:false).
      "fulfillment": {
        "required": true,
        "options": [
          { "id": "inline",   "schema": null },
          { "id": "email",    "schema": { "type": "object", "required": ["email"] } },
          { "id": "xmtp",     "schema": { "type": "object", "required": ["xmtpAddress"] } },
          { "id": "webhook",  "schema": { "type": "object", "required": ["url", "publicKey"] } }
        ]
      },

      "actions": {                                            // see 04-state-machine-and-next-actions.md
        "next": [
          {
            "id": "<impl>-commitOnly",         // commit now, release/redeem separately
            "channels": ["server", "facilitator", "onchain", "mcp"],
            "endpoints": { "server": "https://seller.example/escrow/commit" }
          },
          {
            "id": "<impl>-commitAndRelease",   // commit and release in one transaction
            "channels": ["server", "facilitator", "onchain", "mcp"],
            "endpoints": { "server": "https://seller.example/escrow/commit-and-release" }
          }
        ],
        "fallback": {
          "xmtp": "0xSellerXMTP...",
          "mcp":  "escrow://seller/12345",
          "onchainHints": {
            "escrowContract": "0xEscrowContract...",
            "commitOnlyMethod":       "<impl-specific>",
            "commitAndReleaseMethod": "<impl-specific>"
          }
        }
      }
    }
  ]
}
```

### Field reference (PaymentRequirements)

| Field | Required | Notes |
|---|---|---|
| `scheme` | yes | Must be `"escrow"`. |
| `network` | yes | CAIP-2 (`eip155:<chainId>`). EVM only for v1. |
| `asset` | yes | ERC-20 token contract address. |
| `amount` | yes | Atomic units, decimal string. The server computes and signs this value fresh for each request — it is not pre-registered on-chain. This enables session-specific pricing: the seller can quote an amount that reflects the current request's scope, the buyer's identity, or real-time market conditions. |
| `escrowAddress` | yes | On-chain escrow contract. The custodian. |
| `recipientId` | yes | Routing-only. May be a numeric seller ID, a DID, or a wallet address. |
| `maxTimeoutSeconds` | yes | Upper bound for `validBefore` in token-auth signatures. |
| `offer.commitment` | yes | Implementation-defined OfferCommitment object. Used to compute the offer hash and as the payload for the on-chain commit call. |
| `offer.sellerSig` | yes | EIP-712 signature over `commitment` under the escrow contract's domain. Validated on-chain by the escrow contract. |
| `offer.creator` | yes | Address whose key signed `sellerSig`. |
| `tokenAuthStrategies` | yes | Subset of `["none", "erc3009", "permit", "permit2"]`. The token-transfer authorization strategies the escrow contract accepts for this asset. `none` requires the buyer to pre-approve the escrow contract. |
| `fulfillment` | optional | Three forms: **(1) Absent or `{required: false}`** — resource always returned inline; no buyer input needed. **(2) `{required: true, options: [...]}`** — buyer must pick a channel. Include `inline` in `options[]` to allow same-response delivery alongside out-of-band options; omit `inline` if the resource is never returned in the HTTP body. |
| `actions` | yes | Initial `nextActions` envelope. Always lists at least one of `<impl>-commitOnly` / `<impl>-commitAndRelease`. |

### Action-id namespacing

Action IDs carry an implementation-defined prefix (e.g. `boson-`, `coinbase-`). The `escrow` scheme is designed to be implementation-agnostic; the prefix allows multiple implementations to coexist in a single `nextActions` envelope without collision. Clients that don't recognise an action's prefix MUST skip it rather than attempt dispatch.

### x402 v1 fallback

For x402 v1 deployments that lack a top-level `extensions` field, the same `escrow`-shaped object is the payload object — the v1 `extra` slot is unused. v1 clients that don't recognize `scheme: "escrow"` fail loudly per x402 v1 semantics, which is the desired behaviour.

## 3. PaymentPayload (client → server, in `X-PAYMENT` header)

The header value is base64(JSON):

```jsonc
{
  "x402Version": 2,
  "scheme": "escrow",
  "network": "eip155:8453",
  "payload": {
    "action": "<impl>-commitAndRelease",   // or "<impl>-commitOnly"
    "tokenAuthStrategy": "erc3009",        // "none" | "erc3009" | "permit" | "permit2"

    "offerRef": {                          // echo of the offer the server proposed
      "commitment": { /* echoed verbatim */ },
      "sellerSig":  "0x..."
    },

    "buyer": "0xBuyer...",

    // Buyer's headline signature: a meta-transaction authorising execution of `<action>`
    // on behalf of `buyer`. EIP-712 domain = escrow contract; type is implementation-defined
    // but MUST include at minimum: nonce, from (buyer), the action to execute, and action arguments.
    "metaTx": {
      "from":      "0xBuyer...",
      "nonce":     "0",
      // implementation-defined fields (action selector, encoded arguments, etc.)
      "sig":       { "v": 27, "r": "0x...", "s": "0x..." }
    },

    // Token-transfer authorization payload. Shape depends on tokenAuthStrategy.
    // Passed through to the escrow contract's meta-tx entry-point as a queue entry.

    // tokenAuthStrategy = "none" — payload omitted; buyer has pre-approved the escrow contract.

    // tokenAuthStrategy = "erc3009"
    "tokenAuth": {
      "kind": "erc3009",
      "data": {
        "from": "0xBuyer...", "to": "0xEscrowContract...", "value": "1000000",
        "validAfter": 0, "validBefore": 1730000000, "nonce": "0xabcd...",
        "v": 27, "r": "0x...", "s": "0x..."
      }
    }

    // tokenAuthStrategy = "permit"
    // "tokenAuth": {
    //   "kind": "permit",
    //   "data": {
    //     "owner": "0xBuyer...", "spender": "0xEscrowContract...", "value": "1000000",
    //     "deadline": 1730000000, "nonce": "<token internal nonce>",
    //     "v": 27, "r": "0x...", "s": "0x..."
    //   }
    // }

    // tokenAuthStrategy = "permit2"
    // "tokenAuth": {
    //   "kind": "permit2",
    //   "data": {
    //     "permitted": { "token": "0x...", "amount": "1000000" },
    //     "spender":   "0xEscrowContract...",
    //     "nonce":     "<permit2 nonce>",
    //     "deadline":  1730000000,
    //     "signature": "0x..."
    //   }
    // }
  },

  "fulfillment": {
    "option": "email",
    "data":   { "email": "buyer@example.com" }
  }
}
```

### Field reference (PaymentPayload)

| Field | Required | Notes |
|---|---|---|
| `payload.action` | yes | Which transition the buyer is invoking. Must appear in `actions.next[].id` from the requirements. |
| `payload.tokenAuthStrategy` | yes | One of `tokenAuthStrategies` from the requirements. |
| `payload.offerRef` | yes | Echoed; binds the payload to the specific offer the server signed. |
| `payload.buyer` | yes | Buyer wallet (recovered from sigs by the contract; included for routing). |
| `payload.metaTx` | yes | Escrow meta-tx authorising execution of `<action>` on behalf of `buyer`. EIP-712 signed under the **escrow contract** domain. Implementation-defined type, but the domain binding is normative. |
| `payload.tokenAuth` | iff `tokenAuthStrategy ≠ "none"` | Token-transfer authorization for *this exact spend*. The facilitator passes it to the escrow contract's meta-tx entry-point as a queued authorization consumed during fund transfer. |
| `fulfillment.option` | iff requirements `fulfillment.required = true` | Must be one of `fulfillment.options[].id`. |
| `fulfillment.data` | per the option's schema | Validated against `fulfillment.options[i].schema`. |

## 4. Signatures

### 4.1 Seller — OfferCommitment signature

The seller signs the `OfferCommitment` with EIP-712 under the escrow contract's domain:

```
domain:  { name: "<impl-name>", version: "<version>", chainId, verifyingContract: <escrowContract> }
type:    <implementation-defined OfferCommitment type>
```

The signature is verified on-chain by the escrow contract and MUST support both ECDSA and ERC-1271 (contract wallets).

### 4.2 Buyer — meta-tx (always required)

The buyer signs **one** EIP-712 meta-transaction authorising execution of `<action>` on the escrow contract. This is the only escrow-side signature required regardless of which token-auth strategy the buyer picks.

```
domain:  { name: "<impl-name>", version: "<version>", chainId, verifyingContract: <escrowContract> }
type:    <implementation-defined MetaTransaction type>
         MUST include at minimum:
           - nonce (replay protection)
           - from (buyer address)
           - the action to execute
           - the action arguments (including the echoed OfferCommitment + sellerSig)
```

The escrow contract's existing meta-tx replay protection applies (`usedNonce[from][nonce]` or equivalent).

This nonce provides **authorization replay protection at the action level**: the same signed meta-tx cannot be submitted twice, and cannot be reused against a different escrow contract (the EIP-712 domain binds the signature to `verifyingContract`). Combined with the token-auth nonce enforced independently by the token contract (or Permit2), this gives two independent on-chain replay barriers — one at the action level, one at the funds-transfer level.

### 4.3 Buyer — token-transfer authorization

The buyer picks one of four strategies advertised in `requirements.tokenAuthStrategies`. The chosen strategy's authorization payload is queued in the escrow contract's meta-tx entry-point and consumed during fund transfer.

| Strategy | Buyer signs | Domain | Replay protection | Prior tx required |
|---|---|---|---|---|
| `none` | — | — | — | yes — buyer pre-approves the escrow contract |
| `erc3009` | `ReceiveWithAuthorization(from, to, value, validAfter, validBefore, nonce)` | token's EIP-712 domain | random `nonce`, single-use, enforced by token | no |
| `permit` | `Permit(owner, spender, value, nonce, deadline)` | token's EIP-712 domain | sequential `nonce`, enforced by token | no |
| `permit2` | Uniswap Permit2 `PermitTransferFrom(permitted, spender, nonce, deadline)` | Permit2's EIP-712 domain | bitmap `nonce`, enforced by Permit2 | one-time `approve(Permit2, MaxUint)` per token |

For `erc3009`, `permit`, and `permit2`: `to` / `spender` MUST equal the **escrow contract** address, and `value` MUST equal `requirements.amount`. The token-auth signature is not tied to a forwarder; if replayed elsewhere it just authorizes its own token spend.

### 4.4 No separate release signature for commit-and-release

For `action = <impl>-commitAndRelease`, the release step happens atomically inside the escrow contract's commit-and-release entry-point. The committer is `_msgSender()` of the meta-tx, so the meta-tx signature in §4.2 already authorises the release. **No additional buyer signature is needed.** Note: this is independent of fulfillment timing — the resource may still be fulfilled later via the negotiated fulfillment channel.

## 5. Validation rules (server side, before forwarding to the facilitator)

1. `payload.scheme === requirements.scheme === "escrow"`.
2. `payload.network === requirements.network`.
3. `payload.offerRef.commitment` deep-equals `requirements.offer.commitment` (canonical JSON ordering).
4. `payload.offerRef.sellerSig === requirements.offer.sellerSig`.
5. `payload.action ∈ requirements.actions.next[].id`.
6. `payload.tokenAuthStrategy ∈ requirements.tokenAuthStrategies`.
7. `payload.metaTx` encodes arguments that include the same `commitment` and `sellerSig` as `requirements.offer`.
8. The recovered signer of `payload.metaTx.sig` equals `payload.buyer` and equals `payload.metaTx.from`.
9. For `tokenAuthStrategy = "erc3009"`: `tokenAuth.data.value === requirements.amount`, `tokenAuth.data.to === requirements.escrowAddress`, `tokenAuth.data.validBefore − now ≤ requirements.maxTimeoutSeconds`.
10. For `tokenAuthStrategy = "permit"`: `tokenAuth.data.value === requirements.amount`, `tokenAuth.data.spender === requirements.escrowAddress`, `tokenAuth.data.deadline − now ≤ requirements.maxTimeoutSeconds`.
11. For `tokenAuthStrategy = "permit2"`: `tokenAuth.data.permitted.amount === requirements.amount`, `tokenAuth.data.permitted.token === requirements.asset`, `tokenAuth.data.spender === requirements.escrowAddress`, `tokenAuth.data.deadline − now ≤ requirements.maxTimeoutSeconds`.
12. For `tokenAuthStrategy = "none"`: server SHOULD pre-flight `IERC20.allowance(buyer, escrowContract) ≥ amount` and reject early on insufficient allowance.
13. If `requirements.fulfillment.required`, `payload.fulfillment.option ∈ requirements.fulfillment.options[].id` and `payload.fulfillment.data` validates against the chosen option's `schema`.

A failure on any rule returns `400` with a structured `{ code, field, expected, got }` body. The server does **not** consult the facilitator until rules 1–13 pass.

## 6. JSON Schemas

The canonical JSON Schemas (`payment_requirements.schema.json`, `payment_payload.schema.json`) live in `x402-escrow-core/schemas/`. They are validated in CI against every example in this repo with `ajv`.

Implementation-specific extensions to `offer.commitment` MUST be additive and MUST not overlap with the top-level fields defined in §2 and §3.

## 7. Compatibility with vanilla x402

A vanilla x402 client receiving `{ accepts: [{ scheme: "escrow", ... }] }` does not match any registered scheme handler and surfaces a structured `UnsupportedSchemeError`. **No accidental settle is possible.** The buyer must opt in by installing an escrow-scheme-aware client library.

A server may simultaneously advertise an `exact` and an `escrow` accept entry, letting vanilla x402 clients use the trusted-counterparty path while escrow-aware clients prefer the escrow path. The choice belongs to the client.
