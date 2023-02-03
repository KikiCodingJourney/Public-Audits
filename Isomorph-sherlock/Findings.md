### Issues Template
| Letter | Name | Description |
|:--:|:-------:|:-------:|
| H  | High risk | Loss of funds |
| M |  Medium risk | Unexpected behavior |
| L  | Low risk | Potentially a risk |

| Total Found Issues | 8 |
|:--:|:--:|

### Findings template
| Count | Explanation | Risk |
|:--:|:-------|:--:|
| H-01 | Malicious user can burn the approved NFTs to the depositer contract and successfully steal owner's AMM tokens. | High |
| H-02 | The variable `totalUSDborrowed` is wrongly calculated in the function openLoan in Vault_Synths. Users who want to increase their loans will receive less isoUSD. | High |
| H-03 | Owner won't be able to burn his NFTs and receive his AMM tokens back, duo to important missed check to ensure the NFTs burned were minted on this particular depositer contract. | High |
| M-01 | Parameters of paused collateral tokens can't be changed. | Medium |
| M-02 | Missing sanity check to ensure that the currency key isn't already used on existing collateral token. | Medium |
| M-03 | `CHANGE_COLLATERAL_DELAY` contains a wrong number. | Medium |
| L-01 | Users can spam mint new NFTs in the function split in DepositReceipt_Base providing zero as _percentageSplit. | Low |
| L-02 | Unused require statement in changeCollateralType(). | Low |

### [H-01] Malicious user can burn the approved NFTs to the depositer contract and successfully steal owner's AMM tokens.

### Summary
Malicious user can burn the approved NFTs to the depositer contract and successfully steal owner's AMM tokens.

### Vulnerability Detail
The purpose of the depositer contract is simple, anyone who calls the function makeNewDepositor in Templater.sol receives a new depositer contract and gets the ownership of it. The owner can deposit to gauge, withdraw from gauge and claim rewards accrued from the gauge.

The functions depositToGauge, claimRewards are callable only by the owner of the contract and few of the rest of the functions
partialWithdrawFromGauge, withdrawFromGauge are public and can be called by anyone.

The problem here occurs with the external calls to the contract "depositReceipt" in the functions partialWithdrawFromGauge, withdrawFromGauge.

```solidity
Velo-Deposit-Tokens/contracts/Depositor.sol

function partialWithdrawFromGauge(uint256 _NFTId, uint256 _percentageSplit, address[] memory _tokens) public { 
     uint256 newNFTId = depositReceipt.split(_NFTId, _percentageSplit); 
     //then call withdrawFromGauge on portion removing. 
     withdrawFromGauge(newNFTId, _tokens); 
```

```solidity
Velo-Deposit-Tokens/contracts/Depositor.sol

function withdrawFromGauge(uint256 _NFTId, address[] memory _tokens)  public  { 
     uint256 amount = depositReceipt.pooledTokens(_NFTId); 
     depositReceipt.burn(_NFTId); 
     gauge.getReward(address(this), _tokens); 
     gauge.withdraw(amount); 
     //AMMToken adheres to ERC20 spec meaning it reverts on failure, no need to check return 
     //slither-disable-next-line unchecked-transfer 
     AMMToken.transfer(msg.sender, amount); 
 } 
```

`By the docs:`
To split or burn the inputted NFT the caller must either own the NFT or have added the Depositor contract to the NFTs approved list
This is simply not true for the owner part - as the msg.sender whose calling the functions `split` and `burn` in depositReceipt_base will always be the depositer contract.

So the functions `split` and `burn` in the contract depositReceipt_base will check if the depositer contract is the owner of the NFT or it's approved.

```solidity
Velo-Deposit-Tokens/contracts/DepositReceipt_Base.sol

function split(uint256 _NFTId, uint256 _percentageSplit) external returns (uint256) { 
   require(_percentageSplit < BASE, "split must be less than 100%"); 
   require(_isApprovedOrOwner(msg.sender, _NFTId), "ERC721: caller is not token owner or approved"); 
  
   uint256 existingPooledTokens = pooledTokens[_NFTId]; 
   uint256 newPooledTokens = (existingPooledTokens * _percentageSplit)/ BASE; 
   pooledTokens[_NFTId] = existingPooledTokens - newPooledTokens; 
   uint256 newNFTId = _mintNewNFT(newPooledTokens, relatedDepositor[_NFTId]); 
    
  
   emit NFTSplit(_NFTId, newNFTId); 
   emit NFTDataModified(_NFTId, existingPooledTokens, existingPooledTokens - newPooledTokens); 
   emit NFTDataModified(newNFTId, 0, newPooledTokens); 
   return newNFTId; 
   } 
```

