### Issues Template
| Letter | Name | Description |
|:--:|:-------:|:-------:|
| H  | High risk | Loss of funds |
| M |  Medium risk | Unexpected behavior |
| L  | Low risk | Potentially a risk |

| Total Found Issues | 4 |
|:--:|:--:|

### Findings template
| Count | Explanation | Risk |
|:--:|:-------|:--:|
| H-01 | Malicious user can send the quest reward tokens to the protocol fee contract preventing users from claiming their rewards. | High |
| M-01 | In ERC1155 quests the owner withdraws all of the remaining tokens even for the unclaimed receipts. Leaving users who didn't claim their receipts before the quest end time unable to claim rewards. | Medium |
| M-02 | Incase a malicious attack occurs and the quest is paused, the owner won't be able to withdraw some of his tokens back. | Medium |
| M-03 | In contract Quest the function `claim` shouldn't only set the receipt as claimed, but to burn it as well. As this problem brings the risk, where users can sell already claimed receipts to other people | Medium |
| M-04 | The function `mintReceipt` should check if the quest has expired on-chain as well | Medium |

### [H-01] Malicious user can send the quest reward tokens to the protocol fee contract preventing users from claiming their rewards.

## Impact
Malicious user can take advantage of the function `withdrawFee` after the quest end time and successfuly send the quest reward tokens to the protocol fee contract preventing users from claiming their rewards.

## Proof of Concept
Every receipt minted should still be able to claim rewards with the function `claim` even after the end time of the quest.

`The below comment is related to the function withdrawRemainingTokens, where the owner withdraws the remaining tokens.`

`The owner can't withdraw the tokens for the unclaimed minted receipts, so they can be claimed even after the end time.`

```solidity
contracts/Erc20Quest.sol
       
79: /// @dev Every receipt minted should still be able to claim rewards (and cannot be withdrawn).
```

```solidity
contracts/Quest.sol

88:  modifier onlyQuestActive() {
89:        if (!hasStarted) revert NotStarted();
90:        if (block.timestamp < startTime) revert ClaimWindowNotStarted();
91:        _;
92:    }
```

```solidity
contracts/Quest.sol

96:  function claim() public virtual onlyQuestActive {
97:        if (isPaused) revert QuestPaused();
98:
99:        uint[] memory tokens = rabbitHoleReceiptContract.getOwnedTokenIdsOfQuest(questId, msg.sender);
100:
101:       if (tokens.length == 0) revert NoTokensToClaim();
102:
103:       uint256 redeemableTokenCount = 0;
104:       for (uint i = 0; i < tokens.length; i++) {
105:           if (!isClaimed(tokens[i])) {
106:               redeemableTokenCount++;
107:           }
108:       }
109:
110:       if (redeemableTokenCount == 0) revert AlreadyClaimed();
111:
112:       uint256 totalRedeemableRewards = _calculateRewards(redeemableTokenCount);
113:       _setClaimed(tokens);
114:       _transferRewards(totalRedeemableRewards);
115:       redeemedTokens += redeemableTokenCount;
116:
117:       emit Claimed(msg.sender, totalRedeemableRewards);
118:   }
```

The problem here occurs in the function `withdrawFee`, this function is made to be called only after the quest end time. The modifier `onlyAdminWithdrawAfterEnd` is used on it, which reverts if block.timestamp is before the end time. This leads to the main scenario where users who didn't claim their rewards before the end time of the quest can be exploited by malicious users. 

Lets say we have 3 receipt redeemers - Jake, Kiki and Finn:
1. Kiki claimed his rewards before the quest end time, but Jake and Finn didn't.
2. Malicious user can call the function `withdrawFee` numerous number of times as there is no check to see if the protocol fee was already sent or if the msg.sender calling the function is the owner. 
3. As a result the malicious user can send the contract rewards allocated for Jake and Finn to the protocol fee recipient. 

This scenario leaves receipt redeemers like Finn and Jake unable to claim their rewards.

Duo to this problem, malicious users can sabotage the people who didn't claim their receipts before the quest end time,     
by calling the function `withdrawFee` numerous number of times and successfuly sending the quest contract balance to the protocol fee recipient. As a receipt can be claimed only on its allocated quest contract, this scenario leaves users unable to claim their rewards.


```solidity
contracts/Erc20Quest.sol

102:  function withdrawFee() public onlyAdminWithdrawAfterEnd {
103:      IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
104:  }
```
```solidity
contracts/Quest.sol

76:  modifier onlyAdminWithdrawAfterEnd() {
77:        if (block.timestamp < endTime) revert NoWithdrawDuringClaim();
78:        _;
79:    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider applying the onlyOwner modifier on the function `withdrawFee`, so the function can be called only by the owner.
```solidity
102:  function withdrawFee() public onlyOwner onlyAdminWithdrawAfterEnd {
103:      IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
104:  }
```

### [M-01] In ERC1155 quests the owner withdraws all of the remaining tokens even for the unclaimed receipts. Leaving users who didn't claim their receipts before the quest end time unable to claim rewards.

## Impact
In ERC1155 quests the owner is able to withdraw all of the remaining tokens even for the unclaimed receipts.
This problem prevents users from claiming their rewards, as a receipt can be claimed only on its allocated quest.

## Proof of Concept
In ERC1155 quests, the function `withdrawRemainingTokens` is called by the owner to withdraw the remaining tokens.
The problem here is that the owner withdraws all of the ERC1155 tokens even the unclaimed ones, which are allocated to the users that finished the quest. As there is no check to see how many receipts were minted for this particular quest and how many receipts were claimed. The owner is able to withdraw all of the remaining tokens, preventing users from claiming their rewards as a minted receipt can be claimed only on its allocated quest contract.

Example:
We have 4 people - Jake, Finn, Alice and Kiki
1. All of the people above finished the quest and received minted receipts.
2. But only Jake and Finn claimed their receipts before the quest end time.
3. The owner calls the function `withdrawRemainingTokens` and withdraws all of the tokens, as there is no check to see how many receipts were minted and how many were actually claimed.
4. Leaving both Alice and Kiki unable to claim rewards, as their receipts can be used only on this quest contract.

This problem prevents users from claiming their rewards, as the owner isn't supposed to withdraw the tokens allocated for the unclaimed receipts after the quest end time.

```solidity
contracts/Erc1155Quest.sol

