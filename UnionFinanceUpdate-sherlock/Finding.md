# Users can lose their staking rewards.

## Summary
By following the steps described in `Vulnerability Detail`, user is able to lose all of his staking rewards.

## Vulnerability Detail
The issue occurs in the following steps described below:
1. Kiki calls the function `unstake` and unstakes all of his funds, as a result the internal function `_updateStakedCoinAge` is called to update his staked coin age till the current block.

```solidity
contracts/user/UserManager.sol

711:  function unstake(uint96 amount) external whenNotPaused nonReentrant {
712:        Staker storage staker = stakers[msg.sender];
713:
714:        // Stakers can only unstaked stake balance that is unlocked. Stake balance
715:        // becomes locked when it is used to underwrite a borrow.
716:        if (staker.stakedAmount - staker.locked < amount) revert InsufficientBalance();
717:
718:        comptroller.withdrawRewards(msg.sender, stakingToken);
719:
720:        uint256 remaining = IAssetManager(assetManager).withdraw(stakingToken, msg.sender, amount);
721:        if (uint96(remaining) > amount) {
722:            revert AssetManagerWithdrawFailed();
723:        }
724:        uint96 actualAmount = amount - uint96(remaining);
725:
726:        _updateStakedCoinAge(msg.sender, staker);
727:        staker.stakedAmount -= actualAmount;
728:        totalStaked -= actualAmount;
729:
730:        emit LogUnstake(msg.sender, actualAmount);
731:    }
```
```solidity
contracts/user/UserManager.sol

1064:  function _updateStakedCoinAge(address stakerAddress, Staker storage staker) private {
1065:        uint64 currentBlock = uint64(block.number);
1066:        uint256 lastWithdrawRewards = getLastWithdrawRewards[stakerAddress];
1067:        uint256 blocksPast = (uint256(currentBlock) - _max(lastWithdrawRewards, uint256(staker.lastUpdated)));
1068:        staker.stakedCoinAge += blocksPast * uint256(staker.stakedAmount);
1069:        staker.lastUpdated = currentBlock;
1070:    }
```
2. After that Kiki calls the function `withdrawRewards` in order to withdraw his staking rewards. 
Everything executes fine, but the contract lacks union tokens and can't transfer the tokens to Kiki, so the else statement is triggered and the amount of tokens is added to his accrued balance, so he can still be able to withdraw them after.

```solidity
contracts/token/Comptroller.sol

224:  function withdrawRewards(address account, address token) external override whenNotPaused returns (uint256) {
225:        IUserManager userManager = _getUserManager(token);
226:
227:        // Lookup account state from UserManager
228:        (UserManagerAccountState memory user, Info memory userInfo, uint256 pastBlocks) = _getUserInfo(
229:            userManager,
230:            account,
231:            token,
232:            0
233:        );
234:
235:        // Lookup global state from UserManager
236:        uint256 globalTotalStaked = userManager.globalTotalStaked();
237:
238:        uint256 amount = _calculateRewardsByBlocks(account, token, pastBlocks, userInfo, globalTotalStaked, user);
239:
240:        // update the global states
241:        gInflationIndex = _getInflationIndexNew(globalTotalStaked, block.number - gLastUpdatedBlock);
242:        gLastUpdatedBlock = block.number;
243:        users[account][token].updatedBlock = block.number;
244:        users[account][token].inflationIndex = gInflationIndex;
245:        if (unionToken.balanceOf(address(this)) >= amount && amount > 0) {
246:            unionToken.safeTransfer(account, amount);
247:            users[account][token].accrued = 0;
248:            emit LogWithdrawRewards(account, amount);
249:
250:            return amount;
251:        } else {
252:            users[account][token].accrued = amount;
253:            emit LogWithdrawRewards(account, 0);
254:
255:            return 0;
256:        }
257:    }
```
3. This is where the issue occurs, next time Kiki calls the function `withdrawRewards`, he is going to lose all of his rewards.

Explanation of how this happens:
- First the internal function _getUserInfo will return the struct `UserManagerAccountState memory user`, which contains zero amount for effectiveStaked, because Kiki unstaked all of his funds and already called the function withdrawRewards once.
This happens because Kiki has `stakedAmount = 0, stakedCoinAge = 0, lockedCoinAge = 0, frozenCoinAge = 0`.

