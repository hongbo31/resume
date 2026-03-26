# 手撕Prompt面试准备

> **场景**：面试官要求现场写Prompt，考察Prompt Engineering能力
> **准备目标**：2个高质量Prompt模板，可直接背诵或现场默写

---

## Prompt 1：生成Staking业务测试用例

### 完整Prompt（推荐背诵版）

```markdown
你是一位资深区块链测试工程师，擅长设计加密货币Staking业务的测试用例。

## 任务描述
请根据以下业务需求，生成完整的BDD格式测试用例（Feature文件）。

## 业务背景
Staking是用户将加密货币锁定在平台获取收益的业务。核心流程：
1. 用户发起Stake请求，指定币种和金额
2. 系统冻结用户资金（OEX扣款）
3. 系统按Cycle批次处理，在链上发起质押交易
4. 链上确认后，用户开始获得收益
5. 到期后用户可发起Unstake，解除质押并取回本金+收益

## 输入信息
业务需求：{business_requirement}
支持币种：{supported_tokens}
最小Stake金额：{min_amount}
Cycle周期：{cycle_duration}

## 输出要求
请生成Behave格式的Feature文件，包含：

1. **正常路径场景**（Happy Path）
   - 用户成功Stake并到期赎回的完整流程
   - 收益计算准确

2. **异常场景**
   - 余额不足（边界值：刚好等于最小金额、小于最小金额）
   - 无效的币种
   - 重复Stake同一笔资金
   - 提前Unstake（未到期）

3. **边界条件**
   - 最小Stake金额
   - 最大Stake金额（如有上限）
   - 金额精度（小数位）

4. **数据一致性校验点**
   - Request金额 = Cycle计算金额
   - Cycle金额 = 链上实际质押金额
   - 用户收益计算准确

## 输出格式
```gherkin
Feature: {业务名称} Staking业务流程

  Background:
    Given 用户已登录且有足够余额
    And 系统支持{币种}质押

  Scenario: 用户成功Stake并到期赎回
    Given 用户钱包有{balance} {token}
    When 用户发起Stake {amount} {token}请求
    Then 系统创建Request记录，status=1
    And 冻结金额等于{amount}
    When OEX系统完成资金冻结
    Then Request状态变为status=4
    When 执行Cycle日切操作
    Then 系统在链上发起质押交易
    When 链上交易确认
    Then Request状态变为status=8
    And 验证：Request金额 = Cycle金额 = 链上金额
    When 等待{cycle_duration}后发起Unstake
    Then 用户收到本金+收益，金额正确

  Scenario: Stake失败-余额不足
    ...

  Scenario: Stake失败-小于最小金额
    ...
```

## 质量要求
- 每个Scenario必须有明确的Given-When-Then步骤
- 包含具体的数据（金额、币种、状态码）
- 覆盖正向、反向、边界三种情况
- 数据一致性校验必须明确写出

请直接输出Feature文件内容，不需要额外解释。
```

---

### 具体例子（已填好参数，可直接使用）

