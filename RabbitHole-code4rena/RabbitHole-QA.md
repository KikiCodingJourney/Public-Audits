### Issues Template
| Letter | Name | Description |
|:--:|:-------:|:-------:|
| L  | Low risk | Potential risk |
| NC |  Non-critical | Non risky findings |
| R  | Refactor | Changing the code |
| O | Ordinary | Often found issues |

| Total Found Issues | 20 |
|:--:|:--:|

### Low Risk Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [L-01] | `createQuest` doesn't check if the reward token address is in the allow list on ERC-1155 quest type | 1 |
| [L-02] | The function `mintReceipt` should check if the quest has expired on-chain as well | 1 |
| [L-03] | The reverting functions `_calculateRewards` and `_transferRewards` should be removed, as they are already implemented in the child contracts | 1 |
| [L-04] | The function `withdrawRemainingTokens` can be changed in a safer way to handle the withdraw from the owner and the protocol fee as well. This prevent risks allocated with the protocol fee | 1 |
| [L-05] | The function `royaltyInfo` doesn't check if the receipt was already claimed | 1 |
| [L-06] | In contract Quest the function `claim` shouldn't only set the receipt as claimed, but to burn it as well. As this problem brings the risk, where users can sell already claimed receipts to other people | 1 |
| [L-07] | The function `mintReceipt` shouldn't mint receipts to users, if the quest is paused | 1 |

| Total Low Risk Issues | 7 |
|:--:|:--:|

### Non-Critical Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [N-01] | Confusing modifier name | 1 |
| [N-02] | Deploying a storage variable with its default value | 1 |
| [N-03] | Modifiers not applied on the functions `start` and `withdrawRemainingTokens` | 2 |
| [N-04] | Mandatory checks for extra safety in the setters | 3 |
| [N-05] | Lack of address(0) checks in the constructor | 1 |
| [N-06] | Upgradeable contract is missing a __gap[50] storage variable | 2 |


| Total Non-Critical Issues | 6 |
|:--:|:--:|

### Refactor Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [R-01] | Shorthand way to write if / else statement | 1 |
| [R-02] | Unnecessary true statement is applied in the function `isClaimed` | 1 |
| [R-03] | `isPaused` check can be added to the modifier `onlyQuestActive`, as its used only on the claim function | 1 |
| [R-04] | Total minted check in `mintReceipt` can be refactored | 1 |

| Total Refactor Issues | 4 |
|:--:|:--:|

### Ordinary Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [O-01] | Floating pragma | 8 |
| [O-02] | Code contains empty blocks | 1 |
| [O-03] | Create your own import names instead of using the regular ones | 4 |


| Total Ordinary Issues | 3 |
|:--:|:--:|

### [L-01] `createQuest` doesn't check if the reward token address is in the allow list on ERC-1155 quest type.
The function `createQuest` is called by users with the quest role. The main purpose of the function is to create quests, which can be either erc20 or erc1155 type. When the type is erc20, a check is made to ensure the rewardTokenAddress_ is allowed to be used as a reward - `if (rewardAllowlist[rewardTokenAddress_] == false) revert RewardNotAllowed();`. The problem is that the same check isn't made when the quest is erc1155, as a result when erc1155 quest is created the function createQuest doesn't check if the rewardTokenAddress_ is in the allow list.

