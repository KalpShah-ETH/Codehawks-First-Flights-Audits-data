### [H-1] Frontrunning attack via ```amendSellOrder``` Function Before Buyer Transaction confirmation 

**Description:** 

The ```OrderBook::amendSellOrder``` function allows sellers to update their price ,amount and deadline.
Since Ethereum mempool is public, A malicoius seller can monitor pending buyorder transaction and frontrun
them by modifying (increasing price and reducing token),causing buyers to receive less token then expected
and paying more price.  

**Impact:** 

1.Buyer tricked into accepeting ammended orders with worse terms

2.loss of buyers trust in the protcol due to unexpected situations

3.If price or amount changes before mining, even trusted frontend UIs become unreliable.

**Proof of Concept:**

1.Seller lists 100 wETH for 1000 USDC.

2.Buyer sees this and calls buyOrder(orderId) paying 1000 USDC.

3.Buyer’s transaction is pending in the mempool.

4.Seller quickly frontruns it with amendSellOrder() changing amount to 50 wETH or price to 2000 USDC.

5.Depending on timing, buyer:

Gets fewer tokens than expected.

Or transaction fails (deadline or price mismatch on frontend vs blockchain).

6.This all happens on-chain and undetectable pre-transaction, resulting in buyer exploitation.

**Recommended Mitigation:**  

1.Add order hash commitment to ensure no changes were made

2.Buyers compute the hash off-chain before sending the transaction.


```
function buyOrder(uint256 _orderId, bytes32 expectedHash) public {
    require(
        keccak256(abi.encode(order.amountToSell, order.priceInUSDC, order.deadlineTimestamp)) == expectedHash,
        "Order modified"
    );
}


```


### [L-1] No deletion of Cancelled Orders

**Description:** 

The ```cancelSellOrder``` Function marks the cancelled order as inactive but does not delete 
the Order struct from storage.Due to this contract retains unnecessary data.This leads to
unbounded storage and storage bloat over the time.

**Impact:** 

1.Increased storage costs especially in high trading volumes.

2.Elevated and increased gas costs when reading/writting to storages

3.Due to Higher gas fees ,users gets discourages to buy.

4.waste of on-chain storage leads to inefficiency.

**Proof of Concept:**

1.All users creates hundreds of orders and can cancel that.

2.However it does not get deleted from the storage.

3.Unnecessary gas costs increases over the time.

**Recommended Mitigation:**  

Add delete statement to delete from the mappings and free up the storage.

```diff

 function cancelSellOrder(uint256 _orderId) public {
        Order storage order = orders[_orderId];

        if (order.seller == address(0)) revert OrderNotFound();
        if (order.seller != msg.sender) revert NotOrderSeller();
        if (!order.isActive) revert OrderAlreadyInactive(); // Already inactive (filled or cancelled)

        // Mark as inactive
        order.isActive = false;

        IERC20(order.tokenToSell).safeTransfer(
            order.seller,
            order.amountToSell
        );

+delete orders[_orderid];

        emit OrderCancelled(_orderId, order.seller);
    }

```


### [I-1] CEI Pattern violation in ```OrderBook::withdrawFees``` Function

**Description:** 

The ```OrderBook::withdrawFees``` Function Violates the CEI (Checks-Effects-Interactions) Patterns
by performing an external call before updating internal state variable.

This violation increases chances of getting disrupt through reentrancy attacks.

**Impact:** 

1.Minor risk of inconsistent state due to reentancy attacks.

2.Breaks solidity best practices.

3.Static analysis and formal tools may flag this as unsafe.



**Recommended Mitigation:**  

```diff

 function withdrawFees(address _to) external onlyOwner {
        if (totalFees == 0) {
            revert InvalidAmount();
        }
        if (_to == address(0)) {
            revert InvalidAddress();
        }
        
+        uint amount=totalfees;
+        totalFees = 0;

+        iUSDC.safeTransfer(_to, amount);
        
        emit FeesWithdrawn(_to);
    }

```

### [I-2] Missing Balance Check in ```OrderBook::emergencyWithdrawERC20```


**Description:** 

The emergencyWithdrawERC20 function allows the contract owner to withdraw any non-core ERC20 token from the contract. However, it does not validate that the contract has a sufficient balance before attempting the transfer. This violates defensive programming practices.

**Impact:** 

1.Poor developer experience during debugging.

2.Wasted gas on failed transactions.

3.Potentially confusing behavior during emergency recovery situations.

**Proof of Concept:**

1.Owner tries to withdraw more amount then contract balance.

2.Transaction reverts with inefficient balance.

3.Gas wastages,poor experience.

**Recommended Mitigation:**  

```diff

 function emergencyWithdrawERC20(
        address _tokenAddress,
        uint256 _amount,
        address _to
    ) external onlyOwner {
        if (
            _tokenAddress == address(iWETH) ||
            _tokenAddress == address(iWBTC) ||
            _tokenAddress == address(iWSOL) ||
            _tokenAddress == address(iUSDC)
        ) {
            revert(
                "Cannot withdraw core order book tokens via emergency function"
            );
        }
        if (_to == address(0)) {
            revert InvalidAddress();
        }

        IERC20 token = IERC20(_tokenAddress);
+       uint balance=token.balanceOf(address(this));

+        if(_amount>balance){
+            revert insufficientbalance();
+        }

        token.safeTransfer(_to, _amount);

     
        emit EmergencyWithdrawal(_tokenAddress, _amount, _to);
    }

```


### [I-3] Incorrect Deadline Check in `buyOrder` 

**Description:** 
It was intially assumed that the deadline check in buyOrder function should use >=:

``` 
 if (block.timestamp >= order.deadlineTimestamp) revert OrderExpired();

```
However ,this is an incorrect behaviour .An order with block.timestamp==deadline should 
still be valid and buyable at the exact moment.Using >= prematurely restrict buyers to buy
at exact moment.

**Impact:** 
1.Buyers failes to buy order at precise deadline.

2.Could discourage last-moment trades.

**Proof of Concept:**

1.Seller creates an order with deadlineTimestamp = 3:00 PM.

2.Buyer sends a transaction at exactly 3:00 PM (block.timestamp == deadlineTimestamp).

3.If >= is used, the transaction reverts — though the order should still be valid.


**Recommended Mitigation:**  

Use Only >,Which will allow buyers to buy at exact moment and reverting after that.

```diff

  function buyOrder(uint256 _orderId) public {
        Order storage order = orders[_orderId];

        // Validation checks
        if (order.seller == address(0)) revert OrderNotFound();
        if (!order.isActive) revert OrderNotActive();
-if(block.timestamp >= order.deadlineTimestamp) revert OrderExpired();
+if(block.timestamp > order.deadlineTimestamp) revert OrderExpired();

        order.isActive = false;
  }

```
