### Issues Template
| Letter | Name | Description |
|:--:|:-------:|:-------:|
| L  | Low risk | Potential risk |
| NC |  Non-critical | Non risky findings |
| R  | Refactor | Changing the code |
| O | Ordinary | Often found issues |

| Total Found Issues | 28 |
|:--:|:--:|

### Low Risk Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [L-01] | `_handleProtocolFee` should revert if feeTo returns address(0) instead of ignoring it | 1 |
| [L-02] | The function `_burn` should check if the msg.sender is the owner of the token or approved | 1 |
| [L-03] | Vault `deposit` function should check if the allowList is enabled | 1 |
| [L-04] | Vault can be shutdown, but can't be turn on again | 1 |


| Total Low Risk Issues | 4 |
|:--:|:--:|

### Non-Critical Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [N-01] | initialize function lacks address(0) checks | 1 |
| [N-02] | Owner shouldn't be able to approve himself | 1 |
| [N-03] | Typo in the function `_redeemFutureEpoch` | 1 |
| [N-04] | Modifier exists but isn't used when needed | 1 |
| [N-05] | Missing require strings | 26 |
| [N-06] | Unnecessary check in the function `incrementNonce` | 1 |
| [N-07] | Function `_setPayee` should check if the newPayee is address(0) | 1 |
| [N-08] | Event should be emitted regarding important changes | 1 |


| Total Non-Critical Issues | 8 |
|:--:|:--:|

### Refactor Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [R-01] | Modifier should be used instead of require on owner functions | 7 |
| [R-02] | Shorthand way to write if / else statement | 2 |
| [R-03] | Inputted value in the function should equals the safe cast value | 3 |
| [R-04] | Emergancy pause/unpause function should emit an event to notify users | 2 |
| [R-05] | Modifying an if statement in `_transferAndDepositAssetIfAble` | 1 |
| [R-06] | Unnecessary return statement applied | 1 |
| [R-07] | Use delete to clear variables instead of zero assignment | 5 |
| [R-08] | Some number values can be refactored with _ | 1 |


| Total Refactor Issues | 8 |
|:--:|:--:|

### Ordinary Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [O-01] | Import not used anywhere in the contract | 4 |
| [O-02] | Events is missing indexed fields | 7 |
| [O-03] | Proper use of get as a function name prefix | 4 | 
| [O-04] | Function Naming suggestions | 2 |
| [O-05] | NatSpec comments missing on some functions | 10 |
| [O-06] | Code contains empty blocks | 2 |
| [O-07] | Imported names on Interfaces should start with "I" | 4 |
| [O-08] | Hardcoded values can't be changed | 2 | 


| Total Ordinary Issues | 8 |
|:--:|:--:|

### [L-01] `_handleProtocolFee` should revert if feeTo returns address(0) instead of ignoring it
The function `_handleProtocolFee` uses an if statement, which is triggered if the feeTo address isn't address(0).
As how it is now the function ignores, when the local variable feeTo returns address(0).
This is problematic, as if this ever happens fees will be skipped and won't be paid.

```solidity
397:  function _handleProtocolFee(uint256 amount) internal returns (uint256) {
398:    address feeTo = ROUTER().feeTo();
399:    bool feeOn = feeTo != address(0);
400:    if (feeOn) {
401:      uint256 fee = ROUTER().getProtocolFee(amount);
402:
403:      unchecked {
404:        amount -= fee;
405:      }
406:      ERC20(asset()).safeTransfer(feeTo, fee);
407:    }
408:    return amount;
409:  }
```

Consider applying an else statement, if feeOn returns false to revert.

```solidity
} else {
  revert feeToReturnsAddressZero;
}
```

### [L-02] The function `_burn` should check if the msg.sender is the owner of the token or approved
The function `_burn` in ERC721.sol is used to burn tokens, but there is no actual check to ensure, that the msg.sender is the actual owner of the token id or approved. 

