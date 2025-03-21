## Safe

### 核心：Safe smart account

### 创建 Safe smart account
用户账号与0x4e1DCf7AD4e460CfD30791CCC4F9c8a4f820ec67(SafeProxyFactory)合约地址交互，调用createProxyWithNonce函数；  
创建一个SafeProxy代理合约。
```solidity
// SafeProxyFactory.sol
function deployProxy(address _singleton, bytes memory initializer, bytes32 salt) internal returns (SafeProxy proxy) {
    require(isContract(_singleton), "Singleton contract not deployed");

    bytes memory deploymentData = abi.encodePacked(type(SafeProxy).creationCode, uint256(uint160(_singleton)));
    // solhint-disable-next-line no-inline-assembly
    assembly {
        proxy := create2(0x0, add(0x20, deploymentData), mload(deploymentData), salt)
    }
    require(address(proxy) != address(0), "Create2 call failed");

    if (initializer.length > 0) {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            if eq(call(gas(), proxy, 0, add(initializer, 0x20), mload(initializer), 0, 0), 0) {
                revert(0, 0)
            }
        }
    }
}

function createProxyWithNonce(address _singleton, bytes memory initializer, uint256 saltNonce) public returns (SafeProxy proxy) {
    // If the initializer changes the proxy address should change too. Hashing the initializer data is cheaper than just concatinating it
    bytes32 salt = keccak256(abi.encodePacked(keccak256(initializer), saltNonce));
    proxy = deployProxy(_singleton, initializer, salt);
    emit ProxyCreation(proxy, _singleton);
}
```
_singleton:是已部署的safe合约(0x41675C099F32341bf84BFc5382aF534df5C7461a)。  
initializer:用于生成唯一代理地址的salt值,且是发送到新代理地址的消息调用的有效负载。  
saltNonce:随机数,也用于生成唯一代理地址。  

initializer截取前四个字节执行的是setup函数(safe.sol 逻辑合约)，合约函数详情请参考https://docs.safe.global/reference-smart-account/setup/setup 。  

补充:solidity的代理模式是数据与逻辑分成两个合约即代理合约和逻辑合约，调用者通过代理合约委托执行逻辑合约里的函数，数据存储在代理合约中，执行结果返回调用者。

### safe smart account 管理
管理safe smart account的相关信息(对Owners的添加、更改等以及更改threshold)  
操作逻辑代码在OwnerManager.sol用于更改合约存储内容。  
说明owners是一个存储owner的循环链表结构，key(address)=>value(address);SENTINEL_OWNERS是一个address(0x1);
1. 在初始化safe smart account的时候调用了setupOwners函数：  
```solidity
// Threshold can only be 0 at initialization.
// Check ensures that the setup function can only be called once.
if (threshold > 0) revertWithError("GS200");
// Validate that the threshold is smaller than the number of added owners.
if (_threshold > _owners.length) revertWithError("GS201");
// There has to be at least one Safe owner.
if (_threshold == 0) revertWithError("GS202");
```
检查合约的初始化状态，以及输入的参数是否符合要求。
```solidity
// Initializing Safe owners.
address currentOwner = SENTINEL_OWNERS;
uint256 ownersLength = _owners.length;
for (uint256 i = 0; i < ownersLength; ++i) {
    // Owner address cannot be null.
    address owner = _owners[i];
    if (owner == address(0) || owner == SENTINEL_OWNERS || owner == address(this) || currentOwner == owner)
        revertWithError("GS203");
    // No duplicate owners allowed.
    if (owners[owner] != address(0)) revertWithError("GS204");
    owners[currentOwner] = owner;
    currentOwner = owner;
}
owners[currentOwner] = SENTINEL_OWNERS;
ownerCount = ownersLength;
threshold = _threshold;
```
通过链表存储owners,用owners[0x1]当作链表第一个元素,并记录owners数量和threshold。

2. addOwnerWithThreshold函数;用于添加owner以及更改threshold:  
```solidity
function addOwnerWithThreshold(address owner, uint256 _threshold) public override authorized {
    // Owner address cannot be null, the sentinel or the Safe itself.
    if (owner == address(0) || owner == SENTINEL_OWNERS || owner == address(this)) revertWithError("GS203");
    // No duplicate owners allowed.
    if (owners[owner] != address(0)) revertWithError("GS204");
    owners[owner] = owners[SENTINEL_OWNERS];
    owners[SENTINEL_OWNERS] = owner;
    ++ownerCount;
    emit AddedOwner(owner);
    // Change threshold if threshold was changed.
    if (threshold != _threshold) changeThreshold(_threshold);
}
```
在owners中创建一个链表元素owner指向0x1指向的元素,再将0x1指向owner;然后增加ownerCount;然后调用changeThreshold函数。  

3. changeThreshold函数,用于更改threshold:  
```solidity
function changeThreshold(uint256 _threshold) public override authorized {
    // Validate that threshold is smaller than number of owners.
    if (_threshold > ownerCount) revertWithError("GS201");
    // There has to be at least one Safe owner.
    if (_threshold == 0) revertWithError("GS202");
    threshold = _threshold;
    emit ChangedThreshold(_threshold);
}
```
和初始化一样，检测输入的参数是否符合要求，符合更改threshold。