```markdown
你是一位资深区块链测试工程师，擅长设计加密货币Staking业务的测试用例。

## 任务描述
请根据以下业务需求，生成完整的BDD格式测试用例（Feature文件）。

## 业务背景
Staking是用户将加密货币锁定在平台获取收益的业务。核心流程：
1. 用户发起Stake请求，指定币种和金额
2. 系统冻结用户资金（OEX扣款）
3. 系统按Cycle批次处理，在链上发起质押交易
4. 链上确认后，用户开始获得收益
5. 到期后用户可发起Unstake，解除质押并取回本金+收益

## 输入信息
业务需求：支持ETH和USDT的Staking质押，用户锁定资产后按日计息，到期自动赎回
支持币种：ETH、USDT
最小Stake金额：ETH 0.1个，USDT 100个
Cycle周期：每日日切，质押周期7天、30天、90天可选
日收益率：ETH 0.03%，USDT 0.05%

## 输出要求
请生成Behave格式的Feature文件，包含：

1. **正常路径场景**（Happy Path）
   - 用户成功Stake ETH并7天后赎回的完整流程
   - 收益计算准确（本金×日收益率×7天）

2. **异常场景**
   - 余额不足（ETH余额0.05，小于最小0.1）
   - 无效的币种（如输入BTC）
   - 提前Unstake（质押3天后就想赎回）
   - 金额精度错误（ETH输入0.1234567，超过6位小数）

3. **边界条件**
   - 最小Stake金额（刚好0.1 ETH）
   - 最大Stake金额（单笔上限1000 ETH）
   - 金额精度边界（ETH最多6位小数）

4. **数据一致性校验点**
   - Request.qty = Cycle.net_amount
   - Cycle.net_amount = StakingTransaction.amount（链上）
   - 用户收益 = 本金 × 0.03% × 实际质押天数

## 输出格式
```gherkin
Feature: ETH Staking业务流程

  Background:
    Given 用户已登录
    And 用户钱包有10 ETH可用余额
    And 系统支持ETH质押，最小金额0.1 ETH

  Scenario: 用户成功Stake 1 ETH并7天后赎回
    Given 用户钱包有10 ETH
    When 用户发起Stake 1 ETH请求，选择7天周期
    Then 系统创建Request记录，status=1（待扣款）
    And Request.qty等于1 ETH
    When OEX系统完成资金冻结，冻结金额1 ETH
    Then 用户钱包可用余额变为9 ETH
    And Request状态变为status=4（冻结成功）
    When 执行Cycle日切操作
    Then 系统在链上发起质押交易，amount=1 ETH
    And 创建StakingTransaction记录，status=pending
    When 链上交易确认，status=success
    Then Request状态变为status=8（链上成功）
    And StakingTransaction状态变为status=confirmed
    And 验证数据一致性：Request.qty(1) = Cycle.net_amount(1) = 链上amount(1)
    When 等待7天后用户发起Unstake
    Then 系统计算收益：1 ETH × 0.03% × 7 = 0.0021 ETH
    And 用户收到本金1 ETH + 收益0.0021 ETH = 1.0021 ETH
    And Request状态变为status=16（已完成）

  Scenario: Stake失败-ETH余额不足
    Given 用户钱包有0.05 ETH
    When 用户发起Stake 1 ETH请求
    Then 系统返回错误"余额不足"
    And 不创建Request记录

  Scenario: Stake失败-小于最小金额
    Given 用户钱包有10 ETH
    When 用户发起Stake 0.05 ETH请求
    Then 系统返回错误"最小Stake金额为0.1 ETH"
    And 不创建Request记录

  Scenario: Stake失败-无效的币种
    Given 用户钱包有1 BTC
    When 用户发起Stake 0.5 BTC请求
    Then 系统返回错误"不支持的币种：BTC"
    And 不创建Request记录

  Scenario: Stake失败-金额精度错误
    Given 用户钱包有10 ETH
    When 用户发起Stake 0.1234567 ETH请求（7位小数）
    Then 系统返回错误"ETH金额最多支持6位小数"
    And 不创建Request记录

  Scenario: 提前Unstake失败
    Given 用户已Stake 1 ETH，周期7天
    And 当前已质押3天
    When 用户发起Unstake请求
    Then 系统返回错误"质押未到期，无法提前赎回"
    And Request状态保持为status=8（质押中）

  Scenario: 边界值-Stake刚好最小金额0.1 ETH
    Given 用户钱包有0.1 ETH
    When 用户发起Stake 0.1 ETH请求
    Then 系统创建Request记录，status=1
    And 流程正常执行，最终成功赎回
```

## 质量要求
- 每个Scenario必须有明确的Given-When-Then步骤
- 包含具体的数据（金额、币种、状态码）
- 覆盖正向、反向、边界三种情况
- 数据一致性校验必须明确写出

请直接输出Feature文件内容，不需要额外解释。
```

---

### 简化版（时间紧张时用）

```markdown
你是一位区块链测试专家。请为Staking业务生成BDD测试用例。

业务：用户锁定加密货币获取收益，流程包括发起请求→资金冻结→链上质押→到期赎回。

请生成包含以下场景的Feature文件：
1. 正常路径：成功Stake并到期赎回
2. 异常场景：余额不足、无效币种、提前赎回
3. 边界条件：最小金额、金额精度
4. 数据一致性：Request金额=Cycle金额=链上金额

输出格式：Behave的Given-When-Then格式，包含具体数据和状态码。
```