Consider adding this check in the function `_burn`:

```solidity
require(
      msg.sender == owner ||
        s.isApprovedForAll[owner][msg.sender] ||
        msg.sender == s.getApproved[id],
      "NOT_AUTHORIZED"
    );
```

```solidity
src/ERC721.sol

214:  function _burn(uint256 id) internal virtual {
215:    ERC721Storage storage s = _loadERC721Slot();
216:
217:    address owner = s._ownerOf[id];
218:
219:    require(owner != address(0), "NOT_MINTED");
220:
221:    // Ownership check above ensures no underflow.
222:    unchecked {
223:      s._balanceOf[owner]--;
224:    }
225:
226:    delete s._ownerOf[id];
227:
228:    delete s.getApproved[id];
229:
230:    emit Transfer(owner, address(0), id);
231:  }
```

### [L-03] Vault `deposit` function should check if the allowList is enabled
The deposit function check if the msg.sender calling the function is in the allowList.
This check isn't mandatory, when the allowList isn't enabled by the owner.

```solidity
src/VaultImplementation.sol

104:  function disableAllowList() external virtual {
105:    require(msg.sender == owner()); //owner is "strategist"
106:    _loadVISlot().allowListEnabled = false;
107:    emit AllowListEnabled(false);
108:  }
```

```solidity
src/Vault.sol

59:  function deposit(uint256 amount, address receiver)
60:    public
61:    virtual
62:    returns (uint256)
63:  {
64:    VIData storage s = _loadVISlot();
65:    require(s.allowList[msg.sender] && receiver == owner());
66:    ERC20(asset()).safeTransferFrom(msg.sender, address(this), amount);
67:    return amount;
68:  }
```

Consider adding a check and refactor the above function to:

```solidity
59:  function deposit(uint256 amount, address receiver)
60:    public
61:    virtual
62:    returns (uint256)
63:  {
64:    VIData storage s = _loadVISlot();
65:    require(receiver == owner());
66:    if (s.allowListEnabled){
67:      require(s.allowList[msg.sender]
68:    }
69:    ERC20(asset()).safeTransferFrom(msg.sender, address(this), amount);
70:    return amount;
71:  }
```

### [L-04] Vault can be shutdown, but can't turn on again
The function `shutdown` is used by the owner to shutdown the vault.
Which leads the core functions `commitToLien` and `buyoutLien` uncallable duo to the modifier whenNotPaused.
As how it is right now calling this function will lead to permanent and unusable vault, as there is no function to turn the vault back on again.

```solidity
src/VaultImplementation.sol

146:  function shutdown() external {
147:    require(msg.sender == owner()); //owner is "strategist"
148:    _loadVISlot().isShutdown = true;
149:    emit VaultShutdown();
150:  }
```

```solidity
131:  modifier whenNotPaused() {
132:    if (ROUTER().paused()) {
133:      revert InvalidRequest(InvalidRequestReason.PAUSED);
134:    }
135:
136:    if (_loadVISlot().isShutdown) {
137:      revert InvalidRequest(InvalidRequestReason.SHUTDOWN);
138:    }
139:    _;
140:  }
```

Consider applying a function, which will turn on the vault again:

```solidity
function turnOnVault() external {
  require(msg.sender == owner())
  _loadVISlot().isShutdown = false;
  emit VaultBackOn();
```

### [N-01] initialize function lacks address(0) checks
Zero address check should be used in the initialize function to prevent setting smth as mistake to address(0).

```solidity
src/AstariaRouter.sol

83: function initialize(
84:    Authority _AUTHORITY,
85:    ICollateralToken _COLLATERAL_TOKEN,
86:    ILienToken _LIEN_TOKEN,
87:    ITransferProxy _TRANSFER_PROXY,
88:    address _VAULT_IMPL,
89:    address _SOLO_IMPL,
90:    address _WITHDRAW_IMPL,
91:    address _BEACON_PROXY_IMPL,
92:    address _CLEARING_HOUSE_IMPL
93:  ) external initializer {
```

