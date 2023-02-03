### Issues Template
| Letter | Name | Description |
|:--:|:-------:|:-------:|
| H  | High risk | Loss of funds |
| M |  Medium risk | Unexpected behavior |
| L  | Low risk | Potentially a risk |

| Total Found Issues | 2 |
|:--:|:--:|

### Findings template
| Count | Explanation | Risk |
|:--:|:-------|:--:|
| H-01 | Users won't be able to repay their overduе borrows, duo to a simple mistake made in the function updateFrozenInfo. | High |
| M-01 | Interest can potentially be wrongly calculated, when a borrower repays. | Medium |

### [H-01] Users won't be able to repay their overduе borrows, duo to a simple mistake made in the function updateFrozenInfo.

## Summary
Users won't be able to repay their overduе borrows, duo to a simple mistake made in the function updateFrozenInfo.

## Vulnerability Detail
In the function _repayBorrowFresh(), the if statement if (isOverdue) is triggered for borrowers that are paying back overdue balances, so the function updateFrozenInfo() is called to update their frozen balance and the global total frozen balance on the UserManager contract.

```solidity
 if (isOverdue) { 
     // For borrowers that are paying back overdue balances we need to update their 
     // frozen balance and the global total frozen balance on the UserManager 
     IUserManager(userManager).updateFrozenInfo(borrower, 0); 
 } 
```

The problem here is that the function updateFrozenInfo() can be called only by the comptroller contract.
So every time the function is called by the uToken contract, it will revert.

```solidity
function updateFrozenInfo(address staker, uint256 pastBlocks) external onlyComptroller returns (uint256, uint256) { 
     return _updateFrozen(staker, pastBlocks); 
 } 
```

```solidity
modifier onlyComptroller() { 
     if (address(comptroller) != msg.sender) revert AuthFailed(); 
     _; 
 } 
```

As how it is right now, borrowers with overdue balances won't be able to repay back their borrows.
Because everytime the function updateFrozenInfo() is called by the uToken contract, it will revert.
And since updateFrozenInfo() reverts, the whole function _repayBorrowFresh() in the uToken contract will revert as well.

## Impact
Duo to the issue described in Vulnerability Detail, borrowers with overdue balances won't be able to repay back their borrows.

## Tool used
Manual Review

## Recommendation
Fix the function updateFrozenInfo(), so it can be called by the uToken contract and the comptroller contract.

https://gist.github.com/CodingNameKiki/87f68bac0cba93ef9c19ff3416df70ab

### [M-01] Interest can potentially be wrongly calculated, when a borrower repays.

## Summary
Interest can potentially be wrongly calculated, when a borrower repays.

## Vulnerability Detail
Example:

When a borrower wants to repay a loan, he calls the function repayBorrow(), which calls the internal function _repayBorrowFresh(). This is where the problem occurs:

```solidity
function _repayBorrowFresh( 
     address payer, 
     address borrower, 
     uint256 amount 
 ) internal { 
     if (!accrueInterest()) revert AccrueInterestFailed(); 
  
     uint256 interest = calculatingInterest(borrower); 
     uint256 borrowedAmount = borrowBalanceStoredInternal(borrower); 
     uint256 repayAmount = amount > borrowedAmount ? borrowedAmount : amount; 
     if (repayAmount == 0) revert AmountZero(); 
  
     uint256 toReserveAmount; 
     uint256 toRedeemableAmount; 
  
     if (repayAmount >= interest) { 
         // If the repayment amount is greater than the interest (min payment) 
         bool isOverdue = checkIsOverdue(borrower); 
  
         // Interest is split between the reserves and the uToken minters based on 
         // the reserveFactorMantissa When set to WAD all the interest is paid to teh reserves. 
         // any interest that isn't sent to the reserves is added to the redeemable amount 
         // and can be redeemed by uToken minters. 
         toReserveAmount = (interest * reserveFactorMantissa) / WAD; 
         toRedeemableAmount = interest - toReserveAmount; 
  
         // Update the total borrows to reduce by the amount of principal that has 
         // been paid off 
         totalBorrows -= (repayAmount - interest); 
  
         // Update the account borrows to reflect the repayment 
         accountBorrows[borrower].principal = borrowedAmount - repayAmount; 
         accountBorrows[borrower].interest = 0; 
  
         if (getBorrowed(borrower) == 0) { 
             // If the principal is now 0 we can reset the last repaid block to 0. 
             // which indicates that the borrower has no outstanding loans. 
             accountBorrows[borrower].lastRepay = 0; 
         } else { 
             // Save the current block number as last repaid 
             accountBorrows[borrower].lastRepay = getBlockNumber(); 
         } 
  
         // Call update locked on the userManager to lock this borrowers stakers. This function 
         // will revert if the account does not have enough vouchers to cover the repay amount. ie 
         // the borrower is trying to repay more than is locked (owed) 
         IUserManager(userManager).updateLocked(borrower, uint96(repayAmount - interest), false); 
  
         if (isOverdue) { 
             // For borrowers that are paying back overdue balances we need to update their 
             // frozen balance and the global total frozen balance on the UserManager 
             IUserManager(userManager).updateFrozenInfo(borrower, 0); 
         } 
     } else { 
         // For repayments that don't pay off the minimum we just need to adjust the 
         // global balances and reduce the amount of interest accrued for the borrower 
         toReserveAmount = (repayAmount * reserveFactorMantissa) / WAD; 
         toRedeemableAmount = repayAmount - toReserveAmount; 
         accountBorrows[borrower].interest = interest - repayAmount; 
     } 
  
     totalReserves += toReserveAmount; 
     totalRedeemable += toRedeemableAmount; 
  
     accountBorrows[borrower].interestIndex = borrowIndex; 
  
     // Transfer underlying token that have been repaid and then deposit 
     // then in the asset manager so they can be distributed between the 
     // underlying money markets 
     IERC20Upgradeable(underlying).safeTransferFrom(payer, address(this), repayAmount); 
     _depositToAssetManager(repayAmount); 
  
     emit LogRepay(payer, borrower, repayAmount); 
 } 
```

