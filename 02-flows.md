# 02 — Flows

> **Status:** detailed spec (v0.1). Sequence diagrams are the canonical source for the implementation.

## Flow A — Commit and escrow

The buyer commits to a signed offer; funds are locked in the escrow contract; the escrow contract issues a **proof of commitment** (an implementation-defined on-chain record — e.g. a receipt hash, an NFT voucher, an exchange ID). The server verifies the on-chain state and returns the resource or a delivery receipt. All subsequent steps — delivery confirmation, fund release, and dispute resolution — are driven by the `nextActions` envelope returned with every response.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant S as Resource Server
    participant F as Facilitator
    participant E as Escrow Contract

    Note over S: One-time setup: register seller identity on-chain.<br/>No per-offer on-chain call needed — offers are signed off-chain.

    C->>S: GET /resource
    S->>S: Sign OfferCommitment off-chain (seller's key)
    S-->>C: 402 PaymentRequirements<br/>(scheme=escrow, OfferCommitment + sellerSig,<br/>tokenAuthStrategies, fulfillment options, nextActions)

    Note over C: Buyer chooses commit action,<br/>token-auth strategy, and fulfillment option

    C->>C: Sign meta-tx authorising commit (escrow contract domain)
    C->>C: Sign token-transfer authorization<br/>— omitted when strategy="none"

    C->>S: GET /resource + X-PAYMENT (commit action)
    S->>S: Validate payload (§5 of escrow-scheme.md)
    S->>F: POST /verify, then /settle

    F->>E: submit meta-tx (metaTxParams, tokenAuthorization[], sig)
    Note over E: Verify meta-tx sig<br/>→ transfer funds in (consume token-auth)<br/>→ lock funds in escrow<br/>→ emit CommitEvent, state = COMMITTED<br/>→ issue proof of commitment

    E-->>F: proofOfCommitment, exchangeId, txHash
    F-->>S: proofOfCommitment, exchangeId, txHash

    S->>S: Query escrow contract — verify state=COMMITTED,<br/>seller=self, asset and amount match requirements
    S-->>C: 200 OK<br/>X-PAYMENT-RESPONSE: { exchangeId, proofOfCommitment, txHash }<br/>nextActions: [<impl>-release, <impl>-openDispute, <impl>-cancel, ...]

    Note over C,E: Fulfillment proceeds via the negotiated channel (inline, email, xmtp, ...)<br/>Subsequent state transitions are driven by nextActions.
```

Notes:

- **Proof of commitment** is implementation-defined. It may be an NFT voucher (tradable on secondary markets before redemption), a plain exchange ID, or any on-chain record that proves the buyer's committed stake. The client MUST persist it for subsequent actions.
- The facilitator call is the escrow contract's meta-tx entry-point. The inner commit function locks funds atomically using the queued token-transfer authorization.
- All subsequent actions (delivery confirmation, fund release, dispute) use the `nextActions` envelope. The buyer can invoke any action through any advertised channel — server, facilitator, on-chain direct, MCP, or XMTP — without depending on the server remaining available.
- Whether the resource is fulfilled inline (in the same HTTP 200 body) or asynchronously (via email, XMTP, webhook, etc.) is governed by the negotiated `fulfillment.option`, not by this flow. The on-chain commit and fulfillment are independent dimensions.

## Flow B — Dispute and resolution

After the buyer has committed and the delivery window opens, the buyer may open a dispute if delivery fails or is unsatisfactory. The `escrow` scheme requires that every implementation support at least one resolution method reachable by the buyer on-chain, independently of the seller's server. The specific resolution mechanism is implementation-defined.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client (buyer)
    participant Sv as Resource Server (seller)
    participant R as Resolution Handler
    participant E as Escrow Contract

    Note over C,E: Starting state: post-commit. Dispute window open.

    C->>E: openDispute(exchangeId)<br/>(via server, facilitator, on-chain direct, or MCP — see nextActions)
    E-->>E: state = DISPUTED

    alt Mutual agreement
        Note over C,Sv: Both parties agree on terms off-chain
        Sv->>E: submitResolution(exchangeId, terms, sellerSig)
        C->>E: confirmResolution(exchangeId, terms, buyerSig)
        E-->>E: releaseFunds() per agreed terms → RESOLVED
    else Third-party resolution
        Note over C: Buyer (or seller) escalates to a registered resolver
        C->>E: escalate(exchangeId)
        E-->>R: dispute queued for resolution
        R->>E: resolve(exchangeId, allocation)
        E-->>E: releaseFunds() per resolution → DECIDED
    else Timeout / auto-release
        Note over E: Dispute window expires without buyer action
        E-->>E: releaseFunds() to seller → COMPLETED / RETRACTED
    end
```

Notes:

- **The resolution mechanism is implementation-defined.** An implementation may support only mutual agreement, only a third-party resolver, or both. It may enforce a seller bond that can be slashed on bad delivery. All that this schema requires is that a resolution path exists, that it is reachable on-chain without the seller's cooperation, and that it is described in the `nextActions` envelope.
- The buyer MUST be able to open a dispute directly on-chain (via `onchain` channel) without needing the seller's server. Censorship resistance is a protocol guarantee.
- Timeout behaviour (auto-release to the seller when the buyer does nothing within the dispute window) is recommended but implementation-defined.

## Flow C — Channel fallback (when the server is unreachable)

After the initial 402, every subsequent step has an off-server alternative. Concrete fallback ordering for any post-commit action when the server endpoint is offline:

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant Sv as Server
    participant F as Facilitator
    participant M as MCP
    participant E as Escrow Contract

    C->>Sv: POST /escrow/<action> (preferred)
    Sv-->>C: timeout / 5xx

    C->>F: POST /<action> (next preferred)
    F-->>C: timeout / 5xx

    C->>M: tool: <action>(exchangeId)
    M-->>C: error / unavailable

    C->>E: <action>(exchangeId) (direct on-chain — always available)
    E-->>C: tx confirmed
```

Order is set by `nextActions[i].channels[]` in the most recent server response (or, after server loss, by `requirements.actions.fallback`). The client SDK exposes a `tryAllChannels()` helper that walks the list and returns on the first success.

## Verification points (per flow)

| Flow | Server-side verify | Client-side verify |
|---|---|---|
| A. Commit and escrow | `state === COMMITTED`, `seller === self`, `exchangeToken === asset`, `price === amount` | `txHash` mined; `proofOfCommitment` matches expected OfferCommitment hash |
| B. Dispute and resolution | state transitions match invoked resolution method | event log entries match the resolution path taken |
| C. Fallback | n/a | each channel returns a structured success or moves to the next |
