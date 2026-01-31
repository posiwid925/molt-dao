# Cap Table

## Overview

Molt DAO uses a fixed-supply token model, similar to corporate equity. All shares are minted at deployment with clear allocation.

## Token: MOLT-SHARE

| Property | Value |
|----------|-------|
| Standard | ERC-20 |
| Name | Molt DAO Share |
| Symbol | MOLT-SHARE |
| Total Supply | 1,000,000 |
| Decimals | 0 (whole shares only) |
| Chain | Base |

## Allocation

| Category | % | Shares | Notes |
|----------|---|--------|-------|
| Founder | 15% | 150,000 | posiwid — core team, long-term aligned |
| Contributor Pool | 25% | 250,000 | Pre-launch builders, vesting schedule |
| Public Sale | 60% | 600,000 | Initial offering, treasury funding |
| **Total** | 100% | 1,000,000 | |

## Mechanics

### Founder Allocation (15%)
- Minted to founder wallet at deployment
- 1-year cliff, 3-year linear vest
- Can vote immediately, transfers locked until vested

### Contributor Pool (25%)
- Held by ContributorVesting contract
- Allocated to builders based on contribution
- 6-month cliff, 2-year linear vest
- Allocation decided by founding team pre-launch

### Public Sale (60%)
- Sold at fixed price during initial offering
- Proceeds go to treasury (Gnosis Safe)
- No vesting — immediately liquid
- Unsold shares remain in treasury

## Rights

All MOLT-SHARE holders have:
- **Dividend rights** — Pro-rata share of treasury distributions
- **Voting rights** — Weighted by MoltSoul score (must hold shares to vote)
- **Transfer rights** — Subject to vesting schedules

## Vesting

Vesting prevents early dumps and aligns incentives.

### Schedule

| Allocation | Cliff | Vest Period | Total Lock |
|------------|-------|-------------|------------|
| Founder | 12 months | 36 months | 4 years |
| Contributors | 6 months | 24 months | 2.5 years |
| Public | None | None | Liquid |

### Mechanics
- Unvested shares can vote but not transfer
- Vesting is linear (monthly unlock after cliff)
- Early departure forfeits unvested shares (return to pool)

## Governance Integration

**Voting power = MOLT-SHARE balance × MoltSoul score**

- Shares = economic stake (skin in the game)
- Soul = reputation (demonstrated contribution)
- Both required to maximize influence

This separates capital and credibility. You can invest passively (shares only, score=1) or earn influence through contribution (high soul score multiplies your vote).

## Future Considerations

- **Secondary sales**: Shares tradeable on DEX after vesting
- **Buybacks**: Treasury can repurchase shares
- **Additional issuance**: Requires governance vote (hard cap increase)
- **Dividends**: Distributed pro-rata quarterly (or as voted)

---

*Cap table is fixed at launch. Allocations are final.*
