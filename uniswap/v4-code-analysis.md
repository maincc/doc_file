# uniswap-v4 大额swap处理
通过uniswap客户端生成交易发送到uniswapV4的通用智能合约，然后根据交易里的command分别执行actions；actions是一个可以操作poolManager智能合约函数的函数集。    
例如command如果是10（映射是V4_SWAP）的话，生成的actions是0x070b0e(对应的poolmanager的函数是swap、settle、take)：  
swap记账->settle更新余额->take转账  
如果进行大额代币swap处理，开发者可以在创建池子的时候加入自己的hooks进行处理，官方提供了几个hooks样本：  
1. 时间加权平均做市商 (TWAMM)
2. 定制的链上预言机
...


## 1、uniswap/interface对于v4 swap交易处理
## 2、uniswap v4:Universal Router合约代码（polygon）
**1、通过uniswap客户端生成的最佳route路径提交到uniswap v4:Universal Router智能合约中进行处理**  
**2、进行解码，调用Pool Manager合约**  

参考交易：0xd9a77731a694dce75a0c161a3c78049e49735cc8901db6335eff6f86f6630658  
调用：Function: execute(bytes commands,bytes[] inputs,uint256 deadline)  
进行解码的到的input Data：  
0	commands	bytes	0x1010  
1	inputs	bytes[]  
0x0000000000000000...000000000000000000000000000000000  
0x0000000000000000...000000000000000000000000000000000  
2	deadline	uint256	1740021222


** UniversalRouter.sol**
```solidity
/// @inheritdoc Dispatcher
function execute(bytes calldata commands, bytes[] calldata inputs) public payable override isNotLocked {
    bool success;
    bytes memory output;
    uint256 numCommands = commands.length;
    if (inputs.length != numCommands) revert LengthMismatch();

    // loop through all given commands, execute them and pass along outputs as defined
    for (uint256 commandIndex = 0; commandIndex < numCommands; commandIndex++) {
        bytes1 command = commands[commandIndex];

        bytes calldata input = inputs[commandIndex];

        (success, output) = dispatch(command, input);

        if (!success && successRequired(command)) {
            revert ExecutionFailed({commandIndex: commandIndex, message: output});
        }
    }
}
```
因为每个command是1个字节，所以该交易有两条命令分别是0x10、0x10  

**Dispatcher.sol**  
**abstract contract Dispatcher is Payments, V2SwapRouter, V3SwapRouter, V4SwapRouter, V3ToV4Migrator, Lock**
```solidity
function dispatch(bytes1 commandType, bytes calldata inputs) internal returns (bool success, bytes memory output) {
    uint256 command = uint8(commandType & Commands.COMMAND_TYPE_MASK);

    success = true;

    // 0x00 <= command < 0x21
    if (command < Commands.EXECUTE_SUB_PLAN) {
        // 0x00 <= command < 0x10
        if (command < Commands.V4_SWAP) {
            // 0x00 <= command < 0x08
            if (command < Commands.V2_SWAP_EXACT_IN) {
                if (command == Commands.V3_SWAP_EXACT_IN) { ...
                } else if (command == Commands.V3_SWAP_EXACT_OUT) { ...
                } ... { ...
                } 
            } else { ... }
        } else {
            // 0x10 <= command < 0x21
            if (command == Commands.V4_SWAP) {
                // pass the calldata provided to V4SwapRouter._executeActions (defined in BaseActionsRouter)
                _executeActions(inputs);
                // This contract MUST be approved to spend the token since its going to be doing the call on the position manager
            } ... { ... }
        }
    ...
```
根据command执行相应的分支，具体分支对应的含义看https://docs.uniswap.org/contracts/universal-router/technical-reference#command。  
对于uniswap-v4，重要的分支是V4_SWAP。用于调用Pool Manager合约内的相关function。

**abstract contract V4SwapRouter is V4Router, Permit2Payments**  
**abstract contract V4Router is IV4Router, BaseActionsRouter, DeltaResolver**  
**abstract contract BaseActionsRouter is SafeCallback**  
**abstract contract SafeCallback is ImmutableState, IUnlockCallback**  
以上抽象合约依次继承，各自实现自己对应的函数。

