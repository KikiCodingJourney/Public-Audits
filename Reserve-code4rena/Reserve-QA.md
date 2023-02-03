### Issues Template
| Letter | Name | Description |
|:--:|:-------:|:-------:|
| L  | Low risk | Potential risk |
| NC |  Non-critical | Non risky findings |
| R  | Refactor | Changing the code |
| O | Ordinary | Often found issues |

| Total Found Issues | 27 |
|:--:|:--:|

### Low Risk Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [L-01] | Melt function should be only callable by the Furnance contract | 1 |
| [L-02] | Stake function shouldn't be accessible, when the status is paused or frozen | 1 |
| [L-03] | Draft openzeppelin dependencies | 1 |


| Total Low Risk Issues | 3 |
|:--:|:--:|

### Non-Critical Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [N-01] | Create your own import names instead of using the regular ones | 17 |
| [N-02] | Max value can't be applied in the setters | 9 |
| [N-03] | Using while for unbounded loops isn't recommended | 3 |
| [N-04] | Inconsistent visibility on the bool "disabled" | 2 |
| [N-05] | Modifier exists, but not used when needed | 6 |
| [N-06] | Unused constructor | 2 |
| [N-07] | Unnecessary check in both the _mint and _burn function | 2 |


| Total Non-Critical Issues | 7 |
|:--:|:--:|

### Refactor Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [R-01] | Numeric values having to do with time should use time units for readability | 5 |
| [R-02] | Use require instead of assert | 9 |
| [R-03] | Unnecessary overflow check can be rafactored in a better way | 1 |
| [R-04] | If statement should check first, if the status is disabled | 1 |
| [R-05] | Some number values can be refactored with _ | 2 |
| [R-06] | Revert should be used on some functions instead of return | 9 |
| [R-07] | Modifier can be applied on the function instead of creating require statement | 2 |
| [R-08] | Shorthand way to write if / else statement | 1 |
| [R-09] | Function should be deleted, if a modifier already exists doing its job | 1 |
| [R-10] | The right value should be used instead of downcasting from uint256 to uint192 | 2 |

| Total Refactor Issues | 10 |
|:--:|:--:|

### Ordinary Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [O-01] | Code contains empty blocks | 3 |
| [O-02] | Use a more recent pragma version | 17 |
| [O-03] | Function Naming suggestions | 6 |
| [O-04] | Events is missing indexed fields | 2 | 
| [O-05] | Proper use of get as a function name prefix | 12 |
| [O-06] | Commented out code | 3 |
| [O-07] | Value should be unchecked | 1 |

| Total Ordinary Issues | 7 |
|:--:|:--:|

### [L-01] Melt function should be only callable by the Furnance contract
The function `melt` in RToken.sol is supposed to be called only by Furnace.sol, but as how it is right now the function can be called by anyone. This is problematic considering that this function burns tokens, if a user calls it by mistake. His tokens will be lost and he won't be able to get them back.

```solidity
contracts/p1/RToken.sol

569:  function melt(uint256 amtRToken) external notPausedOrFrozen {
570:        _burn(_msgSender(), amtRToken);
571:        emit Melted(amtRToken);
572:        requireValidBUExchangeRate();
573:    }
```

Consider applying a require statement in the function `melt` that the msg.sender is the furnance contract:

```solidity
569:  function melt(uint256 amtRToken) external notPausedOrFrozen {
569:        require(_msgSender() == address(furnance), "not furnance contract");
570:        _burn(_msgSender(), amtRToken);
571:        emit Melted(amtRToken);
572:        requireValidBUExchangeRate();
573:    }
```

### [L-02] Stake function shouldn't be accessible, when the status is paused or frozen
The function `stake` in StRSR.sol is used by users to stake a RSR amount on the corresponding RToken to earn yield and over-collateralize the system. If the contract is in paused or frozen status, some of the main functions `payoutRewards`, `unstake`, 
`withdraw` and `seizeRSR` can't be used. The `stake` function will keep operating but will skip to payoutRewards, this is problematic considering if the status is paused or frozen and a user stakes without knowing that. He won't be able to unstake or call any of the core functions, the only option he has is to wait for the status to be unpaused or unfrozen.
Consider if a contract is in paused or frozen status to turn off all of the core functions including staking aswell.

