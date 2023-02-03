### Issues Template
| Letter | Name | Description |
|:--:|:-------:|:-------:|
| L  | Low risk | Potential risk |
| NC |  Non-critical | Non risky findings |
| R  | Refactor | Changing the code |
| O | Ordinary | Often found issues |

| Total Found Issues | 33 |
|:--:|:--:|

### Low Risk Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [L-01] | Unnecessary KYC check on the payer | 1 |
| [L-02] | Wrong if statement in the function `_acceptAdmin` | 1 |
| [L-03] | Token transfer to address(0) should be avoided | 1 |
| [L-04] | The maximum fee of 10_000 isn't allowed in the function `setMintFee` | 1 |
| [L-05] | Giving KYC status to address(0) should be forbidden | 1 |
| [L-06] | Redeem limit shouldn't be set below the `currentMintAmount` | 1 |
| [L-07] | `completeRedemptions` is vulnerable to admin mistakes | 1 |

| Total Low Risk Issues | 7 |
|:--:|:--:|

### Non-Critical Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [N-01] | Mandatory checks for extra safety | 3 |
| [N-02] | Constructor lacks address(0) check | 5 |
| [N-03] | Unrealistic if statement | 9 |
| [N-04] | Missing check to ensure epoch duration isn't set as zero seconds | 1 |
| [N-05] | Upgradeable contract is missing a __gap[50] storage variable | 2 |
| [N-06] | Create your own import names instead of using the regular ones | 19 | 
| [N-07] | Initialize function does not use the initializer modifier | 2 |
| [N-08] | Use delete to clear variables instead of zero assignment | 2 |
| [N-09] | Confusing function comment on `setMinimumDepositAmount` | 1 |
| [N-10] | Unnecessary if statement in `_checkAndUpdateRedeemLimit` | 1 |
| [N-11] | Pause/Unpause functions should emit event to notify users | 2 |
| [N-12] | Two KYC checks made on the same redeemers | 1 |
| [N-13] | Unused constructor | 2 |
| [N-14] | NatSpec is incomplete in the pausing functions | 2 |

| Total Non-Critical Issues | 14 |
|:--:|:--:|

### Refactor Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [R-01] | Shorthand way to write if / else statement | 3 |
| [R-02] | Modifier should be used instead of require on admin functions | 9 |
| [R-03] | Unused variables should be deleted | 2 |
| [R-04] | Use require instead of assert | 1 |
| [R-05] | Immutable should be used on variables that can't be changed | 1 |
| [R-06] | Unnecessary return statement applied | 1 |
| [R-07] | Numeric values having to do with time should use time units for readability | 1 |

| Total Refactor Issues | 7 |
|:--:|:--:|

### Ordinary Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [O-01] | Floating pragma | 10 |
| [O-02] | Outdated Compiler Version | 3 |
| [O-03] | Function Naming suggestions | 2 |
| [O-04] | Proper use of get as a function name prefix | 2 | 
| [O-05] | Events is missing indexed fields | 1 |

| Total Ordinary Issues | 5 |
|:--:|:--:|


### [L-01] Unnecessary KYC check on the payer
The main use of the function `repayBorrowBehalf` is that a user can pay back a loan on behalf of the borrower.
The problem here is that the function `repayBorrowFresh` checks if both the payer and the borrower are KYCed.
In my opinion the check for the payer is unnecessary and shouldn't be restricted like that.
As long as the borrower is KYCed, everything should be fine.

```solidity
contracts/lending/tokens/cCash/CCash.sol

121:  function repayBorrowBehalf(
122:    address borrower,
123:    uint repayAmount
124:  ) external override returns (uint) {
125:   repayBorrowBehalfInternal(borrower, repayAmount);
126:    return NO_ERROR;
127:  }

contracts/lending/tokens/cCash/CTokenCash.sol

767:  function repayBorrowFresh(
768:    address payer,
769:    address borrower,
770:    uint repayAmount
771:  ) internal returns (uint) {
772:    /* Revert if not KYC'd */
773:    require(_getKYCStatus(payer), "Payer not KYC'd");
774:    require(_getKYCStatus(borrower), "Borrower not KYC'd");
```