First it will call the function accrueInterest, which will update the accrualBlockNumber with the currentBlockNumber and borrowIndex with borrowIndexNew

```solidity
function accrueInterest() public override returns (bool) { 
     uint256 borrowRate = borrowRatePerBlock(); 
     uint256 currentBlockNumber = getBlockNumber(); 
     uint256 blockDelta = currentBlockNumber - accrualBlockNumber; 
  
     uint256 simpleInterestFactor = borrowRate * blockDelta; 
     uint256 borrowIndexNew = (simpleInterestFactor * borrowIndex) / WAD + borrowIndex; 
  
     accrualBlockNumber = currentBlockNumber; 
     borrowIndex = borrowIndexNew; 
```

The problem here is, when the function calls calculatingInterest() to calculate the interest.
Since accrualBlockNumber and borrowIndex were already updated with the currentBlockNumber and borrowIndexNew in the function accrueInterest().

The outcome here will be:

// accrualBlockNumber will equal to currentBlockNumber, which was already updated in accrueInterest(), so the outcome is 0.

uint256 blockDelta = currentBlockNumber - accrualBlockNumber => uint256 blockDelta = 0

// Since blockDelta is zero, the outcome of simpleInterestFactor will be also zero

uint256 simpleInterestFactor = borrowRate * blockDelta => borrowRate * 0 => uint256 simpleInterestFactor = 0

// Since simpleInterestFactor is zero, (simpleInterestFactor * borrowIndex) / WAD => (0 * borrowIndex) / 1e18 => 0

// All left in the calculation will be borrowIndex, so borrowIndexNew will be equal to borrowIndex.

uint256 borrowIndexNew = (simpleInterestFactor * borrowIndex) / WAD + borrowIndex => borrowIndexNew = borrowIndex

```solidity
function calculatingInterest(address account) public view override returns (uint256) { 
     BorrowSnapshot memory loan = accountBorrows[account]; 
  
     if (loan.principal == 0) { 
         return 0; 
     } 
  
     uint256 borrowRate = borrowRatePerBlock(); 
     uint256 currentBlockNumber = getBlockNumber(); 
     uint256 blockDelta = currentBlockNumber - accrualBlockNumber; 
     uint256 simpleInterestFactor = borrowRate * blockDelta; 
     uint256 borrowIndexNew = (simpleInterestFactor * borrowIndex) / WAD + borrowIndex; 
  
     uint256 principalTimesIndex = (loan.principal + loan.interest) * borrowIndexNew; 
     uint256 balance = principalTimesIndex / loan.interestIndex; 
  
     return balance - getBorrowed(account); 
```

The outcome of this will be that, when borrowers call the function repayBorrow() their interest can be wrongly calculated.
The interest will be calculated based on the old borrowRatePerBlock in accrueInterest() and not with the new borrowRatePerBlock in the function calculatingInterest.

```solidity
if (!accrueInterest()) revert AccrueInterestFailed(); 
  
 uint256 interest = calculatingInterest(borrower); 
```

If borrowRatePerBlock changes after the function repayBorrowFresh() is done with if (!accrueInterest()) and goes to
uint256 interest, the function will still calculate the interest based on the old borrowRatePerBlock used in accrueInterest().

Duo to how it is now, the code in calculatingInterest() L471-L475 is irrelevant.

```solidity
uint256 borrowRate = borrowRatePerBlock(); 
 uint256 currentBlockNumber = getBlockNumber(); 
 uint256 blockDelta = currentBlockNumber - accrualBlockNumber; 
 uint256 simpleInterestFactor = borrowRate * blockDelta; 
 uint256 borrowIndexNew = (simpleInterestFactor * borrowIndex) / WAD + borrowIndex; 
```

## Impact
Duo to the problem described in Vulnerability Detail, a scenario can happen when the interest will be calculated wrongly based on the old borrowRatePerBlock.

## Tool used
Manual Review

## Recommendation
Consider creating a function which will do both of the jobs for accrueInterest() and calculatingInterest().

https://gist.github.com/CodingNameKiki/14374f7b39ab87c0337905eb6db74835

// This function can be accessed only when a person repays, and won't lead to the problem occuring in "Vulnerability Detail".
