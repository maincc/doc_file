## uniswap-v4:Hooks  TWAMM

具体实际流程可能是  
1、部署Hooks合约  
2、在poolManage合约中创建pool,触发beforeInitialize函数。  
3、可以添加流动性或者提交大额长期订单。不同的是添加流动性触发beforeAddLiquidity函数,提交订单是提交到部署的合约中。  
4、在添加流动性和提交了订单,可以在交易所进行交易,触发beforeSwap函数。  
5、后续可以选择更新订单也可以从合约中领取代币。

在其过程中核心函数是executeTWAMMOrders,在提交、更新订单和触发函数中都有执行executeTWAMMOrders。

### 1、创建部署TWAMM Hooks合约以及提交订单
```solidity
constructor(IPoolManager _manager, uint256 _expirationInterval) BaseHook(_manager) {
    expirationInterval = _expirationInterval;
}
```
智能合约的构造函数，设置expirationInterval参数。  
expirationInterval：到期间隔，用于分割时间。
***

```solidity
/// @inheritdoc ITWAMM
function submitOrder(PoolKey calldata key, OrderKey memory orderKey, uint256 amountIn)
    external
    returns (bytes32 orderId)
{
    PoolId poolId = PoolId.wrap(keccak256(abi.encode(key)));
    State storage twamm = twammStates[poolId];
    executeTWAMMOrders(key);

    uint256 sellRate;
    unchecked {
        // checks done in TWAMM library
        uint256 duration = orderKey.expiration - block.timestamp;
        sellRate = amountIn / duration;
        orderId = _submitOrder(twamm, orderKey, sellRate);
        IERC20Minimal(orderKey.zeroForOne ? Currency.unwrap(key.currency0) : Currency.unwrap(key.currency1))
            .safeTransferFrom(msg.sender, address(this), sellRate * duration);
    }

    emit SubmitOrder(
        poolId,
        orderKey.owner,
        orderKey.expiration,
        orderKey.zeroForOne,
        sellRate,
        _getOrder(twamm, orderKey).earningsFactorLast
    );
}

/// @notice Submits a new long term order into the TWAMM
/// @dev executeTWAMMOrders must be executed up to current timestamp before calling submitOrder
/// @param orderKey The OrderKey for the new order
function _submitOrder(State storage self, OrderKey memory orderKey, uint256 sellRate)
    internal
    returns (bytes32 orderId)
{
    if (orderKey.owner != msg.sender) revert MustBeOwner(orderKey.owner, msg.sender);
    if (self.lastVirtualOrderTimestamp == 0) revert NotInitialized();
    if (orderKey.expiration <= block.timestamp) revert ExpirationLessThanBlocktime(orderKey.expiration);
    if (sellRate == 0) revert SellRateCannotBeZero();
    if (orderKey.expiration % expirationInterval != 0) revert ExpirationNotOnInterval(orderKey.expiration);

    orderId = _orderId(orderKey);
    if (self.orders[orderId].sellRate != 0) revert OrderAlreadyExists(orderKey);

    OrderPool.State storage orderPool = orderKey.zeroForOne ? self.orderPool0For1 : self.orderPool1For0;

    unchecked {
        orderPool.sellRateCurrent += sellRate;
        orderPool.sellRateEndingAtInterval[orderKey.expiration] += sellRate;
    }

    self.orders[orderId] = Order({sellRate: sellRate, earningsFactorLast: orderPool.earningsFactorCurrent});
}

struct Order {
    uint256 sellRate;
    uint256 earningsFactorLast;
}
    
function _orderId(OrderKey memory key) private pure returns (bytes32) {
    return keccak256(abi.encode(key));
}
```
用于提交TWAMM订单。  
**submitOrder:**  
1. 先通过key获取poold，然后通过twammStates映射获取v4流动池的twamm(应该是这样)，再执行executeTWAMMOrder函数。
2. 获取sellRate(出售率 每秒卖出出售的数量)，根据_submitOrder获取到orderId。
3. 通过IERC20Minimal(TransferHelper)转移代币到TWAMM智能合约中；记录log，返回orderId。