```solidity
contracts/QuestFactory.sol

61:  function createQuest(
62:        address rewardTokenAddress_,
63:        uint256 endTime_,
64:        uint256 startTime_,
65:        uint256 totalParticipants_,
66:        uint256 rewardAmountOrTokenId_,
67:        string memory contractType_,
68:        string memory questId_
69:    ) public onlyRole(CREATE_QUEST_ROLE) returns (address) {
70:        if (quests[questId_].questAddress != address(0)) revert QuestIdUsed();
71:
72:        if (keccak256(abi.encodePacked(contractType_)) == keccak256(abi.encodePacked('erc20'))) {
73:            if (rewardAllowlist[rewardTokenAddress_] == false) revert RewardNotAllowed();
74:
75:            Erc20Quest newQuest = new Erc20Quest(
76:                rewardTokenAddress_,
77:                endTime_,
78:                startTime_,
79:                totalParticipants_,
80:                rewardAmountOrTokenId_,
81:                questId_,
82:                address(rabbitholeReceiptContract),
83:                questFee,
84:                protocolFeeRecipient
85:            );
86:
87:            emit QuestCreated(
88:                msg.sender,
89:                address(newQuest),
90:                questId_,
91:                contractType_,
92:                rewardTokenAddress_,
93:                endTime_,
94:                startTime_,
95:                totalParticipants_,
96:                rewardAmountOrTokenId_
97:            );
98:            quests[questId_].questAddress = address(newQuest);
99:            quests[questId_].totalParticipants = totalParticipants_;
100:           newQuest.transferOwnership(msg.sender);
101:           ++questIdCount;
102:           return address(newQuest);
103:        }
104:
105:        if (keccak256(abi.encodePacked(contractType_)) == keccak256(abi.encodePacked('erc1155'))) {
106:            if (msg.sender != owner()) revert OnlyOwnerCanCreate1155Quest();
107:
108:            Erc1155Quest newQuest = new Erc1155Quest(
109:                rewardTokenAddress_,
110:                endTime_,
111:                startTime_,
112:                totalParticipants_,
113:                rewardAmountOrTokenId_,
114:                questId_,
115:                address(rabbitholeReceiptContract)
116:            );
117:
118:            emit QuestCreated(
119:                msg.sender,
120:                address(newQuest),
121:                questId_,
122:                contractType_,
123:                rewardTokenAddress_,
124:                endTime_,
125:                startTime_,
126:                totalParticipants_,
127:                rewardAmountOrTokenId_
128:            );
129:            quests[questId_].questAddress = address(newQuest);
130:            quests[questId_].totalParticipants = totalParticipants_;
131:            newQuest.transferOwnership(msg.sender);
132:            ++questIdCount;
133:            return address(newQuest);
134:        }
135:
136:        revert QuestTypeInvalid();
137:    }
```

Consider adding a check to ensure the contract address is allowed to be used as a reward on erc1155 quests as well:

```solidity
105:        if (keccak256(abi.encodePacked(contractType_)) == keccak256(abi.encodePacked('erc1155'))) {
106:            if (msg.sender != owner()) revert OnlyOwnerCanCreate1155Quest();
+               if (rewardAllowlist[rewardTokenAddress_] == false) revert RewardNotAllowed();
```

### [L-02] The function mintReceipt should check if the quest has expired on-chain as well
The main function mintReceipt responsible for minting receipts lacks an important check to ensure the quest end time hasn't finished yet. Considering the fact that on quest creation every quest is enforced with a startTime and endTime, which represents the quest starting time and ending time. Users should not be allowed to mint receipts after the quest is expired.

By the sponsor comment, the `claimSignerAddress` takes care of that on the off-chain side and won't issue hashes before the quest start or after the quest ends. But mistakes always can occur and it is recommended to have a check on the smart contract level as well.

```solidity
contracts/QuestFactory.sol

219:  function mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_) public {
220:        if (quests[questId_].numberMinted + 1 > quests[questId_].totalParticipants) revert OverMaxAllowedToMint();
221:        if (quests[questId_].addressMinted[msg.sender] == true) revert AddressAlreadyMinted();
222:        if (keccak256(abi.encodePacked(msg.sender, questId_)) != hash_) revert InvalidHash();
223:        if (recoverSigner(hash_, signature_) != claimSignerAddress) revert AddressNotSigned();
224:
225:        quests[questId_].addressMinted[msg.sender] = true;
226:        quests[questId_].numberMinted++;
227:        emit ReceiptMinted(msg.sender, questId_);
228:        rabbitholeReceiptContract.mint(msg.sender, questId_);
229:    }
```

Here is a recommended change, which takes care of this problem:

1. Add a storage variable in the struct `Quest`, which will hold the end time of the quest.

```solidity
struct Quest {
        mapping(address => bool) addressMinted;
        address questAddress;
        uint totalParticipants;
        uint numberMinted;
+       uint256 expires;
    }
```

