## 多链部署Safe账号(safe smart account)

### 1、合约层面说明：
1、部署safe账号，需要在链上部署两个合约SafeProxyFactory(代理合约工厂)、safe合约(逻辑合约)。  
2、构建交易通过SafeProxyFactory使用create2部署safe账号(safeproxy)，在SafeProxyFactory内部调用safe里的setup方法完成safe账号的初始化。  

注:   
1、通过create2部署合约，可以提前预测contract address。create2通过sender(调用者), salt(盐), bytecode(待部署合约的字节码)生成地址。  
2、safe账号通过初始化的参数以及用户传入的saltNonce使用keccak256生成create2的salt;大体这样，根据实际业务可能需要将saltNonce与其他参数进行keccak256处理然后再与初始化参数一起处理。  
3、对于是否可以多链部署，根据部署交易调用是否是createProxyWithNonc。

```solidity
// SafeProxyFactory.sol
// 交易调用的是createProxyWithNonce或createProxyWithCallback
// initializer是内部调用safe交易的calldata
function deployProxyWithNonce(
    address _singleton,
    bytes memory initializer,
    uint256 saltNonce
) internal returns (GnosisSafeProxy proxy) {
    // If the initializer changes the proxy address should change too. Hashing the initializer data is cheaper than just concatinating it
    bytes32 salt = keccak256(abi.encodePacked(keccak256(initializer), saltNonce));
    bytes memory deploymentData = abi.encodePacked(type(GnosisSafeProxy).creationCode, uint256(uint160(_singleton)));
    // solhint-disable-next-line no-inline-assembly
    assembly {
        proxy := create2(0x0, add(0x20, deploymentData), mload(deploymentData), salt)
    }
    require(address(proxy) != address(0), "Create2 call failed");
}

function createProxyWithNonce(
    address _singleton,
    bytes memory initializer,
    uint256 saltNonce
) public returns (GnosisSafeProxy proxy) {
    proxy = deployProxyWithNonce(_singleton, initializer, saltNonce);
    if (initializer.length > 0)
        // solhint-disable-next-line no-inline-assembly
        assembly {
            if eq(call(gas(), proxy, 0, add(initializer, 0x20), mload(initializer), 0, 0), 0) {
                revert(0, 0)
            }
        }
    emit ProxyCreation(proxy, _singleton);
}

function createProxyWithCallback(
    address _singleton,
    bytes memory initializer,
    uint256 saltNonce,
    IProxyCreationCallback callback
) public returns (GnosisSafeProxy proxy) {
    uint256 saltNonceWithCallback = uint256(keccak256(abi.encodePacked(saltNonce, callback)));
    proxy = createProxyWithNonce(_singleton, initializer, saltNonceWithCallback);
    if (address(callback) != address(0)) callback.proxyCreated(proxy, _singleton, initializer, saltNonce);
}

// Safe.sol或SafeL2.sol
function setup(
    address[] calldata _owners,
    uint256 _threshold,
    address to,
    bytes calldata data,
    address fallbackHandler,
    address paymentToken,
    uint256 payment,
    address payable paymentReceiver
) external {
    // setupOwners checks if the Threshold is already set, therefore preventing that this method is called twice
    setupOwners(_owners, _threshold);
    if (fallbackHandler != address(0)) internalSetFallbackHandler(fallbackHandler);
    // As setupOwners can only be called if the contract has not been initialized we don't need a check for setupModules
    setupModules(to, data);

    if (payment > 0) {
        // To avoid running into issues with EIP-170 we reuse the handlePayment function (to avoid adjusting code of that has been verified we do not adjust the method itself)
        // baseGas = 0, gasPrice = 1 and gas = payment => amount = (payment + 0) * 1 = payment
        handlePayment(payment, 0, 1, paymentToken, paymentReceiver);
    }
    emit SafeSetup(msg.sender, _owners, _threshold, to, fallbackHandler);
}
```

### 2、功能实现
#### 1) 根据safe账号配置进行多链部署。
```javascript
const safeAccountConfig = {
  threshold: 2,
  owners: [owner1, owner3],
}
const predictedSafe = {
  safeAccountConfig,
  safeDeploymentConfig: {
    saltNonce: 1,
    deploymentType: "canonical"
  }
}
```
根据自己需求设置Safe账号配置threshold、owners。与平时不同需要safeDeploymentConfig配置生成address相关参数saltNonce、deploymentType。