### [L-02] Wrong if statement in the function _acceptAdmin
In the contract CTokenCash.sol in order to change the current admin, two step procces is made.
The problem occurs in the function `_acceptAdmin`, where the if statement is wrongly written.
As how it is now the if statement is triggered if the msg.sender isn't the pending admin and the msg sender is address(0).
A mistake was made instead of `msg.sender == address(0)`, it should be `pendingAdmin != address(0)`.
The function `_acceptAdmin` is supposed to revert if `_setPendingAdmin` isn't called first.

The if statement in the function should look like this:
`if (msg.sender != pendingAdmin || pendingAdmin != address(0)) {`

```solidity
function _acceptAdmin() external override returns (uint) {
    // Check caller is pendingAdmin and pendingAdmin ≠ address(0)
    if (msg.sender != pendingAdmin || msg.sender == address(0)) {
      revert AcceptAdminPendingAdminCheck();
    }

    // Save current values for inclusion in log
    address oldAdmin = admin;
    address oldPendingAdmin = pendingAdmin;

    // Store admin with value pendingAdmin
    admin = pendingAdmin;

    // Clear the pending value
    pendingAdmin = payable(address(0));

    emit NewAdmin(oldAdmin, admin);
    emit NewPendingAdmin(oldPendingAdmin, pendingAdmin);

    return NO_ERROR;
  }
```

### [L-03] Token transfer to address(0) should be avoided
The internal function `_beforeTokenTransfer` ignores the use of address(0).
As how it is now the two if statements won't be triggered on address(0) and the function will finish successfuly.

```solidity
56:  function _beforeTokenTransfer(
57:    address from,
58:    address to,
59:    uint256 amount
60:  ) internal override {
61:    super._beforeTokenTransfer(from, to, amount);
62:
63:    require(
64:      _getKYCStatus(_msgSender()),
65:      "CashKYCSenderReceiver: must be KYC'd to initiate transfer"
66:    );
67:
68:    if (from != address(0)) {
69:      // Only check KYC if not minting
70:      require(
71:        _getKYCStatus(from),
72:        "CashKYCSenderReceiver: `from` address must be KYC'd to send tokens"
73:      );
74:    }
75:
76:    if (to != address(0)) {
77:      // Only check KYC if not burning
78:      require(
79:        _getKYCStatus(to),
80:        "CashKYCSenderReceiver: `to` address must be KYC'd to receive tokens"
81:      );
82:    }
83:  }
84: }
```

Consider applying a check, which will revert if the "from" or "to" address are set as the zero address and remove 
the two if statements:

```solidity
56:  function _beforeTokenTransfer(
57:    address from,
58:    address to,
59:    uint256 amount
60:  ) internal override {
61:    super._beforeTokenTransfer(from, to, amount);
62:    
63:    require(from != address(0), "")
64:    require(to != address(0), "")
65
66:    require(
67:      _getKYCStatus(_msgSender()),
68:      "CashKYCSenderReceiver: must be KYC'd to initiate transfer"
69:    );
70:
71:      // Only check KYC if not minting
72:      require(
73:        _getKYCStatus(from),
74:        "CashKYCSenderReceiver: `from` address must be KYC'd to send tokens"
75:      );
76:
77:      // Only check KYC if not burning
78:      require(
79:        _getKYCStatus(to),
80:        "CashKYCSenderReceiver: `to` address must be KYC'd to receive tokens"
81:      );
83:  }
84: }
```

### [L-04] Wrong if statement in the function `setMintFee`
The function `setMintFee` in CashManager.sol is used by the admin to change the mint fee.
By the dev comment above the function, the maximum fee that can be set is 10_000 bps, or 100%.
As the value of the BPS_DENOMINATOR is set as 10_000 and can't be changed.
The if statement in the function is wrong as it doesn't allow to be set as the maximum fee 10_000 bps.
`if (_mintFee >= BPS_DENOMINATOR)` should be changed to `if (_mintFee > BPS_DENOMINATOR)`.