**_submitOrder:**  
1. 先通过一些检测，确定信息是否准确。
2. 通过_orderId将orderKey转化成orderId，通过submitOrder中获取的twamm.orders映射，判断订单是否已经存在。
3. 根据orderKey的zeroForOne(交易方向)获取对应的orderPool，更新orderPool;在twamm.orders数组中添加order。

### 2、executeTWAMMOrders(核心函数)用于执行TWAMM订单
```solidity
using StateLibrary for IPoolManager;
using CurrencySettler for Currency;

/// @inheritdoc ITWAMM
function executeTWAMMOrders(PoolKey memory key) public {
    PoolId poolId = key.toId();
    (uint160 sqrtPriceX96,,,) = manager.getSlot0(poolId);
    State storage twamm = twammStates[poolId];

    (bool zeroForOne, uint160 sqrtPriceLimitX96) =
        _executeTWAMMOrders(twamm, manager, key, PoolParamsOnExecute(sqrtPriceX96, manager.getLiquidity(poolId)));

    if (sqrtPriceLimitX96 != 0 && sqrtPriceLimitX96 != sqrtPriceX96) {
        manager.unlock(abi.encode(key, IPoolManager.SwapParams(zeroForOne, type(int256).max, sqrtPriceLimitX96)));
    }
}

function _unlockCallback(bytes calldata rawData) internal override returns (bytes memory) {
    (PoolKey memory key, IPoolManager.SwapParams memory swapParams) =
        abi.decode(rawData, (PoolKey, IPoolManager.SwapParams));

    BalanceDelta delta = manager.swap(key, swapParams, ZERO_BYTES);

    if (swapParams.zeroForOne) {
        if (delta.amount0() < 0) {
            key.currency0.settle(manager, address(this), uint256(uint128(-delta.amount0())), false);
        }
        if (delta.amount1() > 0) {
            key.currency1.take(manager, address(this), uint256(uint128(delta.amount1())), false);
        }
    } else {
        if (delta.amount1() < 0) {
            key.currency1.settle(manager, address(this), uint256(uint128(-delta.amount1())), false);
        }
        if (delta.amount0() > 0) {
            key.currency0.take(manager, address(this), uint256(uint128(delta.amount0())), false);
        }
    }
    return bytes("");
}
```
**executeTWAMMOrders:**  
1. 通过PoolKey得到poolId，从manager(IPoolManager)中获取sqrtPriceX96和从twammStates映射中获取twamm。
2. 通过_executeTWAMMOrders获取zeroForOne(交易方向)、sqrtPriceLimitX96(价格限制)。
3. 如果sqrtPriceLimitX96不等于0以及不等于sqrtPriceX96，就通过manager将PoolManager智能合约解锁，再通过
PoolManager调用_unlockCallback。

**_unlockCallback:**
1. 解压rawData数据，获取manager.swap参数并执行swap，得到delta(BalanceDelta)数据。
2. 通过交易方向再根据delta的amount0、amount1是否大小于0进行分类操作。

**注:** manager.getSlot0在StateLibrary中实现;key.currency.settle、.take在CurrencySettler中实现;
***

**_executeTWAMMOrders:**  
```solidity
if (!_hasOutstandingOrders(self)) {
    self.lastVirtualOrderTimestamp = block.timestamp;
    return (false, 0);
}
```
_hasOutstandingOrders:用于检测是否还有长期订单;如果没有长期订单，则更新twamm.lastVirtualOrderTimestamp返回。

```solidity
uint160 initialSqrtPriceX96 = pool.sqrtPriceX96;
uint256 prevTimestamp = self.lastVirtualOrderTimestamp;
uint256 nextExpirationTimestamp = prevTimestamp + (expirationInterval - (prevTimestamp % expirationInterval));

OrderPool.State storage orderPool0For1 = self.orderPool0For1;
OrderPool.State storage orderPool1For0 = self.orderPool1For0;
```
获取initialSqrtPriceX96、prevTimestamp、nextExpirationTimestamp、orderPool0For1和orderPool1For0： 
prevTimestamp:上一个已处理的结束时间戳戳。  
nextExpirationTimestamp:当前处理的结束时间戳。  
orderPool0For1:token0兑换token1的单向池。  
orderPool1For0:token1兑换token0的单向池。