54:  function withdrawRemainingTokens(address to_) public override onlyOwner {
55:        super.withdrawRemainingTokens(to_);
56:        IERC1155(rewardToken).safeTransferFrom(
57:            address(this),
58:            to_,
59:            rewardAmountInWeiOrTokenId,
60:            IERC1155(rewardToken).balanceOf(address(this), rewardAmountInWeiOrTokenId),
61:            '0x00'
62:        );
63:    }
```


You can see that this is enforced in the ERC20 quests and the tokens for the unclaimed receipts can't be withdrawn by the owner at the end of the quest. This is clearly stated as well in the function comment on L79 - minted receipts should still be able to claim rewards even after the quest end time and can't be withdrawn by the owner. 

```solidity
contracts/Erc20Quest.sol

79:  /// @dev Every receipt minted should still be able to claim rewards (and cannot be withdrawn).

81:  function withdrawRemainingTokens(address to_) public override onlyOwner {
82:        super.withdrawRemainingTokens(to_);
83:
84:        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
85:        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
86:        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
87:    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider adding the function `receiptRedeemers` to Erc1155Quest.sol, which checks how many people actually finished the quest and got minted receipts. And refactor the function `withdrawRemainingTokens` to withdraw the ERC1155 tokens without the unclaimed ones:

```solidity
contracts/Erc1155Quest.sol

+  function receiptRedeemers() public view returns (uint256) {
+         return questFactoryContract.getNumberMinted(questId);
+     }

54:  function withdrawRemainingTokens(address to_) public override onlyOwner {
55:        super.withdrawRemainingTokens(to_);
+          uint256 unclaimedTokens = receiptRedeemers() - redeemedTokens;

56:        IERC1155(rewardToken).safeTransferFrom(
57:            address(this),
58:            to_,
59:            rewardAmountInWeiOrTokenId,
60:            IERC1155(rewardToken).balanceOf(address(this), rewardAmountInWeiOrTokenId) - unclaimedTokens,
61:            '0x00'
62:        );
63:    }
```

### [M-02] Incase a malicious attack occurs and the quest is paused, the owner won't be able to withdraw some of his tokens back.

## Impact
Incase a malicious attack occurs and the quest is paused, the owner won't be able to withdraw some of his tokens back, as a result the tokens will be stuck in the quest contract.

## Proof of Concept
By asking the sponsor - a quest can be in paused state as an emergency situation (malicious attack, some issue on their 
off-chain signature etc). When a quest is paused by the owner, the function `claim` is uncallable and reverts, duo to that users are unable to claim their receipts and receive rewards. 

Lets say a malicious attack occurs and the owner successfuly pauses the quest contract, so the attackers can't claim the receipts.
Basically this stops the malicious attack, but doesn't fix the main problem as at the end of the quest the owner won't be able to withdraw the remaining tokens for the minted receipts the attackers posses. Yes the attackers won't be able to claim their receipts as the quest contract is paused, but neither the owner will be able to withdraw the tokens back, as a result the tokens will be stuck in the contract.

The owner should have a way to withdraw the tokens allocated for the unclaimed receipts incase of malicious attack.

```solidity
contracts/Erc20Quest.sol

79: /// @dev Every receipt minted should still be able to claim rewards (and cannot be withdrawn).

81:  function withdrawRemainingTokens(address to_) public override onlyOwner {
82:        super.withdrawRemainingTokens(to_);
83:
84:        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
85:        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
86:        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
87:    }
```

## Tools Used
Manual review.

## Recommended Mitigation Steps
Consider applying a check in the function `withdrawRemainingTokens`, which incase the contract is paused. 
The owner will be able to withdraw all of the remaining tokens in the contract:

```solidity
function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);

        if (isPaused) {
        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee();
        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);

        } else {
         uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
         uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
         IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
        } 
    }
```

### [M-03] In contract Quest the function `claim` shouldn't only set the receipt as claimed, but to burn it as well. As this problem brings the risk, where users can sell already claimed receipts to other people

The function `claim` is used by users to claim their ERC721 receipts for rewards. By using the function the receipt is set as claimed with a simple mapping id => bool, but it isn't burned. In the protocol docs it is clearly stated that users are free to sell or trade their receipts. Since the claimed receipts aren't burned, this bring the risk where already claimed receipts can be sold to other people. A burn function already exists in RabbitHoleReceipt, but isn't used.

### [M-04] The function mintReceipt should check if the quest has expired on-chain as well

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
