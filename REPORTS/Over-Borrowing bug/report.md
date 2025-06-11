Protocol: Lend Protocol
Contract: CoreRouter.sol#145
Function: borrow(_amount, _token);
Link: https:sherlock-audit-report


        **FUNCTION CALL FLOW:**
borrow(_amount, _token);    
 ├─ lendStorage.underlyingTolToken(_token);    
 ├─ LTokenInterface(_lToken).accrueInterest();    
 ├─             lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);    
 ├─ lendStorage.getBorrowBalance(msg.sender, _lToken);    
 ├─ LTokenInterface(_lToken).borrowIndex();    
 ├─ enterMarkets(_lToken);    
 └─ LErc20Interface(_lToken).borrow(_amount) == 0, "Borrow failed");


         **FUNCTION DEPENDENCIES:**               CoreRouter.sol    
 ├─ lendStorage.sol    
 ├─ LTokenInterface.sol            
 └─ LErc20Interface.sol


         **FUNCTION LOGIC SUMMARY:**   
A. This function is used to borrow asset from the FAsset protocol, B. The function will first check the lToken address using underlying asset ( token ), after function got lToken address,
C. It will accrueInterest of the lToken via LTokenInterface contract, after accumulate the interest
D. The function will calculate the hypothetical account liquidity collateral ( which is current borrow + _amount ) and see if the collateral is sufficient for this post-borrow debt.
E. After that, the function will check the current borrow balance via lendStorage contract, which is "pre-borrow debt"if a user has borrowed before.
F. The function will check if a user collateral is equal or greater than the borrowed,
G. After this, the lToken will enterMarkets ( if it is not there already ).
H. The borrow will be executed via LTokenInterface contract and the amount will be transfered to the user.
I. The function will distribute borrower lend via lendStorage contract.
J. The function will then update record/states whether borrower has a current borrow balance or not.
K. Lastly, the function with add this lToken to the user borrowed asset list in order to track all the assets that user borrowed and emet the event.
     
 
     
 ## **FULL FUNCTION WITH INLINE COMMENTS:**
```function borrow(uint256 _amount, address _token) external {        require(_amount != 0, "Zero borrow amount");
    
    // Looking for lToken address using underlying token
        address _lToken = lendStorage.underlyingTolToken(_token);
        
        // accruedInterest of the lToken entirely 
        LTokenInterface(_lToken).accrueInterest();
        
        // calculate the hypothetical balance (postBorrowAmount) which is currentBorrow + _amount and calculate the collateral 
        (uint256 borrowed, uint256 collateral) = lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);
        
        // checking for user's market state and see if user has borrowed before 
        LendStorage.BorrowMarketState memory currentBorrow = lendStorage.getBorrowBalance(msg.sender, _lToken);
        
        // if user has borrowed before, it calculates the borrowAmount here (pre-borrow debt)
        uint256 borrowAmount = currentBorrow.borrowIndex != 0            ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)            : 0;
        // require condition to prevent user from Over-Borrowing
  @audit--> The protocol check preBorrowAmount (borrowAmount) instead of postBorrowAmount (borrowed) this will allow any user to Over-Borrowing beyond the collateral limit
  require(collateral >= borrowAmount, "Insufficient collateral");
  
  // Enter the Compound market if lToken is not there already
  enterMarkets(_lToken);
  
        // Borrow tokens will execute here, if it's 0 the borrow has been successfully otherwise it failed
        require(LErc20Interface(_lToken).borrow(_amount) == 0, "Borrow failed");
        
        // Transfer borrowed tokens to the user and distribute rewards (lend)
        IERC20(_token).transfer(msg.sender, _amount);
        lendStorage.distributeBorrowerLend(_lToken, msg.sender);
        
        // Update user balance if he has borrowed before 
        if (currentBorrow.borrowIndex != 0) {
        uint256 _newPrinciple =                (currentBorrow.amount * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex;
            lendStorage.updateBorrowBalance(msg.sender, _lToken, _newPrinciple + _amount, LTokenInterface(_lToken).borrowIndex()            );        } else {
            
         // update user balance if he hasn't borrowed before       
            lendStorage.updateBorrowBalance(msg.sender, _lToken, _amount, LTokenInterface(_lToken).borrowIndex());
            }
            // add lToken to the user borrowed assets list
        lendStorage.addUserBorrowedAsset(msg.sender, _lToken);
        
      // Emit BorrowSuccess event 
        emit BorrowSuccess(msg.sender, _lToken, lendStorage.getBorrowBalance(msg.sender, _lToken).amount);
        }```