```solidity
contracts/p1/StRSR.sol

212:  function stake(uint256 rsrAmount) external {
213:        require(rsrAmount > 0, "Cannot stake zero");
214:
215:        if (!main.pausedOrFrozen()) _payoutRewards();
216:
217:        // Compute stake amount
218:        // This is not an overflow risk according to our expected ranges:
219:        //   rsrAmount <= 1e29, totalStaked <= 1e38, 1e29 * 1e38 < 2^256.
220:        // stakeAmount: how many stRSR the user shall receive.
221:        // pick stakeAmount as big as we can such that (newTotalStakes <= newStakeRSR * stakeRate)
222:        uint256 newStakeRSR = stakeRSR + rsrAmount;
223:        // newTotalStakes: {qStRSR} = D18{qStRSR/qRSR} * {qRSR} / D18
224:        uint256 newTotalStakes = (stakeRate * newStakeRSR) / FIX_ONE;
225:        uint256 stakeAmount = newTotalStakes - totalStakes;
226:
227:        // Update staked
228:        address account = _msgSender();
229:        stakeRSR += rsrAmount;
230:        _mint(account, stakeAmount);
231:
232:        // Transfer RSR from account to this contract
233:        emit Staked(era, account, rsrAmount, stakeAmount);
234:
235:        // == Interactions ==
236:        IERC20Upgradeable(address(rsr)).safeTransferFrom(account, address(this), rsrAmount);
237:    }
```


### [L-03] Draft openzeppelin dependencies
The contract StRSR.sol uses a draft openzeppelin contract called `draft-EIP712Upgradeable.sol`.
Since it is still draft, it might not be ready for mainnet use. Consider using it at your own risk, openzeppelin contracts are draft
if they are still under development and didn't receive enough security measures to make it safe for use.

```solidity
contracts/p1/StRSR.sol

7: import "@openzeppelin/contracts-upgradeable/utils/cryptography/draft-EIP712Upgradeable.sol";
```

### [N-01] Create your own import names instead of using the regular ones
For better readability, you should name the imports instead of using the regular ones.

Example:
```solidity
6: {IStRSRVotes} import "../interfaces/IStRSRVotes.sol";
```

Instances - All of the contracts.

### [N-02] Max value can't be applied in the setters
The function `setTradingDelay` is used by the governance to change the tradingDelay.
However in the require statement applying the maximum delay is not allowed.

Consider changing the require statement to: `require(val < MAX_TRADING_DELAY, "invalid tradingDelay")`

Other instances:

```solidity
contracts/p1/BackingManager.sol

263: function setBackingBuffer

contracts/p1/Broker.sol

133: function setAuctionLength

contracts/p1/Furnace.sol

88: function setPeriod
96: function setRatio

contracts/p1/RToken.sol

589: function setIssuanceRate
602: function setScalingRedemptionRate

contracts/p1/StRSR.sol

812: function setUnstakingDelay
820: function setRewardPeriod
828: function setRewardRatio
```

### [N-03] Using while for unbounded loops isn't recommended
Don't write loops that are unbounded as this can hit the gas limit, causing your transaction to fail.
For the reason above, while and do while loops are rarely used.

```solidity
contracts/p1/BasketHandler.sol

523: while (_targetNames.length() > 0)

contracts/p1/StRSR.sol

449: while (left < right - 1) {

contracts/p1/StRSRVotes.sol

103: while (low < high) {
```

### [N-04] Inconsistent visibility on the bool "disabled"
In some contracts the visibility of the bool `disabled` is set as private, while on others it is set as public.

Instances:
```solidity
contracts/p1/BasketHandler.sol

139: bool private disabled;

contracts/p1/Broker.sol

41: bool public disabled;
```

### [N-05] Modifier exists, but not used when needed
In the RToken contract, a lot of private calls are made to `requireNotPausedOrFrozen()` checking if it's paused or frozen.
While there is already modifier used for this purpose in the contract.

function without the modifier:

```solidity
contracts/p1/RToken.sol

520:  function claimRewards() external {
521:        requireNotPausedOrFrozen();
522:        RewardableLibP1.claimRewards(assetRegistry);
523:    }
```

function with the modifier used:

```solidity
contracts/p1/RToken.sol

378: function vest(address account, uint256 endId) external notPausedOrFrozen {
```

As you can see there is already modifier with this purpose, but it isn't used on all of the functions.
Consider applying it on the other instances as well.

```solidity
contracts/p1/RToken.sol

520: function claimRewards
527: function claimRewardsSingle
534: function sweepRewards
541: function sweepRewardsSingle
556: function mint
579: function setBasketsNeeded
```

### [N-06] Unused constructor
The constructor does nothing.

```solidity
contracts/p1/Main.sol

23: constructor() initializer {}

contracts/p1/mixins/Component.sol

25: constructor() initializer {}
```

### [N-07] Unnecessary check in both the _mint and _burn function
The function `_mint` and burn in StRSR.sol is called only by someone calling the stake and unstake functions.
A check is made in the functions to ensure the account to mint and burn the amounts isn't address(0).
However this isn't possible as both the stake and unstake function input the address of the msg.sender.
And address(0) can't call this functions, so this checks are unnecessary.

```solidity
contracts/p1/StRSR.sol

694:  function _mint(address account, uint256 amount) internal virtual {
695:        require(account != address(0), "ERC20: mint to the zero address");
696:        assert(totalStakes + amount < type(uint224).max);
697:
698:        stakes[era][account] += amount;
699:        totalStakes += amount;
700:
701:        emit Transfer(address(0), account, amount);
702:        _afterTokenTransfer(address(0), account, amount);
703:    }

708:  function _burn(address account, uint256 amount) internal virtual {
709:        // untestable:
710:        //      _burn is only called from unstake(), which uses msg.sender as `account`
711:        require(account != address(0), "ERC20: burn from the zero address");
712:
713:        mapping(address => uint256) storage eraStakes = stakes[era];
714:        uint256 accountBalance = eraStakes[account];
715:        // untestable:
716:        //      _burn is only called from unstake(), which already checks this
717:        require(accountBalance >= amount, "ERC20: burn amount exceeds balance");
718:        unchecked {
719:            eraStakes[account] = accountBalance - amount;
720:        }
721:        totalStakes -= amount;
722:
723:        emit Transfer(account, address(0), amount);
724;        _afterTokenTransfer(account, address(0), amount);
725:    }
```

As you can see in the below instance, everytime the address given to the _mint and _burn functions will be the msg.sender of stake and unstake:

```solidity
contracts/p1/StRSR.sol

212:  function stake(uint256 rsrAmount) external {

228:  address account = _msgSender();
229:  stakeRSR += rsrAmount;
230:  _mint(account, stakeAmount);
```

```solidity
contracts/p1/StRSR.sol

257:  function unstake(uint256 stakeAmount) external notPausedOrFrozen {
258:        address account = _msgSender();

267:        _burn(account, stakeAmount);
```

### [R-01] Numeric values having to do with time should use time units for readability
Suffixes like seconds, minutes, hours, days and weeks after literal numbers can be used to specify units of time where seconds are the base unit and units are considered naively in the following way:

`1 == 1 seconds`
`1 minutes == 60 seconds`
`1 hours == 60 minutes`
`1 days == 24 hours`
`1 weeks == 7 days`

```solidity
contracts/p1/BackingManager.sol

33: uint48 public constant MAX_TRADING_DELAY = 31536000; // {s} 1 year

contracts/p1/Broker.sol

24: uint48 public constant MAX_AUCTION_LENGTH = 604800; // {s} max valid duration - 1 week

contracts/p1/Furnace.sol

16: uint48 public constant MAX_PERIOD = 31536000; // {s} 1 year

contracts/p1/StRSR.sol

37: uint48 public constant MAX_UNSTAKING_DELAY = 31536000; // {s} 1 year
38: uint48 public constant MAX_REWARD_PERIOD = 31536000; // {s} 1 year
``` 

### [R-02] Use require instead of assert 
The Solidity assert() function is meant to assert invariants.
Properly functioning code should never reach a failing assert statement.