```solidity
408:  // * @dev The maximum fee that can be set is 10_000 bps, or 100%
```

```solidity
contracts/cash/CashManager.sol

410:  function setMintFee(
411:    uint256 _mintFee
412:  ) external override onlyRole(MANAGER_ADMIN) {
413:    if (_mintFee >= BPS_DENOMINATOR) {
414:      revert MintFeeTooLarge();
415    }
416:    uint256 oldMintFee = mintFee;
417:   mintFee = _mintFee;
418:    emit MintFeeSet(oldMintFee, _mintFee);
419:  }
```

### [L-05] Giving KYC status to address(0) should be forbidden
The function `addKYCAddressViaSignature` in KYCRegistry.sol is restricted to the KYC requirement group,
which is allowed to give KYC status to user's addresses. Considering the KYC status is checked all over the protocol and 
if KYCed the zero address can be used. A check should be made in the function `addKYCAddressViaSignature` to make sure
the KYC status isn't given to the zero address.

Add this check to the function `addKYCAddressViaSignature`:
```solidity
require(user != address(0), "KYC status for address(0) not allowed")
```

```solidity
79:  function addKYCAddressViaSignature(
80:    uint256 kycRequirementGroup,
81:    address user,
82:    uint256 deadline,
83:    uint8 v,
84:    bytes32 r,
85:    bytes32 s
86:  ) external {
```

### [L-06] Redeem limit shouldn't be set below the `currentMintAmount`
The function `setRedeemLimit` is used by the admin to update the amount of token that can be redeemed during one epoch.
A check should be made to ensure the new redeem limit isn't set below the `currentMintAmount`.
As this problem will lead to the users not being able to request redeem their minted amount of cash on the current epoch.

```solidity
contracts/cash/CashManager.sol

// Before
609:  function setRedeemLimit(
610:    uint256 _redeemLimit
612:  ) external onlyRole(MANAGER_ADMIN) {
613:    uint256 oldRedeemLimit = redeemLimit;
614:    redeemLimit = _redeemLimit;
615:    emit RedeemLimitSet(oldRedeemLimit, _redeemLimit);
616:  }

// After 
609:  function setRedeemLimit(
610:    uint256 _redeemLimit
612:  ) external onlyRole(MANAGER_ADMIN) {
613:    require(_redeemLimit > currentMintAmount, "RedeemLimit below the currentMintAmount")
614:    uint256 oldRedeemLimit = redeemLimit;
615:    redeemLimit = _redeemLimit;
616:    emit RedeemLimitSet(oldRedeemLimit, _redeemLimit);
617:  }
```

### [L-07] `completeRedemptions` is vulnerable to admin mistakes
The function `completeRedemptions` allows an admin account to distribute collateral to users.
The problem is that the collateral calculation is based on the inputed spec `collateralAmountToDist`.
Duo to that a single admin mistake is not allowed, as it can lead to users receiving less funds back or in the worst case
receiving nothing and the collateral being stuck in the assetSender contract.

```solidity
contracts/cash/CashManager.sol

707:  function completeRedemptions(
708:    address[] calldata redeemers,
709:    address[] calldata refundees,
710:    uint256 collateralAmountToDist,
711:    uint256 epochToService,
712:    uint256 fees
713;  ) external override updateEpoch onlyRole(MANAGER_ADMIN) {
714:    _checkAddressesKYC(redeemers);
715:    _checkAddressesKYC(refundees);
716:    if (epochToService >= currentEpoch) {
717:      revert MustServicePastEpoch();
718:    }
719:    // Calculate the total quantity of shares tokens burned w/n an epoch
720:    uint256 refundedAmt = _processRefund(refundees, epochToService);
721:    uint256 quantityBurned = redemptionInfoPerEpoch[epochToService]
722:      .totalBurned - refundedAmt;
723:    uint256 amountToDist = collateralAmountToDist - fees;
724:    _processRedemption(redeemers, amountToDist, quantityBurned, epochToService);
725:    collateral.safeTransferFrom(assetSender, feeRecipient, fees);
726:    emit RedemptionFeesCollected(feeRecipient, fees, epochToService);
727:  }
```
```solidity
743:  function _processRedemption
755:  uint256 collateralAmountDue = (amountToDist * cashAmountReturned) /
756:        quantityBurned;
```