**BaseActionsRouter.sol**
```solidity
/// @notice internal function that triggers the execution of a set of actions on v4
/// @dev inheriting contracts should call this function to trigger execution
function _executeActions(bytes calldata unlockData) internal {
    poolManager.unlock(unlockData);
}
```
解锁poolManager从而可以操作poolManager。
```solidity
/// @notice function that is called by the PoolManager through the SafeCallback.unlockCallback
/// @param data Abi encoding of (bytes actions, bytes[] params)
/// where params[i] is the encoded parameters for actions[i]
function _unlockCallback(bytes calldata data) internal override returns (bytes memory) {
    // abi.decode(data, (bytes, bytes[]));
    (bytes calldata actions, bytes[] calldata params) = data.decodeActionsRouterParams();
    _executeActionsWithoutUnlock(actions, params);
    return "";
}

function _executeActionsWithoutUnlock(bytes calldata actions, bytes[] calldata params) internal {
    uint256 numActions = actions.length;
    if (numActions != params.length) revert InputLengthMismatch();

    for (uint256 actionIndex = 0; actionIndex < numActions; actionIndex++) {
        uint256 action = uint8(actions[actionIndex]);

        _handleAction(action, params[actionIndex]);
    }
}

/// @notice function to handle the parsing and execution of an action and its parameters
function _handleAction(uint256 action, bytes calldata params) internal virtual;
```
**_unlockCallback：**这个是poolManager通过safeCallback.unlockCallback调用的函数；将command对应的input datas进行解码分出actions和params。  
**_executeActionsWithoutUnlock：**先检查actions元素长度和params元素长度是否一致，如果一致循环actions，分别执行_handleAction函数。

**SafeCallback.sol**
```solidity
/// @title Safe Callback
/// @notice A contract that only allows the Uniswap v4 PoolManager to call the unlockCallback
abstract contract SafeCallback is ImmutableState, IUnlockCallback {
    /// @notice Thrown when calling unlockCallback where the caller is not PoolManager
    error NotPoolManager();

    constructor(IPoolManager _poolManager) ImmutableState(_poolManager) {}

    /// @notice Only allow calls from the PoolManager contract
    modifier onlyPoolManager() {
        if (msg.sender != address(poolManager)) revert NotPoolManager();
        _;
    }

    /// @inheritdoc IUnlockCallback
    /// @dev We force the onlyPoolManager modifier by exposing a virtual function after the onlyPoolManager check.
    function unlockCallback(bytes calldata data) external onlyPoolManager returns (bytes memory) {
        return _unlockCallback(data);
    }

    /// @dev to be implemented by the child contract, to safely guarantee the logic is only executed by the PoolManager
    function _unlockCallback(bytes calldata data) internal virtual returns (bytes memory);
}
```
**SafeCallback.sol：**用于确保是poolManager调用unlockCallback;先声明一个onlyPoolManager修饰符，用于修饰unlockCallback函数，
在其执行前先判断调用者是否是poolManager。

**V4Router.sol**
```solidity
function _handleAction(uint256 action, bytes calldata params) internal override {
    // swap actions and payment actions in different blocks for gas efficiency
    if (action < Actions.SETTLE) {
        if (action == Actions.SWAP_EXACT_IN) {
            IV4Router.ExactInputParams calldata swapParams = params.decodeSwapExactInParams();
            _swapExactInput(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_IN_SINGLE) {
            IV4Router.ExactInputSingleParams calldata swapParams = params.decodeSwapExactInSingleParams();
            _swapExactInputSingle(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_OUT) {
            IV4Router.ExactOutputParams calldata swapParams = params.decodeSwapExactOutParams();
            _swapExactOutput(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_OUT_SINGLE) {
            IV4Router.ExactOutputSingleParams calldata swapParams = params.decodeSwapExactOutSingleParams();
            _swapExactOutputSingle(swapParams);
            return;
        }
    } else {
        if (action == Actions.SETTLE_ALL) {
            (Currency currency, uint256 maxAmount) = params.decodeCurrencyAndUint256();
            uint256 amount = _getFullDebt(currency);
            if (amount > maxAmount) revert V4TooMuchRequested(maxAmount, amount);
            _settle(currency, msgSender(), amount);
            return;
        } else if (action == Actions.TAKE_ALL) {
            (Currency currency, uint256 minAmount) = params.decodeCurrencyAndUint256();
            uint256 amount = _getFullCredit(currency);
            if (amount < minAmount) revert V4TooLittleReceived(minAmount, amount);
            _take(currency, msgSender(), amount);
            return;
        } else if (action == Actions.SETTLE) {
            (Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
            _settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
            return;
        } else if (action == Actions.TAKE) {
            (Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
            _take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
            return;
        } else if (action == Actions.TAKE_PORTION) {
            (Currency currency, address recipient, uint256 bips) = params.decodeCurrencyAddressAndUint256();
            _take(currency, _mapRecipient(recipient), _getFullCredit(currency).calculatePortion(bips));
            return;
        }
    }
    revert UnsupportedAction(action);
}
```
通过对第一个command对应的inputs进行解码的到actions、params：  
actions：0x070b0e  
params：  
0x000000000000000000000000000000...0000000000000000000000000000000  
0x000000000000000000000000000000...0000000000000000000000000000000  
0x000000000000000000000000000000...0000000000000000000000000000000  
**主要关注command产生的actions，对params（参数）不做讨论**  
具体actions映射看https://github.com/Uniswap/v4-periphery/blob/main/src/libraries/Actions.sol  