```solidity
while (nextExpirationTimestamp <= block.timestamp) {
    if (
        orderPool0For1.sellRateEndingAtInterval[nextExpirationTimestamp] > 0
            || orderPool1For0.sellRateEndingAtInterval[nextExpirationTimestamp] > 0
    ) {
        if (orderPool0For1.sellRateCurrent != 0 && orderPool1For0.sellRateCurrent != 0) {
            pool = _advanceToNewTimestamp(
                self,
                manager,
                key,
                AdvanceParams(
                    expirationInterval,
                    nextExpirationTimestamp,
                    nextExpirationTimestamp - prevTimestamp,
                    pool
                )
            );
        } else {
            pool = _advanceTimestampForSinglePoolSell(
                self,
                manager,
                key,
                AdvanceSingleParams(
                    expirationInterval,
                    nextExpirationTimestamp,
                    nextExpirationTimestamp - prevTimestamp,
                    pool,
                    orderPool0For1.sellRateCurrent != 0
                )
            );
        }
        prevTimestamp = nextExpirationTimestamp;
    }
    nextExpirationTimestamp += expirationInterval;

    if (!_hasOutstandingOrders(self)) break;
}
```
一个循环，根据当前处理的结束时间戳是否小于等于当前时间戳，处理order pool里的部分内容，如果当前order pool没有order
则退出循环。  
在循环中，根据orderPool0For1或orderPool1For0的sellRateEndingAtInterval映射是否大于0（用于检测当前处理的时间间隔是否存在订单）。
存在订单则根据orderPool0For1和orderPool1For0的sellRateCurrent是否不等于0（用于区分双向订单还是单向订单）；如果是双向则通过
_advanceToNewTimestamp获取执行后的pool，反之则通过 _advanceTimestampForSinglePoolSell获取执行后的pool。

```solidity
if (prevTimestamp < block.timestamp && _hasOutstandingOrders(self)) {
    if (orderPool0For1.sellRateCurrent != 0 && orderPool1For0.sellRateCurrent != 0) {
        pool = _advanceToNewTimestamp(
            self,
            manager,
            key,
            AdvanceParams(expirationInterval, block.timestamp, block.timestamp - prevTimestamp, pool)
        );
    } else {
        pool = _advanceTimestampForSinglePoolSell(
            self,
            manager,
            key,
            AdvanceSingleParams(
                expirationInterval,
                block.timestamp,
                block.timestamp - prevTimestamp,
                pool,
                orderPool0For1.sellRateCurrent != 0
            )
        );
    }
}
```
退出循环后，根据已经处理的结束时间戳和是否存在订单决定是否再一次执行循环内的程序(可能存在最后一个不完整时间间隔)。

```solidity
self.lastVirtualOrderTimestamp = block.timestamp;
newSqrtPriceX96 = pool.sqrtPriceX96;
zeroForOne = initialSqrtPriceX96 > newSqrtPriceX96;
```
更新所处理的twamm的lastVirtualOrderTimestamp为当前时间戳，根据上文的initialSqrtPriceX96是否大于pool.sqrtPriceX96（newSqrtPriceX96）
返回zeroForOne(交易方向)。  

该函数返回{zeroForOne, newSqrtPriceX96}。
***

**_advanceToNewTimestamp:**  
```solidity
struct AdvanceParams {
    uint256 expirationInterval;
    uint256 nextTimestamp;
    uint256 secondsElapsed;
    PoolParamsOnExecute pool;
}
```
TWAMM.sol 创建的结构体。

```solidity
State storage self, IPoolManager manager, PoolKey memory poolKey, AdvanceParams memory params
```
参数

```solidity
uint160 finalSqrtPriceX96;
uint256 secondsElapsedX96 = params.secondsElapsed * FixedPoint96.Q96;

OrderPool.State storage orderPool0For1 = self.orderPool0For1;
OrderPool.State storage orderPool1For0 = self.orderPool1For0;
```
定义finalSqrtPriceX96；  
secondsElapsedX96:将secondsElapsed(nextExpirationTimestamp - prevTimestamp)转化成Q96定点数格式。

