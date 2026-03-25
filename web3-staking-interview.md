# Web3 Staking业务面试文档

> **角色定位**：后端业务测试专家  
> **核心项目**：Crypto.AI - AI+Web3测试平台  
> **文档版本**：v1.0  
> **更新日期**：2026-03-25

---

## 一、业务全景概述

### 1.1 业务模型
Staking业务是完整的"质押-解押-收益"闭环，采用**三层架构设计**：

| 层级 | 职责 | 核心表 |
|-----|------|-------|
| 用户请求层 | 处理用户Stake/Unstake请求 | request |
| Cycle批次层 | 日切批量聚合计算 | cycle |
| 链上执行层 | 实际链上操作+收益派发 | staking_transaction |

### 1.2 核心数据表

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   request   │──────│    cycle    │──────│ staking_tx  │
│  用户请求表  │ N:1  │  批次计算表  │ 1:N  │ 链上交易表   │
├─────────────┤      ├─────────────┤      ├─────────────┤
│ user_id     │      │ cycle_id    │      │ tx_hash     │
│ side(1/2)   │      │ side        │      │ amount      │
│ qty         │      │ inflow      │      │ status      │
│ status      │      │ outflow     │      │ confirmed_at│
│ cycle_id    │      │ net_amount  │      │             │
└─────────────┘      └─────────────┘      └─────────────┘
       │
       │ 1:N
       ↓
┌─────────────────────────────────────────────────────┐
│              position 持仓表                         │
├─────────────────────────────────────────────────────┤
│ user_id | stake_qty | reward_eligible_qty | residual │
│         |           | (reward计算基数)   | _balance │
└─────────────────────────────────────────────────────┘
```

**字段说明**：
- `side`: 1=Stake(资金流入), 2=Unstake(资金流出)
- `status`: 1(new)→3(pending_approve)→4(pending)→8(staked)→10(completed)
- `reward_eligible_qty`: 用于reward派发的计算基数

---

## 二、核心业务流程

### 2.1 Stake流程（side=1）

```
用户Stake 32 ETH
    ↓
request表: side=1, status=1(new)
    ↓
定时Job → Kafka → OEX系统扣款(positionTransfer)
    ↓
status=3(pending_approve) → OEX回调 → status=4(pending)
    ↓
【等待第二天10点Cycle日切】
    ↓
Cycle创建: status=1(new)
    ↓
Rebalancing计算: net_inflow = inflow - outflow
    ↓
链上操作: main_account → stake_account → 合约Stake
    ↓
staking_transaction记录tx_hash
    ↓
status=8(staked) → 定时任务 → status=10(completed)
```

### 2.2 Unstake流程（side=2）

```
用户Unstake 16 ETH
    ↓
request表: side=2, status=1(new) → 直接status=4(pending)
    ↓
【等待第二天10点Cycle日切】
    ↓
Rebalancing计算: 判断是否能Early Redemption
    ↓
├─ 可以提前 → status=6(early_redemption) → 下轮Cycle优先处理
│
└─ 不能提前 → 链上withdraw: 合约 → stake_account → house_account
                   ↓
              position.residual_balance += 金额
                   ↓
              资金返回用户
                   ↓
              status=8(staked) → 10(completed)
```

**Early Redemption判断逻辑**：
> Cycle的净流入(net_inflow)是否能覆盖早期Unstake的流出。如果当天新Stake金额 > 当天新Unstake金额，说明有新资金流入，可以优先满足部分pending的Unstake请求。

### 2.3 Reward派发流程

```
链上自动产生reward
    ↓
withdraw到staking_account(公司账户)
    ↓
定时reward_cycle启动
    ↓
根据reward_eligible_qty比例计算每个用户应得金额
    ↓