### [N-01] Mandatory checks for extra safety
In the folowing functions below, there are some checks that can be made in order to achieve more safe and efficient code.

1. In the function `setPrice` a require statement can be made to check if the new price is non-zero.

```solidity
contracts/lending/OndoPriceOracle.sol

80:  function setPrice(address fToken, uint256 price) external override onlyOwner {
81:    uint256 oldPrice = fTokenToUnderlyingPrice[fToken];
82:    fTokenToUnderlyingPrice[fToken] = price;
83:    emit UnderlyingPriceSet(fToken, oldPrice, price);
84:  }
```

2. In the function `setFTokenToCToken` a require statement can be made to check if the inputed addresses aren't the same.

```solidity
contracts/lending/OndoPriceOracle.sol

92:  function setFTokenToCToken(
93:    address fToken,
94:    address cToken
95:  ) external override onlyOwner {
96:    address oldCToken = fTokenToCToken[fToken];
97:    _setFTokenToCToken(fToken, cToken);
98:    emit FTokenToCTokenSet(fToken, oldCToken, cToken);
99:  }
```

3.  In the function `setOracle` a check can be made to ensure the newOracle isn't set as address(0).

```solidity
contracts/lending/OndoPriceOracle.sol

106:  function setOracle(address newOracle) external override onlyOwner {
107:    address oldOracle = address(cTokenOracle);
108:    cTokenOracle = CTokenOracle(newOracle);
109:    emit CTokenOracleSet(oldOracle, newOracle);
110:  }
```
Other instances in:
```solidity
contracts/lending/OndoPriceOracleV2.sol
```

### [N-02] Constructor lacks address(0) check
Zero-address check should be used in the constructors, to avoid the risk of setting smth as address(0) at deploying time.

Instances:
```solidity
contracts/cash/factory/CashFactory.sol

53: constructor(address _guardian) {

contracts/cash/factory/CashKYCSenderFactory.sol

53: constructor(address _guardian) {

contracts/cash/factory/CashKYCSenderReceiverFactory.sol

53: constructor(address _guardian) {

contracts/lending/JumpRateModelV2.sol

59: constructor() - owner

contracts/cash/kyc/KYCRegistry.sol

51: constructor() - admin
```

### [N-03] Unrealistic if statement
The if statement in the internal function `mintFresh` checks if accrualBlockNumber doesn't equal the current block number

```solidity
contracts/lending/tokens/cCash/CTokenCash.sol

502:    if (accrualBlockNumber != getBlockNumber()) {
503:      revert MintFreshnessCheck();
504:    }
```
This if statement is unrealistic considering that the function `accrueInterest` is called before that
and accrualBlockNumber is updated to the current block number. Duo to this, the if statement
in the function `mintFresh` is useless and won't be triggered.

```solidity
contracts/lending/tokens/cCash/CTokenCash.sol

479:  function mintInternal(uint mintAmount) internal nonReentrant {
480:    accrueInterest();
481:    // mintFresh emits the actual Mint event if successful and logs on errors, so we don't need to
482:    mintFresh(msg.sender, mintAmount);
483:  }
        
        // In the function accrueInterest, the variable accrualBlockNumber is updated to the currentBlockNumber.
394:  function accrueInterest() public virtual override returns (uint) {
458:  accrualBlockNumber = currentBlockNumber;
```