2. When creating a quest with the function `createQuest` consider adding the endTime to the new stor variable `expires`.

```solidity
// Add the same check if contractType is erc1155 as well.

if (keccak256(abi.encodePacked(contractType_)) == keccak256(abi.encodePacked('erc20'))) {
            if (rewardAllowlist[rewardTokenAddress_] == false) revert RewardNotAllowed();

            Erc20Quest newQuest = new Erc20Quest(
                rewardTokenAddress_,
                endTime_,
                startTime_,
                totalParticipants_,
                rewardAmountOrTokenId_,
                questId_,
                address(rabbitholeReceiptContract),
                questFee,
                protocolFeeRecipient
            );

            emit QuestCreated(
                msg.sender,
                address(newQuest),
                questId_,
                contractType_,
                rewardTokenAddress_,
                endTime_,
                startTime_,
                totalParticipants_,
                rewardAmountOrTokenId_
            );
            quests[questId_].questAddress = address(newQuest);
            quests[questId_].totalParticipants = totalParticipants_;
+           quests[questId_].expires = endTime_;
            newQuest.transferOwnership(msg.sender);
            ++questIdCount;
            return address(newQuest);
        }
```

3. And finally add a check in the function `mintReceipt` to check if the quest expired already.

```solidity
function mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_) public {
+       if (quests[questId_].expires > block.timestamp) revert QuestAlreadyExpired();
        if (quests[questId_].numberMinted + 1 > quests[questId_].totalParticipants) revert OverMaxAllowedToMint();
        if (quests[questId_].addressMinted[msg.sender] == true) revert AddressAlreadyMinted();
        if (keccak256(abi.encodePacked(msg.sender, questId_)) != hash_) revert InvalidHash();
        if (recoverSigner(hash_, signature_) != claimSignerAddress) revert AddressNotSigned();

        quests[questId_].addressMinted[msg.sender] = true;
        quests[questId_].numberMinted++;
        emit ReceiptMinted(msg.sender, questId_);
        rabbitholeReceiptContract.mint(msg.sender, questId_);
    }
```

### [L-03] The reverting functions `_calculateRewards` and `_transferRewards` should be removed, as they are already implemented in the child contract
There are two functions in Quest.sol, which reverts incase they are called. By the revert names, we can understand that this two functions need to be implemented in the child contracts - Erc20Quest.sol, Erc1155Quest.sol. Since this is already done and they are implemented in the child contracts, this two functions are unnecessary and should be removed.

```solidity
contracts/Quest.sol

122:  function _calculateRewards(uint256 redeemableTokenCount_) internal virtual returns (uint256) {
123:        revert MustImplementInChild();
124:    }

129:  function _transferRewards(uint256 amount_) internal virtual {
130:        revert MustImplementInChild();
131:    }
```

### [L-04] The function `withdrawRemainingTokens can be changed in a safer way to handle the withdraw from the owner and the protocol fee as well. This prevent risks allocated with the protocol fees.
By the docs this function is called in two different scenarios, if a quest is full and receipt redeemers equals the max amount of total participants allowed in the quest - only withdrawFee is called. If a quest doesn't hit the max total participants, first the owner calls the function `withdrawRemainingTokens` to withdraw the remaining tokens and then the fee should be paid with the function `withdrawFee`.

Overall the best solution of this problem is that the function `withdrawRemainingTokens`, both does the withdrawing part to the owner and pays the fee to the protocol as well. This is considered the safest way:

First - if the receipt redeemers are below the totalParticipants, can withdraw the remaining tokens and pay the fee at the same time, second if the quest is full and receipt redemeers hits the total amount of people allowed, only the fee will be paid to the protocol and will skip the withdraw remaining rewards part.

```solidity
function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);

        if (receiptRedeemers() < totalParticipants) {

        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);

        IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());

        } else {

        IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());

        }
    }
```

### [L-05] The function `royaltyInfo` doesn't check if the receipt was already claimed
The function `royaltyInfo` is used by users to check sale details regarding a particular ERC721 token.
The problem here is that the function check if the token exists, but doesn't check if the token was already claimed.
Consider applying a check, which will revert if the token was already claimed.

