### [H-1] Grace Period Can Be Bypassed Indefinitely Throne Claims — DoS Vulnerability in declareWinner

**Description:** 

The ```declareWinner()``` function checks if ```block.timestamp > lastClaimTime + gracePeriod``` to determine
if the game should end.However, since claimThrone() updates lastClaimTime every time a new user claims the throne,
any actor can continuously re-claim the throne right before the gracePeriod expires, indefinitely resetting the timer.

This creates a denial-of-service vector where a malicious actor (or bot) can grief the game and prevent the winner from ever being declared, unless the attacker stops or runs out of gas/ETH.

**Impact:** 

1.The declareWinner() function can be permanently blocked by malicious users.

2.Honest players can never win the game despite waiting through the grace period.

3.The entire game can be held hostage by a spammer.

**Proof of Concept:**

1.Bob calls claim throne-lastclaimtime=1

2.alice calls claim throne-lastclaimtime=36001

3.malicious actor calls claimthrone-lastclaimtime=72001

4.Since declareWinner() requires current time > lastClaimTime + TIMEOUT, the game can be 
indefinitely stalled by malicious actors.

```solidity
 function test_dos() public {

        vm.startPrank(bob);
        game.claimThrone{value: 1 ether}();
        vm.stopPrank();

        vm.warp(36000);

        vm.startPrank(alice);
        game.claimThrone{value: 1.1 ether}();
        vm.stopPrank();

        vm.warp(36000);

        vm.startPrank(maliciousActor);
        game.claimThrone{value: 1.5 ether}();
        vm.stopPrank();

        vm.expectRevert();
        game.declareWinner();
    }
Traces:
  [302401] GameTest::test_dos()
    ├─ [0] VM::startPrank(bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e])
    │   └─ ← [Return] 
    ├─ [152621] Game::claimThrone{value: 1000000000000000000}()
    │   ├─ emit ThroneClaimed(newKing: bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e], claimAmount: 1000000000000000000 [1e18], newClaimFee: 1010000000000000000 [1.01e18], newPot: 990000000000000000 [9.9e17], timestamp: 1)
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(36000 [3.6e4])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6])
    │   └─ ← [Return] 
    ├─ [52921] Game::claimThrone{value: 1100000000000000000}()
    │   ├─ emit ThroneClaimed(newKing: alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], claimAmount: 1100000000000000000 [1.1e18], newClaimFee: 1020100000000000000 [1.02e18], newPot: 2079000000000000000 [2.079e18], timestamp: 36000 [3.6e4])
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(36000 [3.6e4])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(maliciousActor: [0x195Ef46F233F37FF15b37c022c293753Dc04A8C3])
    │   └─ ← [Return] 
    ├─ [50121] Game::claimThrone{value: 1500000000000000000}()
    │   ├─ emit ThroneClaimed(newKing: maliciousActor: [0x195Ef46F233F37FF15b37c022c293753Dc04A8C3], claimAmount: 1500000000000000000 [1.5e18], newClaimFee: 1030301000000000000 [1.03e18], newPot: 3564000000000000000 [3.564e18], timestamp: 36000 [3.6e4])
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return] 
    ├─ [6786] Game::declareWinner()
    │   ├─ [0] console::log("in contract:", 36000 [3.6e4]) [staticcall]
    │   │   └─ ← [Stop] 
    │   └─ ← [Revert] revert: Game: Grace period has not expired yet.
    └─ ← [Stop]

 ```

**Recommended Mitigation:**  

Lock the throne after grace period starts:

```diff

function claimThrone() external payable gameNotEnded nonReentrant {
+require(block.timestamp <=+ gracePeriod, "Game: Grace period started, no more claims allowed.");
    ...
}

```


### [M-1] Throne Cannot Be Claimed — Game Is Permanently Bricked 

**Description:** 
The claimThrone() function contains a faulty require statement that prevents any user from ever claiming the throne.

Specifically, it checks:

```solidity
require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
```
This condition only allows the current king to call claimThrone(), which completely contradicts the purpose of the game — where new players are supposed to outbid the current king to claim the throne.

At deployment:

currentKing == address(0)

First caller (e.g., Bob) will never be address(0)

⇒ claimThrone() always reverts

⇒ No one can ever claim the throne


**Impact:** 

1.Critical game-breaking flaw

2.No one can ever become King

3.All ETH sent is wasted or locked

4.Prize pool never accumulates

5.Game logic never progresses

6.Contract is permanently bricked

**Proof of Concept:**

```

// assume: initial claim fee = 1 ETH, currentKing == address(0)

claimThrone() called by Bob (EOA != address(0)):

→ require(msg.sender == currentKing) fails
→ reverts: "Game: You are already the king."
→ Bob cannot claim throne
→ No state updates
→ Game cannot begin

// Game is now in a permanently unusable state

```

**Recommended Mitigation:**  

1. Correct the require logic:

Replace:

```solidity
require(msg.sender == currentKing, "Game: You are already the king.");
```

With:

```solidity
require(msg.sender != currentKing, "Game: You are already the king.");

```

### [M-2] Precision Truncation May Lead to Platform Fee Loss and Pot Misallocation


**Description:** 

In the claimThrone() function, the platform fee is calculated as:
```
currentPlatformFee = (sentAmount * platformFeePercentage) / 100;

```
This calculation involves integer division in Solidity. If sentAmount is very small (e.g., 1 wei), and platformFeePercentage is low (e.g., 1%), the result will truncate to 0 due to Solidity’s lack of decimals, causing:

1.platformFeesBalance to receive zero even though a fee is expected.

2.amountToPot to misrepresent what remains after fee deduction.

3.Event logs and game state to become inconsistent with economic expectations.

4.This becomes particularly relevant if the owner sets a low claimFee, or a user intentionally sends the minimum ETH possible to manipulate this logic.



**Impact:** 

1.Users can circumvent platform fees by exploiting small-value claims.

2.Over time, the platform may lose significant value in micro-farming scenarios or sybil loops.

3.Fee percentage logic becomes non-deterministic at low scales, harming auditability.

**Proof of Concept:**

1.Users can circumvent platform fees by exploiting small-value claims.

2.Over time, the platform may lose significant value in micro-farming scenarios or sybil loops.

3.Fee percentage logic becomes non-deterministic at low scales, harming auditability.
**Recommended Mitigation:**  

Enforce Minimum Claim Value:

```
require(msg.value >= MIN_CLAIM_THRESHOLD, "Game: Claim fee too low.");
```

### [M-3]  Missing payout to previous player breaks documented incentive model


**Description:** 

The documentation states that when a player joins and calls claim(), part of the claimFee is sent to 
the previous player. However, the smart contract does not contain any logic to actually perform this 
payout. As a result, previous players receive nothing, which directly contradicts the intended reward 
model and creates an unfair loss.


**Impact:** 
This breaks the game’s incentive mechanism. Players are led to believe they will receive a portion of the claimFee when someone else joins after them, but in practice, they receive nothing. This creates:

1.An unfair economic loss for participants.

2.Broken trust and reduced participation due to false incentives.

3.A game model collapse if no one receives any reward beyond the chance to be the winner.

**Proof of Concept:**

1.Player A joins by calling claim() and pays the claimFee.

2.Player B joins afterward, calling claim() and also pays the claimFee.

3.Expected: Part of Player B’s fee goes to Player A (as per docs).

4.Actual: There is no transfer or payout to Player A in code.

5.Player A ends up with 0 reward, despite the documented claim.

**Recommended Mitigation:**  

Add logic in the claim() function to transfer the specified share of the claimFee to the previous claimer, e.g.:

```
if (lastClaimer != address(0)) {
    payable(lastClaimer).transfer(rewardAmount);
}

```
Ensure proper validation and state updates to prevent abuse.

Alternatively, if the reward is intentionally removed, update the documentation to reflect the actual behavior — no fee sharing — to avoid misleading players.



### [L-1] Misleading GameEnded Event Emitted After pot is Zeroed


**Description:** 

The ```declareWinner()``` function emits the GameEnded event after setting pot = 0, resulting in prizeAmount always being emitted as 0, even though the winner's actual reward was non-zero.

This behavior misleads off-chain systems, dashboards, users, or auditors who rely on event data for historical game outcomes.

**Impact:** 

1.Users/frontends will see a prize amount of 0 in the GameEnded event, even if the actual winnings were much higher.

2.Analytics tools or dashboards built on event logs (e.g., The Graph, Dune, Nansen) will display incorrect prize history.

3.Trust in contract transparency may be undermined due to inaccurate emitted data.

**Proof of Concept:**

In the declareWinner() function:

```solidity
pendingWinnings[currentKing] += pot;
pot = 0;
emit GameEnded(currentKing, pot, block.timestamp, gameRound); // `pot` is already 0

```

```solidity
emit GameEnded(winner, 0, timestamp, round); // always shows 0 prize

```
**Recommended Mitigation:**  

Cache the pot value in a local variable before zeroing it, and use the cached value in the event:

```solidity
uint256 reward = pot;
pendingWinnings[currentKing] += reward;
pot = 0;

emit GameEnded(currentKing, reward, block.timestamp, gameRound);
```

### [I-1] CEI pattern not followed in withdrawWinnings


**Description:** 
The withdrawWinnings function updates the state after making the external call to msg.sender. This 
violates the Checks-Effects-Interactions (CEI) pattern, which is a standard best practice in Solidity to 
prevent reentrancy issues and unexpected behavior.

While the function uses nonReentrant, following CEI adds defense-in-depth and makes the code more robust and auditable.

**Impact:** 

1.In general, reentrancy is prevented by nonReentrant, so this is not a direct vulnerability.

2.However, future maintainers may remove the nonReentrant modifier or introduce new logic that becomes vulnerable.

3.Also, it increases the cognitive load of reasoning about state safety during audits or upgrades.

**Proof of Concept:**

```solidity
function withdrawWinnings() external nonReentrant {
    uint256 amount = pendingWinnings[msg.sender];
    require(amount > 0, "Game: No winnings to withdraw.");

    //  Interaction before effect
    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Game: Failed to withdraw winnings."); 

    //  State update after external call
    pendingWinnings[msg.sender] = 0;

    emit WinningsWithdrawn(msg.sender, amount);
}

```

**Recommended Mitigation:**  

Update the state before the external call:

```solidity

function withdrawWinnings() external nonReentrant {
    uint256 amount = pendingWinnings[msg.sender];
    require(amount > 0, "Game: No winnings to withdraw.");

    // Effect before interaction
    pendingWinnings[msg.sender] = 0;

    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Game: Failed to withdraw winnings."); 

    emit WinningsWithdrawn(msg.sender, amount);
}


```