```solidity
(UserManagerAccountState memory user, Info memory userInfo, uint256 pastBlocks) = _getUserInfo(
            userManager,
            account,
            token,
            0
        );
```
- The cache `uint256 amount` will have a zero value because of the if statement applied in the internal function `_calculateRewardsByBlocks`, the if statement will be triggered as Kiki's effectiveStaked == 0, and as a result the function will return zero.
```solidity
uint256 amount = _calculateRewardsByBlocks(account, token, pastBlocks, userInfo, globalTotalStaked, user);
```
```solidity
if (user.effectiveStaked == 0 || totalStaked == 0 || startInflationIndex == 0 || pastBlocks == 0) {
            return 0;
        }
```
- Since the cache `uint256 amount` have a zero value, the if statement in the function `withdrawRewards` will actually be ignored because of `&& amount > 0`. And the else statement will be triggered, which will override Kiki's accrued balance with "amount", which is actually zero. As a result Kiki will lose his rewards.
```solidity
if (unionToken.balanceOf(address(this)) >= amount && amount > 0) {
            unionToken.safeTransfer(account, amount);
            users[account][token].accrued = 0;
            emit LogWithdrawRewards(account, amount);

            return amount;
        } else {
            users[account][token].accrued = amount;
            emit LogWithdrawRewards(account, 0);

            return 0;
        }
```


## Impact
The impact here is that users can lose their staking rewards. 

To understand the scenario which is described in `Vulnerability Detail`, you'll need to know how the codebase works.
Here in the impact section, l will describe in little more details and trace the functions.

The issue occurs in 3 steps like described in `Vulnerability Detail`:
1. User unstakes all of his funds.
2. Then he calls the function `withdrawRewards` in order to withdraw his rewards, everything executes fine but the contract lacks union tokens, so instead of transferring the tokens to the user, they are added to his accrued balance so he can still withdraw them after.
3. The next time the user calls the function `withdrawRewards` in order to withdraw his accrued balance of tokens, he will lose all of his rewards.

Explanation in details:

1. User unstakes all of his funds by calling the function `unstake`.

- His stakedAmount will be reduced to zero in the struct `Staker`.
- His stakedCoinAge will be updated to the current block with the internal function `_updateStakedCoinAge`.

2. Then he calls the function withdrawRewards in order to withdraw his rewards, everything executes fine but the contract lacks union tokens, so instead of transferring the tokens to the user, they are added to his accrued balance so he can still withdraw them after.

- User's stakedCoinAge, lockedCoinAge and frozenCoinAge are reduced to zero in the function `onWithdrawRewards`. 

3. The next time the user calls the function `withdrawRewards` in order to withdraw his accrued balance of tokens, he will lose all of his rewards.

- In order to withdraw his accrued rewards stored in his struct balance `Info`. He calls the function `withdrawRewards` again and this is where the issue occurs, as the user has `stakedAmount = 0, stakedCoinAge = 0, lockedCoinAge = 0, frozenCoinAge = 0 `.
- Duo to that the outcome of the function _getCoinAge, which returns a memory struct of CoinAge to the function `_getEffectiveAmounts` will look like this:
```
CoinAge memory coinAge = CoinAge({
            lastWithdrawRewards: lastWithdrawRewards,
            diff: diff,
            stakedCoinAge: staker.stakedCoinAge + diff * uint256(staker.stakedAmount),
            lockedCoinAge: staker.lockedCoinAge,
            frozenCoinAge: frozenCoinAge[stakerAddress]
        });

// The function will return:
CoinAge memory coinAge = CoinAge({
            lastWithdrawRewards: random number,
            diff: random number,
            stakedCoinAge: 0 + random number * 0,
            lockedCoinAge: 0, 
            frozenCoinAge: 0
        });

```
- As a result the function `_getEffectiveAmounts` will return zero values for effectiveStaked and effectiveLocked to the function `onWithdrawRewards`.
```
return (
            // staker's total effective staked = (staked coinage - frozen coinage) / (# of blocks since last reward claiming)
            coinAge.diff == 0 ? 0 : (coinAge.stakedCoinAge - coinAge.frozenCoinAge) / coinAge.diff,
            // effective locked amount = (locked coinage - frozen coinage) / (# of blocks since last reward claiming)
            coinAge.diff == 0 ? 0 : (coinAge.lockedCoinAge - coinAge.frozenCoinAge) / coinAge.diff,
            memberTotalFrozen
        );

return (
            // staker's total effective staked = (staked coinage - frozen coinage) / (# of blocks since last reward claiming)
            coinAge.diff == 0 ? 0 : (0 - 0) / random number,
             // effective locked amount = (locked coinage - frozen coinage) / (# of blocks since last reward claiming)
            coinAge.diff == 0 ? 0 : (0 - 0) / random number,
            0
        );
```

- After that the function `withdrawRewards` caches the returning value from the internal function `_calculateRewardsByBlocks`.
What happens is that in the function `_calculateRewardsByBlocks` the if statement is triggered because the user's effectiveStaked == 0. As a result the internal function will return 0 and the cache `uint256 amount` will equal zero.