Consider adding zero address check like this:
```solidity
require(_VAULT_IMPL != address(0), "Address Zero");
```

### [N-02] Owner shouldn't be able to approve himself
The function `approve` is used by a token owner, to approve a certain user of using his token.
In general the owner shouldn't be able to approve himself, as this is best practice in [openzeppelin-ERC721](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L114).

```solidity
src/ERC721.sol

87:  function approve(address spender, uint256 id) external virtual {
88:    ERC721Storage storage s = _loadERC721Slot();
89:    address owner = s._ownerOf[id];
90:    require(
91:      msg.sender == owner || s.isApprovedForAll[owner][msg.sender],
92:      "NOT_AUTHORIZED"
93:    );
94:
95:    s.getApproved[id] = spender;
96:
97:    emit Approval(owner, spender, id);
98:  }
```

### [N-03] Typo
In the following instance, the comment tell us that the storage variable will underflow, if not enough balance.
This is simply not true, as it isn't unchecked and it will just revert.

```solidity
PublicVault.sol

173:  //this will underflow if not enough balance
174:    es.balanceOf[owner] -= shares;
```

### [N-04] Modifier exists but isn't used when needed
The function `mint` in WithdrawProxy.sol is supposed to be called only by the vault contract.
A require statement is used to ensure this is true, but a modifier exist already in the contract for this purpose and isn't used.

```solidity
src/WithdrawProxy.sol

132:  function mint(uint256 shares, address receiver)
133:    public
134:    virtual
135:    override(ERC4626Cloned, IERC4626)
136:    returns (uint256 assets)
137:  {
138:    require(msg.sender == VAULT(), "only vault can mint");
139:    _mint(receiver, shares);
140:    return shares;
141:  }

230:  modifier onlyVault() {
231:    require(msg.sender == VAULT(), "only vault can call");
232:    _;
233:  }
```

### [N-05] Missing require strings
Rquire statements should have descriptive strings to describe why the revert occurs.

```solidity
src/VaultImplementation.sol

78: require(msg.sender == owner());
96: require(msg.sender == owner());
105: require(msg.sender == owner());
114: require(msg.sender == owner());
147: require(msg.sender == owner());
191: require(msg.sender == address(ROUTER()));
211: require(msg.sender == owner());

src/LienToken.sol

505: require(msg.sender == address(s.COLLATERAL_TOKEN.getClearingHouse(collateralId)));
860: require(position < length);

src/AstariaRouter.sol

341: require(msg.sender == s.guardian);
347: require(msg.sender == s.guardian);
354: require(msg.sender == s.newGuardian);
361: require(msg.sender == address(s.guardian));

src/PublicVault.sol

241: require(s.allowList[receiver]);
259: require(s.allowList[receiver]);
508: require(msg.sender == owner());
680: require(msg.sender == address(LIEN_TOKEN()));

src/CollateralToken.sol

266: require(ownerOf(collateralId) == msg.sender);
535: require(msg.sender == s.clearingHouse[collateralId]);
564: require(ERC721(msg.sender).ownerOf(tokenId_) == address(this));

src/ClearingHouse.sol

72: require(msg.sender == address(ASTARIA_ROUTER.LIEN_TOKEN()));
199: require(msg.sender == address(ASTARIA_ROUTER.COLLATERAL_TOKEN()));
216: require(msg.sender == address(ASTARIA_ROUTER.COLLATERAL_TOKEN()));
223: require(msg.sender == address(ASTARIA_ROUTER.COLLATERAL_TOKEN()));

src/Vault.sol

65: require(s.allowList[msg.sender] && receiver == owner());
71: require(msg.sender == owner());
```

