### Issues Template
| Letter | Name | Description |
|:--:|:-------:|:-------:|
| L  | Low risk | Potential risk |
| NC |  Non-critical | Non risky findings |
| R  | Refactor | Changing the code |
| O | Ordinary | Often found issues |

| Total Found Issues | 21 |
|:--:|:--:|

### Low Risk Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [L-01] | Dangerous use of the `burn` function | 1 |
| [L-02] | `refundETH` can be front-run preventing users from getting their eth back | 1 |
| [L-03] | The function `collect` forgets to accrue position interest before the user collects the interest of his position | 1 |
| [L-04] | Minting tokens to the zero address should be avoided | 1 | 

| Total Low Risk Issues | 4 |
|:--:|:--:|

### Non-Critical Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [N-01] | The function `collect` should revert incase the collateral is zero | 1 |
| [N-02] | The check for liquidity in `mint` is unrealistic, as it can never happen | 1 |
| [N-03] | Unnecessary if statement applied in the function `sweepToken` | 2 |
| [N-04] | Require statements missing strings | 3 |
| [N-05] | Constructor lacks address(0) check | 4 |
| [N-06] | Confusing revert statement | 1 |


| Total Non-Critical Issues | 6 |
|:--:|:--:|

### Refactor Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [R-01] | `invariant` could just return false if the liquidity is zero | 1 |
| [R-02] | Some number values can be refactored with _ | 1 |
| [R-03] | Value should be unchecked | 1 |
| [R-04] | `2**<n> - 1` can be refactored as `type(uint<n>).max` | 1 |


| Total Refactor Issues | 4 |
|:--:|:--:|

### Ordinary Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [O-01] | Floating pragma | 12 |
| [O-02] | Use a more recent pragma version | 15 |
| [O-03] | Events is missing indexed fields | 1 |
| [O-04] | Function Naming suggestions | 13 |
| [O-05] | Proper use of get as a function name prefix | 3 |
| [O-06] | Hardcoded values can't be changed | 1 | 
| [O-07] | PositionMath contains outdated compiler version | 1 |


| Total Ordinary Issues | 7 |
|:--:|:--:|

### [L-01] Dangerous use of the `burn` function
The function `burn` is used by users to burn an option position by minting the required liquidity and unlocking the collateral.
As how the function is designed right now in order to do that, the user needs to send his shares to the contract balance.
This is simply too risky, as anyone can call the function and basically burn the shares deposited by the users, before they even get the chance to call the function first.

Instead of the need to send the shares to the contract balance, the function can be refactored to check the balance of shares the user posses and to burn them in the moment of execution or on top of that to input a uint value of how many shares the user wants to burn. 

```solidity
src/core/Lendgine.sol

105:  function burn(address to, bytes calldata data) external override nonReentrant returns (uint256 collateral) {
106:    _accrueInterest();
107:
108:    uint256 shares = balanceOf[address(this)];
109:    uint256 liquidity = convertShareToLiquidity(shares);
110:    collateral = convertLiquidityToCollateral(liquidity);
111:
112:    if (collateral == 0 || liquidity == 0 || shares == 0) revert InputError();
113:
114:    totalLiquidityBorrowed -= liquidity;
115:    _burn(address(this), shares);
116:    SafeTransferLib.safeTransfer(token1, to, collateral); // optimistically transfer
117:    mint(liquidity, data);
118:
119:    emit Burn(msg.sender, collateral, shares, liquidity, to);
120:  }
```

### [L-02] `refundETH` can be front-run preventing users from getting their eth back
The function `refundETH` in Payment.sol is used by users to get their ether back if they send more than the needed amount when using the function `pay`. The problem here is as how the function is designed, any eth values in the contract can be withdrawn by anyone. This bring the risk, where a malicious users can front-run users and successfuly steal their refunds.

```solidity
src/periphery/Payment.sol

44:  function refundETH() external payable {
45:    if (address(this).balance > 0) SafeTransferLib.safeTransferETH(msg.sender, address(this).balance);
46:  }
```