---

### 具体例子（已填好参数，可直接使用）

```markdown
你是一位自动化测试专家，擅长分析测试失败日志并定位根因。

## 任务描述
请分析以下测试失败日志，判断失败类型、定位根因、给出修复建议。

## 分析维度
1. **失败类型**（单选）
   - CODE_DEFECT：代码缺陷（断言失败、空指针、业务逻辑错误）
   - ENV_ISSUE：环境问题（超时、连接拒绝、服务不可用）
   - DATA_ISSUE：数据问题（数据不存在、状态错误、脏数据）
   - ACCOUNT_ISSUE：账号问题（登录失败、权限不足、Token过期）
   - THIRD_PARTY：第三方问题（支付接口、外部服务异常）
   - UNKNOWN：无法判断

2. **具体根因**
   - 失败的具体原因（如：数据库连接池耗尽、断言期望值不匹配）
   - 涉及的代码位置（如有堆栈信息）

3. **修复建议**
   - 针对该问题的修复方案
   - 预防措施

4. **是否需要重试**
   - YES：瞬时问题（超时、网络波动），建议重试
   - NO：确定性问题（代码错误、数据错误），重试无效

## 输入信息
测试名称：test_eth_staking_full_flow
测试环境：test_env_03
失败时间：2026-03-26 14:32:15

### 错误日志
```
2026-03-26 14:32:15,234 ERROR TestExecutor - Test failed: test_eth_staking_full_flow
2026-03-26 14:32:15,235 ERROR TestExecutor - Error message: Timeout waiting for transaction receipt
2026-03-26 14:32:15,236 ERROR ChainClient - Transaction 0xabc123def456... not found after 300 seconds
2026-03-26 14:32:15,237 ERROR ChainClient - Current block: 18452367, Target block: unknown
2026-03-26 14:32:15,238 WARNING HttpConnection - Retrying (Retry(total=2, connect=None, read=None, redirect=None, status=None)) after connection broken by 'ReadTimeoutError("HTTPSConnectionPool(host='sepolia.infura.io', port=443): Read timed out. (read timeout=30)")'
2026-03-26 14:32:45,567 ERROR ChainClient - Failed to query transaction status after 3 retries
```

### 堆栈信息（如有）
```
File "/app/tests/staking/test_staking_flow.py", line 87, in test_eth_staking_full_flow
    receipt = chain_client.wait_for_transaction(tx_hash, timeout=300)
File "/app/core/chain_client.py", line 156, in wait_for_transaction
    raise TimeoutError(f"Transaction timeout: {tx_hash}")
TimeoutError: Transaction timeout: 0xabc123def456789...
```

### 历史相似案例（RAG检索结果）
```
案例1（相似度0.92）:
- 错误：Timeout waiting for transaction receipt
- 根因：以太坊测试网（Sepolia）拥堵，gas price设置过低导致交易长时间pending
- 解决：提高gas price到推荐值的1.5倍，或增加超时时间到600秒
- 是否重试：YES，重试后成功

案例2（相似度0.85）:
- 错误：ReadTimeoutError connecting to Infura
- 根因：Infura节点临时不可用，网络波动
- 解决：切换到备用RPC节点（Alchemy或QuickNode）
- 是否重试：YES，更换节点后成功

