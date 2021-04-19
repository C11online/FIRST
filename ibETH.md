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

- function `mint`(uint mintAmount) external returns (uint) {mintAmount;`delegateAndReturn();`}

- function `redeem`(uint redeemTokens) external returns (uint) {redeemTokens;`delegateAndReturn();`}    

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
  if (allowResign) {delegateToImplementation(abi.encodeWithSignature("_resignImplementation()"));}
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
  uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);
  if (allowed != 0) {return (failOpaque(Error.COMPTROLLER_REJECTION, FailureInfo.MINT_COMPTROLLER_REJECTION, allowed), 0);}
  if (accrualBlockNumber != getBlockNumber()) {return (fail(Error.MARKET_NOT_FRESH, FailureInfo.MINT_FRESHNESS_CHECK), 0);}

  MintLocalVars memory vars;

  vars.`exchangeRateMantissa` = `exchangeRateStoredInternal()`;
  * `exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply`

  `function exchangeRateStoredInternal() internal view returns (uint) {
    uint _totalSupply = totalSupply;
    if (_totalSupply == 0) {
      return initialExchangeRateMantissa;
      } else {
        uint totalCash = getCashPrior();
        uint cashPlusBorrowsMinusReserves = sub_(add_(totalCash, totalBorrows), totalReserves);
        uint exchangeRate = div_(cashPlusBorrowsMinusReserves, Exp({mantissa: _totalSupply}));
        return exchangeRate;
      }
    }`


  vars.`actualMintAmount` = doTransferIn(minter, mintAmount);

  vars.`mintTokens` = `div_ScalarByExpTruncate`(vars.`actualMintAmount`, Exp({mantissa: vars.`exchangeRateMantissa`}));
  *  (`actualMintAmount` * `expScale` / `exchangeRateMantissa`) / `expScale`
  *  How it works: Exp = a / b; Scalar = s; s / (a / b) = b * s / a and since for an Exp a = mantissa, b = `expScale`
  *  uint constant `expScale` = 1e18;
  *   `function div_ScalarByExpTruncate(uint scalar, Exp memory divisor) pure internal returns (uint) {
      Exp memory fraction = div_ScalarByExp(scalar, divisor);
      return truncate(fraction);
      }`
  *   `function div_ScalarByExp(uint scalar, Exp memory divisor) pure internal returns (Exp memory) {
      uint numerator = mul_(expScale, scalar);
      return Exp({mantissa: div_(numerator, divisor)});
      }`
  *   `function truncate(Exp memory exp) pure internal returns (uint) {
      return exp.mantissa / expScale;
      } `

  vars.totalSupplyNew = add_(totalSupply, vars.mintTokens);

  vars.accountTokensNew = add_(accountTokens[minter], vars.mintTokens);

  totalSupply = vars.totalSupplyNew;

  accountTokens[minter] = vars.accountTokensNew;

  emit Mint(minter, vars.actualMintAmount, vars.mintTokens);

  emit Transfer(address(this), minter, vars.mintTokens);

  comptroller.mintVerify(address(this), minter, vars.actualMintAmount, vars.mintTokens);

  return (uint(Error.NO_ERROR), vars.actualMintAmount);
  }