### [L-03] The function `collect` forgets to accrue position interest before the user collects the interest of his position
In Lendgine the function `collect` is used by the users to collect the interest that has been gathered to their liquidity position.
The problem here occurring is that the function is supposed to accrue both the global interest and the user's liquidity position interest to the current block.timestamp. Note that what l just described is true and it's already applied in a similar function in LiquidityManager - collect(). Consider calling `accruePositionInterest` prior to executing the function `collect`, so the interest can accrued till the current time of the block.timestamp.

```solidity
src/core/Lendgine.sol

194:  function collect(address to, uint256 collateralRequested) external override nonReentrant returns (uint256 collateral) {
195:    Position.Info storage position = positions[msg.sender]; // SLOAD
196:    uint256 tokensOwed = position.tokensOwed;
197:
198:    collateral = collateralRequested > tokensOwed ? tokensOwed : collateralRequested;
199:
200:    if (collateral > 0) {
201:      position.tokensOwed = tokensOwed - collateral; // SSTORE
202:      SafeTransferLib.safeTransfer(token1, to, collateral);
203:    }
204:
205:    emit Collect(msg.sender, to, collateral);
206:  }
```

You can see that this is already applied in a similar function:

```solidity
src/periphery/LiquidityManager.sol

230:  function collect(CollectParams calldata params) external payable returns (uint256 amount) {
231:    ILendgine(params.lendgine).accruePositionInterest();
232:
233:    address recipient = params.recipient == address(0) ? address(this) : params.recipient;
234:
235:    Position memory position = positions[msg.sender][params.lendgine]; // SLOAD
236:
237:    (, uint256 rewardPerPositionPaid,) = ILendgine(params.lendgine).positions(address(this));
238:    position.tokensOwed += FullMath.mulDiv(position.size, rewardPerPositionPaid - position.rewardPerPositionPaid, 1e18);
239:    position.rewardPerPositionPaid = rewardPerPositionPaid;
240:
241:    amount = params.amountRequested > position.tokensOwed ? position.tokensOwed : params.amountRequested;
242:    position.tokensOwed -= amount;
243:
244:    positions[msg.sender][params.lendgine] = position; // SSTORE
245:
246:    uint256 collectAmount = ILendgine(params.lendgine).collect(recipient, amount);
247:    if (collectAmount != amount) revert CollectError(); // extra check for safety
248:
249:    emit Collect(msg.sender, params.lendgine, amount, recipient);
250:  }
```

### [L-04] Minting tokens to the zero address should be avoided
The core function `mint` is used by users to mint an option position by providing token1 as collateral and borrowing the max amount of liquidity. Address(0) check is missing in both this function and the internal function `_mint`, which is triggered to mint the tokens to the `to` address. Consider applying a check in the function to ensure tokens aren't minted to the zero address.

```solidity
src/core/Lendgine.sol

71:  function mint(
72:    address to,
73:    uint256 collateral,
74:    bytes calldata data
75:  )
76:    external
77:    override
78:    nonReentrant
79:    returns (uint256 shares)
80:  {
81:    _accrueInterest();
82:
83:    uint256 liquidity = convertCollateralToLiquidity(collateral);
84:    shares = convertLiquidityToShare(liquidity);
85:
86:    if (collateral == 0 || liquidity == 0 || shares == 0) revert InputError();
87:    if (liquidity > totalLiquidity) revert CompleteUtilizationError();
88:    // next check is for the case when liquidity is borrowed but then was completely accrued
89:    if (totalSupply > 0 && totalLiquidityBorrowed == 0) revert CompleteUtilizationError();
90:
91:    totalLiquidityBorrowed += liquidity;
92:    (uint256 amount0, uint256 amount1) = burn(to, liquidity);
93:    _mint(to, shares);
94:
95:    uint256 balanceBefore = Balance.balance(token1);
96:    IMintCallback(msg.sender).mintCallback(collateral, amount0, amount1, liquidity, data);
97:    uint256 balanceAfter = Balance.balance(token1);
98:
99:    if (balanceAfter < balanceBefore + collateral) revert InsufficientInputError();
100:
101:    emit Mint(msg.sender, collateral, shares, liquidity, to);
102:  }
```