案例3（相似度0.71）:
- 错误：Transaction not found
- 根因：交易实际上链失败（nonce冲突），但系统未正确处理错误
- 解决：检查nonce管理逻辑，确保交易真正提交成功
- 是否重试：NO，代码逻辑问题
```

## 输出格式
请以JSON格式输出分析结果：

```json
{
  "failure_type": "THIRD_PARTY",
  "failure_type_confidence": 0.88,
  "root_cause": "以太坊测试网Sepolia临时拥堵/Infura节点不稳定，导致交易查询超时。具体表现为：300秒内未收到交易确认，且RPC连接出现ReadTimeoutError。",
  "location": "ChainClient.wait_for_transaction() line 156",
  "fix_suggestion": "1. 短期：增加超时时间到600秒，或切换到备用RPC节点（Alchemy）；2. 中期：实现多节点轮询机制，主节点失败时自动切换；3. 长期：监控链上gas price，动态调整以加速确认",
  "prevention": "1. 建立RPC节点健康检查机制，自动剔除慢节点；2. 设置合理的gas price策略（参考eth_gasPrice API）；3. 对超时交易增加补偿查询机制（10分钟后再次确认）",
  "should_retry": true,
  "retry_count": 3,
  "reasoning": "根据日志分析：1) 错误类型是'Timeout waiting for transaction receipt'，属于外部服务响应超时；2) 堆栈显示在ChainClient.wait_for_transaction等待链上确认；3) 历史相似案例显示Sepolia网络拥堵时提高gas price或切换节点可解决；4) 有ReadTimeoutError表明是网络/服务问题，非代码逻辑错误。因此判断为THIRD_PARTY问题，建议重试。"
}
```

## 分析步骤（请按此步骤思考）
1. 首先识别错误关键字（如timeout、NullPointer、AssertionError）
2. 结合堆栈信息定位代码位置
3. 参考历史相似案例，验证判断
4. 根据错误类型决定是否需要重试
5. 给出具体可执行的修复建议

请确保：
- failure_type必须从给定的6个选项中选择
- confidence表示置信度（0-1）
- 修复建议要具体可操作，不要泛泛而谈
- reasoning要展示你的分析逻辑

直接输出JSON结果，不需要额外解释。
```

---

### 面试表达技巧

**如果面试官让你解释这个Prompt的设计思路**：

> "这个Prompt用了**结构化+角色设定+Few-shot**的组合：
> 1. **角色设定**：'资深区块链测试工程师'，让模型用专业视角思考
> 2. **业务背景**：详细描述Staking流程，给模型足够的上下文
> 3. **明确分类**：要求正常路径、异常场景、边界条件、数据一致性四类用例，避免遗漏
> 4. **输出格式**：用XML标签和代码块明确格式要求，减少格式混乱
> 5. **质量要求**：明确列出验收标准，让模型自检"

**如何使用这个具体例子**：
- 这个例子展示了ETH Staking的完整测试用例，包含6个场景
- 你可以根据实际情况修改：币种（ETH→USDT）、金额（0.1→其他）、周期（7天→30天）
- 重点是展示你理解BDD格式，能覆盖正常路径、异常场景、边界条件

---

## Prompt 2：分析测试失败日志

### 完整Prompt（推荐背诵版）

```markdown
你是一位自动化测试专家，擅长分析测试失败日志并定位根因。

## 任务描述
请分析以下测试失败日志，判断失败类型、定位根因、给出修复建议。

## 分析维度
1. **失败类型**（单选）
   - CODE_DEFECT：代码缺陷（断言失败、空指针、业务逻辑错误）
   - ENV_ISSUE：环境问题（超时、连接拒绝、服务不可用）
   - DATA_ISSUE：数据问题（数据不存在、状态错误、脏数据）
   - ACCOUNT_ISSUE：账号问题（登录失败、权限不足、Token过期）
   - THIRD_PARTY：第三方问题（支付接口、外部服务异常）
   - UNKNOWN：无法判断

2. **具体根因**
   - 失败的具体原因（如：数据库连接池耗尽、断言期望值不匹配）
   - 涉及的代码位置（如有堆栈信息）

3. **修复建议**
   - 针对该问题的修复方案
   - 预防措施

4. **是否需要重试**
   - YES：瞬时问题（超时、网络波动），建议重试
   - NO：确定性问题（代码错误、数据错误），重试无效

## 输入信息
测试名称：{test_name}
测试环境：{environment}
失败时间：{failure_time}

### 错误日志
```
{error_logs}
```

### 堆栈信息（如有）
```
{stack_trace}
```

### 历史相似案例（RAG检索结果）
```
{similar_cases}
```

## 输出格式
请以JSON格式输出分析结果：

```json
{
  "failure_type": "ENV_ISSUE",
  "failure_type_confidence": 0.85,
  "root_cause": "数据库连接池耗尽，导致查询超时",
  "location": "DBHelper.query() line 45",
  "fix_suggestion": "1. 增加连接池大小 2. 检查连接是否正确释放 3. 添加连接超时监控",
  "prevention": "1. 代码Review检查资源释放 2. 添加连接池使用率监控告警",
  "should_retry": true,
  "retry_count": 3,
  "reasoning": "错误日志显示'Connection pool exhausted'，这是典型的资源耗尽问题。历史案例显示增加连接池后问题解决。属于瞬时性问题，重试可能成功。"
}
```

## 分析步骤（请按此步骤思考）
1. 首先识别错误关键字（如timeout、NullPointer、AssertionError）
2. 结合堆栈信息定位代码位置
3. 参考历史相似案例，验证判断
4. 根据错误类型决定是否需要重试
5. 给出具体可执行的修复建议

请确保：
- failure_type必须从给定的6个选项中选择
- confidence表示置信度（0-1）
- 修复建议要具体可操作，不要泛泛而谈
- reasoning要展示你的分析逻辑

直接输出JSON结果，不需要额外解释。
```

