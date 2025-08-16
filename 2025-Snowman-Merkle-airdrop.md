### [L-1] Unnecessary ```s_earntimer``` State Update in ```Snow::buySnow``` Function.

**Description:** 

The ```s_earntimer``` State gets Update in ```Snow::buySnow``` Function ,even when the transaction 
involves an amount of zero.As per documentation, users can either buy or can mint their free tokens.
Due to ```s_earntimer``` state updation users cannot  claim their first week Free Tokens,
Resulting into loss of one Token .

**Impact:** 

1.Users cannot mint their first week free token ,leading to loss of one token.

2.Unnecessary state updats without efficient checks leads to more gas consumption.

3.Unfair to users who did not complete their valid token purchase.

**Proof of Concept:**
1. Here we try to buy token with zero amount,and it got executed.

2.No Tokens we minted by Bob.

3.Despite of zero amount,s_earntimer got update ,which prevents bob to mint its 
first week Free Token till first week.

```
 function test_lossoffirstweektoken() public {
        vm.deal(bob, 10 ether);
        
        vm.startPrank(bob);
        //buying token with amount zero
        snow.buySnow{value: 1e18}(0);
        
        snow.earnSnow();
        
        vm.stopPrank();
        /*
        Ran 1 test for test/Owntest.sol:OwnTest
[FAIL: S__Timer()] test_lossoffirstweektoken() (gas: 82999)
Traces:
  [82999] OwnTest::test_lossoffirstweektoken()
    ├─ [0] VM::deal(bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e], 10000000000000000000 [1e19])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e])
    │   └─ ← [Return] 
    ├─ [60096] Snow::buySnow{value: 1000000000000000000}(0)
    │   ├─ [16355] MockWETH::transferFrom(bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e], Snow: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 0)
    │   │   ├─ emit Transfer(from: bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e], to: Snow: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], value: 0)
    │   │   └─ ← [Return] true
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e], value: 0)
    │   ├─ emit SnowBought(buyer: bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e], amount: 0)
    │   └─ ← [Return] 
    ├─ [1315] Snow::earnSnow()
    │   └─ ← [Revert] S__Timer()
    └─ ← [Revert] S__Timer()
    */
    }
 ```

**Recommended Mitigation:**  

```
function buySnow(uint256 amount) external payable canFarmSnow {
    if (msg.value == (s_buyFee * amount)) {
        if (amount > 0) {
            _mint(msg.sender, amount);
            s_earnTimer = block.timestamp; 
        }
    } else {
        if (amount > 0) {
            i_weth.safeTransferFrom(
                msg.sender,
                address(this),
                (s_buyFee * amount)
            );
            _mint(msg.sender, amount);
            s_earnTimer = block.timestamp; 
        }
    }

    emit SnowBought(msg.sender, amount);
} 

```
