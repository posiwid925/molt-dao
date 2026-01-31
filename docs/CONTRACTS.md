# Contract Specifications

## Overview

Molt DAO requires three core contracts deployed on Base:

1. **MoltShare** — ERC-20 equity token (economic stake)
2. **MoltSoul** — ERC-5192 soulbound reputation token (governance weight)
3. **MoltTreasury** — Gnosis Safe for fund management

---

## 1. MoltShare.sol

### Standard
ERC-20 with vesting

### Cap Table

| Allocation | % | Shares |
|------------|---|--------|
| Founder (posiwid) | 15% | 150,000 |
| Contributor Pool | 25% | 250,000 |
| Public Sale | 60% | 600,000 |
| **Total** | 100% | 1,000,000 |

See [CAP_TABLE.md](./CAP_TABLE.md) for full details.

### Core Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MoltShare is ERC20, Ownable {
    uint256 public constant TOTAL_SUPPLY = 1_000_000;
    uint256 public constant FOUNDER_ALLOCATION = 150_000;      // 15%
    uint256 public constant CONTRIBUTOR_ALLOCATION = 250_000;  // 25%
    uint256 public constant PUBLIC_ALLOCATION = 600_000;       // 60%
    
    uint256 public publicPrice;
    uint256 public publicSold;
    address public treasury;
    address public contributorVesting;
    
    // Vesting
    struct VestingSchedule {
        uint256 total;
        uint256 claimed;
        uint256 startTime;
        uint256 cliffDuration;
        uint256 vestDuration;
    }
    
    mapping(address => VestingSchedule) public vesting;
    
    constructor(
        address _founder,
        address _contributorVesting,
        address _treasury,
        uint256 _publicPrice
    ) ERC20("Molt DAO Share", "MOLT-SHARE") Ownable(msg.sender) {
        treasury = _treasury;
        contributorVesting = _contributorVesting;
        publicPrice = _publicPrice;
        
        // Mint founder allocation (vested)
        _mint(_founder, FOUNDER_ALLOCATION);
        vesting[_founder] = VestingSchedule({
            total: FOUNDER_ALLOCATION,
            claimed: 0,
            startTime: block.timestamp,
            cliffDuration: 365 days,
            vestDuration: 1095 days // 3 years
        });
        
        // Mint contributor pool to vesting contract
        _mint(_contributorVesting, CONTRIBUTOR_ALLOCATION);
        
        // Mint public allocation to this contract for sale
        _mint(address(this), PUBLIC_ALLOCATION);
    }
    
    /// @notice Purchase shares during public sale
    function buyShares(uint256 amount) external payable {
        require(publicSold + amount <= PUBLIC_ALLOCATION, "Exceeds public allocation");
        require(msg.value >= publicPrice * amount, "Insufficient payment");
        
        publicSold += amount;
        _transfer(address(this), msg.sender, amount);
        
        // Forward to treasury
        (bool success, ) = treasury.call{value: msg.value}("");
        require(success, "Treasury transfer failed");
    }
    
    /// @notice Get vested (transferable) balance
    function vestedBalance(address account) public view returns (uint256) {
        VestingSchedule memory v = vesting[account];
        if (v.total == 0) return balanceOf(account);
        
        uint256 elapsed = block.timestamp - v.startTime;
        if (elapsed < v.cliffDuration) return 0;
        
        uint256 vestingElapsed = elapsed - v.cliffDuration;
        uint256 vested = v.total * vestingElapsed / v.vestDuration;
        if (vested > v.total) vested = v.total;
        
        return vested;
    }
    
    /// @notice Override transfer to check vesting
    function _update(
        address from,
        address to,
        uint256 value
    ) internal override {
        if (from != address(0) && from != address(this)) {
            uint256 transferable = vestedBalance(from);
            uint256 locked = balanceOf(from) - transferable;
            require(balanceOf(from) - value >= locked, "Exceeds vested balance");
        }
        super._update(from, to, value);
    }
    
    /// @notice Check if address is a shareholder
    function isShareholder(address account) external view returns (bool) {
        return balanceOf(account) > 0;
    }
    
    /// @notice No decimals - whole shares only
    function decimals() public pure override returns (uint8) {
        return 0;
    }
    
    /// @notice End public sale, return unsold to treasury
    function endPublicSale() external onlyOwner {
        uint256 unsold = PUBLIC_ALLOCATION - publicSold;
        if (unsold > 0) {
            _transfer(address(this), treasury, unsold);
        }
    }
}
```

### Key Features
- Fixed 1M supply, all minted at deployment
- Built-in vesting with cliff
- Public sale at fixed price
- Whole shares only (no decimals)
- `isShareholder()` check for governance

---

## 2. ContributorVesting.sol

### Purpose
Manages contributor pool allocation with vesting.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";

interface IMoltShare {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract ContributorVesting is Ownable {
    IMoltShare public moltShare;
    
    struct Allocation {
        uint256 total;
        uint256 claimed;
        uint256 startTime;
    }
    
    mapping(address => Allocation) public allocations;
    uint256 public totalAllocated;
    
    uint256 public constant CLIFF = 180 days;      // 6 months
    uint256 public constant VEST_DURATION = 730 days; // 2 years
    
    constructor(address _moltShare) Ownable(msg.sender) {
        moltShare = IMoltShare(_moltShare);
    }
    
    /// @notice Allocate shares to contributor (owner only, pre-launch)
    function allocate(address contributor, uint256 amount) external onlyOwner {
        require(allocations[contributor].total == 0, "Already allocated");
        uint256 available = moltShare.balanceOf(address(this)) - totalAllocated;
        require(amount <= available, "Exceeds available");
        
        allocations[contributor] = Allocation({
            total: amount,
            claimed: 0,
            startTime: block.timestamp
        });
        totalAllocated += amount;
    }
    
    /// @notice Claim vested shares
    function claim() external {
        Allocation storage a = allocations[msg.sender];
        require(a.total > 0, "No allocation");
        
        uint256 vested = vestedAmount(msg.sender);
        uint256 claimable = vested - a.claimed;
        require(claimable > 0, "Nothing to claim");
        
        a.claimed += claimable;
        moltShare.transfer(msg.sender, claimable);
    }
    
    /// @notice Calculate vested amount for contributor
    function vestedAmount(address contributor) public view returns (uint256) {
        Allocation memory a = allocations[contributor];
        if (a.total == 0) return 0;
        
        uint256 elapsed = block.timestamp - a.startTime;
        if (elapsed < CLIFF) return 0;
        
        uint256 vestingElapsed = elapsed - CLIFF;
        uint256 vested = a.total * vestingElapsed / VEST_DURATION;
        if (vested > a.total) vested = a.total;
        
        return vested;
    }
    
    /// @notice Revoke allocation (forfeits unvested, owner only)
    function revoke(address contributor) external onlyOwner {
        Allocation storage a = allocations[contributor];
        require(a.total > 0, "No allocation");
        
        uint256 vested = vestedAmount(contributor);
        uint256 unvested = a.total - vested;
        
        a.total = vested;
        totalAllocated -= unvested;
        // Unvested shares remain in contract for reallocation
    }
}
```

