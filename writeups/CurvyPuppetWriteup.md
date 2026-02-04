# damn-vulnerable-defi CurvyPuppet Writeup

## 题目大意 & 目标汇率

根据 `test/curvy-puppet/CurvyPuppet.t.sol` 的初始化配置和 `src/curvy-puppet/CurvyPuppetLending.sol` 的业务逻辑，该用户头寸的详细数据如下：

| 参数 | 数值 | 计算过程 / 依据 |
| :--- | :--- | :--- |
| **抵押物数量** | 2500 DVT | `USER_INITIAL_COLLATERAL_BALANCE` |
| **债务数量** | 1 LP | `USER_BORROW_AMOUNT` |
| **抵押物价值** | **25,000** | $2500 	imes 10$ (`DVT_PRICE`) |
| **债务价值** | **~4,000** | $1 \times \text{LP 价格} (\approx 4,000)$ |
| **当前抵押率** | **625%** | $	ext{抵押物价值} \div 	ext{债务价值} = 25,000 \div 4,000$ |
| **清算边界 (抵押率)** | **175%** | 合约逻辑：`collateralValue < borrowValue * 1.75` |

抵押物价值永远都是 25000，因此只能通过修改债务价值，从而触发清算。

1. **LP 价格计算公式:**
   合约中 `_getLPTokenPrice` 定义如下：
   $$\text{LP 价格} = \text{ETH 价格 (4,000)} \times \text{Curve 池 virtual price}$$ 

2. **清算触发场景 (假设抵押品价格不变):**
   清算发生的临界点是 $\text{债务价值} = \text{抵押物价值} \div 1.75$。
   - **目标 LP 价格**: $25,000 \div 1.75 \approx \mathbf{14,285.7}$
   - **目标 virtual price**: $\mathbf{14,285.7} \div 4,000 \approx \mathbf{3.57}$

3. **结论:**
   - 当 Curve 池的 **`virtual_price`** 被拉升至 **3.57** 以上时（初始约为 1.0），1 个 LP 的价格将达到 **14,285.7**。
   - 此时债务价值 ($14,285.7 \times 1.75 \approx 25,000$) 刚好开始超过抵押物价值，导致该头寸可以被清算。

## 任务简报

- **初始资金**：200 WETH、6.5 LP。
- **任务目标**：去清算 3 个用户的各自的 1 个 LP。
- **关键操作**：操纵 Curve 池的汇率。

## 漏洞

Curve 池是 `ETH/stETH`，因此，它的 `ETH` 转入转出时，会触发 `fallback` 和 `receive`，在回调函数里，ETH 已经被转出，但 Curve 池内部状态还没有更新完毕，会看到非常抽象的汇率，此时调用 `liquidate`，LendingPool 会认为资不抵债，从而允许我们清算。

## 一定要使用漏洞吗？一定！

### 1. Curve 汇率极其稳定，难以通过交易操纵

我们的目标是将 `virtual_price` 从 ~1.0 拉升至 **3.57**。
然而，Curve stETH/ETH 池的设计极其抗操纵，普通的 Swap 很难大幅撼动汇率。

### 2. 实验数据：30 万 ETH 都砸不动

即使我们拥有巨额资金进行砸盘尝试，汇率的变化也微乎其微。

**测试记录：**
尝试使用 311,999 stETH (约 30 万 ETH) 进行单边砸盘（Swap stETH -> ETH）：

```text
  Balances AFTER Stalk:
    stETH: 311999999999999999999998
  已存入 stETH: 当前汇率 (1 ETH = 1000x stETH): 1096
  已存入 stETH: 当前汇率 (1 ETH = 1000x stETH): 1096
  已存入 stETH: 当前汇率 (1 ETH = 1000x stETH): 1096
  ...
  已存入 stETH: 当前汇率 (1 ETH = 1000x stETH): 1096
  已存入 stETH: 当前汇率 (1 ETH = 1000x stETH): 1097
  ...
  已存入 stETH: 当前汇率 (1 ETH = 1000x stETH): 1097
```

**结果：**
投入 30 万 stETH，`virtual_price` 仅仅从 ~1.0 维持在 ~1.097 附近，距离目标 **3.57** 遥不可及。即使是 30 万 ETH 的体量，都完全无法撼动汇率。

### 3. 资金限制

在主网环境中，我们也无法获得无限的资金：
1.  **Aave Flashloan**: stETH 的池子有限，最多能借出约 **170,000 stETH**。
2.  **Lido Stake**: 每日存款有配额限制 (Daily Limit)，通常最多存入约 **150,000 ETH** 换取 stETH。

即使我们将这两者结合，凑出约 32 万 stETH，如上所述，这依然远远不足以将汇率拉升至 3.57。