Other instances:

```solidity
contracts/lending/tokens/cCash/CTokenCash.sol

576:  function redeemFresh()
679:  function borrowFresh()
767:  function repayBorrowFresh()
870:  function liquidateBorrowFresh()
1151:  function _setReserveFactorFresh()
1198:  function _addReservesFresh()
1253:  function _reduceReservesFresh()
1314:  function _setInterestRateModelFresh()
```

### [N-04] Missing check to ensure epoch duration isn't set as zero
The function `setEpochDuration` is used to change the epoch duration.
Considering the fact that setting an epoch's duration as 0 seconds might lead to undesired behaviour behavior.
Adding a simple check to prevent this from happening is recommended.

```solidity
contracts/cash/CashManager.sol

546:  function setEpochDuration(
547:    uint256 _epochDuration
548:  ) external onlyRole(MANAGER_ADMIN) {
549:    uint256 oldEpochDuration = epochDuration;
550:    epochDuration = _epochDuration;
551:    emit EpochDurationSet(oldEpochDuration, _epochDuration);
552:  }
```

### [N-05] Upgradeable contract is missing a __gap[50] storage variable
Reference: [Storage_gaps](https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps)

You may notice that every contract includes a state variable named __gap. This is empty reserved space in storage that is put in place in Upgradeable contracts. It allows us to freely add new state variables in the future without compromising the storage compatibility with existing deployments.

Instances:
```solidity
contracts/cash/token/CashKYCSender.sol

22:  contract CashKYCSender is
23:  ERC20PresetMinterPauserUpgradeable,
24:  KYCRegistryClientInitializable
```
```solidity
contracts/cash/token/Cash.sol

21: contract Cash is ERC20PresetMinterPauserUpgradeable 
```

### [N-06] Create your own import names instead of using the regular ones
For better readability, you should name the imports instead of using the regular ones.

Example:
```solidity
import {IKYCRegistry} from "contracts/cash/interfaces/IKYCRegistry.sol";
```

Instances - All of the contracts.

### [N-07] Initialize function does not use the initializer modifier
Without the modifier, the function may be called multiple times, overwriting prior initializations

```solidity
contracts/lending/tokens/cCash/CCash.sol

30: function initialize

contracts/lending/tokens/cToken/CErc20.sol

30: function initialize
```

### [N-08] Use delete to clear variables instead of zero assignment
You can use the delete keyword instead of setting the variable as zero.

```solidity
contracts/cash/CashManager.sol

259: mintRequestsPerEpoch[epochToClaim][user] = 0
790: redemptionInfoPerEpoch[epochToService].addressToBurnAmt[refundee] = 0
```

### [N-09] Confusing function comment on `setMinimumDepositAmount`
There is a little confusion between the dev comment and the if statement in the function.
As per dev comment the inputed `_minimumDepositAmount` should be larger than the `BPS_DENOMINATOR`.
But the if statement actually allows for the ``_minimumDepositAmount` to equal the `BPS_DENOMINATOR`.

```solidity
contracts/cash/CashManager.sol

427: // @dev Must be larger than BPS_DENOMINATOR due to keep our `_getMintFees`

433:  function setMinimumDepositAmount(
434:    uint256 _minimumDepositAmount
435:  ) external override onlyRole(MANAGER_ADMIN) {
436:    if (_minimumDepositAmount < BPS_DENOMINATOR) {
437:      revert MinimumDepositAmountTooSmall();
438:    }
439:    uint256 oldMinimumDepositAmount = minimumDepositAmount;
440:    minimumDepositAmount = _minimumDepositAmount;
441:    emit MinimumDepositAmountSet(
442:      oldMinimumDepositAmount,
443:      _minimumDepositAmount
444:    );
445:  }
```

### [N-10] Unnecessary if statement in _checkAndUpdateRedeemLimit
In the private function `_checkAndUpdateRedeemLimit` an if statement occurs, which is triggered if the inputed amount is zero.
This if statement is unnecessary and useless as there is a check already made in the core function `requestRedemption` to ensure the requested redemption isn't below the `minimumRedeemAmount`

```solidity
contracts/cash/CashManager.sol

