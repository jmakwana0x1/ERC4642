# ERC-4626 Tokenized Vault Implementation
## Research Phase & Technical Specification

**Document Version:** 1.0  
**Date:** January 9, 2026  
**Status:** Research Complete - Ready for Implementation  
**Author:** Development Team

---

## Executive Summary

This document outlines the research, design decisions, and implementation strategy for building an ERC-4626 compliant tokenized vault. The ERC-4626 standard provides a unified API for tokenized vaults, enabling better composability across DeFi protocols.

**Key Objectives:**
- Full ERC-4626 compliance
- Gas-optimized implementation

---

## Table of Contents

1. [Standard Overview](#1-standard-overview)
2. [Core Requirements](#2-core-requirements)
3. [Architecture Design](#3-architecture-design)
4. [Security Considerations](#4-security-considerations)


---

## 1. Standard Overview

### What is ERC-4626?

ERC-4626 is a standard API for tokenized vaults that represent shares of a single underlying ERC-20 token. It standardizes the technical parameters of yield-bearing vaults while maintaining flexibility for different strategies.

### Benefits

| Benefit | Description |
|---------|-------------|
| **Composability** | Unified interface enables seamless integration across protocols |
| **User Experience** | Consistent behavior reduces user confusion and errors |
| **Developer Efficiency** | Standard API reduces integration time and testing burden |
| **Security** | Well-audited pattern reduces novel attack vectors |

### Architecture Flow

```
┌─────────────────────────────────────────────────────────────┐
│                        ERC-4626 Vault                        │
│                                                              │
│  ┌────────────┐         ┌──────────────┐                   │
│  │   Users    │────────▶│ Deposit/Mint │                   │
│  │            │         └──────┬───────┘                   │
│  │            │                │                            │
│  │            │                ▼                            │
│  │            │         ┌──────────────┐                   │
│  │            │         │Share Issuance│                   │
│  │            │         └──────┬───────┘                   │
│  │            │                │                            │
│  │            │                ▼                            │
│  │            │         ┌──────────────┐                   │
│  │            │         │ Asset Pool   │◀────Yield         │
│  │            │         └──────┬───────┘     Strategy      │
│  │            │                │                            │
│  │            │                ▼                            │
│  │            │         ┌──────────────┐                   │
│  │            │◀────────│Withdraw/Redeem│                  │
│  └────────────┘         └──────────────┘                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Flow: Asset → Shares → Yield Generation → Asset Return
```

---

## 2. Core Requirements

### 2.1 Mandatory Functions

#### Deposit & Withdrawal Operations

| Function | Input | Output | Purpose |
|----------|-------|--------|---------|
| `deposit` | assets, receiver | shares | Deposit exact assets, receive shares |
| `mint` | shares, receiver | assets | Mint exact shares, deposit assets |
| `withdraw` | assets, receiver, owner | shares | Withdraw exact assets, burn shares |
| `redeem` | shares, receiver, owner | assets | Redeem exact shares, receive assets |

#### Accounting Functions

| Function | Returns | Purpose |
|----------|---------|---------|
| `totalAssets` | uint256 | Total assets managed by vault |
| `convertToShares` | uint256 | Convert asset amount to shares |
| `convertToAssets` | uint256 | Convert shares to asset amount |
| `previewDeposit` | uint256 | Simulate deposit operation |
| `previewMint` | uint256 | Simulate mint operation |
| `previewWithdraw` | uint256 | Simulate withdraw operation |
| `previewRedeem` | uint256 | Simulate redeem operation |

#### Limit Functions

| Function | Returns | Purpose |
|----------|---------|---------|
| `maxDeposit` | uint256 | Maximum deposit for user |
| `maxMint` | uint256 | Maximum mint for user |
| `maxWithdraw` | uint256 | Maximum withdrawal for user |
| `maxRedeem` | uint256 | Maximum redemption for user |

#### Metadata

| Function | Returns | Purpose |
|----------|---------|---------|
| `asset` | address | Underlying asset token |
| `totalSupply` | uint256 | Total vault shares |
| `balanceOf` | uint256 | User's share balance |

### 2.2 State Diagram

```
                    ┌──────────────┐
                    │              │
                    │  User Wallet │
                    │              │
                    └──────┬───────┘
                           │
                           │ approve()
                           ▼
                    ┌──────────────┐
                    │              │
          ┌─────────│    Vault     │◀─────────┐
          │         │              │          │
          │         └──────┬───────┘          │
          │                │                  │
  deposit/mint             │            withdraw/redeem
          │                │                  │
          │                ▼                  │
          │         ┌──────────────┐          │
          │         │              │          │
          └────────▶│  Asset Pool  │──────────┘
                    │              │
                    └──────────────┘
                           │
                           │ Yield Accrual
                           ▼
                    [Share Value ↑]
```

### 2.3 Events

| Event | Parameters | Emitted When |
|-------|------------|--------------|
| `Deposit` | caller, owner, assets, shares | Assets deposited |
| `Withdraw` | caller, receiver, owner, assets, shares | Assets withdrawn |

---

## 3. Architecture Design

### 3.1 Contract Structure

```
ERC4626Vault
│
├── Inherits: ERC20 (OpenZeppelin)
├── Uses: SafeERC20 (OpenZeppelin)
│
├── State Variables
│   ├── _asset (immutable IERC20)
│   └── _decimals (immutable uint8)
│
├── External Functions
│   ├── Deposit Operations
│   │   ├── deposit()
│   │   └── mint()
│   ├── Withdrawal Operations
│   │   ├── withdraw()
│   │   └── redeem()
│   └── View Functions
│       ├── Preview Functions
│       ├── Conversion Functions
│       └── Limit Functions
│
└── Internal Functions
    ├── _convertToShares()
    ├── _convertToAssets()
    ├── _deposit()
    └── _withdraw()
```

### 3.2 Share Calculation Formula

**Shares on Deposit:**
```
shares = (assets × totalSupply) / totalAssets

Special case (first deposit):
shares = assets
```

**Assets on Withdrawal:**
```
assets = (shares × totalAssets) / totalSupply
```

### 3.3 Rounding Strategy

| Operation | Rounding Direction | Reason |
|-----------|-------------------|--------|
| `deposit` → shares | Round Down | Favors vault, prevents share inflation attack |
| `mint` → assets | Round Up | Favors vault, user pays ceiling |
| `withdraw` → shares | Round Up | Favors vault, burns more shares |
| `redeem` → assets | Round Down | Favors vault, user receives floor |

**Rationale:** Always round in favor of the vault to prevent economic attacks and share manipulation.

---

## 4. Security Considerations

### 4.1 Known Attack Vectors

| Attack | Description | Mitigation |
|--------|-------------|------------|
| **Share Inflation** | Attacker makes first deposit of 1 wei, then donates large amount to inflate share price | Round down on deposits; consider minimum first deposit |
| **Reentrancy** | Malicious token callbacks during transfer | Follow CEI pattern; use SafeERC20 |
| **Front-running** | MEV bots sandwich user transactions | Use preview functions; implement slippage protection in UI |
| **Donation Attack** | Direct asset transfer to inflate share value | Not preventable; document expected behavior |

### 4.2 Security Checklist

- [ ] Use OpenZeppelin's audited contracts
- [ ] Implement proper rounding (favor vault)
- [ ] Follow Checks-Effects-Interactions pattern
- [ ] Use SafeERC20 for all token transfers
- [ ] Validate all inputs
- [ ] Emit events for all state changes
- [ ] Test edge cases (zero amounts, first deposit, etc.)
- [ ] Fuzz test conversion functions
- [ ] Consider minimum deposit requirement
- [ ] Document all assumptions

### 4.3 Threat Model

```
┌─────────────────────────────────────────────────────────┐
│                    Threat Landscape                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  High Risk                                               │
│  ├── Share Manipulation (First Deposit Attack)          │
│  └── Reentrancy on Token Transfer                       │
│                                                          │
│  Medium Risk                                             │
│  ├── Front-running Deposits/Withdrawals                 │
│  ├── Donation Attack (Direct Transfers)                 │
│  └── Integer Overflow/Underflow                         │
│                                                          │
│  Low Risk                                                │
│  ├── Access Control Issues                              │
│  └── Gas Griefing                                        │
│                                                          │
└─────────────────────────────────────────────────────────┘
```


#### Critical Risks

**Share Inflation Attack**
- Strategy: Round all conversions in favor of vault
- Implementation: Custom rounding logic in `_convertToShares` and `_convertToAssets`
- Validation: Fuzz test edge cases
- Additional: Consider minimum first deposit of 1e6 wei

#### High Risks

**Reentrancy**
- Strategy: Use OpenZeppelin's SafeERC20
- Implementation: Follow CEI (Checks-Effects-Interactions) pattern
- Validation: Test with malicious ERC20 mock
- Additional: Consider ReentrancyGuard if custom logic added

### 7.3 Acceptance Criteria

Before marking complete, the implementation must satisfy:

| Criteria | Requirement | Status |
|----------|-------------|--------|
| **Standards Compliance** | 100% ERC-4626 compliant | ⏳ Pending |
| **Test Coverage** | ≥95% line coverage | ⏳ Pending |
| **Security** | All high/critical risks mitigated | ⏳ Pending |
| **Gas Efficiency** | Deposit/withdraw <150k gas | ⏳ Pending |
| **Documentation** | Full NatSpec + README | ⏳ Pending |
| **Code Quality** | Passes Slither analysis | ⏳ Pending |

---

---

## References & Resources

### ERC-4626 Resources
- [EIP-4626 Specification](https://eips.ethereum.org/EIPS/eip-4626)
- [OpenZeppelin ERC4626 Implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC4626.sol)
- [Solmate ERC4626 Implementation](https://github.com/transmissions11/solmate/blob/main/src/mixins/ERC4626.sol)

### Security Resources
- [ERC-4626 Security Considerations](https://docs.openzeppelin.com/contracts/4.x/erc4626)
- [Trail of Bits - ERC4626 Inflation Attack](https://blog.trailofbits.com/)
- [Consensys Best Practices](https://consensys.github.io/smart-contract-best-practices/)

### Testing Resources
- [Foundry Book](https://book.getfoundry.sh/)
- [Fuzz Testing Guide](https://book.getfoundry.sh/forge/fuzz-testing)
- [Invariant Testing](https://book.getfoundry.sh/forge/invariant-testing)

---


### A. Glossary

| Term | Definition |
|------|------------|
| **Vault** | Smart contract holding assets and issuing shares |
| **Shares** | ERC20 tokens representing vault ownership |
| **Assets** | Underlying ERC20 tokens deposited in vault |
| **Yield** | Additional assets earned by vault strategy |
| **Share Price** | Ratio of totalAssets to totalSupply |

### B. Comparison with Alternatives

| Feature | ERC-4626 | Custom Vault | Yield Aggregator |
|---------|----------|--------------|------------------|
| Standardization | ✅ Yes | ❌ No | ⚠️ Partial |
| Composability | ✅ High | ❌ Low | ⚠️ Medium |
| Flexibility | ⚠️ Medium | ✅ High | ⚠️ Medium |
| Integration Cost | ✅ Low | ❌ High | ⚠️ Medium |
| Security | ✅ Well-tested | ⚠️ Varies | ⚠️ Varies |



---

**Document Status:** ✅ Approved for Implementation  
**Next Review Date:** Before deployment to mainnet  
**Version History:**
- v1.0 (2026-01-09): Initial research document