### [N-01] The function `collect` should revert incase the collateral is zero
The function `collect` is used by user to collect their position interest. As how it's designed the function ignores if the outcome of the collateral is zero and still executes the function. This is problematic considering an event is emitted, the function not reverting on zero collateral will lead to spamming zero values events. Apply a revert statement, so the function will revert instead of simply ignoring it.

```solidity
src/core/Lendgine.sol

194:  function collect(address to, uint256 collateralRequested) external override nonReentrant returns (uint256 collateral) {
195:    Position.Info storage position = positions[msg.sender]; // SLOAD
196:    uint256 tokensOwed = position.tokensOwed;
197:
198:    collateral = collateralRequested > tokensOwed ? tokensOwed : collateralRequested;
199:
200:    if (collateral > 0) {
201:      position.tokensOwed = tokensOwed - collateral; // SSTORE
202:      SafeTransferLib.safeTransfer(token1, to, collateral);
203:    }
204:
205:    emit Collect(msg.sender, to, collateral);
206:  }
```

Refactor the above instance to:

```solidity
function collect(address to, uint256 collateralRequested) external override nonReentrant returns (uint256 collateral) {
    Position.Info storage position = positions[msg.sender]; // SLOAD
    uint256 tokensOwed = position.tokensOwed;

    collateral = collateralRequested > tokensOwed ? tokensOwed : collateralRequested;

    if (collateral > 0) revert ZeroCollater();
      position.tokensOwed = tokensOwed - collateral; // SSTORE
      SafeTransferLib.safeTransfer(token1, to, collateral);

    emit Collect(msg.sender, to, collateral);
  }
```


### [N-02] The check for liquidity in `mint` is unrealistic, as it can never happen
In the core function `mint` a check is made to revert in any of the amount collateral, liquidity, shares is zero.
The outcome of the liquidity can never be zero, if the collateral is non zero. Considering the fact that first it check if the collateral is zero and revert, the check for the liquidity is unnecessary and can be removed.

```solidity
src/core/Lendgine.sol

71:  function mint(
72:    address to,
73:    uint256 collateral,
74:    bytes calldata data
75:  )
76:    external
77:    override
78:    nonReentrant
79:    returns (uint256 shares)
80:  {
81:    _accrueInterest();
82:
83:    uint256 liquidity = convertCollateralToLiquidity(collateral);
84:    shares = convertLiquidityToShare(liquidity);
85:
86:    if (collateral == 0 || liquidity == 0 || shares == 0) revert InputError();
```

Consider removing `liquidity == 0` check on L86, as it's unrealistic from occurring:

```solidity
86:    if (collateral == 0 || shares == 0) revert InputError();
```

### [N-03] Unnecessary if statement applied in the function sweepToken
In the function `sweepToken` an if statement is made, which is triggered only if balanceToken is non zero.
This if statement is completely unnecessary, as before that another if statement is made to revert if the balance of the contract is below the minimum amount. As the minimum amount is over zero, there is no need for the second if statement after that.

```solidity
src/periphery/Payment.sol

35:  function sweepToken(address token, uint256 amountMinimum, address recipient) public payable {
36:    uint256 balanceToken = Balance.balance(token);
37:    if (balanceToken < amountMinimum) revert InsufficientOutputError();
38:
39:    if (balanceToken > 0) {
40:      SafeTransferLib.safeTransfer(token, recipient, balanceToken);
41:    }
42:  }
```

In the above instance `if (balanceToken > 0)` is not needed and should be removed:

```solidity
function sweepToken(address token, uint256 amountMinimum, address recipient) public payable {
    uint256 balanceToken = Balance.balance(token);
    if (balanceToken < amountMinimum) revert InsufficientOutputError();

      SafeTransferLib.safeTransfer(token, recipient, balanceToken);
  }
```

Other instance:
```solidity
src/periphery/Payment.sol

25: function unwrapWETH
```

### [N-04] Require statements missing strings
Rquire statements should have descriptive strings to describe why the revert occurs.

Instances:
```solidity
src/periphery/SwapHelper.sol

116: require(amountOutReceived == params.amount);

src/libraries/SafeCast.sol

9: require((z = uint120(y)) == y);
16: require(y < 2 ** 255);
```

### [N-05] Constructor lacks address(0) check
Zero-address check should be used in the constructors, to avoid the risk of setting a storage variable as address(0) at deploying time.

Instances:
```solidity
src/periphery/LiquidityManager.sol

75: constructor(address _factory, address _weth) Payment(_weth) {

src/periphery/LendgineRouter.sol

49: constructor

src/periphery/Payment.sol

17: constructor(address _weth) {

src/periphery/SwapHelper.sol

29: constructor(address _uniswapV2Factory, address _uniswapV3Factory) 
```

### [N-06] Confusing revert statement
The modifier checkDeadline is used on both of the core functions `mint` and `burn`, the main use of the modifier is to check if the block.timestamp crossed the deadline, so the function can revert. A confusing revert name is used in the modifier, users which got the error won't understand the reason why the function reverts.

```solidity
src/periphery/LendgineRouter.sol

65:  modifier checkDeadline(uint256 deadline) {
66:    if (deadline < block.timestamp) revert LivelinessError();
67:    _;
68:  }
```

Change the revert statement name, so it can be more understandable

Example:
```solidity

revert Deadline();
```

### [R-01] `invariant` could just return false if the liquidity is zero 
In the function `invariant` a check is made, so that the function will revert incase liquidity is zero.
If triggered the statement returns `(amount0 == 0 && amount1 == 0)`, so it can revert as inputted amount0 and amount1 can never be zero. Instead of doing all of that a simple false can be applied, so the function can return false and revert.

```solidity
src/core/Pair.sol

53:  function invariant(uint256 amount0, uint256 amount1, uint256 liquidity) public view override returns (bool) {
54:    if (liquidity == 0) return (amount0 == 0 && amount1 == 0);
55:
56:    uint256 scale0 = FullMath.mulDiv(amount0, 1e18, liquidity) * token0Scale;
57:    uint256 scale1 = FullMath.mulDiv(amount1, 1e18, liquidity) * token1Scale;
58:
59:    if (scale1 > 2 * upperBound) revert InvariantError();
60:
61:    uint256 a = scale0 * 1e18;
62:    uint256 b = scale1 * upperBound;
63:    uint256 c = (scale1 * scale1) / 4;
64:    uint256 d = upperBound * upperBound;
65:
66:    return a + b >= c + d;
67:  }
```

The above instance can be refactored to:
```solidity
function invariant(uint256 amount0, uint256 amount1, uint256 liquidity) public view override returns (bool) {
+   if (liquidity == 0) return false;

    uint256 scale0 = FullMath.mulDiv(amount0, 1e18, liquidity) * token0Scale;
    uint256 scale1 = FullMath.mulDiv(amount1, 1e18, liquidity) * token1Scale;

    if (scale1 > 2 * upperBound) revert InvariantError();

    uint256 a = scale0 * 1e18;
    uint256 b = scale1 * upperBound;
    uint256 c = (scale1 * scale1) / 4;
    uint256 d = upperBound * upperBound;

    return a + b >= c + d;
  }
```

### [R-02] Some number values can be refactored with _
Consider using underscores for number values to improve readability.