第一个command产生的actions分别是07（SWAP_EXACT_IN）、0b（SETTLE）、0e（TAKE）。  
**SWAP_EXACT_IN:**  
```solidity
function _swapExactInput(IV4Router.ExactInputParams calldata params) private {
    unchecked {
        // Caching for gas savings
        uint256 pathLength = params.path.length;
        uint128 amountOut;
        Currency currencyIn = params.currencyIn;
        uint128 amountIn = params.amountIn;
        if (amountIn == ActionConstants.OPEN_DELTA) amountIn = _getFullCredit(currencyIn).toUint128();
        PathKey calldata pathKey;

        for (uint256 i = 0; i < pathLength; i++) {
            pathKey = params.path[i];
            (PoolKey memory poolKey, bool zeroForOne) = pathKey.getPoolAndSwapDirection(currencyIn);
            // The output delta will always be positive, except for when interacting with certain hook pools
            amountOut = _swap(poolKey, zeroForOne, -int256(uint256(amountIn)), pathKey.hookData).toUint128();

            amountIn = amountOut;
            currencyIn = pathKey.intermediateCurrency;
        }

        if (amountOut < params.amountOutMinimum) revert V4TooLittleReceived(params.amountOutMinimum, amountOut);
    }
}
```
先将params进行解码然后进入这个函数中，进行一系列相关处理后根据path（开头设计好的路径）分别调用_swap函数。
```solidity
function _swap(PoolKey memory poolKey, bool zeroForOne, int256 amountSpecified, bytes calldata hookData)
    private
    returns (int128 reciprocalAmount)
{
    // for protection of exactOut swaps, sqrtPriceLimit is not exposed as a feature in this contract
    unchecked {
        BalanceDelta delta = poolManager.swap(
            poolKey,
            IPoolManager.SwapParams(
                zeroForOne, amountSpecified, zeroForOne ? TickMath.MIN_SQRT_PRICE + 1 : TickMath.MAX_SQRT_PRICE - 1
            ),
            hookData
        );

        reciprocalAmount = (zeroForOne == amountSpecified < 0) ? delta.amount1() : delta.amount0();
    }
}
```
调用poolManager.swap函数进行pool交换（flash accounting）。

**DeltaResolver.sol**

```solidity
/// @notice Take an amount of currency out of the PoolManager
/// @param currency Currency to take
/// @param recipient Address to receive the currency
/// @param amount Amount to take
/// @dev Returns early if the amount is 0
function _take(Currency currency, address recipient, uint256 amount) internal {
    if (amount == 0) return;
    poolManager.take(currency, recipient, amount);
}

/// @notice Pay and settle a currency to the PoolManager
/// @dev The implementing contract must ensure that the `payer` is a secure address
/// @param currency Currency to settle
/// @param payer Address of the payer
/// @param amount Amount to send
/// @dev Returns early if the amount is 0
function _settle(Currency currency, address payer, uint256 amount) internal {
    if (amount == 0) return;

    poolManager.sync(currency);
    if (currency.isAddressZero()) {
        poolManager.settle{value: amount}();
    } else {
        _pay(currency, payer, amount);
        poolManager.settle();
    }
}
```
_take() 和 _settle()分别由v4Router._handleAction()调用，用于调用poolManager里的take()和settle函数，使其命令形成闭环。

## 3、Pool Manager合约代码
**用于管理所有的v4-pool；将所有的池的状态和逻辑封装在Pool Manager合约中。**
```solidity
/// @inheritdoc IPoolManager
function unlock(bytes calldata data) external override returns (bytes memory result) {
    if (Lock.isUnlocked()) AlreadyUnlocked.selector.revertWith();

    Lock.unlock();

    // the caller does everything in this callback, including paying what they owe via calls to settle
    result = IUnlockCallback(msg.sender).unlockCallback(data);

    if (NonzeroDeltaCount.read() != 0) CurrencyNotSettled.selector.revertWith();
    Lock.lock();
}
```
用于解锁Pool Manager合约，从而调用合约进行各种操作。先检测合约是否已经解锁，没解锁则继续解锁，反之返回错误，
然后通过调用者合约实现的unlockCallback函数。
***