### [N-06] Unnecessary check in the function `incrementNonce`
The function `incrementNonce` is used by the owner to increment the nonce. 
An if statement occurs in the function to check if the msg.sender is the owner and if the msg.sender is delegate.
The problem is that the owner can just delegate himself with the function `setDelegate`, so there is no actually need
to check if the msg.sender is delegate in `incrementNonce`.

```solidity
64:  function incrementNonce() external {
65:    VIData storage s = _loadVISlot();
66:    if (msg.sender != owner() && msg.sender != s.delegate) {
67:      revert InvalidRequest(InvalidRequestReason.NO_AUTHORITY);
68:    }
69:    s.strategistNonce++;
70:    emit NonceUpdated(s.strategistNonce);
71:  }

210:  function setDelegate(address delegate_) external {
211:    require(msg.sender == owner()); //owner is "strategist"
212:    VIData storage s = _loadVISlot();
213:    s.delegate = delegate_;
214:    emit DelegateUpdated(delegate_);
215;    emit AllowListUpdated(delegate_, true);
216:  }
```

### [N-07] Function `_setPayee` should check if the newPayee is address(0) 
The internal function `_setPayee` is used to change the current payee to a new one.
Consider adding a check to ensure the newPayee isn't set as adress(0).

```solidity
src/LienToken.sol

911:  function _setPayee(
912:    LienStorage storage s,
913:    uint256 lienId,
914:    address newPayee
915:  ) internal {
916:    s.lienMeta[lienId].payee = newPayee;
917:    emit PayeeChanged(lienId, newPayee);
918:  }
```
Above instance can be changed to:

```solidity
911:  function _setPayee(
912:    LienStorage storage s,
913:    uint256 lienId,
914:    address newPayee
915:  ) internal {
916:    require(newPayee != address(0), "AddressZero");
917:    s.lienMeta[lienId].payee = newPayee;
918:    emit PayeeChanged(lienId, newPayee);
919:  }
```

### [N-08] The function should emit an event to notify users regarding some changes
The function `modifyDepositCap` is used by the owner to modify the deposit cap.
An event should be emitted to notify users, when the deposit cap is changed.

```solidity
src/VaultImplementation.sol

77:  function modifyDepositCap(uint256 newCap) external {
78:    require(msg.sender == owner()); //owner is "strategist"
79:    _loadVISlot().depositCap = newCap.safeCastTo88();
80:  }
```

### [R-01] Modifier should be used instead of require on owner functions
If functions are only allowed to be called by the owner, modifier should be used instead of checking 
with require statement, if owner is the msg.sender calling the function.

Example:
```solidity
77:  function modifyDepositCap(uint256 newCap) external {
78:    require(msg.sender == owner()); //owner is "strategist"
79:    _loadVISlot().depositCap = newCap.safeCastTo88();
80:  }
```
Modifier should be created only accessible by the owner and the instance above can be refactored in:

```solidity
77:  function modifyDepositCap(uint256 newCap) external onlyOwner {
78:    _loadVISlot().depositCap = newCap.safeCastTo88();
80:  }
```

Other instances:
```solidity
src/VaultImplementation.sol

64: function incrementNonce
95: function modifyAllowList
104: function disableAllowList
113: function enableAllowList
146: function shutdown
210: function setDelegate
```

### [R-02] Shorthand way to write if / else statement
The normal if / else statement can be refactored in a shorthand way to write it:
1. Increases readability
2. Shortens the overall SLOC.

```solidity
src/VaultImplementation.sol

366:  function recipient() public view returns (address) {
367:    if (IMPL_TYPE() == uint8(IAstariaRouter.ImplementationType.PublicVault)) {
368:      return address(this);
369:    } else {
370:      return owner();
371:    }
372:  }
```

The above instance can be refactored to:

```solidity
366:  function recipient() public view returns (address) {
367:    return IMPL_TYPE() == uint8(IAstariaRouter.ImplementationType.PublicVault) ? address(this) : owner();
368:  }
```