```solidity
while (true) {
    TwammMath.ExecutionUpdateParams memory executionParams = TwammMath.ExecutionUpdateParams(
        secondsElapsedX96,
        params.pool.sqrtPriceX96,
        params.pool.liquidity,
        orderPool0For1.sellRateCurrent,
        orderPool1For0.sellRateCurrent
    );

    finalSqrtPriceX96 = TwammMath.getNewSqrtPriceX96(executionParams);

    (bool crossingInitializedTick, int24 tick) =
        _isCrossingInitializedTick(params.pool, manager, poolKey, finalSqrtPriceX96);
    unchecked {
        if (crossingInitializedTick) {
            uint256 secondsUntilCrossingX96;
            (params.pool, secondsUntilCrossingX96) = _advanceTimeThroughTickCrossing(
                self,
                manager,
                poolKey,
                TickCrossingParams(tick, params.nextTimestamp, secondsElapsedX96, params.pool)
            );
            secondsElapsedX96 = secondsElapsedX96 - secondsUntilCrossingX96;
        } else {
            (uint256 earningsFactorPool0, uint256 earningsFactorPool1) =
                TwammMath.calculateEarningsUpdates(executionParams, finalSqrtPriceX96);

            if (params.nextTimestamp % params.expirationInterval == 0) {
                orderPool0For1.advanceToInterval(params.nextTimestamp, earningsFactorPool0);
                orderPool1For0.advanceToInterval(params.nextTimestamp, earningsFactorPool1);
            } else {
                orderPool0For1.advanceToCurrentTime(earningsFactorPool0);
                orderPool1For0.advanceToCurrentTime(earningsFactorPool1);
            }
            params.pool.sqrtPriceX96 = finalSqrtPriceX96;
            break;
        }
    }
}
```
定义一个死循环,在无tick跨越的情况下,更新pool状态退出循环。  
先通过TwammMath.getNewSqrtPriceX96获得finalSqrtPriceX96;通过_isCrossingInitializedTick检测交易过程中是否存在初始化的tick
并返回crossingInitializedTick和tick;  
根据crossingInitializedTick判断是否发生了tick跨越;发生tick跨越,通过_advanceTimeThroughTickCrossing得到更新的pool以及到下一个tick的时间(secondsUntilCrossingX96),
更新secondsElapsedX96(更新剩余时间段);没有发生tick跨越,计算pool里双向流动池的收益,根据交易结束时间是否能被定义的时间间隔整出,能的话更新当前收益、区间收益、卖出
率，反之则只更新当前收益。  
最后,更新pool.sqrtPriceX96。
***

**_advanceTimestampForSinglePoolSell:**
```solidity
struct AdvanceSingleParams {
    uint256 expirationInterval;
    uint256 nextTimestamp;
    uint256 secondsElapsed;
    PoolParamsOnExecute pool;
    bool zeroForOne;
}
```
TWAMM.sol 创建的结构体。
```solidity
State storage self, IPoolManager manager, PoolKey memory poolKey, AdvanceSingleParams memory params
```
参数

```solidity
OrderPool.State storage orderPool = params.zeroForOne ? self.orderPool0For1 : self.orderPool1For0;
uint256 sellRateCurrent = orderPool.sellRateCurrent;
uint256 amountSelling = sellRateCurrent * params.secondsElapsed;
uint256 totalEarnings;
```
先根据zeroForOne判断出是哪一个单向流动池,获取这个流动池的卖出率(sellRateCurrent),计算出卖出的数量，定义一个totalEarnings。

```solidity
while (true) {
    ...
}

return params.pool;
```
定义一个死循环,退出循环返回pool。  

**循环内逻辑:**  
_TwammMath:_ 用于TWAMM数学计算的纯函数。  
_SqrtPriceMath:_ 基于Q64.94的平方根价格和流动性函数。