```
uint256 amount = _calculateRewardsByBlocks(account, token, pastBlocks, userInfo, globalTotalStaked, user);
```
```
if (user.effectiveStaked == 0 || totalStaked == 0 || startInflationIndex == 0 || pastBlocks == 0) {
            return 0;
        }
```

- Since the cache `uint256 amount` have a zero value, the if statement in the function `withdrawRewards` will actually be ignored because of `&& amount > 0`. And the else statement will be triggered, which will override Kiki's accrued balance with "amount", which is actually zero. 

```
if (unionToken.balanceOf(address(this)) >= amount && amount > 0) {
            unionToken.safeTransfer(account, amount);
            users[account][token].accrued = 0;
            emit LogWithdrawRewards(account, amount);

            return amount;
        } else {
            users[account][token].accrued = amount;
            emit LogWithdrawRewards(account, 0);

            return 0;
        }
```

Below you can see the functions which are invoked starting from the function `_getUserInfo`:

```solidity
(UserManagerAccountState memory user, Info memory userInfo, uint256 pastBlocks) = _getUserInfo(
            userManager,
            account,
            token,
            0
        );
```
```solidity
function _getUserInfo(
        IUserManager userManager,
        address account,
        address token,
        uint256 futureBlocks
    ) internal returns (UserManagerAccountState memory user, Info memory userInfo, uint256 pastBlocks) {
        userInfo = users[account][token];
        uint256 lastUpdatedBlock = userInfo.updatedBlock;
        if (block.number < lastUpdatedBlock) {
            lastUpdatedBlock = block.number;
        }

        pastBlocks = block.number - lastUpdatedBlock + futureBlocks;

        (user.effectiveStaked, user.effectiveLocked, user.isMember) = userManager.onWithdrawRewards(
            account,
            pastBlocks
        );
    }
```
```solidity
function onWithdrawRewards(address staker, uint256 pastBlocks)
        external
        returns (
            uint256 effectiveStaked,
            uint256 effectiveLocked,
            bool isMember
        )
    {
        if (address(comptroller) != msg.sender) revert AuthFailed();
        uint256 memberTotalFrozen = 0;
        (effectiveStaked, effectiveLocked, memberTotalFrozen) = _getEffectiveAmounts(staker, pastBlocks);
        stakers[staker].stakedCoinAge = 0;
        stakers[staker].lastUpdated = uint64(block.number);
        stakers[staker].lockedCoinAge = 0;
        frozenCoinAge[staker] = 0;
        getLastWithdrawRewards[staker] = block.number;

        uint256 memberFrozenBefore = memberFrozen[staker];
        if (memberFrozenBefore != memberTotalFrozen) {
            memberFrozen[staker] = memberTotalFrozen;
            totalFrozen = totalFrozen - memberFrozenBefore + memberTotalFrozen;
        }

        isMember = stakers[staker].isMember;
    }
```
```solidity
function _getEffectiveAmounts(address stakerAddress, uint256 pastBlocks)
        private
        view
        returns (
            uint256,
            uint256,
            uint256
        )
    {
        uint256 memberTotalFrozen = 0;
        CoinAge memory coinAge = _getCoinAge(stakerAddress);

        uint256 overdueBlocks = uToken.overdueBlocks();
        uint256 voucheesLength = vouchees[stakerAddress].length;
        // Loop through all of the stakers vouchees sum their total
        // locked balance and sum their total currDefaultFrozenCoinAge
        for (uint256 i = 0; i < voucheesLength; i++) {
            // Get the vouchee record and look up the borrowers voucher record
            // to get the locked amount and lastUpdated block number
            Vouchee memory vouchee = vouchees[stakerAddress][i];
            Vouch memory vouch = vouchers[vouchee.borrower][vouchee.voucherIndex];

            uint256 lastRepay = uToken.getLastRepay(vouchee.borrower);
            uint256 repayDiff = block.number - _max(lastRepay, coinAge.lastWithdrawRewards);
            uint256 locked = uint256(vouch.locked);

            if (overdueBlocks < repayDiff && (coinAge.lastWithdrawRewards != 0 || lastRepay != 0)) {
                memberTotalFrozen += locked;
                if (pastBlocks >= repayDiff) {
                    coinAge.frozenCoinAge += (locked * repayDiff);
                } else {
                    coinAge.frozenCoinAge += (locked * pastBlocks);
                }
            }

            uint256 lastUpdateBlock = _max(coinAge.lastWithdrawRewards, uint256(vouch.lastUpdated));
            coinAge.lockedCoinAge += (block.number - lastUpdateBlock) * locked;
        }

        return (
            // staker's total effective staked = (staked coinage - frozen coinage) / (# of blocks since last reward claiming)
            coinAge.diff == 0 ? 0 : (coinAge.stakedCoinAge - coinAge.frozenCoinAge) / coinAge.diff,
            // effective locked amount = (locked coinage - frozen coinage) / (# of blocks since last reward claiming)
            coinAge.diff == 0 ? 0 : (coinAge.lockedCoinAge - coinAge.frozenCoinAge) / coinAge.diff,
            memberTotalFrozen
        );
    }
```
```solidity
function _getCoinAge(address stakerAddress) private view returns (CoinAge memory) {
        Staker memory staker = stakers[stakerAddress];

        uint256 lastWithdrawRewards = getLastWithdrawRewards[stakerAddress];
        uint256 diff = block.number - _max(lastWithdrawRewards, uint256(staker.lastUpdated));

        CoinAge memory coinAge = CoinAge({
            lastWithdrawRewards: lastWithdrawRewards,
            diff: diff,
            stakedCoinAge: staker.stakedCoinAge + diff * uint256(staker.stakedAmount),
            lockedCoinAge: staker.lockedCoinAge,
            frozenCoinAge: frozenCoinAge[stakerAddress]
        });

        return coinAge;
    }
```