```solidity
contracts/RabbitHoleReceipt.sol

178:  function royaltyInfo(
179:        uint256 tokenId_,
180:        uint256 salePrice_
181:    ) external view override returns (address receiver, uint256 royaltyAmount) {
182:        require(_exists(tokenId_), 'Nonexistent token');
183:
184:        uint256 royaltyPayment = (salePrice_ * royaltyFee) / 10_000;
185:        return (royaltyRecipient, royaltyPayment);
186:    }
```

### [L-06] In contract Quest the function `claim` shouldn't only set the receipt as claimed, but to burn it as well. As this problem brings the risk, where users can sell already claimed receipts to other people
The function `claim` is used by users to claim their ERC721 receipts for rewards. By using the function the receipt is set as claimed with a simple mapping id => bool, but it isn't burned. In the protocol docs it is clearly stated that users are free to sell or trade their receipts. Since the claimed receipts aren't burned, this bring the risk where already claimed receipts can be sold to other people. A burn function already exists in RabbitHoleReceipt, but isn't used.

### [L-07] The function `mintReceipt` shouldn't mint receipts to users, if the quest is paused
For now the function `mintReceipt` doesn't issue hashes before the quest has started or after the quest has ended. 
This is done off-chain with the help of `claimSignerAddress`, but the off-chain side doesn't check if a quest is in paused state.
So even if a quest is in paused state duo to some sort of issue occurring, the function `mintReceipt` can still mint receipts for this particular quest.

```solidity
contracts/QuestFactory.sol

219:  function mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_) public {
220:        if (quests[questId_].numberMinted + 1 > quests[questId_].totalParticipants) revert OverMaxAllowedToMint();
221:        if (quests[questId_].addressMinted[msg.sender] == true) revert AddressAlreadyMinted();
222:        if (keccak256(abi.encodePacked(msg.sender, questId_)) != hash_) revert InvalidHash();
223:        if (recoverSigner(hash_, signature_) != claimSignerAddress) revert AddressNotSigned();
224:
225:        quests[questId_].addressMinted[msg.sender] = true;
226:        quests[questId_].numberMinted++;
227:        emit ReceiptMinted(msg.sender, questId_);
228:        rabbitholeReceiptContract.mint(msg.sender, questId_);
229:    }
```

A recommended change l thought of:

1. Create a private mapping, which will check if the quest address is paused

```solidity
mapping(string => bool) private isPaused;
```

2. Create an owner function, so the owner can change the state of the mapping. 

```solidity
function setQuestState(string questId_, bool _paused) public onlyOwner {
        isPaused[questId_] = _paused;
    }
```

3. Apply the check in `mintReceipt`, so users won't be able claim receipts, when the quest is paused.

```solidity
function mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_) public {
+       if (isPaused[questId_] == true) revert QuestPaused();
        if (quests[questId_].numberMinted + 1 > quests[questId_].totalParticipants) revert OverMaxAllowedToMint();
        if (quests[questId_].addressMinted[msg.sender] == true) revert AddressAlreadyMinted();
        if (keccak256(abi.encodePacked(msg.sender, questId_)) != hash_) revert InvalidHash();
        if (recoverSigner(hash_, signature_) != claimSignerAddress) revert AddressNotSigned();

        quests[questId_].addressMinted[msg.sender] = true;
        quests[questId_].numberMinted++;
        emit ReceiptMinted(msg.sender, questId_);
        rabbitholeReceiptContract.mint(msg.sender, questId_);
    }
}
```


### [N-01] Confusing modifier name 
A confusing name is set on the modifier `onlyAdminWithdrawAfterEnd`. By its name it says only admin withdraw after end time, but at the same time the modifier only check if `block.timestamp < endTime`.

```solidity
contracts/Quest.sol

76:  modifier onlyAdminWithdrawAfterEnd() {
77:        if (block.timestamp < endTime) revert NoWithdrawDuringClaim();
78:        _;
79:    }
```