```solidity
Velo-Deposit-Tokens/contracts/DepositReceipt_Base.sol

function burn(uint256 _NFTId) external onlyMinter{ 
     require(_isApprovedOrOwner(msg.sender, _NFTId), "ERC721: caller is not token owner or approved"); 
     delete pooledTokens[_NFTId]; 
     delete relatedDepositor[_NFTId]; 
     _burn(_NFTId); 
 } 
```

This action leads to the following scenario:

1. The owner calls the function `depositToGauge` couple of times depositing AMM tokens and receiving newly minted NFTs equaled to the price of the AMM tokens deposited.
2. After some time the owner wants to withdraw his AMM tokens and burn his NFTs, but as how it is right now with the issue occurring the only way to withdraw is if the owner approve his NFTs to the depositer contract.
3. A malicious user sees that and calls the function `withdrawFromGauge` and successfully burns the approved NFTs and steals the AMM tokens.

Note: This is possible as the msg.sender calling the external functions split and burn in depositReceipt_base will always be the depositer contract.

Result:
The owners who approve their NFTs to their own depositer contract in order to withdraw from gauge can lose their NFTs.
As anyone can call the function `withdrawFromGauge` and burn the approved NFTs and steal the AMM tokens.
This will result in a big material loss, as owners will lose their NFTs by malicious users.

### Impact
Duo to the issue described in Vulnerability Detail any user can call the functions `partialWithdrawFromGauge`,
`withdrawFromGauge` and successfully burn the approved NFTs to the depositer contract and steal the owner's AMM tokens equaled to the price of the NFTs.

### Tool used
Manual Review

### Recommendation

My recommended change is: https://gist.github.com/CodingNameKiki/66b1d6a4c180b0f217644b555994f8ba

### [H-02] The variable `totalUSDborrowed` is wrongly calculated in the function openLoan in Vault_Synths. Users who want to increase their loans will receive less isoUSD.

### Summary
The variable totalUSDborrowed is wrongly calculated in the function openLoan in Vault_Synths.
Users who want to increase their loans will receive less isoUSD.

### Vulnerability Detail
In the function openLoan(), the variable totalUSDborrowed is wrongly calculated.
With the issue occurring here, the wrong mapping isoUSDLoaned is used instead of isoUSDLoanAndInterest.
And as how it is right now, the variable totalUSDborrowed will be equal to a wrong amount (bigger amount),
preventing users from getting their full amount of isoUSD, when they call openLoan to increase their loan.

When the first time someone opens a loan, everything will be normal.
The problem occurs if they already opened a loan and want to increase it, by calling the function openLoan again.

Note: ignoring the minimum loan size of 100 isoUSD (100 * 10^18), so the example can be more understandable.
The same argument works for larger values.

Example:
Kiki opened a loan with the params: _colAmount = 1000, _USDborrowed = 1000.

// Let's say virtual price is 1.10 * 1 ether.
As a result his mappings will look like this:
collateralPosted[_collateralAddress][msg.sender] = 1000;
isoUSDLoaned[_collateralAddress][msg.sender] = 1000;
isoUSDLoanAndInterest[_collateralAddress][msg.sender] = 909

Later Kiki decides that he wants to increase his loan with the same params as the first one.
This time Kiki will be able to borrow less isoUSD duo to the wrong mapping applied.

// Let's say the virtual price became 1.11 * 1 ether than the last time.

What totalUSDborrowed will show with the wrong mapping:
totalUSDborrowed = _USDborrowed +  (isoUSDLoan[_collateralAddress][msg.sender] * virtualPrice)/LOAN_SCALE;
totalUSDborrowed = 1 000 +  (1 000 * (1.11 * 1 ether)) / 1 ether => totalUSDborrowed = 2111

