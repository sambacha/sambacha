---
title: "MakerDAO is a Lending Protocol not a Stablecoin"
date: 2025-03-06
tags: [MakerDAO, Smart Contracts, Solidity, DeFi]
---

- [Contract Renaming](#contract-renaming)
- [Variable and Naming Changes](#variable-and-naming-changes)
- [CDP Engine (formerly Vat)](#cdp-engine-formerly-vat)
  - [Risk Parameters for Collateral Types](#risk-parameters-for-collateral-types)
  - [Vault Operations](#vault-operations)
  - [Vault Safety Mechanisms](#vault-safety-mechanisms)
- [CDP Engine Implementation](#cdp-engine-implementation)
  - [Authentication](#authentication)
  - [Data Structures](#data-structures)
  - [Administration](#administration)
  - [CDP Manipulation](#cdp-manipulation)
  - [CDP Fungibility](#cdp-fungibility)
  - [CDP Confiscation](#cdp-confiscation)
  - [Settlement](#settlement)
  - [Rates](#rates)

## Semantic Mapping Changes

To make dear reader's life easier, we adopt the following mappings:

- `end.sol` → `globalSettlement.sol`
- `lib.sol` → `commonFunctions.sol`
- `join.sol` → `adapters.sol`
- `flip.sol` → `collateralAuction.sol`
- `flap.sol` → `surplusAuction.sol`
- `vat.sol` → `CDPEngine.sol`
- `cat.sol` → `liquidations.sol`
- `flop.sol` → `debtAuction.sol`
- `jug.sol` → `stabilityFees.sol`
- `spot.sol` → `oracle.sol`
- `vow.sol` → `settlement.sol`

## Variable and Naming Changes

- Some `uint` variables that used only `0` or `1` values were transformed into `bool`:
  - `uint256 ward` → `bool authorizedAccounts`
  - `uint256 live` → `bool DSRisActive`
- The library contract `Note` is now `LogEmitter` (expanded to include the common `auth` pattern).
- **Gems** are now **Tokens**.
  - `gemLike` (a simple token pattern) is now `SimpleToken`.
- **Flip** and **Flap** auctions both exchange **Collateral Tokens** for **DAI**, but in opposite directions.
  - More intuitive names: "buy" and "sell" auctions.

## CDP Engine (formerly Vat)

The **CDP Engine** is the core **Vault engine** of the Dai System. It stores vaults, tracks associated DAI and collateral balances, and defines immutable rules for vault management—essentially the **constitution of DSS**.

> 📖 [MakerDAO System Glossary - Vat (Vault Engine)](https://docs.makerdao.com/other-documentation/system-glossary#vat-vault-engine)

Users (`msg.sender`) can authorize an address (`usr`) to manipulate their collateral balances and outstanding debt within a vault.

### Risk Parameters for Collateral Types

#### `accumulatedRates`

- Accumulated interest rate used for debt calculations.
- Read more in the [Intro to Rate Mechanism](https://github.com/makerdao/developerguides).

```solidity
uint256 accumulatedRates;  // Accumulated Rates [ray]
```

#### `safetyPrice`

- Medianizer-reported price adjusted by the required collateralization ratio.
- Example: If **price = $200** and **collateralization ratio = 120%**, then **safetyPrice = 166.66**.

```solidity
uint256 safetyPrice;  // Price with Safety Margin [ray]
```

#### `debtCeiling`

- Maximum DAI that can be generated per collateral type.
- Example: If **debtCeiling = 3M**, no new DAI can be minted once 3M is reached until governance updates it.

```solidity
uint256 debtCeiling;  // Debt Ceiling [rad]
```

#### `debtFloor`

- Minimum outstanding debt per vault to ensure economic viability of liquidations.
- Prevents uneconomical liquidations due to gas fees.

```solidity
uint256 debtFloor;  // Vault Debt Floor [rad]
```

#### `debt`

- **Total DAI issued** across all collateral types.
- ⚠️ There are **two variables** named `debt`, with different meanings.

```solidity
uint256 public debt;  // Total Dai Issued [rad]
```

### Vault Operations

#### `slip`

Restricted function allowing collateral token adapters to update user balances for adding/removing collateral.

```solidity
function slip(bytes32 collateralType, address usr, int256 amount) external emitLog onlyOwners {
    tokenCollateral[collateralType][usr] = add(tokenCollateral[collateralType][usr], amount);
}
```

#### `frob`

- Manipulates locked collateral and outstanding debt in a vault.
- Changes must respect debt ceilings and ensure vault stability.
- Calculates **normalized DAI debt** using accumulated rates.

```solidity
require(collateralType.accumulatedRates != 0, "CDPEngineInstance/collateralType-not-init");
urn.lockedCollateral = add(urn.lockedCollateral, dink);
urn.debt = add(urn.debt, dart);
collateralType.debtAmount = add(collateralType.debtAmount, dart);

int dtab = mul(collateralType.accumulatedRates, dart);
uint tab = mul(collateralType.accumulatedRates, urn.debt);
debt = add(debt, dtab);
```

### Vault Safety Mechanisms

- **Safe actions** (e.g., paying down debt) can be performed by any user.
- **Unsafe actions** (e.g., removing collateral) require vault ownership or authorization.
- Ensures:
  - `cool`: Actions reduce outstanding debt.
  - `firm`: Additional collateral is locked.
  - `calm`: Debt ceilings are respected.

## CDP Engine Implementation

The CDP Engine is the core Vault engine of Dai System. It stores Vaults and tracks all the associated Dai and Collateral balances. It also defines the rules by which Vaults and balances can be manipulated. The rules defined in the CDP Engine are immutable, so in some sense, the rules in the CDP Engine can be viewed as the constitution of dss.

Learn more at: [https://docs.makerdao.com/other-documentation/system-glossary#vat-vault-engine](https://docs.makerdao.com/other-documentation/system-glossary#vat-vault-engine)

```solidity
pragma solidity 0.5.12;

contract CDPEngineInstance is Permissioned {
```

### Authentication

The caller (`msg.sender`) can authorize an address `usr` to manipulate their collateral balances and locked collateral or outstanding debt in a Vault.

```solidity
function addAuthorization(address usr) external emitLog onlyOwners {
  require(NormalOperation, "CDPEngineInstance/not-NormalOperation");
  authorizedAccounts[usr] = true;
}

function removeAuthorization(address usr) external emitLog onlyOwners {
  require(NormalOperation, "CDPEngineInstance/not-NormalOperation");
  authorizedAccounts[usr] = false;
}

mapping(address => mapping (address => uint)) public can;
function hope(address usr) external emitLog { can[msg.sender][usr] = 1; }
function nope(address usr) external emitLog { can[msg.sender][usr] = 0; }
function wish(address bit, address usr) internal view returns (bool) {
    return either(bit == usr, can[bit][usr] == 1);
}
```

### Data Structures

#### CollateralType Structure

Total `daiBalance` debt of all vaults of a collateral type

> [!IMPORTANT]
> Total Normalised Debt [amount]

```solidity
uint256 debtAmount;
```

Accumulated interest used in the multiplication to calculate the total debt owed either by a vault or an entire collateral type. Please read the "Intro to Rate Mechanism" developer guide to learn more about how rates work in Maker smart contracts [https://github.com/makerdao/developerguides](https://github.com/makerdao/developerguides)

> [!IMPORTANT]
> Accumulated Rates [ray]

```solidity
uint256 accumulatedRates;
```

Price reported by the medianizer scaled with the collateralization ratio required by a vault. Ex: Price: 200 USD, Collateralization ratio for the collateral type: 120%. Spot will have 166.66. Which means a maximum of 166.66 in debt is allowed by the system for every unit of collateral locked in a Vault.

> [!IMPORTANT]
> Price with Safety Margin [ray]

```solidity
uint256 safetyPrice;
```

Debt ceiling allocated for a collateral type. Ex: 3M daiBalance. A maximum of 3M in daiBalance can be generated by all vaults of a collateral type. Once this limit is reached new daiBalance cannot be generated until governance raises it by updating this risk parameter.

> [!IMPORTANT]
> Debt Ceiling [rad]

```solidity
uint256 debtCeiling;
```

Dust limits that limit the minimum amount of outstanding debt that a vault of this collateral type needs to be open. This ensures vaults with tiny amounts of collateral are not present in the system as they won't be economical for Keepers to liquidate because the gas fees incurred would be higher than the profits gained from liquidation.

> [!IMPORTANT]
> Vault Debt Floor [rad]

```solidity
uint256 debtFloor;
```

#### Vault Structure

Collateral locked in a vault.

> [!IMPORTANT]
> Locked Collateral [amount]

```solidity
uint256 lockedCollateral;
```

Total debt of a vault is always calculated by multiplying debt with CollateralType.accumulatedRates.

> [!IMPORTANT]
> Normalised Debt [amount]

```solidity
uint256 debt;
```

#### Global Mappings and Variables

Mapping to store some of the risk parameters of collateral types. Risk parameters for collateral types are also stored in other contracts too.

```solidity
mapping (bytes32 => CollateralType) public collateralTypes;
```

Mapping to store the collateral locked and outstanding daiBalance debt of all vaults

```solidity
mapping (bytes32 => mapping (address => Vault)) public vaults;
```

Collateral balance of a user within the vault database that is not locked within a vault. This balance cannot be ceased during a vault liquidation. A user needs to first get this balance to lock additional collateral within a vault.

> [!IMPORTANT]
> [amount]

```solidity
mapping (bytes32 => mapping (address => uint)) public tokenCollateral;
```

Dai balances of users. This will not reflect the balances of those holding Dai ERC-20 tokens issued by the adapter contract.

> [!IMPORTANT]
> [rad]

```solidity
mapping (address => uint256) public daiBalance;
```

Bad debt balance of addresses. Typically only the address of the Vow contract would have a badDebt balance in the system.

> [!IMPORTANT]
> [rad]

```solidity
mapping (address => uint256) public badDebt;
```

Total Dai issued across all collateral types by the system. ATTENTION: there are currently two variables with name Debt, meaning different things

> [!IMPORTANT]
> Total Dai Issued [rad]

```solidity
uint256 public debt;
```

Total amount of bad debt in the system.

> [!IMPORTANT]
> Total Unbacked Dai [rad]

```solidity
uint256 public totalBadDebt;
```

Total debt ceiling set for the entire system. It need not add up to all the individual debt ceilings set for each collateral type.

> [!IMPORTANT]
> Total Debt Ceiling [rad]

```solidity
uint256 public totalDebtCeiling;
```

Set to false when the system is in Emergency Shutdown mode and stops users from making any changes to their existing vaults. For detailed sequence of events please refer to End.

> [!IMPORTANT]
> Access Flag

```solidity
bool public NormalOperation;
```

#### Event Logging

EmitLog - a general purpose log that can be added to any function from a contract.

```solidity
event EmitLog(
    bytes4   indexed  sig,
    bytes32  indexed  arg1,
    bytes32  indexed  arg2,
    bytes32  indexed  arg3,
    bytes    data
) anonymous;

modifier emitLog {
    _;
    assembly {
        // log an 'anonymous' event with a constant 6 words of calldata
        // and four indexed topics: the selector and the first three args
        let mark := msize                         // end of memory ensures zero
        mstore(0x40, add(mark, 288))              // update free memory pointer
        mstore(mark, 0x20)                        // bytes type data offset
        mstore(add(mark, 0x20), 224)              // bytes size (padded)
        calldatacopy(add(mark, 0x40), 0, 224)     // bytes payload
        log4(mark, 288,                           // calldata
             shl(224, shr(224, calldataload(0))), // msg.sig
             calldataload(4),                     // arg1
             calldataload(36),                    // arg2
             calldataload(68)                     // arg3
            )
    }
}
```

### Administration

Rate for all collateral types is first initialized to 1 a fixed point integer, with 27 decimal places. It increases continuously every second based on the value of collateral-specific risk premium set for the collateral type on the Stability Fees (former jug) contract.

```solidity
function init(bytes32 collateralType) external emitLog onlyOwners {
    require(collateralTypes[collateralType].accumulatedRates == 0, "CDPEngineInstance/collateralType-already-init");
    collateralTypes[collateralType].accumulatedRates = 10 ** 27;
}
```

Setter methods for total debt ceiling, and to update safetyPrice, debtCeiling, and debtFloor for each collateral type.

```solidity
function setSystemDebtCeiling(uint newDebtCeiling) external emitLog onlyOwners {
    require(NormalOperation, "CDPEngineInstance/not-NormalOperation");
    totalDebtCeiling = newDebtCeiling;
}

function setSafetyPrice(bytes32 collateralType, uint newSafetyPrice) external emitLog onlyOwners {
    require(NormalOperation, "CDPEngineInstance/not-NormalOperation");
    collateralTypes[collateralType].safetyPrice = newSafetyPrice;
}

function setCollateralDebtCeiling(bytes32 collateralType, uint newDebtCeiling) external emitLog onlyOwners {
    require(NormalOperation, "CDPEngineInstance/not-NormalOperation");
    collateralTypes[collateralType].debtCeiling = newDebtCeiling;
}

function setDebtFloor(bytes32 collateralType, uint newDebtFloor) external emitLog onlyOwners {
    require(NormalOperation, "CDPEngineInstance/not-NormalOperation");
    collateralTypes[collateralType].debtFloor = newDebtFloor;
}
```

Sets the emergency shutdown flag to true for this contract.

```solidity
function setEmergencyShutdown() external emitLog onlyOwners {
    NormalOperation = false;
}
```

### Fungibility

This is a restricted function that primarily allows collateral token adapters to update internal collateral balances of users when they add or remove collateral. wad is a signed number and lets this function both add or subtract internal collateral balances. The balances here are not locked within vaults and cannot be ceased during liquidations.

```solidity
function slip(bytes32 collateralType, address usr, int256 amount) external emitLog onlyOwners {
    tokenCollateral[collateralType][usr] = add(tokenCollateral[collateralType][usr], amount);
}
```

Public function used to transfer internal collateral balance of collateral type ilk within Vat from src to dst address.

```solidity
function flux(bytes32 collateralType, address src, address dst, uint256 amount) external emitLog {
    require(wish(src, msg.sender), "CDPEngineInstance/not-allowed");
    tokenCollateral[collateralType][src] = sub(tokenCollateral[collateralType][src], amount);
    tokenCollateral[collateralType][dst] = add(tokenCollateral[collateralType][dst], amount);
}
```

Public function used to transfer internal dai balance of user within Vat to another address.

```solidity
function move(address src, address dst, uint256 rad) external emitLog {
    require(wish(src, msg.sender), "CDPEngineInstance/not-allowed");
    daiBalance[src] = sub(daiBalance[src], rad);
    daiBalance[dst] = add(daiBalance[dst], rad);
}
```

### CDP Manipulation

Manipulates the locked collateral and outstanding debt balance of a vault u with collateral balance from address v and dai balance of address w. Positive dink and dart values add to the existing balances while negative value subtract from them. The function also expects normalized values as inputs, which are then multiplied by rate to arrive at the amount of dai to add or subtract from on the user's address.

```solidity
function frob(bytes32 i, address u, address v, address w, int dink, int dart) external emitLog {
    // Ensures the system is still live and isn't going through the emergency shutdown process.
    require(NormalOperation, "CDPEngineInstance/not-NormalOperation");

    Vault memory urn = vaults[i][u];
    CollateralType memory collateralType = collateralTypes[i];

    // Collateral type has to be initialized for the frob operation to succeed.
    require(collateralType.accumulatedRates != 0, "CDPEngineInstance/collateralType-not-init");

    /* Locked collateral and debt of a vault can be decreased even with the
    add function because their inputs are signed numbers. Total amount of debt
    issued against a collateral type is also updated. */
    urn.lockedCollateral = add(urn.lockedCollateral, dink);
    urn.debt = add(urn.debt, dart);
    collateralType.debtAmount = add(collateralType.debtAmount, dart);


    /* The total change in debt of a vault is determined by multiplying the
    input dart with the current rate of the collateral type i.

    If a user wants to increase their debt by 100 dai, they will have to
    pre-calculate this value by looking at the current rate to find the
    dart input to call frob with.

    Similarly, the total debt issued for the collateral type after the
    vault balances are updated is calculated by multiplying rate of
    the collateral type with the final value stored in urn,art */
    int dtab = mul(collateralType.accumulatedRates, dart);
    uint tab = mul(collateralType.accumulatedRates, urn.debt);
    debt = add(debt, dtab);

    /* The set of actions a user can perform on a vault using frob- adding
    collateral, paying down dai debt, drawing more dai debt, etc. can
    be classified as either safe or unsafe actions from the perspective of a vault.

    Frob allows any user is allowed to perform safe actions like paying down
    debt of a vault. Unsafe actions like removing locked collateral can only
    be performed by the owner or an authorized address of a vault.

    Cool is true when the action pays down outstanding debt of a vault.
    Firm is true when additional collateral is locked in the vault.

    Calm is true when the debt ceilings that limit the total amount of dai
    that can be issued for a collateral type and globally across the entire
    system are respected. The changes already made to the vault are also taken into account.

    Safe is true when the vault is safely over collateralized after the
    changes are made to dai balance and locked collateral balance.
    Spot takes into account both the price of token reported by the price
    feed oracles as well as the liquidation ratio set for the collateral type by MKR holders.

    Irrespective of the msg.sender, one of the following has to evaluate
    to true for the frob action to succeed:

    cool && firm
    calm && safe
    cool && safe
    The owner of the vault or an authorized has to call frob if the action
    isn't both cool and firm. The caller has to be authorized if collateral
    balance of v is being debited. The caller has to be authorized to debit
    dai from address w to pay down outstanding debt of a vault.

    Safe actions on a vault are unsafe actions for the owners of addresses whose
    collateral and dai balances are being decreased, hence the negation on cool
    and firm. */

    // either debt has decreased, or debt ceilings are not exceeded
    require(either(dart <= 0, both(mul(collateralType.debtAmount, collateralType.accumulatedRates) <= collateralType.debtCeiling, debt <= totalDebtCeiling)), "CDPEngineInstance/ceiling-exceeded");
    // urn is either less risky than before, or it is safe
    require(either(both(dart <= 0, dink >= 0), tab <= mul(urn.lockedCollateral, collateralType.safetyPrice)), "CDPEngineInstance/not-safe");

    // urn is either more safe, or the owner consents
    require(either(both(dart <= 0, dink >= 0), wish(u, msg.sender)), "CDPEngineInstance/not-allowed-u");
    // collateral src consents
    require(either(dink <= 0, wish(v, msg.sender)), "CDPEngineInstance/not-allowed-v");
    // debt dst consents
    require(either(dart >= 0, wish(w, msg.sender)), "CDPEngineInstance/not-allowed-w");


    /* Dust limits are set to ensure there is a minimum amount of outstanding debt in a vault. */
    // urn has no debt, or a non-dusty amount
    require(either(urn.debt == 0, tab >= collateralType.debtFloor), "CDPEngineInstance/debtFloor");

    /* The amount of collateral to update in a vault is directly specified by the input dink.
    Changes in the vault are now balanced by adding or removing collateral balance from
    address v, and updating the balance of dai on address w as well as the total sum of
    debt issued by Vat.*/
    tokenCollateral[i][v] = sub(tokenCollateral[i][v], dink);
    daiBalance[w] = add(daiBalance[w], dtab);

    vaults[i][u] = urn;
    collateralTypes[i] = collateralType;
}
```

### CDP Fungibility

Moves locked collateral and/or outstanding debt from one vault src to another vault dst by dink and dart amounts. This is a public function that can be called by an address that is authorized by both src and dst vaults.

```solidity
function fork(bytes32 collateralType, address src, address dst, int dink, int dart) external emitLog {
    Vault storage u = vaults[collateralType][src];
    Vault storage v = vaults[collateralType][dst];
    CollateralType storage i = collateralTypes[collateralType];

    /* Positive values of dink and dart would decrease locked collateral and
    outstanding debt amounts on the src vault and the increased by the
    same amounts in the dst vault. */
    u.lockedCollateral = sub(u.lockedCollateral, dink);
    u.debt = sub(u.debt, dart);
    v.lockedCollateral = add(v.lockedCollateral, dink);
    v.debt = add(v.debt, dart);

    uint utab = mul(u.debt, i.accumulatedRates);
    uint vtab = mul(v.debt, i.accumulatedRates);

    // both sides consent
    /* Both vaults need to authorize the msg.sender to perform this action even if
    it could be a safe action on one vault. */
    require(both(wish(src, msg.sender), wish(dst, msg.sender)), "CDPEngineInstance/not-allowed");

    // both sides safe
    /* Checks to ensure both the vaults remain safe after the adjustments */
    require(utab <= mul(u.lockedCollateral, i.safetyPrice), "CDPEngineInstance/not-safe-src");
    require(vtab <= mul(v.lockedCollateral, i.safetyPrice), "CDPEngineInstance/not-safe-dst");

    // both sides non-dusty
    // Checks to ensure both vaults respect the dust limit set for the collateral type.
    require(either(utab >= i.debtFloor, u.debt == 0), "CDPEngineInstance/debtFloor-src");
    require(either(vtab >= i.debtFloor, v.debt == 0), "CDPEngineInstance/debtFloor-dst");
}
```

### CDP Confiscation

Can increase/decrease locked collateral and outstanding debt on a vault. Changes in debt on a vault u is balanced with an increase or decrease of sin balance on address w, typically Vow. Changes in the amount of locked collateral on a vault is balanced with an increase or decrease of collateral balance of address v. Only sin balance on an address can be modified and dai balance of an address cannot be changed. The following addresses/contracts are currently authorized to call this restricted function: Cat

```solidity
function grab(bytes32 i, address u, address v, address w, int dink, int dart) external emitLog onlyOwners {
    Vault storage urn = vaults[i][u];
    CollateralType storage collateralType = collateralTypes[i];

    /* Both locked collateral and debt of a vault can be decreased even
    with the add function because their inputs are signed numbers. Total
    amount of debt issued against a collateral type is also updated. */
    urn.lockedCollateral = add(urn.lockedCollateral, dink);
    urn.debt = add(urn.debt, dart);
    collateralType.debtAmount = add(collateralType.debtAmount, dart);

    /* Collateral balance of address v and bad debt balance sin of
    address w are updated. Global bad debt amount vice is also updated.*/
    int dtab = mul(collateralType.accumulatedRates, dart);

    tokenCollateral[i][v] = sub(tokenCollateral[i][v], dink);
    badDebt[w] = sub(badDebt[w], dtab);
    totalBadDebt = sub(totalBadDebt, dtab);
}
```

### Settlement

Allows anyone to cancel the bad debt present on an address with its own dai balance. All bad debt in the system is currently accounted for on the address of the Vow contract. A MKR debt auction can be triggered to cover the remaining bad debt after all the surplus on Vow is used up.

```solidity
function heal(uint rad) external emitLog {
    address u = msg.sender;
    /* heal accepts a positive number as an input. Dai balance and bad
    debt balance of the address are reduced. Total bad debt and debt
    in the system are reduced.*/
    badDebt[u] = sub(badDebt[u], rad);
    daiBalance[u] = sub(daiBalance[u], rad);
    totalBadDebt = sub(totalBadDebt, rad);
    debt = sub(debt, rad);
}
```

Generates dai on address u and accounted as bad debt on address v without opening a vault. Governors of the system can call this restricted function through an executive vote to create dai and let Vow handle the resulting bad debt in the system. No addresses/contracts are currently authorized to call this restricted function.

```solidity
function suck(address u, address v, uint rad) external emitLog onlyOwners {
    badDebt[u] = add(badDebt[u], rad);
    daiBalance[v] = add(daiBalance[v], rad);

    // Total debt and bad debt in the system are increased.
    totalBadDebt = add(totalBadDebt, rad);
    debt = add(debt, rad);
}
```

### Rates

Fold increases rate of a collateral type i to proportionally increase the outstanding debt of all its vaults, typically to assess the stability fee. It also increases the dai balance of address u to add a surplus to the Vow contract. It can also decrease the debt of all vaults in collateral type i by debiting the dai balance of address u to reduce the debt of all vaults in a collateral type by decreasing the dai balance of Vow. The following addresses/contracts are currently authorized to call this restricted function: Jug. As compared to Single Collateral Dai, we can now:
* Continually assess stability fee every block and not just collect when debt is repaid by the vault.
* Remove the need for vault owners to pay this fee in MKR.

```solidity
function fold(bytes32 i, address u, int accumulatedRates) external emitLog onlyOwners {
    require(NormalOperation, "CDPEngineInstance/not-NormalOperation");
    CollateralType storage collateralType = collateralTypes[i];

    /* A positive value increases the rate of a collateral type to add
    debt to all its vaults simultaneously. Please note that the outstanding
    debt of a vault at all times is calculated by multiplying it's normalized
    debt value with the rate variable stored for it's collateral type.*/
    collateralType.accumulatedRates = add(collateralType.accumulatedRates, accumulatedRates);

    /* Dai balance of the address u and global debt are updated based on whether
    debt was collected or paid back for a collateral type. */
    int rad = mul(collateralType.debtAmount, accumulatedRates);
    daiBalance[u] = add(daiBalance[u], rad);
    debt = add(debt, rad);
}
```