以上是节选三个功能，可通过官方文档查看具体功能:  
https://docs.safe.global/reference-smart-account/owners/addOwnerWithThreshold   
智能合约代码:  
https://github.com/safe-global/safe-smart-account/blob/main/contracts/base/OwnerManager.sol

### 提交议案&签名验证
1. 通过safe客户端进行页面操作
2. 可以通过safe提供的Safe Core SDK进行操作，可参考:  
https://github.com/safe-global/safe-core-sdk/blob/main/guides/integrating-the-safe-core-sdk.md  
提供的三个工具包@safe-global/types-kit、@safe-global/protocol-kit、@safe-global/api-kit  
@safe-global/protocol-kit:应该是链接智能合约功能以及对safe交易签名;  
@safe-global/api-kit:与其提供的后段服务连接,将提交的议案以及签名提交到他们的Safe Transaction Service后端服务。  

注:  
生成的交易是safe交易,通过protocolKit.createTransaction生成;  
生成safe交易后,再通过protocolKit.getTransactionHash计算的到safeTxHash;  
最后通过protocolKit.signHash对safeTxHash进行签名，通过apiKit.proposeTransaction提交到后端服务。
```solidity
await apiKit.proposeTransaction({
  safeAddress,
  safeTransactionData: safeTransaction.data,
  safeTxHash,
  senderAddress,
  senderSignature: senderSignature.data,
  origin
})
```
通过几次原生交易、修改信息交易和erc20代币交易猜测:  
如果是触及ERC20交易、nft交易和safe smart account信息交易构建safe交易时候:
```solidity
const transactions: MetaTransactionData[] = [
  {
    to,
    data,
    value,
    operation
  },
  {
    to,
    data,
    value,
    operation
  }
  // ...
]
```
to是对应的代理合约地址,data是对应的函数签名+参数;value:0。

参考交易(polygon):  
0x5dcec9eb71436b2f642ed530e5baacbc29230862777e80ef36a70074cf00ebde  
其中的data是:  
0xa9059cbb  对应transfer(address,uint256)	
000000000000000000000000646672c0aac59b26499d971741e94bad8d20d710  对应接受方
000000000000000000000000000000000000000000000000168cee8c9b36fcfe  对应数量

如果是原生代币交易:  
to是接受方,data是空;value:数量。 

参考交易(polygon):  
0xcd5dc8be7fedb1dfa22fb8272ddcd6c81f9dc87bae994d3fa5465b64082eac39  

### 执行交易
在提交一个议案，并且达到阙值，由某个用户执行发送交易将议案发送到safe smart account中:  
执行execTransaction函数
```solidity
// 核心代码
// safe.sol
// 检测参数是否正确以及签名是否正确
txHash = getTransactionHash( // Transaction info
    to,
    value,
    data,
    operation,
    safeTxGas,
    // Payment info
    baseGas,
    gasPrice,
    gasToken,
    refundReceiver,
    // Signature info
    // We use the post-increment here, so the current nonce value is used and incremented afterwards.
    nonce++
);
checkSignatures(msg.sender, txHash, signatures);
// 执行
success = execute(to, value, data, operation, gasPrice == 0 ? (gasleft() - 2500) : safeTxGas);

// Executor.sol
function execute(
    address to,
    uint256 value,
    bytes memory data,
    Enum.Operation operation,
    uint256 txGas
) internal returns (bool success) {
    if (operation == Enum.Operation.DelegateCall) {
        /* solhint-disable no-inline-assembly */
        /// @solidity memory-safe-assembly
        assembly {
            success := delegatecall(txGas, to, add(data, 0x20), mload(data), 0, 0)
        }
        /* solhint-enable no-inline-assembly */
    } else {
        /* solhint-disable no-inline-assembly */
        /// @solidity memory-safe-assembly
        assembly {
            success := call(txGas, to, value, add(data, 0x20), mload(data), 0, 0)
        }
        /* solhint-enable no-inline-assembly */
    }
}
```
execute函数是交易执行函数，根据operation判断是否是否需要委托调用(delegatecall);  
根据几次操作,并没有触发delegatecall,所以仅讨论call:
1. 如果提案中to是eov地址,则to是接收方、value为数额、data为null；
2. 如果提安中to是合约地址,则to是合约地址、value为0、data为调用函数+参数；

参考交易(polygon):  
0xcd5dc8be7fedb1dfa22fb8272ddcd6c81f9dc87bae994d3fa5465b64082eac39  
0xd519760fb6ea9be636f30c05ba4f8670c99ff0d30baac2c7348c3f0e3853ba29  

补充:
如果交易的不是原生,则通过不同的ERC20代理合约触发转账。

参考交易(polygon):  
0x5dcec9eb71436b2f642ed530e5baacbc29230862777e80ef36a70074cf00ebde

官方文档参数说明:  
https://docs.safe.global/reference-smart-account/transactions/execTransaction  
智能合约代码:  
https://github.com/safe-global/safe-smart-account/blob/main/contracts/base/Executor.sol

注:  
```solidity
// 检测签名所有者
if (currentOwner <= lastOwner || owners[currentOwner] == address(0) || currentOwner == SENTINEL_OWNERS)
    revertWithError("GS026");
lastOwner = currentOwner;
```
如果但前所有者小于等于上一个所有者 则是重复签名;  
如果但前所有者映射为address(0) 则表示不是合法所有者;  
如果所有者是哨兵地址address(0x1) 则也表示不是合法所有者(第二种的特殊情况);