```solidity
uint160 finalSqrtPriceX96 = SqrtPriceMath.getNextSqrtPriceFromInput(
    params.pool.sqrtPriceX96, params.pool.liquidity, amountSelling, params.zeroForOne
);

(bool crossingInitializedTick, int24 tick) =
    _isCrossingInitializedTick(params.pool, manager, poolKey, finalSqrtPriceX96);
```
和_advanceToNewTimestamp相同,先通过SqrtPriceMath.getNextSqrtPriceFromInput计算出finalSqrtPriceX96，再通过
_isCrossingInitializedTick计算出crossingInitializedTick和tick。

```solidity
if (crossingInitializedTick) {
    (, int128 liquidityNetAtTick) = manager.getTickLiquidity(poolKey.toId(), tick);
    uint160 initializedSqrtPrice = TickMath.getSqrtPriceAtTick(tick);

    uint256 swapDelta0 = SqrtPriceMath.getAmount0Delta(
        params.pool.sqrtPriceX96, initializedSqrtPrice, params.pool.liquidity, true
    );
    uint256 swapDelta1 = SqrtPriceMath.getAmount1Delta(
        params.pool.sqrtPriceX96, initializedSqrtPrice, params.pool.liquidity, true
    );

    params.pool.liquidity = params.zeroForOne
        ? params.pool.liquidity - uint128(liquidityNetAtTick)
        : params.pool.liquidity + uint128(-liquidityNetAtTick);
    params.pool.sqrtPriceX96 = initializedSqrtPrice;

    unchecked {
        totalEarnings += params.zeroForOne ? swapDelta1 : swapDelta0;
        amountSelling -= params.zeroForOne ? swapDelta0 : swapDelta1;
    }
```
如果检测到需要tick跨越则进去上面那段代码。先通过manager.getTickLiquidity()获取liquidityNetAtTick;通过TickMath.getSqrtPriceAtTick将
tick按照sqrt(1.0001^tick) * 2^96这个公式得到initializedSqrtPrice;通过SqrtPriceMath里的getAmount0Delta和getAmount1Delta分别计算出
swapDelta0(token0变化余额)和swapDelta1(token1变化余额);然后更新params.pool的liquidity和sqrtPriceX96、totalEarning、amountSelling。

```solidity
} else {
    if (params.zeroForOne) {
        totalEarnings += SqrtPriceMath.getAmount1Delta(
            params.pool.sqrtPriceX96, finalSqrtPriceX96, params.pool.liquidity, true
        );
    } else {
        totalEarnings += SqrtPriceMath.getAmount0Delta(
            params.pool.sqrtPriceX96, finalSqrtPriceX96, params.pool.liquidity, true
        );
    }

    uint256 accruedEarningsFactor = (totalEarnings * FixedPoint96.Q96) / sellRateCurrent;

    if (params.nextTimestamp % params.expirationInterval == 0) {
        orderPool.advanceToInterval(params.nextTimestamp, accruedEarningsFactor);
    } else {
        orderPool.advanceToCurrentTime(accruedEarningsFactor);
    }
    params.pool.sqrtPriceX96 = finalSqrtPriceX96;
    break;
}
```
如果检测无tick跨越则进入上面这段代码。根据zeroForOne(交易方向)更新totalEarnings;然后根据totalEarnings计算出accruedEarningsFactor;
根据交易结束时间是否能被定义的时间间隔整出,能的话更新当前收益、区间收益、卖出 率，反之则只更新当前收益。  
更新pool.sqrtPriceX96,退出循环。