Instances:
```solidity
contracts/p1/mixins/RecollateralizationLib.sol

110: assert(doTrade);

contracts/p1/mixins/RewardableLib.sol

78: assert(erc20s[i].balanceOf(address(this)) >= liabilities[erc20s[i]]);
102: assert(erc20.balanceOf(address(this)) >= liabilities[erc20]);

contracts/p1/mixins/TradeLib.sol

44: assert(trade.buyPrice > 0 && trade.buyPrice < FIX_MAX && trade.sellPrice < FIX_MAX);
108: assert
168: assert(errorCode == 0x11 || errorCode == 0x12);
170: assert(keccak256(reason) == UIntOutofBoundsHash);

contracts/p1/BackingManager.sol

249: assert(tradesOpen == 0 && !basketHandler.fullyCollateralized());

contracts/p1/BasketHandler.sol

556: assert(targetIndex < targetsLength);

contracts/p1/StRSR.sol

696: assert(totalStakes + amount < type(uint224).max);
```

Recommended: Consider whether the condition checked in the assert() is actually an invariant.
If not, replace the assert() statement with a require() statement.

### [R-03] Unnecessary overflow check can be rafactored in a better way
In the function `quantityMulPrice` an unchecked code is made, where the local variable `rawDelta` is calculated and after that
an if statement is created, where is check if `rawDelta` overflows. This check won't be needed if we just move the variable
above the unchecked block, so it will revert if this ever happens.

```solidity
contracts/p1/BasketHandler.sol

356: function quantityMulPrice(uint192 qty, uint192 p) internal pure returns (uint192) {

365:  unchecked {
366:            // p and mul *are* Fix values, so have 18 decimals (D18)
367:            uint256 rawDelta = uint256(p) * qty; // {D36} = {D18} * {D18}
368:            // if we overflowed *, then return FIX_MAX
369:            if (rawDelta / p != qty) return FIX_MAX;
```

The instance above can be refactored to:

```solidity
356: function quantityMulPrice(uint192 qty, uint192 p) internal pure returns (uint192) {

// rawDelta is moved above the unchecked block and reverts if overflows
364:  uint256 rawDelta = uint256(p) * qty; // {D36} = {D18} * {D18}
365:  unchecked {          
```

### [R-04] If statement should check first, if the status is disabled
The if statement in the function `basketsHeldBy` check first if basket's length equals zero and then checks if the basket is invalid and disabled. Consider first checking if the staus is disabled and then if the length equals zero.

```solidity
432:  function basketsHeldBy(address account) public view returns (uint192 baskets) {
433:        uint256 length = basket.erc20s.length;
434:        if (length == 0 || disabled) return FIX_ZERO;
```

Refactor the instance above to:

```solidity
432:  function basketsHeldBy(address account) public view returns (uint192 baskets) {
433:        uint256 length = basket.erc20s.length;
434:        if (disabled || length == 0) return FIX_ZERO;
```

### [R-05] Some number values can be refactored with _
Consider using underscores for number values to improve readability.

```solidity
contracts/p1/Distributor.sol

165:  require(share.rsrDist <= 10000, "RSR distribution too high");
166:  require(share.rTokenDist <= 10000, "RToken distribution too high");
```

The above instance can be refactored to:

```solidity
165:  require(share.rsrDist <= 10_000, "RSR distribution too high");
166:  require(share.rTokenDist <= 10_000, "RToken distribution too high");
```

### [R-06] Revert should be used on some functions instead of return
Some instances just return without doing anything, consider applying revert statement instead with a discriptive string
why it does that.

```solidity
contracts/p1/BackingManager.sol

109: if (tradesOpen > 0) return;
114: if (block.timestamp < basketTimestamp + tradingDelay) return;

contracts/p1/BasketHandler.sol

96: if (weight == FIX_ZERO) return;

contracts/p1/Furnace.sol

71: if (uint48(block.timestamp) < uint64(lastPayout) + period) return;

contracts/p1/RToken.sol

660: if (left >= right) return;
739: if (queue.left == endId) return;

contracts/p1/StRSR.sol

310: if (endId == 0 || firstId >= endId) return;
327: if (rsrAmount == 0) return;
497: if (block.timestamp < payoutLastPaid + rewardPeriod) return;
```

### [R-07] Modifier can be applied on the function instead of creating require statement
If functions are only allowed to be called by a certain individual, modifier should be used instead of checking
with require statement, if the individual is the msg.sender calling the function.