注:  
1、saltNonce不能为0否则默认为null。  
2、deploymentType原本可要可不要但因为根据调查提供的sdk中的networkAddresses数组顺序不一致，导致默认的networkAddresses[0]不一样导致生成的address不一致。  
**canonical**: 指在以太坊主网（或目标链的原生环境）上部署的标准 Safe 合约。  
特点：  
使用最原始的 Safe 智能合约代码，未经特定链的定制化修改。  
适用于大多数 EVM 兼容链（如 Ethereum、Polygon、BSC 等）。  
适用场景：普通 EVM 链上的标准部署。  
**eip155**: 基于 EIP-155 标准（以太坊改进提案）的链，主要针对以太坊主网及其分叉链。  
特点：  
强调链 ID（由 EIP-155 定义）的兼容性，确保交易重放攻击防护。  
与 canonical 类似，但更明确依赖以太坊的链 ID 机制。  
适用场景：以太坊主网或严格遵循 EIP-155 的链。  
**zksync**: 专为 zkSync Era（ZK-Rollup 的 Layer 2 网络）定制的 Safe 合约部署。  
特点：  
适配 zkSync 的独特架构（如账户抽象、gas 机制、合约代码限制等）。  
可能使用 zkSync 特定的 opcode 或预编译合约。  
适用场景：仅在 zkSync Era 网络上使用。  
以上是询问AI得出的结论。  

```javascript
  const polygonKit = await Safe.init({
    provider: polygonChainUrl,
    signer: owner1_priv,
    predictedSafe
  })
  const sepoliaKit = await Safe.init({
    provider: sepoliaChainUrl,
    signer: owner1_priv,
    predictedSafe
  })
  const polygonTx = await polygonKit.createSafeDeploymentTransaction();
  const walletClient_polygon = await polygonKit.getSafeProvider().getExternalSigner();
    const transactionHash_polygon = await walletClient_polygon.sendTransaction({
    to: polygonTx.to,
    value: BigInt(polygonTx.value),
    data: polygonTx.data,
    // chain: polygon
  })
  console.log("transactionHash: ", transactionHash_polygon);
  const sepoliaTx = await sepoliaKit.createSafeDeploymentTransaction();
  const walletClient_sepolia = await sepoliaKit.getSafeProvider().getExternalSigner();  const transactionHash_sepolia = await walletClient_sepolia.sendTransaction({
    to: sepoliaTx.to,
    value: BigInt(sepoliaTx.value),
    data: sepoliaTx.data,
    // chain: polygon
  })
  console.log("transactionHash: ", transactionHash_sepolia);
```
余下就是根据自己需求进行构建deploymentTx,然后发送到链上。

#### 2) 已经在其他链上部署了safe, 需要再其他链上部署同样的safe
```javascript
const polygonChainUrl = polygon.rpcUrls.default.http[0];
const sepoliaChainUrl = sepolia.rpcUrls.default.http[0];

const apiKit = new SafeApiKit({
  chainId: polygon.id,
});
const polygonKit = await Safe.init({
  provider: polygonChainUrl,
  safeAddress: safeAddress,
});
// 获取safe账号版本
const version = await polygonKit.getContractVersion();
// 获取得到safe账号的最初配置(部署后执行的setup的params)
const createConfig = await apiKit.getSafeCreationInfo(safeAddress);
const createParams = handleCalldataParams(createConfig.dataDecoded.parameters);

const handleCalldataParams = (calldata_params) => {
  const obj = {};
  calldata_params.forEach((param) => {
    // 将object.name前面的_去除
    obj[param.name.replace(/^_+/, "")] = param.value;
  })
  return obj;
}

// 根据信息生成predictedSafe
const safeAccountConfig = createParams;
const predictedSafe = {
  safeAccountConfig: safeAccountConfig,
  safeDeploymentConfig: {
    saltNonce: createConfig.saltNonce,
    safeVersion: version,
  }
}

// 获取safe账号的逻辑合约地址
const singletonAddress = createConfig.singleton;
// 获取polygon service记录的singletons信息数组
const serviceSingletons = await apiKit.getServiceSingletonsInfo();
// 通过singletonAddress获取singleton的信息，可能获取不到，得到undefined
const singleton = serviceSingletons.find((singleton) => singleton.address === singletonAddress);
// 如果获取不到singleton，可能逻辑合约是sale.sol，所以默认isL1SafeSingleton为true
let isL1SafeSingleton = true;
if(singleton) {
  isL1SafeSingleton = !singleton.l2;
}

// 根据以上信息初始化新的Protocol Kit实例
const newKit = await Safe.init({
  provider: sepoliaChainUrl,
  signer: owner1_priv,
  predictedSafe: predictedSafe,
  isL1SafeSingleton: isL1SafeSingleton
});
```
在其他链上部署当前链的safe合约，需要获取初始化safe合约时候的参数、safe version以及isL1SafeSingleton（逻辑合约是否是safe.sol）  

