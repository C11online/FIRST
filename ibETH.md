## ibETHv2 `0xeEa3311250FE4c3268F8E684f7C87A82fF183Ec1`

- ICErc20 public immutable `cToken` = `0x41c84c0e2ee0b740cf0d31f63f3b6f627dc6b393`;

- constructor(ICErc20 `_cToken`,string memory _name,string memory _symbol) public ERC20(_name, _symbol) {
  _setupDecimals(_cToken.decimals());
  IERC20 _uToken = IERC20(_cToken.underlying());
  __Governable__init();
  `cToken` = `_cToken`;
  uToken = _uToken;
  relayer = msg.sender;
  _uToken.safeApprove(address(_cToken), uint(-1));
}

- function `deposit`(uint amount) external nonReentrant {
  uint uBalanceBefore = uToken.balanceOf(address(this));
  uToken.safeTransferFrom(msg.sender, address(this), amount);
  uint uBalanceAfter = uToken.balanceOf(address(this));
  uint cBalanceBefore = cToken.balanceOf(address(this));

  `require(cToken.mint(uBalanceAfter.sub(uBalanceBefore)) == 0, '!mint');`

  uint cBalanceAfter = cToken.balanceOf(address(this));
  _mint(msg.sender, cBalanceAfter.sub(cBalanceBefore));
}


- function `withdraw`(uint amount) public nonReentrant {
  _burn(msg.sender, amount);
  uint uBalanceBefore = uToken.balanceOf(address(this));

  `require(cToken.redeem(amount) == 0, '!redeem');`

  uint uBalanceAfter = uToken.balanceOf(address(this));
  uToken.safeTransfer(msg.sender, uBalanceAfter.sub(uBalanceBefore));
}

- Methods used to change strategy not found    

## cyWETH `0x41c84c0e2EE0b740Cf0d31F63f3B6F627DC6b393`

- function `mint`(uint mintAmount) external returns (uint) {
  mintAmount; // Shh
  `delegateAndReturn();`

}

- function `redeem`(uint redeemTokens) external returns (uint) {
  redeemTokens; // Shh
  `delegateAndReturn();`
}    

- function `delegateAndReturn()` private returns (bytes memory) {
  (bool success, ) = `implementation`.delegatecall(msg.data);

  assembly {
    let free_mem_ptr := mload(0x40)
    returndatacopy(free_mem_ptr, 0, returndatasize)

    switch success
    case 0 { revert(free_mem_ptr, returndatasize) }
    default { return(free_mem_ptr, returndatasize) }
  }
}   

- address `implementation` = `0x1a9e503562ce800ea8e68e2cf0cfa0aec2edb509`;  
- Methods used to change strategy:    

function `_setImplementation`(address implementation_, bool allowResign, bytes memory becomeImplementationData) public {
  require(`msg.sender == admin`, "CErc20Delegator::_setImplementation: Caller must be admin");
  if (allowResign) {
    delegateToImplementation(abi.encodeWithSignature("_resignImplementation()"));
  }
  address oldImplementation = implementation;
  `implementation` = implementation_;
  delegateToImplementation(abi.encodeWithSignature("_becomeImplementation(bytes)", becomeImplementationData));
  emit NewImplementation(oldImplementation, implementation);
}


## implementation `0x1a9e503562ce800ea8e68e2cf0cfa0aec2edb509`


- function `mint`(uint mintAmount) external returns (uint) {
  (uint err,) = `mintInternal`(mintAmount);
  return err;
}

- function `mintInternal`(uint mintAmount) internal nonReentrant returns (uint, uint) {
  uint error = accrueInterest();
  if (error != uint(Error.NO_ERROR)) {return (fail(Error(error), FailureInfo.MINT_ACCRUE_INTEREST_FAILED), 0);}
  return `mintFresh`(msg.sender, mintAmount);
}   