派发给用户
```

**派发规则**：
- 按比例分配，尾差归集到最后一个用户
- 使用`BigDecimal`精确计算，避免浮点误差
- 记录`calculated_amount`和`actual_amount`，方便对账

---

## 三、状态机流转

| 状态码 | 状态名 | 含义 | 适用场景 |
|-------|-------|------|---------|
| 1 | new | 请求创建 | Stake/Unstake初始状态 |
| 3 | pending_approve | 等待扣款确认 | Stake专用，已发Kafka给OEX |
| 4 | pending | 等待Cycle处理 | 资金已冻结/无需扣款 |
| 6 | early_redemption | 可提前赎回 | Unstake专用，已通过资格审核 |
| 8 | staked | 链上操作完成 | 等待定时任务收尾 |
| 10 | completed | 流程完结 | 终态 |

**关键设计**：
- Early Redemption失败时，**不自动回退到status=4**，而是保持在6并增加失败计数器
- 下次Cycle优先处理status=6的请求（比status=4优先级更高）
- 连续失败3次触发告警并人工介入

---

## 四、测试核心关注点

### 4.1 功能测试

| 测试维度 | 关注重点 | 验证方法 |
|---------|---------|---------|
| Stake | 资金冻结→上链→状态完成 | Mock OEX回调 + 链上tx校验 |
| Unstake | Early redemption触发条件 | 边界值：刚好满足/不满足条件 |
| Reward | 按比例分配准确性 | 多用户场景，验证分配总和=链上获得 |
| Cycle | 日切Job幂等性 | 重复执行不重复处理 |

### 4.2 数据一致性测试

**三层对账机制**（每天凌晨2点执行）：

```sql
-- 第一层：request层对账（Stake资金冻结校验）
SELECT * FROM request r
JOIN oex_position op ON r.request_id = op.biz_id
WHERE r.qty != op.frozen_amount 
  AND r.side = 1
  AND r.status >= 4;

-- 第二层：cycle层对账（批次计算校验）
SELECT cycle_id, 
       (SELECT SUM(qty) FROM request WHERE cycle_id = c.cycle_id) as req_sum,
       c.net_amount
FROM cycle c
HAVING req_sum != c.net_amount;

-- 第三层：链上层对账（链上交易校验）
SELECT c.cycle_id, c.net_amount, st.amount as chain_amount
FROM cycle c
JOIN staking_transaction st ON c.cycle_id = st.cycle_id
WHERE c.net_amount != st.amount;
```

### 4.3 并发与异常测试

| 场景 | 处理策略 |
|------|---------|
| 并发冲突 | 乐观锁(version字段) + 业务层重试3次 |
| 精度问题 | BigDecimal计算，尾差归集到最后一个用户 |
| Cycle失败 | 失败计数器 + 下次优先重试 + 3次失败告警 |
| 链上pending超时 | 轮询等待 + 超时告警 + 人工介入 |

---

## 五、高频面试题与标准答案

### Q1: Early Redemption的判断逻辑是什么？

**答案要点**：
> Cycle计算`net_inflow = 当天新Stake金额 - 当天新Unstake金额`。如果net_inflow为正，说明有新资金流入，可以优先用于满足部分pending的Unstake请求，标记为status=6。

### Q2: Early Redemption失败怎么处理？为什么不回退到status=4？

**答案要点**：
> 不回退是为了保证幂等性和简化逻辑。保持在status=6，增加失败计数器，下次Cycle优先重试。连续失败3次触发告警人工介入。

### Q3: Reward精度怎么处理？

**答案要点**：
> 使用BigDecimal精确计算，尾差归集到最后一个用户，保证总和严格等于链上金额。记录calculated_amount和actual_amount方便对账。

### Q4: 并发冲突怎么解决？

**答案要点**：
> 乐观锁(version字段) + 业务层重试。不选悲观锁是因为Cycle是批量操作，持锁时间太长会影响用户体验。Cycle执行期间短暂禁止新Unstake请求，减少冲突。

### Q5: 发现链上和DB不一致，SOP是什么？

**答案要点**：
> 1. 立即告警 2. 暂停相关Cycle自动执行 3. 人工核对三方原始数据 4. 修复后补数据或人工调账 5. 记录事故报告复盘改进

---

## 六、技术亮点总结

1. **三层架构设计**：请求层-Cycle层-链上层，职责清晰
2. **Early Redemption机制**：优化用户体验，提高资金效率
3. **幂等性设计**：状态机+失败计数器，保证重试安全
4. **三层对账**：自动化SQL，及时发现数据不一致
5. **乐观锁策略**：兼顾并发安全和用户体验

---

*文档生成：基于Crypto.AI项目实际业务整理*
