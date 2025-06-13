// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {IERC20} from "../src/interfaces/IERC20.sol";


interface ICoreRouter {
    function balanceOf(address account) external view returns (uint256);
    function supply(uint256 amount, address token) external;
    function borrow(address token, uint amount) external;
    function repayBorrow(address token, uint amount) external;
}


contract LendBorrowAndRepay is Test {

    address public USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address public CoreRouter = 0x55b330049d46380BF08890EC21b88406eBFd20B0;
    address public User = 0x37305B1cD40574E4C5Ce33f8e8306Be057fD7341;
    address public LToken = 0x3BF04b493D844f135E934Fd18480D9FE22A6734B;


    function setUp() public {
    vm.createSelectFork(vm.envString("ETH_RPC_URL"), 19999999);

    deal(USDC, User, 100_000e6);
    console2.log("USDC Balance:", IERC20(USDC).balanceOf(User));
    }

    function test_borrow_and_repay() public {
        vm.startPrank(User);

     // approve
        IERC20(USDC).approve(CoreRouter, type(uint256).max);
        console2.log("USDC Allowance for CoreRouter:", IERC20(USDC).allowance(User, CoreRouter));

     // supply
       (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("supply(uint256,address)", 10_000e6, USDC)
        );
     if (!success) console2.log("Revert reason:", string(data));

     console2.log("User supplied:", ICoreRouter(USDC).balanceOf(User));
     
     vm.warp(block.timestamp + 1 days);

        vm.stopPrank();
    }


    function test_borrow1() public {
     vm.startPrank(User);
        // borrow
        (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("borrow(address,uint256)", LToken, 5_000e18)
        );
    if (!success) console2.log("Revert reason:", string(data));
    vm.stopPrank();
    }

    function test_borrow2() public {
     vm.startPrank(User);
        // borrow
        (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("borrow(address,uint256)", LToken, 4_000e18)
        );
    if (!success) console2.log("Revert reason:", string(data));
    vm.stopPrank();
    }
}

// forge test Test/LendBorrowAndRepay.t.sol -vvvv


![PoC](https://github.com/user-attachments/assets/38c81c75-7017-48b6-8d54-b24e8a9df4bc)