What totalUSDborrowed is supposed to show with the right mapping:
totalUSDborrowed = _USDborrowed +  (isoUSDLoanAndInterest[_collateralAddress][msg.sender] * virtualPrice)/LOAN_SCALE;
totalUSDborrowed = 1 000 +  (909 * (1.11 * 1 ether)) / 1 ether => totalUSDborrowed => 2008

As you can see with the wrong mapping there is a difference.
Kiki won't be able to borrow his full amount of isoUSD duo to the wrong mapping applied.

The issue occurs in the function openLoan() in vault_synths on L120.

```solidity
Isomorph/contracts/Vault_Synths.sol

function openLoan( 
     address _collateralAddress, 
     uint256 _colAmount, 
     uint256 _USDborrowed 
     ) external override whenNotPaused  
     { 
     _collateralExists(_collateralAddress); 
     IERC20 collateral = IERC20(_collateralAddress); 
     require(collateral.balanceOf(msg.sender) >= _colAmount, "User lacks collateral quantity!"); 
     //make sure virtual price is related to current time before fetching collateral details 
     //slither-disable-next-line reentrancy-vulnerabilities-1 
     _updateVirtualPrice(block.timestamp, _collateralAddress);   
      
     (    
         bytes32 currencyKey, 
         uint256 minOpeningMargin, 
         , 
         , 
         , 
         uint256 virtualPrice, 
          
     ) = _getCollateral(_collateralAddress); 
     //check for frozen or paused collateral 
     _checkIfCollateralIsActive(currencyKey); 
  
     //make sure the total isoUSD borrowed doesn't exceed the opening borrow margin ratio 
     uint256 colInUSD = priceCollateralToUSD(currencyKey, _colAmount + collateralPosted[_collateralAddress][msg.sender]); 
     uint256 totalUSDborrowed = _USDborrowed +  (isoUSDLoaned[_collateralAddress][msg.sender] * virtualPrice)/LOAN_SCALE; 
     require(totalUSDborrowed >= ONE_HUNDRED_DOLLARS, "Loan Requested too small");  
     uint256 borrowMargin = (totalUSDborrowed * minOpeningMargin) / LOAN_SCALE; 
     require(colInUSD >= borrowMargin, "Minimum margin not met!"); 
  
     //update mappings with new loan amounts 
     collateralPosted[_collateralAddress][msg.sender] = collateralPosted[_collateralAddress][msg.sender] + _colAmount; 
     isoUSDLoaned[_collateralAddress][msg.sender] = isoUSDLoaned[_collateralAddress][msg.sender] + _USDborrowed; 
     isoUSDLoanAndInterest[_collateralAddress][msg.sender] = isoUSDLoanAndInterest[_collateralAddress][msg.sender] + ((_USDborrowed * LOAN_SCALE) / virtualPrice); 
      
     emit OpenOrIncreaseLoan(msg.sender, _USDborrowed, currencyKey, _colAmount); 
  
     //Now all effects are handled, transfer the assets so we follow CEI pattern 
     _increaseCollateral(collateral, _colAmount); 
     _increaseLoan(_USDborrowed); 
```

```solidity
uint256 totalUSDborrowed = _USDborrowed +  (isoUSDLoaned[_collateralAddress][msg.sender] * virtualPrice)/LOAN_SCALE; 
```

### Impact
Duo to a mistake made in the function openLoan() in Vault_Synths, the variable totalUSDborrowed is wrongly calculated duo to the wrong mapping used. Which leads to the issue described in Vulnerability Detail.

You can see that the right mapping was used in another vault as vault_lyra.

```solidity
Isomorph/contracts/Vault_Lyra.sol

uint256 totalUSDborrowed = _USDborrowed +  (isoUSDLoanAndInterest[_collateralAddress][msg.sender] * virtualPrice)/LOAN_SCALE;
```

### Tool used
Manual Review

### Recommendation
Fix the issue by using the right mapping:
`totalUSDborrowed = _USDborrowed +  (isoUSDLoanAndInterest[_collateralAddress][msg.sender] * virtualPrice)/LOAN_SCALE;`