Other instances:

```solidity
src/PublicVault.sol

96: function minDepositAmount
```

### [R-03] Inputted value in the function should equals the safe cast value
In the function `modifyDepositCap` the inputted value of newCap is set as uint256, but it's later safe casted to uint88 value. 
Consider changing the value of newCap as an uint88 on calling time.

```solidity
src/VaultImplementation.sol

77:  function modifyDepositCap(uint256 newCap) external {
78:    require(msg.sender == owner()); //owner is "strategist"
79:    _loadVISlot().depositCap = newCap.safeCastTo88();
80:  }
```

Refactore the instance above to:

```solidity
77:  function modifyDepositCap(uint88 newCap) external {
78:    require(msg.sender == owner()); //owner is "strategist"
79:    _loadVISlot().depositCap = newCap.safeCastTo88();
80:  }
```

Other instance:

```solidity
src/PublicVault.sol

694: function _setYIntercept

src/WithdrawProxy.sol

302: function setWithdrawRatio
```

### [R-04] Emergancy pause/unpause function should emit an event to notify users
The functions __emergencyPause and __emergencyUnpause are used to freeze and un-freeze some of the core functions, when emergency occurs. Emitting an event in the function will notify users regarding this.

```solidity
src/AstariaRouter.sol

252:  function __emergencyPause() external requiresAuth whenNotPaused {
253:    _pause();
254:  }

259:  function __emergencyUnpause() external requiresAuth whenPaused {
260:    _unpause();
261:  }
```

### [R-05] Modifying an if statement in `_transferAndDepositAssetIfAble
The function `_transferAndDepositAssetIfAble` should revert if the owner of the token isn't the msg.sender.
An else statement should be added to revert in case this happens.

```solidity
src/AstariaRouter.sol

788:  function _transferAndDepositAssetIfAble(
789:    RouterStorage storage s,
790:    address tokenContract,
791:    uint256 tokenId
792:  ) internal {
793:    ERC721 token = ERC721(tokenContract);
794:    if (token.ownerOf(tokenId) == msg.sender) {
795:      token.safeTransferFrom(
796:        msg.sender,
797:       address(s.COLLATERAL_TOKEN),
798:        tokenId,
799:        ""
800:      );
801:    }
```

Refactore the code to:

```solidity
788:  function _transferAndDepositAssetIfAble(
789:    RouterStorage storage s,
790:    address tokenContract,
791:    uint256 tokenId
792:  ) internal {
793:    ERC721 token = ERC721(tokenContract);
794:    if (token.ownerOf(tokenId) == msg.sender) {
795:      token.safeTransferFrom(
796:        msg.sender,
797:       address(s.COLLATERAL_TOKEN),
798:        tokenId,
799:        ""
800:      ) else {
801:          revert NotTokenOwner;
802:      }
803:    }
```

### [R-06] Unnecessary return statement applied
Adding a return statement when the function defines a named return variable, is wrong.

```solidity
src/AstariaRouter.sol

712:  function _newVault(
713:    RouterStorage storage s,
714:    address underlying,
715:    uint256 epochLength,
716:    address delegate,
717:    uint256 vaultFee,
718:    bool allowListEnabled,
719:    address[] memory allowList,
720:    uint256 depositCap
721:  ) internal returns (address vaultAddr) {

758: return vaultAddr;
```

### [R-07] Use delete to clear variables instead of zero assignment
You can use the delete keyword instead of setting the variable as zero.

Instances:
```solidity
src/PublicVault.sol

308: s.liquidationWithdrawRatio = 0;
327: s.withdrawReserve = 0;
377: s.withdrawReserve = 0;
511: s.strategistUnclaimedShares = 0;

src/WithdrawProxy.sol

284: s.finalAuctionEnd = 0;
```

### [R-08] Some number values can be refactored with _
Consider using underscores for number values to improve readability.

