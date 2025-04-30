# Safe 交易安全

## Tenderly 模拟部分

Tenderly官网地址: https://tenderly.co/  

Tenderly 支持通过RPC和API进行单笔交易模拟。模拟在所选网络的最新状态下执行。

Dapp 和钱包可以使用单一模拟功能，让用户在投入真实资产之前进行交易试运行。这让他们可以预览交易的执行情况以及发生的余额和资产变化详情。交易预览可以帮助用户建立对 Dapp 的信任，只批准成功的交易，并避免代价高昂的失败。

#### 1、在UI上操作
在safe/wallet签署交易时，会提供模拟按钮让用户模拟要签署的交易。使用Tenderly为用户提供对潜在成功或失败的完整的技术报告。  

提供一个url，让用户可以可视化查看模拟后的交易对应的token变化，内存变化等。

#### 2、在safe/wallet的代码中如何实现。
```typescript
// 获取模拟交易结果
export const getSimulation = async (
  tx: TenderlySimulatePayload,
  customTenderly: EnvState['tenderly'] | undefined,
): Promise<TenderlySimulation> => {
  const requestObject: RequestInit = {
    method: 'POST',
    body: JSON.stringify(tx),
  }

  if (customTenderly?.accessToken) {
    requestObject.headers = {
      'content-type': 'application/JSON',
      'X-Access-Key': customTenderly.accessToken,
    }
  }

  const data = await fetch(
    customTenderly?.url ? customTenderly.url : TENDERLY_SIMULATE_ENDPOINT_URL,
    requestObject,
  ).then((res) => {
    if (res.ok) {
      return res.json()
    }
    return res.json().then((data) => {
      throw new Error(`${res.status} - ${res.statusText}: ${data?.error?.message}`)
    })
  })

  return data as TenderlySimulation
}

//可视化交易结果url
export const getSimulationLink = (simulationId: string): string => {
  return `https://dashboard.tenderly.co/public/${TENDERLY_ORG_NAME}/${TENDERLY_PROJECT_NAME}/simulator/${simulationId}`
}
```
通过访问自己在tenderly创建的project提供的url进行交易模拟；并提供可视化的url查看交易模拟。

#### 3、分析
```javascript
import * as dotenv from 'dotenv';
 
dotenv.config();
 
const approveDai = async () => {
  const simulateObj = {
    /* Simulation Configuration */
    save: true, // if true simulation is saved and shows up in the dashboard
    save_if_fails: true, // if true, reverting simulations show up in the dashboard
    simulation_type: 'full', // full, abi or quick (full is default)
    network_id: '80002', // network to simulate on
    /* Standard EVM Transaction object */
    from: 'from address',
    to: 'to address',
    input: "0x", //Calldata
    gas: 0,
    gas_price: 0,
    value: 0,
  }

  console.time('Simulation');
 
  const resp = await fetch(`https://api.tenderly.co/api/v1/account/happy_kid/project/project/simulate`, {
    method: 'POST',
    headers: {
      'content-type': 'application/JSON',
      'X-Access-Key': "6BEJlnWIiBI0Oul0RMzOeOWirP5klUsa",
    },
    body: JSON.stringify(simulateObj),
  })

  console.timeEnd('Simulation');
  console.log(resp.ok);
  const simulateRes = await resp.json();
  console.log(JSON.stringify(simulateRes, null, 2));
  console.log("line: ", `https://dashboard.tenderly.co/public/happy_kid/project/simulator/${simulateRes.simulation.id}`);
}
approveDai();
```
对应的api参数、路径详解: https://docs.tenderly.co/reference/api#/operations/simulateTransaction,

响应返回以下对象：

transaction：包含与交易相关的数据，包括其哈希值、区块编号、原始地址和目标地址、gas 详细信息、输入数据、随机数以及其他交易详细信息，如状态、时间戳和涉及的合约地址。

simulation：包含模拟交易的数据。它提供与模拟 ID、项目和所有者 ID、包含交易的区块编号、使用的 Gas、调用的方法、模拟状态以及其他元数据相关的详细信息。

contracts：包含交易所涉及的合约的数据，包括合约的 ID、网络 ID、余额、验证状态、相关标准（如 ERC20）、如果合约是代币则包含代币特定数据、编译器版本，甚至在某些情况下包含合约的源代码。

generated_access_list：包含交易将访问的地址和存储密钥列表，用于优化 Gas 成本。访问列表是以太坊 EIP-2930中引入的功能。


## 其他关于交易安全部分
对于Tenderly部分需要通过用户点击进行模拟，并给个url便于用户可视化查看。
通过调查safe公司与Redefine合作进一步保证用户的交易安全。
https://safe.mirror.xyz/rInLWZwD_sf7enjoFerj6FIzCYmVMGrrV8Nhg4THdwI

相关代码呈现
https://github.com/safe-global/safe-wallet-monorepo/blob/dev/packages/utils/src/services/security/modules/BlockaidModule/index.ts#L87  
根据代码指向的是Blockaid公司（官网：https://www.blockaid.io/）