```solidity
/// @inheritdoc IPoolManager
function swap(PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    external
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta swapDelta)
{
    if (params.amountSpecified == 0) SwapAmountCannotBeZero.selector.revertWith();
    PoolId id = key.toId();
    Pool.State storage pool = _getPool(id);
    pool.checkPoolInitialized();

    BeforeSwapDelta beforeSwapDelta;
    {
        int256 amountToSwap;
        uint24 lpFeeOverride;
        (amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);

        // execute swap, account protocol fees, and emit swap event
        // _swap is needed to avoid stack too deep error
        swapDelta = _swap(
            pool,
            id,
            Pool.SwapParams({
                tickSpacing: key.tickSpacing,
                zeroForOne: params.zeroForOne,
                amountSpecified: amountToSwap,
                sqrtPriceLimitX96: params.sqrtPriceLimitX96,
                lpFeeOverride: lpFeeOverride
            }),
            params.zeroForOne ? key.currency0 : key.currency1 // input token
        );
    }

    BalanceDelta hookDelta;
    (swapDelta, hookDelta) = key.hooks.afterSwap(key, params, swapDelta, hookData, beforeSwapDelta);

    // if the hook doesn't have the flag to be able to return deltas, hookDelta will always be 0
    if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));

    _accountPoolBalanceDelta(key, swapDelta, msg.sender);
}

/// @notice Internal swap function to execute a swap, take protocol fees on input token, and emit the swap event
function _swap(Pool.State storage pool, PoolId id, Pool.SwapParams memory params, Currency inputCurrency)
    internal
    returns (BalanceDelta)
{
    (BalanceDelta delta, uint256 amountToProtocol, uint24 swapFee, Pool.SwapResult memory result) =
        pool.swap(params);

    // the fee is on the input currency
    if (amountToProtocol > 0) _updateProtocolFees(inputCurrency, amountToProtocol);

    // event is emitted before the afterSwap call to ensure events are always emitted in order
    emit Swap(
        id,
        msg.sender,
        delta.amount0(),
        delta.amount1(),
        result.sqrtPriceX96,
        result.liquidity,
        result.tick,
        swapFee
    );

    return delta;
}
```
**swap：**  
先检查交换金额是否为0，为0，触发还原并报错；再检测pool是否初始化；  
先执行一下pool的交换前hook；然后进入_swap执行交换；再执行pool的交换后的hook；  
最后根据前后hook是否影响pool，进行调整。  
**_swap：**  
通过pool.swap更新得到BalanceDelta delta, uint256 amountToProtocol, uint24 swapFee, Pool.SwapResult memory result等结果：  
delta：兑换后余额的变化。  
amountToProtocol：本次动作的协议收取的费用。  
swapFee：本次交易的费用。  
result：本次兑换后的影响包含有关掉期结果的附加信息，例如更新的价格和流动性。
最后更新任何协议收取的费用，确保准确跟踪协议的收入；发出交换事件。
***

```solidity
/// @inheritdoc IPoolManager
function take(Currency currency, address to, uint256 amount) external onlyWhenUnlocked {
    unchecked {
        // negation must be safe as amount is not negative
        _accountDelta(currency, -(amount.toInt128()), msg.sender);
        currency.transfer(to, amount);
    }
}

/// @inheritdoc IPoolManager
function settle() external payable onlyWhenUnlocked returns (uint256) {
    return _settle(msg.sender);
}

// if settling native, integrators should still call `sync` first to avoid DoS attack vectors
function _settle(address recipient) internal returns (uint256 paid) {
    Currency currency = CurrencyReserves.getSyncedCurrency();

    // if not previously synced, or the syncedCurrency slot has been reset, expects native currency to be settled
    if (currency.isAddressZero()) {
        paid = msg.value;
    } else {
        if (msg.value > 0) NonzeroNativeValue.selector.revertWith();
        // Reserves are guaranteed to be set because currency and reserves are always set together
        uint256 reservesBefore = CurrencyReserves.getSyncedReserves();
        uint256 reservesNow = currency.balanceOfSelf();
        paid = reservesNow - reservesBefore;
        CurrencyReserves.resetCurrency();
    }

    _accountDelta(currency, paid.toInt128(), recipient);
}
```
***settle和_settle：***  
根据是否是synced currency分别处理余额，然后使用_accountDelta调整相关账户余额。  
***take:***  
_accountDelta是用于调整账户余额，currency.transfer用户代币转移。