**结论：**
硬性操纵 Curve 池的 `virtual_price` 在数学和资金上都是不可行的。因此，必须寻找逻辑漏洞，即利用 `read-only reentrancy` 在状态更新的中间过程中读取到一个“错误”的临时价格。

## 最低需要多少资金

参考：[CurvyPuppet.t.sol](https://github.com/DeFiHackLabs/Web3-CTF-Intensive-CoLearning/blob/main/Writeup/SunSec/damn-vulnerable-defi/test/curvy-puppet/CurvyPuppet.t.sol)

它使用了 58685 ETH 和 172000 stETH，观测后者，它是 Aave stETH 贷款的资金上限，前者是由 Balancer 和 Aave 的 ETH 组合来的。

如果全部使用 Aave 的贷款，还款时还差 8.3 ETH，所以需要 Balancer 来节省利息。（我图方便，直接把开局的 200 ETH 提高到 210 ETH，可以直接过关）

那么问题来了，(58685, 172000) 是一个解，如果降低 ETH/提高 stETH，如果提高 ETH/降低 stETH，有没有更优的组合？

### 失败的尝试：通过 Lido 将 ETH 转为 stETH，降低 ETH，提高 stETH

经过测试，(25000, 180,000) 可以把汇率打到 3.584，性价比极高，纸面数据上，额外节省 33000 ETH，额外花费 8000 stETH。

看起来是赚的，实际测下来其实并不可行，从原先的差 8.3 ETH 变得更亏损了。

大概原因（可能，不一定对）：通过 Lido 质押获得的汇率大约是 1:1，在清算完成后，手里大量的 stETH 需要转化为 ETH，无法通过 Lido 转化，只能而通过池子转化，汇率大概是 1:1.00x。而且由于 Curve 对池子不平衡的惩罚，也带来了一定损失。

亏损比节省更高，所以无法通过该方式完成清算。

### 58685 是怎么来的？

经过测试，stETH 对汇率的影响更大，因此第一步肯定是把 stETH 借满。

在此基础上，反复调整 ETH 的值，直到恰好完成清算，而 58685 是恰好完成清算的临界值。

| ETH | stETH | `virtual_price` | success | comment |
| :--- | :--- | :--- | :--- | :--- |
| 58,200 | 172,000 | 3.570 | 失败 | 极度接近 |
| 59,000 | 172,000 | 3.572 | 成功 | 勉强过线 |

因此，当前已经是最优解了。

## 知识点

1. 依赖外部的 Oracle 时，必须依赖正规的池子，即便在 2026 年，Makina 也会被此类漏洞攻击。https://github.com/SunWeb3Sec/DeFiHackLabs/blob/main/src/test/2026-01/makina_exp.sol
2. 依赖外部的 Oracle 时，管理员要有滑点保护，对资产价格有大致的预期
3. 即便是 Curve 池，在使用有回调的货币时，在回调函数中也可以操纵汇率

## Exploit

https://github.com/anon-cBE4/damn-vulnerable-defi/blob/master/test/curvy-puppet/CurvyPuppet.t.sol

```text
Ran 1 test for test/curvy-puppet/CurvyPuppet.t.sol:CurvyPuppetChallenge
[PASS] test_curvyPuppet() (gas: 2793435)
Logs:
  --- [CHEAT] Add 10ether, too lazy to borrow Balancer ---
  === Start Exploit ===
  --- Step 1: Set Approvals ---
  --- Step 2: Initiate Aave Flashloan ---
  Balances BEFORE Flashloan:
    WETH: 210000000000000000000
    stETH: 0
  Borrowing from Aave: 172000 stETH & 58491 WETH...
  Aave Flashloan Received.
  Balances AFTER Flashloan:
    WETH: 58701000000000000000000
    stETH: 171999999999999999999999
  --- Step 3: Manipulate Curve Pool (Add Liquidity) ---
  Adding Liquidity to Curve...
  LP token price after add liquidity: 1096900479621201626
  --- Step 4: Remove Liquidity & Trigger Reentrancy ---
  --- Step 5: Readonly Reentrancy ---
  Current LP Virtual Price: 3571436638440058306
  Liquidation sequence complete.
  Exited remove_liquidity. Reentrancy logic over.
  --- Step 6: Asset Swap & Repay Flashloan ---
  Current LP Virtual Price: 1096900479621201626
  Exchange Rate (1 ETH -> stETH): 1005063678242922738
    Swapping ETH for stETH on Curve: 12963923469069977697655
    Wrapping remaining ETH to WETH: 58529366881363086523457
  --- Step 7: Final Cleanup & Return Funds ---
  Treasury Recovery:
    WETH returning: 1724981363086523457
    LP returning: 1
    DVT returning: 7500000000000000000000
  === Exploit Completed ===
```