### [N-02] Deploying a storage variable with its default value
At a deploying time in the contract Quest, the storage variable `redeemedTokens` is set as zero, even tho its default value is already zero.

```solidity
contracts/Quest.sol

26:  constructor(
27:        address rewardTokenAddress_,
28:        uint256 endTime_,
29:        uint256 startTime_,
30:        uint256 totalParticipants_,
31:        uint256 rewardAmountInWeiOrTokenId_,
32:        string memory questId_,
33:        address receiptContractAddress_
34:    ) {
35:        if (endTime_ <= block.timestamp) revert EndTimeInPast();
36:        if (startTime_ <= block.timestamp) revert StartTimeInPast();
37:        if (endTime_ <= startTime_) revert EndTimeLessThanOrEqualToStartTime();
38:        endTime = endTime_;
39:        startTime = startTime_;
40:        rewardToken = rewardTokenAddress_;
41:        totalParticipants = totalParticipants_;
42:        rewardAmountInWeiOrTokenId = rewardAmountInWeiOrTokenId_;
43:        questId = questId_;
44:        rabbitHoleReceiptContract = RabbitHoleReceipt(receiptContractAddress_);
45:        redeemedTokens = 0;
46    }
```

### [N-03] Modifiers not applied on the functions `start` and `withdrawRemainingTokens`
First with the function `start`, a quest should be started only by the owner, even tho the modifier is applied on the function `start` in Quest.sol. It should be added to the child contract as well.

```solidity
contracts/Erc20Quest.sol

58:  function start() public override {
59:        if (IERC20(rewardToken).balanceOf(address(this)) < maxTotalRewards() + maxProtocolReward())
60:            revert TotalAmountExceedsBalance();
61:        super.start();
62:    }
```

Consider adding the onlyOwner modifier to the function above:

```solidity
function start() public override onlyOwner {
        if (IERC20(rewardToken).balanceOf(address(this)) < maxTotalRewards() + maxProtocolReward())
            revert TotalAmountExceedsBalance();
        super.start();
    }
```

Same goes for the function `withdrawRemainingTokens`, an onlyOwner modifier is applied but the modifier which check if the quest ended is not. 

```solidity
contracts/Erc20Quest.sol

81:  function withdrawRemainingTokens(address to_) public override onlyOwner {
82:        super.withdrawRemainingTokens(to_);
83:
84:        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
85:        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
86:        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
87:    }
```

Consider adding the modifier `onlyAdminWithdrawAfterEnd` to the child contract as well:

```solidity
function withdrawRemainingTokens(address to_) public override onlyOwner onlyAdminWithdrawAfterEnd {
        super.withdrawRemainingTokens(to_);

        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
    }
```

### [N-04] Mandatory checks for extra safety in the setters
In the folowing functions below, there are some checks that can be made in order to achieve more safe and efficient code.

Address zero check can be added in the functions `setClaimSignerAddress`, `setRabbitHoleReceiptContract` to ensure the new addresses aren't address(0).

```solidity
contracts/QuestFactory.sol

159:  function setClaimSignerAddress(address claimSignerAddress_) public onlyOwner {
160:        claimSignerAddress = claimSignerAddress_;
161:    }

172:  function setRabbitHoleReceiptContract(address rabbitholeReceiptContract_) public onlyOwner {
173:        rabbitholeReceiptContract = RabbitHoleReceipt(rabbitholeReceiptContract_);
174:    }
```

In the function `setQuestFee` a check can be made to ensure the fee is set as non-zero.

```solidity
contracts/QuestFactory.sol

186:  function setQuestFee(uint256 questFee_) public onlyOwner {
187:        if (questFee_ > 10_000) revert QuestFeeTooHigh();
188:        questFee = questFee_;
189:    }
```

### [N-05] Lack of address(0) checks in the constructor
Zero-address check should be used in the constructors, to avoid the risk of setting smth as address(0) at deploying time.

```solidity
contracts/Quest.sol

26: constructor
```

### [N-06] Upgradeable contract is missing a __gap[50] storage variable
Reference: [Storage_gaps](https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps)