### 3、更新订单 updateOrder
**updateOrder:**  
```solidity
function updateOrder(PoolKey memory key, OrderKey memory orderKey, int256 amountDelta)
    external
    returns (uint256 tokens0Owed, uint256 tokens1Owed)
{
    PoolId poolId = PoolId.wrap(keccak256(abi.encode(key)));
    State storage twamm = twammStates[poolId];

    executeTWAMMOrders(key);

    // This call reverts if the caller is not the owner of the order
    (uint256 buyTokensOwed, uint256 sellTokensOwed, uint256 newSellrate, uint256 newEarningsFactorLast) =
        _updateOrder(twamm, orderKey, amountDelta);

    if (orderKey.zeroForOne) {
        tokens0Owed += sellTokensOwed;
        tokens1Owed += buyTokensOwed;
    } else {
        tokens0Owed += buyTokensOwed;
        tokens1Owed += sellTokensOwed;
    }

    tokensOwed[key.currency0][orderKey.owner] += tokens0Owed;
    tokensOwed[key.currency1][orderKey.owner] += tokens1Owed;

    if (amountDelta > 0) {
        IERC20Minimal(orderKey.zeroForOne ? Currency.unwrap(key.currency0) : Currency.unwrap(key.currency1))
            .safeTransferFrom(msg.sender, address(this), uint256(amountDelta));
    }

    emit UpdateOrder(
        poolId, orderKey.owner, orderKey.expiration, orderKey.zeroForOne, newSellrate, newEarningsFactorLast
    );
}
```
根据key计算出所需要的流动池的poolId,再根据poolId通过twammStates映射得到twamm;通过key执行executeTWAMMOrders函数;
根据_updateOrder获取buyTokensOwed、sellTokensOwed、newSellrate、newEarningsFactorLast;根据orderKey.zeroForOne更新tokens0Owed和tokens1Owed;
根据tokens0Owed和tokens1Owed更新tokensOwed(可领取代币数量数组)里的数据。判断amountDelta大于0,如果符合则从msg.sender交易代币到这个合约内;
发送更新订单事件并记录log。
***

**_updateOrder:**  
```solidity
function _updateOrder(State storage self, OrderKey memory orderKey, int256 amountDelta)
    internal
    returns (uint256 buyTokensOwed, uint256 sellTokensOwed, uint256 newSellRate, uint256 earningsFactorLast)
{
    Order storage order = _getOrder(self, orderKey);
    OrderPool.State storage orderPool = orderKey.zeroForOne ? self.orderPool0For1 : self.orderPool1For0;

    if (orderKey.owner != msg.sender) revert MustBeOwner(orderKey.owner, msg.sender);
    if (order.sellRate == 0) revert OrderDoesNotExist(orderKey);
    if (amountDelta != 0 && orderKey.expiration <= block.timestamp) revert CannotModifyCompletedOrder(orderKey);
```
通过twamm和orderKey获取订单;根据orderKey.zeroForOne获取对应的订单池内单向流动池;  
检测订单的所有者是否和调用合约者一致;检测订单的卖出价不为0;检测要修改的订单是否是已经完成的订单。

```solidity
uint256 earningsFactor = orderPool.earningsFactorCurrent - order.earningsFactorLast;
buyTokensOwed = (earningsFactor * order.sellRate) >> FixedPoint96.RESOLUTION;
earningsFactorLast = orderPool.earningsFactorCurrent;
order.earningsFactorLast = earningsFactorLast;

if (orderKey.expiration <= block.timestamp) {
    delete self.orders[_orderId(orderKey)];
}
```
先通过orderPool当前收益因子减去要修改订单的最新收益因子获取该订单未结算期间的收益因子差值(earningsFactor);
通过earningsFactor和order.sellRate得到buyTokensOwed;最后将order的最新收益因子更新为ordePool的当前收益因子。  
如果订单已经到期则在self.orders(twamm)删除这个订单。

```solidity
if (amountDelta != 0) {
    uint256 duration = orderKey.expiration - block.timestamp;
    uint256 unsoldAmount = order.sellRate * duration;
    if (amountDelta == MIN_DELTA) amountDelta = -(unsoldAmount.toInt256());
    int256 newSellAmount = unsoldAmount.toInt256() + amountDelta;
    if (newSellAmount < 0) revert InvalidAmountDelta(orderKey, unsoldAmount, amountDelta);

    newSellRate = uint256(newSellAmount) / duration;

    if (amountDelta < 0) {
        uint256 sellRateDelta = order.sellRate - newSellRate;
        orderPool.sellRateCurrent -= sellRateDelta;
        orderPool.sellRateEndingAtInterval[orderKey.expiration] -= sellRateDelta;
        sellTokensOwed = uint256(-amountDelta);
    } else {
        uint256 sellRateDelta = newSellRate - order.sellRate;
        orderPool.sellRateCurrent += sellRateDelta;
        orderPool.sellRateEndingAtInterval[orderKey.expiration] += sellRateDelta;
    }
    if (newSellRate == 0) {
        delete self.orders[_orderId(orderKey)];
    } else {
        order.sellRate = newSellRate;
    }
}
```
如果传入的amountDelta不等0:  
先获取订单还剩多少时间;再获取还剩多少未售出数量;如果amountDelta等于MIN_DELTA(-1),则将amountDelta设置成负数的unsoldAmount;
定义一个newSellAmount表示新的出售数量并设置为unsoldAmount.toInt256() + amountDelta;判断newSellAmount是否小于0,是的话回滚;
通过newSellAmount和duration重新计算出newSellRate。 

