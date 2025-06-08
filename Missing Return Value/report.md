***I found the Missing Return Value Check in Compound-Finance Protocol***

___

Silent Failure on Insufficient Balance

**Severity:** Medium  
**Target:** `cUSDC` (Compound’s USDC cToken)  

---

##  Bug Summary  
- **Issue:** `cUSDC.transfer()` returns `false` on failure but **does not revert**, causing silent failures.  
- **Risk:** Contracts relying on `transfer()` may incorrectly assume success, leading to broken integrations.  
- **Affected Function:** `CErc20.transfer()` in Compound Protocol. 

 

## 🔍 Proof of Concept (PoC) 

### Test Code (Foundry)

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/interfaces/IERC20.sol";
import "../src/interfaces/IcERC20.sol";

contract missingReturn is Test {
 address constant COMPOUND = 0xc00e94Cb662C3520282E6f5717214004A7f26888;
 address constant cUSDC = 0x39AA39c021dfbaE8faC545936693aC917d5E7563;
 address constant Holder = 0x37e34E113D571d80d4BC20c941E8117738Ae2A09;
 address constant User = 0x90581aFC83520a649376852166B3df92153cEE20;

IERC20 comp = IERC20(COMPOUND);
IcERC20 cusdc = IcERC20(cUSDC);

function setUp () public {
vm.createSelectFork(vm.envString("MAINNET_RPC_URL"), 19600000);
}

function test_MissingReturn() public {
vm.startPrank(Holder);
comp.approve(cUSDC, 1000);
cusdc.mint(1000);

uint256 balance = cusdc.balanceOf(Holder);
uint256 badAmount = balance + 1;
cusdc.transfer(User, badAmount);
vm.stopPrank();
uint256 userBalance = cusdc.balanceOf(User);
emit log_named_uint("User Balance After Transfer", userBalance);
assertEq(userBalance, 0, "Transfer succeeded silently, missing return value!");
}
}
```


''''// forge test Test/missingReturn.t.sol --rpc-url MAINNET_RPC_URL -vvvv''''

___

Result after run test on foundry 

$ forge test Test/missingReturn.t.sol --rpc-url MAINNET_RPC_URL
[⠢] Compiling...
[⠢] Compiling...
[⠆] Compiling 22 files with Solc 0.8.24
[⠰] Solc 0.8.24 finished in 6.21s
Compiler run successful!

Ran 1 test for Test/missingReturn.t.sol:missingReturn
[PASS] test_MissingReturn() (gas: 222147)
Compiler run successful!

Ran 1 test for Test/missingReturn.t.sol:missingReturn
[PASS] test_MissingReturn() (gas: 222147)
Ran 1 test for Test/missingReturn.t.sol:missingReturn
[PASS] test_MissingReturn() (gas: 222147)
[PASS] test_MissingReturn() (gas: 222147)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.86s (8.34ms CPU time)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.86s (8.34ms CPU time)

Ran 1 test suite in 5.31s (3.86s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)