641:  function _checkAndUpdateRedeemLimit(uint256 amount) private {
642:    if (amount == 0) {
643:      revert RedeemAmountCannotBeZero();
644:    }
645:    if (amount > redeemLimit - currentRedeemAmount) {
646:      revert RedeemExceedsRateLimit();
647:    }
648:
649:    currentRedeemAmount += amount;
650:  }

662:  function requestRedemption(
663:    uint256 amountCashToRedeem
664:  )
665:    external
666:    override
667:    updateEpoch
668:    nonReentrant
669:    whenNotPaused
670:    checkKYC(msg.sender)
671:  {
672:    if (amountCashToRedeem < minimumRedeemAmount) {
673:      revert WithdrawRequestAmountTooSmall();
674:    }
675:
676:    _checkAndUpdateRedeemLimit(amountCashToRedeem);
```

### [N-11] Pause/Unpause functions should emit event to notify users
The two function are used to pause and unpause the contract.
Consider emitting an even to notify the users, when this is happening.

```solidity
contracts/cash/CashManager.sol

526:  function pause() external onlyRole(PAUSER_ADMIN) {
527:    _pause();
528:   }

533:  function unpause() external onlyRole(MANAGER_ADMIN) {
534:    _unpause();
535:  }
```

### [N-12] Two KYC checks made on the same redeemers 
The function `completeRedemptions` allows an admin account distribute collateral to users.
A check is made to ensure all of the redeemers are KYCed, but this check is unnecessary.
As in order to request redemption with the function `requestRedemption`, the function already check if the user calling the function is KYCed.

```solidity
contracts/cash/CashManager.sol

662:  function requestRedemption(
663:    uint256 amountCashToRedeem
664:  )
665:    external
666:    override
667:    updateEpoch
668:    nonReentrant
669:    whenNotPaused
670:    checkKYC(msg.sender)

707:  function completeRedemptions(
708:    address[] calldata redeemers,
709:    address[] calldata refundees,
710:    uint256 collateralAmountToDist,
711:    uint256 epochToService,
712:    uint256 fees
713:  ) external override updateEpoch onlyRole(MANAGER_ADMIN) {
714:    _checkAddressesKYC(redeemers);
```

### [N-13] Unused constructor
The constructor does nothing.

```solidity
contracts/lending/tokens/cCash/CCashDelegate.sol

15: constructor() {}

contracts/lending/tokens/cToken/CTokenDelegate.sol

15: constructor() {}
```

### [N-14] NatSpec is incomplete in the pausing functions
In the both pause and unpause functions a comment is made, that the purpose of this functions is to pause or unpause the
minting functionality. This NatSpec isn't full as it not only applies on the minting, but on the redeeming as well.

```solidity
contracts/cash/CashManager.sol

522:  /**
523:   * @notice Will pause minting functionality of this contract
524:   *
525:   */
526:  function pause() external onlyRole(PAUSER_ADMIN) {
527:    _pause();
528:  }

530:  /**
531:   * @notice Will unpause minting functionality of this contract
532:   */
533:  function unpause() external onlyRole(MANAGER_ADMIN) {
534:    _unpause();
535:  }

662:  function requestRedemption(
663:    uint256 amountCashToRedeem
664:  )
665:    external
666:    override
667:    updateEpoch
668:    nonReentrant
669:    whenNotPaused
670:    checkKYC(msg.sender)
671:  {
```

### [R-01] Shorthand way to write if / else statement
The normal if / else statement can be refactored in a shorthand way to write it:
1. Increases readability
5. Shortens the overall SLOC.

```solidity
contracts/lending/OndoPriceOracle.sol 

61: function getUnderlyingPrice(
62:    address fToken
63:  ) external view override returns (uint256) {
64:    if (fTokenToUnderlyingPrice[fToken] != 0) {
65:      return fTokenToUnderlyingPrice[fToken];
66:    } else {
67:      // Price is not manually set, attempt to retrieve price from Compound's
68:      // oracle
69:      address cTokenAddress = fTokenToCToken[fToken];
70:      return cTokenOracle.getUnderlyingPrice(cTokenAddress);
71:    }
72:  } 
```
The above instance can be refactored in:

```solidity
61: function getUnderlyingPrice(
62:    address fToken
63:  ) external view override returns (uint256) {
64:     address cTokenAddress = fTokenToCToken[fToken];
65:    return fTokenToUnderlyingPrice[fToken] != 0 ? fTokenToUnderlyingPrice[fToken] : cTokenOracle.getUnderlyingPrice(cTokenAddress)
66:  }
```
Other instances:
```solidity
contracts/lending/tokens/cCash/CTokenCash.sol - function exchangeRateStoredInternal()
contracts/lending/tokens/cToken/CTokenModified.sol - function exchangeRateStoredInternal()
```

### [R-02] Modifier should be used instead of require on admin functions
If functions are only allowed to be called by the admin, modifier should be used instead of checking 
with require statement, if admin is the msg.sender calling the function.

```solidity
contracts/lending/tokens/cCash/CCashDelegate.sol

21:  function _becomeImplementation(bytes memory data) public virtual override {
22:    // Shh -- currently unused
23:    data;
24:
25:    // Shh -- we don't ever want this hook to be marked pure
26:    if (false) {
27:      implementation = address(0);
28:    }
29:
30:    require(
31:      msg.sender == admin,
32:      "only the admin may call _becomeImplementation"
33:    );
34:  }
```
Modifier should be created only accessible by the admin and the instance above can be refactored in:

```solidity
21:  function _becomeImplementation(bytes memory data) public virtual override onlyAdmin {
22:    // Shh -- currently unused
23:    data;
24:
25:    // Shh -- we don't ever want this hook to be marked pure
26:    if (false) {
27:      implementation = address(0);
28:    }
29:  }
```
Other instances:
```solidity
contracts/lending/tokens/cCash/CCashDelegate.sol - function _resignImplementation()
contracts/lending/tokens/cToken/CTokenDelegate.sol - function _becomeImplementation()
contracts/lending/tokens/cToken/CTokenDelegate.sol - function _resignImplementation()
contracts/lending/tokens/cCash/CCash.sol - function sweepToken()
contracts/lending/tokens/cCash/CCash.sol - function _delegateCompLikeTo()
contracts/lending/tokens/cToken/CErc20.sol - function sweepToken()
contracts/lending/tokens/cToken/CErc20.sol - function _delegateCompLikeTo()
contracts/lending/tokens/cCash/CTokenCash.sol - function initialize()
```

### [R-03] Unused variables should be deleted
In this case bytes "data" is implemented in the function, but it isn't used.
If not used, the variable shouldn't be used in the first place.

```solidity
contracts/lending/tokens/cCash/CCashDelegate.sol

21:  function _becomeImplementation(bytes memory data) public virtual override {
22:    // Shh -- currently unused
23:    data;
24:
25:    // Shh -- we don't ever want this hook to be marked pure
26:    if (false) {
27:      implementation = address(0);
28:    }
29:
30:    require(
31:      msg.sender == admin,
32:      "only the admin may call _becomeImplementation"
33:    );
34:  }
```
Other instance:
```solidity
contracts/lending/tokens/cToken/CTokenDelegate.sol - function _becomeImplementation()
```

### [R-04] Use require instead of assert
The Solidity assert() function is meant to assert invariants. 
Properly functioning code should never reach a failing assert statement.

```solidity
contracts/cash/factory/CashFactory.sol

97: assert(cashProxyAdmin.owner() == guardian);

contracts/cash/factory/CashKYCSenderFactory.sol

106: assert(cashKYCSenderProxyAdmin.owner() == guardian);
```

Recommended: Consider whether the condition checked in the assert() is actually an invariant. 
If not, replace the assert() statement with a require() statement.

### [R-05] Immutable should be used on variables that can't be changed
State variables, which can't be changed after deploying time should be set as immutable.

```solidity
contracts/lending/JumpRateModelV2.sol

24: address public owner;
```

### [R-06] Unnecessary return statement applied
Adding a return statement when the function defines a named return variable, is wrong.

```solidity
contracts/cash/CashManager.sol

781:  function _processRefund(
782:    address[] calldata refundees,
783:    uint256 epochToService
784:  ) private returns (uint256 totalCashAmountRefunded) {
785:    uint256 size = refundees.length;
786:    for (uint256 i = 0; i < size; ++i) {
787:      address refundee = refundees[i];
788:      uint256 cashAmountBurned = redemptionInfoPerEpoch[epochToService]
789:        .addressToBurnAmt[refundee];
790:      redemptionInfoPerEpoch[epochToService].addressToBurnAmt[refundee] = 0;
791:      cash.mint(refundee, cashAmountBurned);
792:      totalCashAmountRefunded += cashAmountBurned;
793:      emit RefundIssued(refundee, cashAmountBurned, epochToService);
794:    }
795:    return totalCashAmountRefunded;
796  }
```

### [R-07] Numeric values having to do with time should use time units for readability
Suffixes like seconds, minutes, hours, days and weeks after literal numbers can be used to specify units of time where seconds are the base unit and units are considered naively in the following way:

`1 == 1 seconds`
`1 minutes == 60 seconds`
`1 hours == 60 minutes`
`1 days == 24 hours`
`1 weeks == 7 days`

```solidity
contracts/lending/OndoPriceOracleV2.sol

77: uint256 public maxChainlinkOracleTimeDelay = 90000; // 25 hours
```

### [O-01] Floating pragma
Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

Instances:
```solidity
contracts/lending/tokens/cCash/CCashDelegate.sol
contracts/lending/tokens/cToken/CTokenDelegate.sol
contracts/lending/JumpRateModelV2.sol
contracts/lending/tokens/cCash/CCash.sol
contracts/lending/tokens/cToken/CErc20.sol
contracts/lending/tokens/cCash/CTokenInterfacesModifiedCash.sol
contracts/lending/tokens/cToken/CTokenInterfacesModified.sol
contracts/lending/tokens/cErc20ModifiedDelegator.sol
contracts/lending/tokens/cCash/CTokenCash.sol
contracts/lending/tokens/cToken/CTokenModified.sol
```

### [O-02] Outdated Compiler Version
Using an outdated compiler version can be problematic especially if there are publicly disclosed bugs and issues that affect the current compiler version. It is recommended to use a recent version of the Solidity compiler.

Instances:
```solidity
contracts/lending/OndoPriceOracle.sol
contracts/lending/JumpRateModelV2.sol
contracts/lending/tokens/cErc20ModifiedDelegator.sol
```

### [O-03] Function Naming suggestions
Proper use of _ as a function name prefix and a common pattern is to prefix internal and private function names with _. 
This pattern is correctly applied in the Party contracts, however there are some inconsistencies in the libraries.

Instances:
```solidity
contracts/lending/tokens/cToken/CTokenModified.sol
contracts/lending/tokens/cCash/CTokenCash.sol
```

### [O-04] Proper use of get as a function name prefix
Clear function names can increase readability. Follow a standard convertion function names such as using get for getter (view/pure) functions.

Instances:
```solidity
contracts/lending/tokens/cToken/CTokenModified.sol
contracts/lending/tokens/cCash/CTokenCash.sol
```

### [O-05] Events is missing indexed fields
Index event fields make the field more quickly accessible to off-chain.
Each event should use three indexed fields if there are three or more fields.

Instances in:
```solidity
contracts/lending/tokens/cErc20ModifiedDelegator.sol
```