```solidity
src/AstariaRouter.sol

118: s.buyoutFeeDenominator = uint32(1000);
```

The above instance can be refactored to:

```solidity
118: s.buyoutFeeDenominator = uint32(1_000);
```

### [O-01] Import not used anywhere in the contract
File imported is not used anywhere in the contract, if unused consider removing it.

```solidity
src/VaultImplementation.sol

25: import {IPublicVault} from "core/interfaces/IPublicVault.sol";

src/LienToken.sol

34: import {VaultImplementation} from "./VaultImplementation.sol";
38: import {Initializable} from "./utils/Initializable.sol";

src/CollateralToken.sol

33: import {VaultImplementation} from "core/VaultImplementation.sol";
```

### [O-02] Events is missing indexed fields
Index event fields make the field more quickly accessible to off-chain.
Each event should use three indexed fields if there are three or more fields.

Instances in:

```solidity
src/interfaces/IAstariaRouter.sol

294: event Liquidation
295: event NewVault

src/interfaces/IPublicVault.sol

168: event StrategistFee
169:  event LiensOpenForEpochRemaining
170:  event YInterceptChanged
171:  event WithdrawReserveTransferred
172:  event LienOpen
```

### [O-03] Proper use of get as a function name prefix
Clear function names can increase readability. Follow a standard convertion function names such as using get for getter (view/pure) functions.

Instances:

```solidity
src/VaultImplementation.sol

152: function domainSeparator
170: function encodeStrategyData
366: function recipient

src/LienToken.sol

348: function tokenURI
```

### [O-04] Function Naming suggestions
Proper use of _ as a function name prefix and a common pattern is to prefix internal and private function names with _.
This pattern is correctly applied in the Party contracts, however there are some inconsistencies in the libraries.

Instances:
```solidity
src/PublicVault.sol

271: function computeDomainSeparator
575: function afterDeposit
```

### [O-05] NatSpec comments missing on some functions
Some functions don't have description of what they are supposed to do.

Instances:

```solidity
src/VaultImplementation.sol

146: function shutdown
210: function setDelegate
397: function _handleProtocolFee

src/LienToken.sol

277: function stopLiens
424: function _createLien
459: function _appendStack
497: function payDebtViaClearingHouse
512: function _payDebt
608: function makePayment
623: function _paymentAH

// This are some of the instances, there are more.
```

### [O-06] Code contains empty blocks
There are some empty blocks, which are unused. 
The code should do smth or at least have a description why is structured that way.

Instances:
```solidity
src/VaultImplementation.sol

268: function _afterCommitToLien(
269:    uint40 end,
270:    uint256 lienId,
271:    uint256 slope
272:  ) internal virtual {}

274: function _beforeCommitToLien(IAstariaRouter.Commitment calldata)
275:    internal
276:    virtual
277:  {}
```

Other instance:

```solidity
src/ClearingHouse.sol
```

### [O-07] Imported names on Interfaces should start with "I"
It is best practise that interfaces name start with "I", consider applying it in the imports as well.

```solidity
src/CollateralToken.sol

34: import {ZoneInterface} from "seaport/interfaces/ZoneInterface.sol";
40: import {ConduitControllerInterface} from "seaport/interfaces/ConduitControllerInterface.sol";
43: import {ConsiderationInterface} from "seaport/interfaces/ConsiderationInterface.sol";

src/ClearingHouse.sol

24: import {ConduitControllerInterface} from "seaport/interfaces/ConduitControllerInterface.sol";
```

### [O-08] Hardcoded values can't be changed
Ones set hardcoded values can't be changed later, consider this as a reminder.

```solidity
src/AstariaRouter.sol

66: uint256 private constant OUTOFBOUND_ERROR_SELECTOR =
67:    0x571e08d100000000000000000000000000000000000000000000000000000000;
68:  uint256 private constant ONE_WORD = 0x20;
```
