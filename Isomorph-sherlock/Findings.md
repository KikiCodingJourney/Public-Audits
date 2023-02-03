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
| H-02 | The variable totalUSDborrowed is wrongly calculated in the function openLoan in Vault_Synths. Users who want to increase their loans will receive less isoUSD. | High |
| H-03 | Owner won't be able to burn his NFTs and receive his AMM tokens back, duo to important missed check to ensure the NFTs burned were minted on this particular depositer contract. | High |
| M-01 | Parameters of paused collateral tokens can't be changed. | Medium |
| M-02 | Missing sanity check to ensure that the currency key isn't already used on existing collateral token. | Medium |
| M-03 | CHANGE_COLLATERAL_DELAY contains a wrong number. | Medium |
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
function partialWithdrawFromGauge(uint256 _NFTId, uint256 _percentageSplit, address[] memory _tokens) public { 
     uint256 newNFTId = depositReceipt.split(_NFTId, _percentageSplit); 
     //then call withdrawFromGauge on portion removing. 
     withdrawFromGauge(newNFTId, _tokens); 
```

```solidity
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
This is simply not true for the owner part - as the msg.sender whose calling the functions split and burn in depositReceipt_base will always be the depositer contract.

So the functions split and burn in the contract depositReceipt_base will check if the depositer contract is the owner of the NFT or it's approved.

```solidity
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
function burn(uint256 _NFTId) external onlyMinter{ 
     require(_isApprovedOrOwner(msg.sender, _NFTId), "ERC721: caller is not token owner or approved"); 
     delete pooledTokens[_NFTId]; 
     delete relatedDepositor[_NFTId]; 
     _burn(_NFTId); 
 } 
```

This action leads to the following scenario:

1. The owner calls the function depositToGauge couple of times depositing AMM tokens and receiving newly minted NFTs equaled to the price of the AMM tokens deposited.
2. After some time the owner wants to withdraw his AMM tokens and burn his NFTs, but as how it is right now with the issue occurring the only way to withdraw is if the owner approve his NFTs to the depositer contract.
3. A malicious user sees that and calls the function withdrawFromGauge and successfully burns the approved NFTs and steals the AMM tokens.

Note: This is possible as the msg.sender calling the external functions split and burn in depositReceipt_base will always be the depositer contract.

The owners who approve their NFTs to their own depositer contract in order to withdraw from gauge can lose their NFTs.
As anyone can call the function withdrawFromGauge and burn the approved NFTs and steal the AMM tokens.
This will result in a big material loss, as owners will lose their NFTs by malicious users.

### Impact
Duo to the issue described in Vulnerability Detail any user can call the functions partialWithdrawFromGauge,
withdrawFromGauge and successfully burn the approved NFTs to the depositer contract and steal the owner's AMM tokens equaled to the price of the NFTs.

### Tool used
Manual Review

### Recommendation

My recommended change is: https://gist.github.com/CodingNameKiki/66b1d6a4c180b0f217644b555994f8ba


### [H-02] The variable totalUSDborrowed is wrongly calculated in the function openLoan in Vault_Synths. Users who want to increase their loans will receive less isoUSD.

### Summary
The variable totalUSDborrowed is wrongly calculated in the function openLoan in Vault_Synths.
Users who want to increase their loans will receive less isoUSD.

### Vulnerability Detail
In the function openLoan, the variable totalUSDborrowed is wrongly calculated.
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

The issue occurs in the function openLoan in vault_synths on L120.

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
Duo to a mistake made in the function `openLoan` in Vault_Synths, the variable totalUSDborrowed is wrongly calculated duo to the wrong mapping used. Which leads to the issue described in Vulnerability Detail.

You can see that the right mapping was used in another vault as vault_lyra.

```solidity
Isomorph/contracts/Vault_Lyra.sol

uint256 totalUSDborrowed = _USDborrowed +  (isoUSDLoanAndInterest[_collateralAddress][msg.sender] * virtualPrice)/LOAN_SCALE;
```

### Tool used
Manual Review

### Recommendation
Fix the issue by using the right mapping:

totalUSDborrowed = _USDborrowed +  (isoUSDLoanAndInterest[_collateralAddress][msg.sender] * virtualPrice)/LOAN_SCALE;

### [H-03] Owner won't be able to burn his NFTs and receive his AMM tokens back, duo to important missed check to ensure the NFTs burned were minted on this particular depositer contract.

### Summary
Owner won't be able to burn his NFTs and receive his AMM tokens back, duo to important missed check to ensure the NFTs burned were minted on this particular depositer contract.

### Vulnerability Detail
Every user who calls the function makeNewDepositor in Templater.sol receives a new depositer contract and the ownership of it.
Every depositer contract has a different address and owner. The problem here is that the owner is the only one, who can deposit to gauge with the function depositToGauge, but anyone can call the functions partialWithdrawFromGauge, withdrawFromGauge and withdraw from gauge.

This actions can lead to this scenario:

Kiki the owner of the depositer contract calls the function depositToGauge and deposits 1000 AMM tokens, as a result a newly NFT is minted and sent to him equaled to the price of 1000 AMM tokens.

```solidity
function depositToGauge(uint256 _amount) onlyOwner() external returns(uint256){ 
      //AMMToken adheres to ERC20 spec meaning it reverts on failure, no need to check return 
     //slither-disable-next-line unchecked-transfer 
     AMMToken.transferFrom(msg.sender, address(this), _amount); 
  
     AMMToken.safeIncreaseAllowance(address(gauge), _amount); 
     //we are not attaching a veNFT to the deposit so the 2nd arg is always 0. 
     gauge.deposit(_amount, 0); 
     uint256 NFTId = depositReceipt.safeMint(_amount); 
     //safeMint sends the minted DepositReceipt to Depositor so now we forward it to the user. 
     depositReceipt.safeTransferFrom(address(this), msg.sender, NFTId); 
     return(NFTId); 
 } 
```

In the deposit function in gauge the deposited amount is added to the mapping balanceOf[msg.sender] += amount
As a result Kiki's depositer contract has a balance of 1000 AMM tokens now.

https://github.com/velodrome-finance/contracts/blob/de6b2a19b5174013112ad41f07cf98352bfe1f24/contracts/Gauge.sol#L463

Since anyone can withdraw from gauge, as the functions partialWithdrawFromGauge, withdrawFromGauge are public.
And there is no check to ensure the NFTId was minted on this particular depositer contract.

A user calls the function withdrawFromGauge burns his NFT and receives his 500 AMM tokens back.

```solidity
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

As a result the balance of Kiki's depositer contract will be equal to 500 AMM tokens in gauge.

https://github.com/velodrome-finance/contracts/blob/de6b2a19b5174013112ad41f07cf98352bfe1f24/contracts/Gauge.sol#L505

When Kiki decides to withdraw from gauge, he won't be able to do it as his contract's balance in gauge will be only 500 tokens and his NFT price is 1000 AMM tokens.

This issue breaks the purpose of the contracts, as every NFT minted leads to its depositer contract with the mapping
relatedDepositor and every NFT should only be burned on its own depositer contract to prevent issues like this occurring.

### Impact
Duo to the issue described in Vulnerability Detail, the owner of the contract won't be able to withdraw from gauge.

### Tool used
Manual Review

### Recommendation
My recommended change: https://gist.github.com/CodingNameKiki/d0f989eaed118890799067ebe83b659c


### [M-01] Parameters of paused collateral tokens can't be changed.

### Summary
Parameters of paused collateral tokens can't be changed.

### Vulnerability Detail
As how it is now the function queueCollateralChange() can't be called on paused collateral, duo to the modifier collateralExists.
This can be a problem if the following scenario occurs:

Admin calls the function addCollateralType() and successfully adds a new collateral token. Later by checking the mapping collateralProps, he notices a mistake he made in the collateral parameters. The admin doesn't want anyone to open a loan with the wrong parameters of the collateral, so he calls the function pauseCollateralType() and pauses the collateral.

```solidity
function pauseCollateralType( 
     address _collateralAddress, 
     bytes32 _currencyKey 
     ) external collateralExists(_collateralAddress) onlyAdmin { 
     require(_collateralAddress != address(0)); //this should get caught by the collateralExists check but just to be careful 
     //checks two inputs to help prevent input mistakes 
     require( _currencyKey == collateralProps[_collateralAddress].currencyKey, "Mismatched data"); 
     collateralValid[_collateralAddress] = false; 
     collateralPaused[_collateralAddress] = true; 
      
 } 
```

Admin can't call the function queueCollateralChange(), duo to the modifier collateralExists as it check if the collateral exists.
This is potentially a problem because the same collateral address can't be added again, collateral can't be changed duo to the modifier on the function queueCollateralChange() and neither the admin wants to unpause the collateral as someone can open a loan with the wrong parameters of the collateral.

```solidity
function queueCollateralChange( 
     address _collateralAddress, 
     bytes32 _currencyKey, 
     uint256 _minimumRatio, 
     uint256 _liquidationRatio, 
     uint256 _interestPer3Min, 
     uint256 _assetType, 
     address _liquidityPool 
  
 ) external collateralExists(_collateralAddress) onlyAdmin { 
```

```solidity
modifier collateralExists(address _collateralAddress){ 
     require(collateralValid[_collateralAddress], "Unsupported collateral!"); 
     _; 
 } 
```

### Impact
Duo to the issue described in "Vulnerability Detail", parameters of paused collateral tokens can't be changed if necessary.
As the function queueCollateralChange() reverts, because of the modifier collateralExists.

### Tool used
Manual Review

### Recommendation
Consider creating a new modifier which will be applied on the functions queueCollateralChange() and updateVirtualPriceSlowly() and can be called by paused collateral as well. https://gist.github.com/CodingNameKiki/7820aa470ada9bdc57dafdc466a08de6

// Note having the modifier on updateVirtualPriceSlowly() has another plus, which is that the virtual price of temporary paused collateral tokens can be updated as well, so it won't lead to DOS.

### [M-02] Missing sanity check to ensure that the currency key isn't already used on existing collateral token.

### Summary
Missing sanity check to ensure that the currency key isn't already used on existing collateral token.

### Vulnerability Detail
There is no sanity check in the function addCollateralType() to ensure that the currency key isn't already used on an existing collateral token. As a result the currency key will lead to two collateral tokens and the address of the liquidity pool for the second collateral token will override the address of the liquidity pool from the first one in the mapping
mapping(bytes32 => address) public liquidityPoolOf.

```solidity
function addCollateralType( 
     address _collateralAddress, 
     bytes32 _currencyKey, 
     uint256 _minimumRatio, 
     uint256 _liquidationRatio, 
     uint256 _interestPer3Min, 
     uint256 _assetType, 
     address _liquidityPool 
     ) external onlyAdmin { 
  
     require(!collateralValid[_collateralAddress], "Collateral already exists"); 
     require(!collateralPaused[_collateralAddress], "Collateral already exists"); 
     require(_collateralAddress != address(0)); 
     require(_minimumRatio > _liquidationRatio); 
     require(_liquidationRatio > 0); 
     require(vaults[_assetType] != address(0), "Vault not deployed yet"); 
     IVault vault = IVault(vaults[_assetType]); 
  
     //prevent setting liquidationRatio too low such that it would cause an overflow in callLiquidation, see appendix on liquidation maths for details. 
     require( vault.LIQUIDATION_RETURN() *_liquidationRatio >= 10 ** 36, "Liquidation ratio too low"); //i.e. 1 when multiplying two 1 ether scale numbers. 
     collateralValid[_collateralAddress] = true; 
     collateralProps[_collateralAddress] = Collateral( 
         _currencyKey, 
         _minimumRatio, 
         _liquidationRatio, 
         _interestPer3Min, 
         block.timestamp, 
         1 ether, 
         _assetType 
         ); 
     //Then update LiqPool as this isn't stored in the struct and requires the currencyKey also. 
     liquidityPoolOf[_currencyKey]= _liquidityPool;  
 } 
```
```solidity
liquidityPoolOf[_currencyKey]= _liquidityPool;  
```

This action will lead to problems occurring in the vault, as one currency key leads to two collateral tokens and one of the collateral tokens uses the address of a wrong liquidity pool to check the token price and if collateral is still active.

```solidity
function priceCollateralToUSD(bytes32 _currencyKey, uint256 _amount) public view override returns(uint256){ 
      //The LiquidityPool associated with the LP Token is used for pricing 
     ILiquidityPoolAvalon LiquidityPool = ILiquidityPoolAvalon(collateralBook.liquidityPoolOf(_currencyKey)); 
     //we have already checked for stale greeks so here we call the basic price function. 
     uint256 tokenPrice = LiquidityPool.getTokenPrice();           
     uint256 withdrawalFee = _getWithdrawalFee(LiquidityPool); 
     uint256 USDValue  = (_amount * tokenPrice) / LOAN_SCALE; 
     //we remove the Liquidity Pool withdrawalFee  
     //as there's no way to remove the LP position without paying this. 
     uint256 USDValueAfterFee = USDValue * (LOAN_SCALE- withdrawalFee)/LOAN_SCALE; 
     return(USDValueAfterFee); 
```

```solidity
function _checkIfCollateralIsActive(bytes32 _currencyKey) internal view override { 
          
          //Lyra LP tokens use their associated LiquidityPool to check if they're active 
          ILiquidityPoolAvalon LiquidityPool = ILiquidityPoolAvalon(collateralBook.liquidityPoolOf(_currencyKey)); 
          bool isStale; 
          uint circuitBreakerExpiry; 
          //ignore first output as this is the token price and not needed yet. 
          (, isStale, circuitBreakerExpiry) = LiquidityPool.getTokenPriceWithCheck(); 
          require( !(isStale), "Global Cache Stale, can't trade"); 
          require(circuitBreakerExpiry < block.timestamp, "Lyra Circuit Breakers active, can't trade"); 
 } 
```

### Impact
The issue described in "Vulnerability Detail" can lead to problems occurring in the vault, as there is no sanity check in addCollateralType() to ensure that the currency key wasn't already used on other collateral token.

### Tool used
Manual Review

### Recommendation
Add a sanity check to ensure that the currency key isn't already used on an existing token.

https://gist.github.com/CodingNameKiki/33e2d63e73694e7eab35cb66db17b124

There seems to be missing check for that in the function queueCollateralChange() too, might want to add the check there as well.

### [M-03] CHANGE_COLLATERAL_DELAY contains a wrong number.

### Summary
CHANGE_COLLATERAL_DELAY contains a wrong number.

### Vulnerability Detail
As you can see by the comment next to it, CHANGE_COLLATERAL_DELAY is supposed to be 2 days.
But as how it is right now the delay will be only 200 seconds.

```solidity
uint256 public constant CHANGE_COLLATERAL_DELAY = 200; //2 days 
```

As a result the require statement in the function changeCollateralType() can be bypassed after 200 seconds instead of 2 days.

```solidity
require(submissionTimestamp + CHANGE_COLLATERAL_DELAY <= block.timestamp, "Not enough time passed");
```

### Impact
This issue breaks the logic, as the "time delays" are important to the protocol.
And leads to bypassing a certain require statement in a function in a short period of time than it's supposed to be.

### Tool used
Manual Review

### Recommendation
Change to - uint256 public constant CHANGE_COLLATERAL_DELAY = 2 days;

### [L-01] Users can spam mint new NFTs in the function split in DepositReceipt_Base providing zero as _percentageSplit.

### Summary
Users can spam mint new NFTs in the function split in DepositReceipt_Base providing zero as _percentageSplit.

### Vulnerability Detail
l can't see how this will lead to anything bad, as how it is right now the only problem is that the variable currentLastId can be spammed like that. Certainly from looking over the code, splitting a NFT with zero _percentageSplit is forbidden, but the split function is lacking this require statement in DepositReceipt_Base.

```solidity
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

### Impact
Users can spam mint new NFTs with zero values

### Tool used
Manual Review

### Recommendation
Add a require statement to prevent minting new NFTs with zero value.

### [L-02] Unused require statement in changeCollateralType()

### Summary
Unused require statement in changeCollateralType()

### Vulnerability Detail
As how it is the require statement is only useful for the first time someone calls changeCollateralType().
require(submissionTimestamp != 0, "Uninitialized collateral change");
Every time the function is called and collatoral type is changed, the variable submissionTimestamp should be updated to zero.
So the function can't be called second time overriding the same params. The issue doesn't lead to anything bad tho.

```solidity
function changeCollateralType() external onlyAdmin {
        uint256 submissionTimestamp = queuedTimestamp;
        require(submissionTimestamp != 0, "Uninitialized collateral change");
        require(submissionTimestamp + CHANGE_COLLATERAL_DELAY <= block.timestamp, "Not enough time passed");
        address collateralAddress = queuedCollateralAddress;
        bytes32 currencyKey = queuedCurrencyKey;
        uint256 minimumRatio = queuedMinimumRatio;
        uint256 liquidationRatio = queuedLiquidationRatio;
        uint256 interestPer3Min = queuedInterestPer3Min;
        address liquidityPool = queuedLiquidityPool;
        

        //Now we must ensure interestPer3Min changes aren't applied retroactively
        // by updating the assets virtualPrice to current block timestamp
        uint256 timeDelta = (block.timestamp - collateralProps[collateralAddress].lastUpdateTime) / THREE_MIN;
        if (timeDelta != 0){ 
           updateVirtualPriceSlowly(collateralAddress, timeDelta );
        }
        bytes32 oldCurrencyKey = collateralProps[collateralAddress].currencyKey;

        _changeCollateralParameters(
            collateralAddress,
            currencyKey,
            minimumRatio,
            liquidationRatio,
            interestPer3Min
        );
        //Then update LiqPool as this isn't stored in the struct and requires the currencyKey also.
        liquidityPoolOf[oldCurrencyKey]= address(0); 
        liquidityPoolOf[currencyKey]= liquidityPool;
        
    }
```
### Impact
The require statement in changeCollateralType() should be used, so no one can call the function and override the same params.

### Tool used
Manual Review

### Recommendation
Update the variable submissionTimestamp to zero at the end of the function changeCollateralType().