```solidity
contracts/p1/RToken.sol

556:  function mint(address recipient, uint256 amtRToken) external {
557:        requireNotPausedOrFrozen();
558:        require(_msgSender() == address(backingManager), "not backing manager");
559:        _mint(recipient, amtRToken);
560:        requireValidBUExchangeRate();
561:    }
```

Modifier should be created only accessible by the individual and the instance above can be refactored in:

```solidity
556:  function mint(address recipient, uint256 amtRToken) external onlyManager {
557:        requireNotPausedOrFrozen();
558:        _mint(recipient, amtRToken);
559:        requireValidBUExchangeRate();
560:    }
```

Other instances:

```solidity
contracts/p1/RToken.sol

579: function setBasketsNeeded
```

### [R-08] Shorthand way to write if / else statement
The normal if / else statement can be refactored in a shorthand way to write it:

1. Increases readability
2. Shortens the overall SLOC.

```solidity
contracts/p1/BasketHandler.sol

296:  function quantity(IERC20 erc20) public view returns (uint192) {
297:        try assetRegistry.toColl(erc20) returns (ICollateral coll) {
298:            if (coll.status() == CollateralStatus.DISABLED) return FIX_ZERO;
299:
300:            uint192 refPerTok = coll.refPerTok(); // {ref/tok}
301:            if (refPerTok > 0) {
302:                // {tok/BU} = {ref/BU} / {ref/tok}
303:                return basket.refAmts[erc20].div(refPerTok, CEIL);
304:            } else {
305:                return FIX_MAX;
306:            }
307:        } catch {
308:            return FIX_ZERO;
309:        }
310:    }
```

The above instance can be refactored to:

```solidity
296:  function quantity(IERC20 erc20) public view returns (uint192) {
297:        try assetRegistry.toColl(erc20) returns (ICollateral coll) {
298:            if (coll.status() == CollateralStatus.DISABLED) return FIX_ZERO;
299:
300:            uint192 refPerTok = coll.refPerTok(); // {ref/tok}
301:            returns refPerTok > 0 ? basket.refAmts[erc20].div(refPerTok, CEIL) : FIX_MAX;
302:        } catch {
303:            return FIX_ZERO;
304:        }
305:    }
```

### [R-09] Function should be deleted, if a modifier already exists doing its job
The function `requireNotPausedOrFrozen` is created only to hold the modifier notPausedOrFrozen.
And for this purpose in some functions requireNotPausedOrFrozen is called in order to check if its paused or frozen.
This function isn't necessary as the modifier notPausedOrFrozen can just be applied on the functions.

```solidity
contracts/p1/RToken.sol

838: function requireNotPausedOrFrozen() private notPausedOrFrozen {}
```
```solidity
520: function claimRewards() external {
521:        requireNotPausedOrFrozen();
522:        RewardableLibP1.claimRewards(assetRegistry);
523:    }
```

Consider removing `requireNotPausedOrFrozen();` and apply the modifier to the function:

```solidity
520: function claimRewards() external notPausedOrFrozen {
521:        RewardableLibP1.claimRewards(assetRegistry);
522:    }
```

### [R-10] The right value should be used instead of downcasting from uint256 to uint192
In the function `requireValidBUExchangeRate` local variables are used to calculate the outcome of low and high.
After that a require statement is made to ensure the BU rate is in range. The problem is that for the local variables uint256 is used and later in the require statement the value are downcasted to uint192.

```solidity
802:  function requireValidBUExchangeRate() private view {
803:        uint256 supply = totalSupply();
804:        if (supply == 0) return;
805:
806:        // Note: These are D18s, even though they are uint256s. This is because
807:        // we cannot assume we stay inside our valid range here, as that is what
808:        // we are checking in the first place
809:        uint256 low = (FIX_ONE_256 * basketsNeeded) / supply; // D18{BU/rTok}
810:        uint256 high = (FIX_ONE_256 * basketsNeeded + (supply - 1)) / supply; // D18{BU/rTok}
811:
812:        // 1e9 = FIX_ONE / 1e9; 1e27 = FIX_ONE * 1e9
813:        require(uint192(low) >= 1e9 && uint192(high) <= 1e27, "BU rate out of range");
814:    }
```

Consider changing the local variables to use uint192 in the first place, instead of downcasting it:

```solidity
809:        uint192 low = (FIX_ONE_256 * basketsNeeded) / supply; // D18{BU/rTok}
810:        uint192 high = (FIX_ONE_256 * basketsNeeded + (supply - 1)) / supply; // D18{BU/rTok}
811:
812:        // 1e9 = FIX_ONE / 1e9; 1e27 = FIX_ONE * 1e9
813:        require(low >= 1e9 && high <= 1e27, "BU rate out of range");
```

### [O-01] Code contains empty blocks
There are some empty blocks, which are unused.
The code should do something or at least have a description why is structured that way.

```solidity
contracts/p1/Main.sol

64: function _authorizeUpgrade(address newImplementation) internal override onlyRole(OWNER) {}
```

Other instances:
```solidity
contracts/p1/RToken.sol

838: function requireNotPausedOrFrozen() private notPausedOrFrozen {}

contracts/p1/mixins/Component.sol

57: function _authorizeUpgrade(address newImplementation) internal view override governance {}
```

### [O-02] Use a more recent pragma version
Old version of solidity is used, consider using the new one `0.8.17`.
You can see what new versions offer regarding bug fixed [here](https://github.com/ethereum/solidity/blob/develop/Changelog.md)

Instances - All of the contracts.

### [O-03] Function Naming suggestions
Proper use of _ as a function name prefix and a common pattern is to prefix internal and private function names with _.
This pattern is correctly applied in the Party contracts, however there are some inconsistencies in the libraries.

Instances:
```solidity
contracts/p1/BackingManager.sol

154: function handoutExcessAssets

contracts/p1/BasketHandler.sol

68: function empty
75: function setFrom
87: function add
356: function quantityMulPrice
650: function requireValidCollArray
```

### [O-04] Events is missing indexed fields
Index event fields make the field more quickly accessible to off-chain.
Each event should use three indexed fields if there are three or more fields.

Instances in:
```solidity
contracts/interfaces/IDistributor.sol

28: event DistributionSet(address dest, uint16 rTokenDist, uint16 rsrDist);

contracts/interfaces/IRToken.sol

83: event BasketsNeededChanged(uint192 oldBasketsNeeded, uint192 newBasketsNeeded);
```

### [O-05] Proper use of get as a function name prefix
Clear function names can increase readability. Follow a standard convertion function names such as using get for getter (view/pure) functions.

Instances:
```solidity
contracts/p1/BasketHandler.sol

279: function status
296: function quantity
316: function price
325: function lotPrice
394: function basketTokens
407: function quote

contracts/p1/Distributor.sol

141: function totals

contracts/p1/RToken.sol

596: function scalingRedemptionRate
609: function redemptionRateFloor
621: function issueItem
628: function redemptionLimit

contracts/p1/StRSR.sol

425: function exchangeRate
```

### [O-06] Commented out code
Commented code in the protocol.

Instances:
[L373-L384](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BasketHandler.sol#L373-L384)
[L457-L510](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BasketHandler.sol#L457-L510)
[L339-L372](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/StRSR.sol#L339-L372)


### [O-07] Value should be unchecked
The function `_mint` is used to mint tokens to user's accounts.
The storage variable `totalStakes` is an uint256 and there is a check before that preventing it from going overflow.
totalStakes should be unchecked as there is no chance to overflow.

```solidity
contracts/p1/StRSR.sol

694:  function _mint(address account, uint256 amount) internal virtual {
695:        require(account != address(0), "ERC20: mint to the zero address");
696:        assert(totalStakes + amount < type(uint224).max);
697:
698:        stakes[era][account] += amount;
699:        totalStakes += amount;
700:
701:        emit Transfer(address(0), account, amount);
702:        _afterTokenTransfer(address(0), account, amount);
703:    }
```

Consider unchecking totalStakes as how it is done in the `_burn` function as well:

```solidity
694:  function _mint(address account, uint256 amount) internal virtual {
695:        require(account != address(0), "ERC20: mint to the zero address");
696:        assert(totalStakes + amount < type(uint224).max);
697:
698:        stakes[era][account] += amount;
699:        unchecked { totalStakes += amount; }
700:
701:        emit Transfer(address(0), account, amount);
702:        _afterTokenTransfer(address(0), account, amount);
703:    }
```