注:  
Safe.sol: 针对ethereum主网、安全性优先、兼容所有的EVM链;返回的版本号是"safe version"("1.4.1"),**因为某些L2网络（如Polygon）早期直接复用了L1的Safe.sol合约，但为了标识部署环境而在版本号中强制添加了-L2后缀，导致实际逻辑仍是L1版本但版本号显示为L2。**  
SafeL2.sol: 针对Layer 2网络、gas费优先，因为gas优先，所以对比safe.sol合约少了些可能导致gas昂贵的逻辑;返回的版本号是"safe version-L2"("1.4.1-L2")

因为safe合约的master copy(主合约/逻辑合约)影响其safe address生成以及以上原因，所以不能通过获取apiKit.getSafeInfo得到version进而判断是否是Safe.sol。  
故采用的是直接对比master copy方法;先从后端服务获取到所有的master copy信息，查找出符合safe账号的master copy信息，通过l2字段判断，如果没有找出应该使用的是主网的safe.sol地址（主网兼容其他EVM链，其master copy: 0x41675C099F32341bf84BFc5382aF534df5C7461a）

(合约初始化参数、version、saltNonce、isL1SafeSingleton这些值的变化对safe address有影响)  
version决定调用的是哪个版本的SafeProxyFactory,从而导致create2的sender不同;参数、saltNonce、isL1SafeSingleton导致salt、deploymentData不同(代码说明请看合约中deployProxyWithNonce方法)。

```javascript
const sepoliaDeployTx = await newKit.createSafeDeploymentTransaction();
const walletClient = await newKit.getSafeProvider().getExternalSigner();
const sepoliaDeployTxHash = await walletClient.sendTransaction({
  to: sepoliaDeployTx.to,
  value: BigInt(sepoliaDeployTx.value),
  data: sepoliaDeployTx.data,
});
```
因为上面代码应该生成新的对应链的Protocol Kit,所以和一般部署一样，先生成对应的部署tx然后发布到对应链上。

#### 3) 不能多链部署的账号
1、部署交易调用的方法不是createProxyWithNonce则不能多链部署。  
**safe-wallet-monorepo**
```react
const createProxySelector = proxyFactoryInterface.getFunction('createProxyWithNonce').selector

const tx = await provider.getTransaction(creation.transactionHash)
if (!tx) {
  throw new Error(SAFE_CREATION_DATA_ERRORS.TX_NOT_FOUND)
}
const txData = tx.data
const startOfTx = txData.indexOf(createProxySelector.slice(2, 10))
if (startOfTx === -1) {
  throw new Error(SAFE_CREATION_DATA_ERRORS.UNSUPPORTED_SAFE_CREATION)
  // The method this Safe was created with is not supported. safe{wallet}错误提示
}
```

2、不支持小于1.3.0版本的safe账号。  
https://help.safe.global/en/articles/222612-deploying-a-multi-chain-safe

3、链上没有部署相关合约，不能在其部署safe合约  
**可以通过@safe-global/safe-deployments获取部署在链上的合约地址，从而判断是否支持该链**
https://docs.safe.global/advanced/smart-account-supported-networks  
https://github.com/safe-global/safe-wallet-monorepo/blob/dev/apps/web/src/features/multichain/hooks/useCompatibleNetworks.ts#L22  


