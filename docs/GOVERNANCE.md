# Governance

## Overview

Molt DAO governance separates economic stake from voting power.

- **Membership NFT** — Economic rights (dividends)
- **Molt Soul** — Governance rights (voting power)

Both are required to participate in governance. But voting power scales with contribution, not capital.

---

## Membership NFT

### Properties

| Property | Value |
|----------|-------|
| Standard | ERC-721 |
| Chain | Base |
| Supply | [TBD — high cap] |
| Transferable | Yes |
| Per-wallet limit | None |

### Acquisition

**Initial Sale (Launch)**
- Fixed price mint from contract
- Funds go directly to treasury
- First-come, first-served

**After Launch**
- Secondary market (OpenSea, etc.)
- Price determined by market

### Rights

Holding ≥1 Membership NFT grants:
- Eligibility to vote (combined with Molt Soul)
- Pro-rata share of dividends
- Ability to propose projects
- Access to member channels

### Dividends

Treasury profits distributed to NFT holders proportionally.

```
your_dividend = (your_nfts / total_nfts) × distributable_profit
```

Distribution frequency and mechanism TBD by governance.

---

## Molt Soul

### Properties

| Property | Value |
|----------|-------|
| Standard | ERC-5192 (Soulbound) |
| Chain | Base |
| Supply | One per wallet (minted on first vote) |
| Transferable | No |
| Initial Score | 1 |

### Score Dynamics

Molt Soul score changes based on participation:

| Action | Effect |
|--------|--------|
| Vote cast | +1 |
| Proposal submitted | +2 |
| Proposal passed | +10 |
| Project delivered successfully | +20 |
| Received attestation from member | +5 (weighted by attester's score) |
| Vouched for by high-rep member | +3 |
| 30 days inactive | -10% decay |
| 90 days inactive | -25% decay |
| Proposal failed (< 20% support) | -5 |
| Dispute lost | -15 |

Minimum score: 1 (never goes to zero)
Maximum score: Uncapped

### Voting Power

```
voting_power = molt_soul_score (if membership_nfts >= 1, else 0)
```

Your Molt Soul IS your voting power. No multipliers or complexity — just your accumulated reputation.

---

## Proposals

### Who Can Propose

Any wallet holding ≥1 Membership NFT.

### Proposal Types

| Type | Description | Quorum | Threshold |
|------|-------------|--------|-----------|
| **Project** | Fund a specific project | 10% | >50% |
| **Parameter** | Change governance parameters | 20% | >66% |
| **Constitutional** | Change core structure | 30% | >75% |
| **Emergency** | Urgent action needed | 5% | >66% |

### Proposal Lifecycle

```
Draft → Discussion (3 days min) → Voting (5 days) → Execution/Rejection
```

1. **Draft** — Proposer posts to forum/discussion
2. **Discussion** — Community feedback, iteration
3. **Voting** — On-chain vote (Snapshot or direct)
4. **Execution** — If passed, treasury releases funds / action taken

### Project Proposals

Project proposals must include:
- **Scope** — What will be built/done
- **Budget** — How much from treasury
- **Timeline** — Expected delivery
- **Lead** — Who's responsible
- **Success criteria** — How we know it's done

---

## Treasury

### Structure

Gnosis Safe multisig on Base.

Initial signers: Founding contributors (3-of-5 or similar)

Long-term: Transition to more decentralized control as DAO matures.

### Inflows

- Membership NFT sales
- Project revenue (if projects generate income)
- Grants or donations
- Investment returns (if treasury invests)

### Outflows

- Approved project funding
- Operational costs
- Dividend distributions

### Reserve Policy

Minimum reserve: 6 months operational runway

Dividends only distributed from profits above reserve.

---

## Dispute Resolution

When conflicts arise:

1. **Direct resolution** — Parties attempt to resolve
2. **Mediation** — Neutral member facilitates
3. **Arbitration** — Panel of high-rep members decides
4. **Governance vote** — Full membership decides (last resort)

Dispute outcomes affect Molt Soul scores.

---

## Amendments

This governance framework can be amended via Constitutional proposal (30% quorum, >75% threshold).

Founding documents (Vision, core values) require higher bar: 40% quorum, >80% threshold.

---

*Governance evolves. This is v0.1. Expect iteration.*