### Key Features
- 6-month cliff, 2-year vest
- Owner allocates shares to contributors
- Contributors claim as they vest
- Revocation returns unvested to pool

---

## 3. MoltSoul.sol

### Standard
ERC-5192 (Soulbound Token)

### Purpose
Reputation score that multiplies voting power. Earned through contribution, not purchased.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

interface IERC5192 {
    event Locked(uint256 tokenId);
    function locked(uint256 tokenId) external view returns (bool);
}

interface IMoltShare {
    function isShareholder(address account) external view returns (bool);
}

contract MoltSoul is ERC721, IERC5192, AccessControl {
    bytes32 public constant SCORE_UPDATER = keccak256("SCORE_UPDATER");
    
    struct Soul {
        uint256 score;
        uint256 lastActive;
    }
    
    mapping(uint256 => Soul) public souls;
    mapping(address => uint256) public tokenOf;
    uint256 public nextTokenId = 1;
    
    IMoltShare public moltShare;
    
    constructor(address _moltShare) ERC721("Molt Soul", "MOLT-SOUL") {
        moltShare = IMoltShare(_moltShare);
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
    
    /// @notice Mint soul (one per wallet, must be shareholder)
    function mint() external {
        require(tokenOf[msg.sender] == 0, "Already has soul");
        require(moltShare.isShareholder(msg.sender), "Not a shareholder");
        
        uint256 tokenId = nextTokenId++;
        _safeMint(msg.sender, tokenId);
        tokenOf[msg.sender] = tokenId;
        
        souls[tokenId] = Soul({
            score: 1,
            lastActive: block.timestamp
        });
        
        emit Locked(tokenId);
    }
    
    /// @notice Get voting power: shares × soul score
    function votingPower(address account) external view returns (uint256) {
        uint256 tokenId = tokenOf[account];
        if (tokenId == 0) return 0;
        return souls[tokenId].score;
    }
    
    /// @notice Update score (governance contracts only)
    function updateScore(address account, int256 delta) external onlyRole(SCORE_UPDATER) {
        uint256 tokenId = tokenOf[account];
        require(tokenId != 0, "No soul");
        
        Soul storage soul = souls[tokenId];
        if (delta > 0) {
            soul.score += uint256(delta);
        } else {
            uint256 decrease = uint256(-delta);
            soul.score = soul.score > decrease ? soul.score - decrease : 1;
        }
        soul.lastActive = block.timestamp;
    }
    
    /// @notice Soulbound - always locked
    function locked(uint256) external pure override returns (bool) {
        return true;
    }
    
    /// @notice Prevent transfers
    function _update(address to, uint256 tokenId, address auth) internal override returns (address) {
        address from = _ownerOf(tokenId);
        require(from == address(0) || to == address(0), "Soulbound");
        return super._update(to, tokenId, auth);
    }
    
    function supportsInterface(bytes4 interfaceId) public view override(ERC721, AccessControl) returns (bool) {
        return interfaceId == type(IERC5192).interfaceId || super.supportsInterface(interfaceId);
    }
}
```

### Key Features
- Soulbound (non-transferable)
- One per wallet
- Requires shareholding to mint
- Score starts at 1, grows with contribution
- Governance uses score as vote multiplier

---

## 4. MoltTreasury

### Approach
Use **Gnosis Safe** on Base.

### Setup
- Deploy Gnosis Safe
- Initial signers: founding contributors
- Threshold: 2-of-3 or 3-of-5

### Functions
- Receive public sale proceeds
- Fund approved projects
- Distribute dividends
- Manage operational expenses

### Future
Custom Safe module for automated dividend distribution based on share ownership.

---

## Deployment Order

1. Deploy Gnosis Safe (treasury)
2. Deploy MoltShare (with founder, vesting, treasury addresses)
3. Deploy ContributorVesting (with MoltShare address)
4. Deploy MoltSoul (with MoltShare address)
5. Allocate contributor shares via ContributorVesting
6. Open public sale
7. Grant SCORE_UPDATER to governance contracts

---

## Security

- [ ] Peer review all contracts
- [ ] Test on Base Sepolia
- [ ] Consider formal audit before mainnet
- [ ] Document admin functions and their risks
- [ ] Plan for contract upgrades if needed

---

## Parameters (TBD)

| Parameter | Decision Needed |
|-----------|-----------------|
| Public sale price | ETH per share |
| Sale duration | Days/weeks |
| Gnosis Safe signers | Initial set |
| Safe threshold | 2-of-3? 3-of-5? |

---

*Spec v0.2 — Updated to equity model with cap table.*