```solidity
src/periphery/UniswapV2/libraries/UniswapV2Library.sol

64: uint256 denominator = (reserveIn * 1000) + amountInWithFee;
80: uint256 numerator = reserveIn * amountOut * 1000;
```

The instances above can be refactored to:

```solidity
64: uint256 denominator = (reserveIn * 1_000) + amountInWithFee;
80: uint256 numerator = reserveIn * amountOut * 1_000;
```

### [R-03] Value should be unchecked
In the deposit function the storage variable `totalPositionSize` is updated, which represents the total amount of positions issued. Considering the fact the variable is of uint256, an overflow in unrealistic and therefore impossible.

```solidity
src/core/Lendgine.sol

145: totalPositionSize = _totalPositionSize + size;
```

### [R-04] `2**<n> - 1` can be refactored as `type(uint<n>).max`

```solidity
src/libraries/SafeCast.sol

15:  function toInt256(uint256 y) internal pure returns (int256 z) {
16:    require(y < 2 ** 255);
17:    z = int256(y);
18:  }
```

The above instance can be refactored to:

```solidity
function toInt256(uint256 y) internal pure returns (int256 z) {
    require(y < type(uint255).max - 1);
    z = int256(y);
  }
```

### [O-01] Floating pragma
Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

Instances:
```solidity
src/core/libraries/Position.sol
src/libraries/SafeCast.sol
src/libraries/Balance.sol
src/core/Pair.sol
src/periphery/SwapHelper.sol
src/periphery/Payment.sol
src/core/JumpRate.sol
src/core/ImmutableState.sol
src/periphery/LendgineRouter.sol
src/periphery/LiquidityManager.sol
src/core/Lendgine.sol
src/core/Factory.sol
```

### [O-02] Use a more recent pragma version
Old version of solidity is used, consider using the new one `0.8.17`.
You can see what new versions offer regarding bug fixed [here](https://github.com/ethereum/solidity/blob/develop/Changelog.md)

Instances - All of the contracts.

### [O-03] Events is missing indexed fields
Index event fields make the field more quickly accessible to off-chain.
Each event should use three indexed fields if there are three or more fields.

Instances in:
```solidity
src/core/Lendgine.sol 
```

### [O-04] Function Naming suggestions
Proper use of _ as a function name prefix and a common pattern is to prefix internal and private function names with _.
This pattern is correctly applied in the Party contracts, however there are some inconsistencies in the libraries.

Instances:
```solidity
src/periphery/UniswapV2/libraries/UniswapV2Library.sol

10: function sortTokens
17: function pairFor
36: function getReserves
51: function getAmountOut
69: function getAmountIn

src/core/libraries/Position.sol

69: function newTokensOwed
73: function convertLiquidityToPosition
86: function convertPositionToLiquidity

src/periphery/libraries/LendgineAddress.sol

9: function computeAddress

src/libraries/SafeCast.sol

8: function toUint120
15: function toInt256

src/libraries/Balance.sol

12: function balance

src/core/libraries/PositionMath.sol

12: function addDelta
```

### [O-05] Proper use of get as a function name prefix
Clear function names can increase readability. Follow a standard convertion function names such as using get for getter (view/pure) functions.

Instances:
```solidity
src/periphery/UniswapV2/libraries/UniswapV2Library.sol

10: function sortTokens

src/periphery/libraries/LendgineAddress.sol

9: function computeAddress

src/core/Pair.sol

53: function invariant
```

### [O-06] Hardcoded values can't be changed
The storage variables kink, multiplier and jumpMultiplier all use hardcoded values, which can't be changed in the future.
And this values are the base logic for the calculation of the interest rate curve.

Instance:
```solidity
src/core/JumpRate.sol
```

### [O-07] PositionMath contains outdated compiler version
Using an outdated compiler version can be problematic especially if there are publicly disclosed bugs and issues that affect the current compiler version. It is recommended to use a recent version of the Solidity compiler.

Instance:
```solidity
src/core/libraries/PositionMath.sol
```
