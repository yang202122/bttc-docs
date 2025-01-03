# Cross-chain Transaction

### 1. What is a cross-chain transaction?  
  A cross-chain transaction transfers your token asset from one mainnet (e.g., TRON) to another (e.g., Ethereum) through BTTC via a single cross-chain request.  
### 2. How is this achieved?  
  1. The process relies on swapping child tokens on the BTTC chain directly.  
  2. Define one child token as the origin token and correlate it to the other child token. Then, add the swapIn and swapOut functions.  
  3. The two functions are then called by the ChildERC20Exit contract https://bttcscan.com/address/0xbd55ae392f5abbc10eff7d6610b5b2d32293c418#code to perform the token swap, as illustrated in the script below.  

```js
   /**
     * @notice function to swap and withdraw ERC20 token automatically except btt interrelated token
     * @param to token receive address
     * @param tokenWithdraw token to withdraw
     * @param tokenExit token to exit
     * @param amount token amount
     */
    function withdrawTo(address to, IChildToken tokenWithdraw, IChildToken tokenExit, uint256
        amount)
    external override open() {
        emit ExitTokenTo(msg.sender, to, address(tokenWithdraw), address(tokenExit), amount);

        IERC20(tokenWithdraw).safeTransferFrom(msgSender(), address(this), amount);
        if (tokenWithdraw == tokenExit) {
            tokenExit.withdrawTo(to, amount);
            return;
        }
        IChildToken originToken = tokenToOrigin[tokenWithdraw];
        require(address(originToken) != address(0), "originToken can't be zero");
        if (tokenWithdraw != originToken) {
            IChildTokenForExchange(address(tokenWithdraw)).swapOut(amount);
            tokenWithdraw = originToken;
        }
        if (tokenExit == originToken) {
            tokenExit.withdrawTo(to, amount);
            return;
        }
        IERC20(tokenWithdraw).safeIncreaseAllowance(address(tokenExit), amount);
        IChildTokenForExchange(address(tokenExit)).swapIn(amount);
        tokenExit.withdrawTo(to, amount);
    }
```

### 3. Example:
  1. In the following section, we'll use the JST token as a reference. Here, [JST_t](https://bttcscan.com/address/0x17501034df227d8565a8c11f41df2418f5d403b6#code) serves as the origin token, while JST_e and JST_b are the other two child tokens.
  2. The script below illustrates how [JST_t](https://bttcscan.com/address/0x17501034df227d8565a8c11f41df2418f5d403b6#code) is swapped for JST_e (JST_b also undergoes the same process). The key is to use the swap functions and set [JST_t](https://bttcscan.com/address/0x17501034df227d8565a8c11f41df2418f5d403b6#code) as the origin token.

```js
1. just_e token:   
   https://bttcscan.com/address/0x6c0a243302429d3ab54207414aeabb7c6be70aeb#code 

contract JST_e is
    ERC20,
    IChildToken,
    AccessControlMixin,
    NativeMetaTransaction,
    ContextMixin
{
    event SwapIn(address indexed sender, uint256 value);
    event SwapOut(address indexed sender, uint256 value);

    bytes32 public constant DEPOSITOR_ROLE = keccak256("DEPOSITOR_ROLE");

    address public immutable originToken;

    constructor(
        string memory name_,
        string memory symbol_,
        uint8 decimals_,
        address childChainManager,
        address origin_
    ) public ERC20(name_, symbol_) {
        _setupContractId("JST_e");
        _setupDecimals(decimals_);
        _setupRole(DEPOSITOR_ROLE, childChainManager);
        _initializeEIP712(name_);
        originToken = origin_;
    }

    // This is to support Native meta transactions
    // never use msg.sender directly, use _msgSender() instead
    function _msgSender()
        internal
        override
        view
        returns (address payable sender)
    {
        return ContextMixin.msgSender();
    }

    /**
     * @notice called when token is deposited on root chain
     * @dev Should be callable only by ChildChainManager
     * Should handle deposit by minting the required amount for user
     * Make sure minting is done only by this function
     * @param user user address for whom deposit is being done
     * @param depositData abi encoded amount
     */
    function deposit(address user, bytes calldata depositData)
        external
        override
        only(DEPOSITOR_ROLE)
    {
        uint256 amount = abi.decode(depositData, (uint256));
        _mint(user, amount);
    }

    /**
     * @notice called when user wants to withdraw tokens back to root chain
     * @dev Should burn user's tokens. This transaction will be verified when exiting on root chain
     * @param amount amount of tokens to withdraw
     */
    function withdrawTo(address to, uint256 amount) public {
        _burn(_msgSender(), amount);
        emit WithdrawTo(to, address(0x00), amount);
    }

    function withdraw(uint256 amount) external {
        withdrawTo(_msgSender(), amount);
    }

    function swapIn(uint256 amount) public {
        require(originToken != address(0x0), "origin token not set");
        IERC20(originToken).transferFrom(_msgSender(), address(this), amount);
        _mint(_msgSender(), amount);
        emit SwapIn(_msgSender(), amount);
    }

    function swapOut(uint256 amount) public {
        require(originToken != address(0x0), "origin token not set");
        _burn(_msgSender(), amount);
        IERC20(originToken).transfer(_msgSender(), amount);
        emit SwapOut(_msgSender(), amount);
    }

}
```

JST_t token
1. [JST_t](https://bttcscan.com/address/0x17501034df227d8565a8c11f41df2418f5d403b6#code)
2. There's no need to add the swap functions.