---

### 简化版（时间紧张时用）

```markdown
你是一位自动化测试专家。请分析以下测试失败日志。

判断维度：
1. 失败类型（代码缺陷/环境问题/数据问题/账号问题/第三方问题）
2. 具体根因
3. 修复建议
4. 是否需要重试（是/否）

输入：
测试名称：{test_name}
错误日志：{error_logs}
堆栈：{stack_trace}

输出：JSON格式，包含failure_type、root_cause、fix_suggestion、should_retry、reasoning。

要求：给出具体可执行的修复建议，不要泛泛而谈。
```

---

### 面试表达技巧

**如果面试官让你解释这个Prompt的设计思路**：

> "这个Prompt的核心是**结构化输出+思维链+置信度评估**：
> 1. **明确分类**：6种失败类型，减少模型幻觉
> 2. **JSON格式**：结构化输出，方便程序解析和统计
> 3. **置信度**：让模型评估自己的判断，低置信度时人工介入
> 4. **思维链**：要求展示reasoning，让分析过程可追溯
> 5. **可操作**：修复建议要具体，不要'检查代码'这种空话
> 6. **RAG增强**：输入相似案例，让模型参考历史经验"

**如何使用这个具体例子**：
- 这个例子展示了一个典型的链上交易超时问题（Sepolia网络拥堵）
- 你可以根据实际情况修改：测试名称、错误日志内容、历史案例
- 重点是展示你理解如何结构化分析失败，区分代码问题/环境问题/第三方问题
- 输出JSON格式体现了工程化思维，方便后续程序化处理

---

## 快速记忆口诀

### Prompt设计原则（STAR-F）
- **S**tructure：结构化（用标签、代码块分隔）
- **T**ask：任务描述清晰（做什么）
- **A**udience：角色设定（谁来做）
- **R**eference：Few-shot示例（参考什么）
- **F**ormat：输出格式明确（输出什么）

### 面试现场写Prompt checklist
1. [ ] 开头设定角色（"你是一位...专家"）
2. [ ] 描述任务背景（给足上下文）
3. [ ] 明确输入信息（占位符标记）
4. [ ] 定义输出格式（XML/JSON/Markdown）
5. [ ] 提供示例（Few-shot）
6. [ ] 设定质量要求（验收标准）

---

## 现场演示建议

如果面试官说："现场写一个Prompt，让AI生成测试用例"

**步骤**：
1. **先问清楚需求**："请问是什么业务类型的测试用例？有具体的业务需求文档吗？"
   - 体现你理解"好Prompt需要上下文"
   
2. **写结构化Prompt**（5分钟）：
   - 角色设定
   - 任务描述
   - 输出要求
   - 格式示例
   
3. **解释设计思路**（3分钟）：
   - 为什么这样设计
   - 每个部分的作用
   - 预期效果

4. **讨论优化**（2分钟）：
   - "如果要进一步提升准确率，我可以加Few-shot示例..."
   - "如果业务需求经常变，我可以把需求部分参数化..."

**加分点**：
- 提到"我会用A/B测试验证不同Prompt的效果"
- 提到"关键业务场景会加人工Review环节"
- 提到"Prompt会版本化管理，方便回滚"

---

*整理时间：2026-03-26*  
*适用场景：AI+测试开发岗位面试*