You may notice that every contract includes a state variable named __gap. This is empty reserved space in storage that is put in place in Upgradeable contracts. It allows us to freely add new state variables in the future without compromising the storage compatibility with existing deployments.

Instances:

```solidity
contracts/QuestFactory.sol

16: contract QuestFactory is Initializable, OwnableUpgradeable, AccessControlUpgradeable, IQuestFactory {

contracts/RabbitHoleReceipt.sol

15: contract RabbitHoleReceipt is
```

### [R-01] Shorthand way to write if / else statement
The normal if / else statement can be refactored in a shorthand way to write it:

1. Increases readability
2. Shortens the overall SLOC.

```solidity
contracts/QuestFactory.sol

142:  function changeCreateQuestRole(address account_, bool canCreateQuest_) public onlyOwner {
143:        if (canCreateQuest_) {
144:            _grantRole(CREATE_QUEST_ROLE, account_);
145:        } else {
146:            _revokeRole(CREATE_QUEST_ROLE, account_);
147:        }
148:    }
```

The above instance can be refactored in:

```solidity
function changeCreateQuestRole(address account_, bool canCreateQuest_) public onlyOwner {
        canCreateQuest_ ? _grantRole(CREATE_QUEST_ROLE, account_); : _revokeRole(CREATE_QUEST_ROLE, account_);
    }
```

### [R-02] Unnecessary true statement is applied in the function `isClaimed`
The function `isClaimed` checks if the given token id is claimed, for some reason there is an unnecessary true statement applied, which doesn't do anything.

```solidity
contracts/Quest.sol

135:  function isClaimed(uint256 tokenId_) public view returns (bool) {
136:        return claimedList[tokenId_] == true;
137:    }
```

Consider changing the above instance to:
```solidity

function isClaimed(uint256 tokenId_) public view returns (bool) {
        return claimedList[tokenId_];
   }
```

### [R-03] `isPaused` check can be added to the modifier `onlyQuestActive`, as its used only on the claim function
In the `claim` function a check is made to ensure the quest isn't paused. Considering the fact that the modifier `onlyQuestActive` is only used once and its on this particular function. The check can be refactored in the modifier instead of applying it in the function.

```solidity
contracts/Quest.sol

96:  function claim() public virtual onlyQuestActive {
97:        if (isPaused) revert QuestPaused();

88:  modifier onlyQuestActive() {
89:        if (!hasStarted) revert NotStarted();
90:        if (block.timestamp < startTime) revert ClaimWindowNotStarted();
91:        _;
92:    }
```

Refactor the modifier `onlyQuestActive` and remove the check from the function claim:

```solidity
modifier onlyQuestActive() {
        if (!hasStarted) revert NotStarted();
        if (block.timestamp < startTime) revert ClaimWindowNotStarted();
        if (isPaused) revert QuestPaused();
        _;
    }
```

### [R-04] Total minted check in `mintReceipt` can be refactored
In the function `mintReceipt` a check is made to see if the amount of already minted receipts doesn't exceed the amount of total participants allowed and the following if statement is used below. Instead of adding 1 to the total amount of minted receipts, the if statement can just be changed to `>=`, as it does the same thing.
`if (quests[questId_].numberMinted + 1 > quests[questId_].totalParticipants)` 

### [O-01] Floating pragma

Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

Instances:
```solidity
contracts/QuestFactory.sol
contracts/RabbitHoleReceipt.sol
contracts/Quest.sol
contracts/RabbitHoleTickets.sol
contracts/Erc20Quest.sol
contracts/Erc1155Quest.sol
contracts/ReceiptRenderer.sol
contracts/TicketRenderer.sol
```

### [O-02] Code contains empty blocks

There are some empty blocks, which are unused.
The code should do smth or at least have a description why is structured that way.

Instances:
```solidity
contracts/QuestFactory.sol

35: constructor() initializer {}
```

### [O-03] Create your own import names instead of using the regular ones
For better readability, you should name the imports instead of using the regular ones.

Instances:
```solidity
contracts/RabbitHoleReceipt.sol
contracts/RabbitHoleTickets.sol
contracts/ReceiptRenderer.sol
contracts/TicketRenderer.sol
```