- function `mintFresh`(address minter, uint mintAmount) internal returns (uint, uint) {
  /* Fail if mint not allowed */
  uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);
  if (allowed != 0) {
    return (failOpaque(Error.COMPTROLLER_REJECTION, FailureInfo.MINT_COMPTROLLER_REJECTION, allowed), 0);
  }

  /* Verify market's block number equals current block number */
  if (accrualBlockNumber != getBlockNumber()) {
    return (fail(Error.MARKET_NOT_FRESH, FailureInfo.MINT_FRESHNESS_CHECK), 0);
  }

  MintLocalVars memory vars;

  vars.`exchangeRateMantissa` = `exchangeRateStoredInternal()`;

  function `exchangeRateStoredInternal()` internal view returns (uint) {
    uint _totalSupply = totalSupply;
    if (_totalSupply == 0) {
      return initialExchangeRateMantissa;
      } else {
        /*
        ** Otherwise:
        **  `exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply`
        */
        uint totalCash = getCashPrior();
        uint cashPlusBorrowsMinusReserves = sub_(add_(totalCash, totalBorrows), totalReserves);
        uint exchangeRate = div_(cashPlusBorrowsMinusReserves, Exp({mantissa: _totalSupply}));
        return exchangeRate;
      }
    }


    /////////////////////////
    // EFFECTS & INTERACTIONS
    // (No safe failures beyond this point)
    //
    //  We call doTransferIn for the minter and the mintAmount.
    //  Note: The cToken must handle variations between ERC-20 and ETH underlying.
    //  doTransferIn reverts if anything goes wrong, since we can't be sure if
    //  side-effects occurred. The function returns the amount actually transferred,
    //  in case of a fee. On success, the cToken holds an additional actualMintAmount
    //  of cash.
    //

    vars.`actualMintAmount` = doTransferIn(minter, mintAmount);

    //
    // We get the current exchange rate and calculate the number of cTokens to be minted:
    // `mintTokens = actualMintAmount / exchangeRate`
    //

    vars.`mintTokens` = `div_ScalarByExpTruncate`(vars.`actualMintAmount`, Exp({mantissa: vars.`exchangeRateMantissa`}));

    //
    //  We calculate the new total supply of cTokens and minter token balance, checking for overflow:
    //  totalSupplyNew = totalSupply + mintTokens
    //  accountTokensNew = accountTokens[minter] + mintTokens
    //

    vars.totalSupplyNew = add_(totalSupply, vars.mintTokens);
    vars.accountTokensNew = add_(accountTokens[minter], vars.mintTokens);

    /* We write previously calculated values into storage */
    totalSupply = vars.totalSupplyNew;
    accountTokens[minter] = vars.accountTokensNew;

    /* We emit a Mint event, and a Transfer event */
    emit Mint(minter, vars.actualMintAmount, vars.mintTokens);
    emit Transfer(address(this), minter, vars.mintTokens);

    /* We call the defense hook */
    comptroller.mintVerify(address(this), minter, vars.actualMintAmount, vars.mintTokens);

    return (uint(Error.NO_ERROR), vars.actualMintAmount);
  }

  /*
  ** Divide a scalar by an Exp, then truncate to return an unsigned integer.
  */
  function `div_ScalarByExpTruncate`(uint scalar, Exp memory divisor) pure internal returns (uint) {
    Exp memory fraction = `div_ScalarByExp`(scalar, divisor);
    return `truncate`(fraction);
  }

    uint constant `expScale` = 1e18;

  /*
  **  Divide a scalar by an Exp, returning a new Exp.
  */
  function `div_ScalarByExp`(uint scalar, Exp memory divisor) pure internal returns (Exp memory) {
    /*
    ** How it works:
    ** Exp = a / b;
    ** Scalar = s;
    ** s / (a / b) = b * s / a and since for an Exp a = mantissa, b = expScale
    */
    uint numerator = mul_(expScale, scalar);
    return Exp({mantissa: div_(numerator, divisor)});
  }

  /*
   ** Truncates the given exp to a whole number value.
   **      For example, truncate(Exp{mantissa: 15 * expScale}) = 15
   */
  function `truncate`(Exp memory exp) pure internal returns (uint) {
      // Note: We are not using careful math here as we're performing a division that cannot fail
      return exp.mantissa / expScale;
  }