判断amountDelta是否小于0:  
小于0(减少订单数量):计算出出售速率的减少量,更新orderPool的sellRateCurrent和sellRateEndingAtInterval;将sellTokensOwed设置成
负数的amountDelta。  
大于0(增加订单数量):计算出出售速率的增加量,更新orderPool的sellRateCurrent和sellRateEndingAtInterval;

判断newSellRate是否等于0:  
等于0:删除订单。  
不等于0:更新order.sellRate。

### 4、claimTokens 领取代币
```solidity
/// @inheritdoc ITWAMM
function claimTokens(Currency token, address to, uint256 amountRequested)
    external
    returns (uint256 amountTransferred)
{
    uint256 currentBalance = token.balanceOfSelf();
    amountTransferred = tokensOwed[token][msg.sender];
    if (amountRequested != 0 && amountRequested < amountTransferred) amountTransferred = amountRequested;
    if (currentBalance < amountTransferred) amountTransferred = currentBalance; // to catch precision errors
    tokensOwed[token][msg.sender] -= amountTransferred;
    IERC20Minimal(Currency.unwrap(token)).safeTransfer(to, amountTransferred);
}
```
先获取要领取代币的数量(currentBalance),再将amountTransferred设置成tokensOwed[token][msg.sender] (存储的用户可领取的代币);  
判断amountRequested不等于0并且小于amountTransferred:符合条件则将amountTransferred设置成amountRequested;  
判断currentBalance小于amountTransferred;符合条件则将amountTransferred设置成currentBalance;  
更新tokensOwed[token][msg.sender]数据,然后通过safeTransfer进行转账。

### 5、beforeInitialize (uniswap-v4 创建池之前触发函数)
```solidity
struct State {
    uint256 lastVirtualOrderTimestamp;
    OrderPool.State orderPool0For1;
    OrderPool.State orderPool1For0;
    mapping(bytes32 => Order) orders;
}

function beforeInitialize(address, PoolKey calldata key, uint160, bytes calldata)
    external
    virtual
    override
    onlyByManager
    returns (bytes4)
{
    // one-time initialization enforced in PoolManager
    initialize(_getTWAMM(key));
    return BaseHook.beforeInitialize.selector;
}

/// @notice Initialize TWAMM state
function initialize(State storage self) internal {
    self.lastVirtualOrderTimestamp = block.timestamp;
}

function _getTWAMM(PoolKey memory key) private view returns (State storage) {
    return twammStates[PoolId.wrap(keccak256(abi.encode(key)))];
}
```
根据调用合约方提供的PoolKey获取twammStates对应的映射,如果没找到就返回默认结构体,再将state.lastVirtualOrderTimestamp设置成当前区块时间。

### 6、beforeAddLiquidity和beforeSwap (uniswap-v4 添加流动性和交易前触发函数)
```solidity
function beforeAddLiquidity(
    address,
    PoolKey calldata key,
    IPoolManager.ModifyLiquidityParams calldata,
    bytes calldata
) external override onlyByManager returns (bytes4) {
    executeTWAMMOrders(key);
    return BaseHook.beforeAddLiquidity.selector;
}

function beforeSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata, bytes calldata)
    external
    override
    onlyByManager
    returns (bytes4, BeforeSwapDelta, uint24)
{
    executeTWAMMOrders(key);
    return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
}
```
在uniswap-v4中进行添加流动性和交易之前,执行executeTWAMMOrders函数。