Below you can see the function _calculateRewardsByBlocks:

```solidity
function _calculateRewardsByBlocks(
        address account,
        address token,
        uint256 pastBlocks,
        Info memory userInfo,
        uint256 totalStaked,
        UserManagerAccountState memory user
    ) internal view returns (uint256) {
        uint256 startInflationIndex = users[account][token].inflationIndex;

        if (user.effectiveStaked == 0 || totalStaked == 0 || startInflationIndex == 0 || pastBlocks == 0) {
            return 0;
        }

        uint256 rewardMultiplier = _getRewardsMultiplier(user);

        uint256 curInflationIndex = _getInflationIndexNew(totalStaked, pastBlocks);

        if (curInflationIndex < startInflationIndex) revert InflationIndexTooSmall();

        return
            userInfo.accrued +
            (curInflationIndex - startInflationIndex).wadMul(user.effectiveStaked).wadMul(rewardMultiplier);
    }
```

## Code Snippet
Function `unstake` - https://github.com/sherlock-audit/2023-02-union-CodingNameKiki/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L711-L731

Function `_updateStakedCoinAge` - https://github.com/sherlock-audit/2023-02-union-CodingNameKiki/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L1064-L1070

Function `withdrawRewards` - https://github.com/sherlock-audit/2023-02-union-CodingNameKiki/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L224-L257

Function `_getUserInfo` - https://github.com/sherlock-audit/2023-02-union-CodingNameKiki/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L311-L329

Function `onWithdrawRewards` - https://github.com/sherlock-audit/2023-02-union-CodingNameKiki/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L969-L993

Function `_getEffectiveAmounts` - https://github.com/sherlock-audit/2023-02-union-CodingNameKiki/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L892-L938

Function `_getCoinAge` - https://github.com/sherlock-audit/2023-02-union-CodingNameKiki/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L1072-L1087

Function `_calculateRewardsByBlocks` - https://github.com/sherlock-audit/2023-02-union-CodingNameKiki/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L341-L364

## Tool used

Manual Review

## Recommendation
One way of fixing this problem, that l can think of is to refactor the function _calculateRewardsByBlocks.
First the function _calculateRewardsByBlocks will revert if `(totalStaked == 0 || startInflationIndex == 0 || pastBlocks == 0)`.
Second new if statement is created, which is triggered if `user.effectiveStaked == 0`.
- if `userInfo.accrued == 0`, it will return 0
- if `userInfo.accrued != 0`, it will return the accrued balance.

```solidity
function _calculateRewardsByBlocks(
        address account,
        address token,
        uint256 pastBlocks,
        Info memory userInfo,
        uint256 totalStaked,
        UserManagerAccountState memory user
    ) internal view returns (uint256) {
        uint256 startInflationIndex = users[account][token].inflationIndex;

        if (totalStaked == 0 || startInflationIndex == 0 || pastBlocks == 0) {
            revert ZeroNotAllowed();
        }
        
        if (user.effectiveStaked == 0) {
           if (userInfo.accrued == 0) return 0;
           else return userInfo.accrued
        }

        uint256 rewardMultiplier = _getRewardsMultiplier(user);

        uint256 curInflationIndex = _getInflationIndexNew(totalStaked, pastBlocks);

        if (curInflationIndex < startInflationIndex) revert InflationIndexTooSmall();

        return
            userInfo.accrued +
            (curInflationIndex - startInflationIndex).wadMul(user.effectiveStaked).wadMul(rewardMultiplier);
    }
```
