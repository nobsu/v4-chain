# 订单交易时序分析 - 附录

> 本文档是 [订单交易在多节点环境下的时序与时延分析](./order-transaction-timing-analysis.md) 的附录部分

---

## 附录

### A. 术语表

- **CheckTx**: 交易进入Mempool前的验证阶段
- **DeliverTx**: 交易在区块中最终执行的阶段
- **MemClob**: 内存中的中央限价订单簿
- **PrepareProposal**: 提议者准备区块的ABCI回调
- **ProcessProposal**: 验证者验证区块提议的ABCI回调
- **FinalizeBlock**: 执行区块的ABCI回调
- **PrepareCheckState**: 为下一区块准备CheckState的ABCI回调
- **Prevote/Precommit**: Tendermint共识的两阶段投票
- **GoodTilBlock**: 短期订单的有效期（区块高度）
- **GoodTilBlockTime**: 长期订单的有效期（时间戳）

### B. 相关文件索引

**订单处理核心**：
- [protocol/x/clob/ante/clob.go](protocol/x/clob/ante/clob.go) - CheckTx订单处理
- [protocol/x/clob/keeper/process_operations.go](protocol/x/clob/keeper/process_operations.go) - 撮合操作处理
- [protocol/x/clob/memclob/memclob.go](protocol/x/clob/memclob/memclob.go) - 内存订单簿

**ABCI回调**：
- [protocol/x/clob/abci.go](protocol/x/clob/abci.go) - PreBlocker, BeginBlocker, EndBlocker
- [protocol/app/prepare/prepare_proposal.go](protocol/app/prepare/prepare_proposal.go) - PrepareProposal
- [protocol/app/process/process_proposal.go](protocol/app/process/process_proposal.go) - ProcessProposal

**节点类型处理**：
- [protocol/app/prepare/full_node_prepare_proposal.go](protocol/app/prepare/full_node_prepare_proposal.go) - 全节点PrepareProposal
- [protocol/app/process/full_node_process_proposal.go](protocol/app/process/full_node_process_proposal.go) - 全节点ProcessProposal

### C. 参考资料

1. [CometBFT ABCI++ Specification](https://docs.cometbft.com/v0.38/spec/abci/)
2. [dYdX v4 Technical Architecture](https://dydx.exchange/blog/v4-technical-architecture-overview)
3. [Cosmos SDK Documentation](https://docs.cosmos.network/)
4. [Tendermint Consensus Algorithm](https://arxiv.org/abs/1807.04938)

---

## 附录D: 深入研究专题

### D.1 dYdX v4 Chain的主要状态管理

#### D.1.1 状态类型概览

dYdX v4 Chain采用**多层状态管理架构**，在不同的存储层和生命周期中管理不同类型的状态。

#### D.1.2 订单相关状态

##### 1. 短期订单（Short-Term Orders）

**存储位置**：
- **主存储**：MemClob内存（`MemClobPriceTimePriority.orderbooks`）
- **数据结构**：`Orderbook`结构体
- **文件位置**：[protocol/x/clob/memclob/memclob.go](protocol/x/clob/memclob/memclob.go)

**特性**：
```go
// 仅在内存中存在，不持久化到KVStore
type Orderbook struct {
    Asks                           map[types.Subticks]*types.Level
    Bids                           map[types.Subticks]*types.Level
    BestAsk, BestBid              types.Subticks
    orderIdToLevelOrder            map[types.OrderId]*types.LevelOrder
    blockExpirationsForOrders      map[uint32]map[types.OrderId]bool
    orderIdToCancelExpiry          map[types.OrderId]uint32
    // ...更多索引
}
```

**生命周期**：
- 创建：在CheckTx阶段添加到MemClob
- 有效期：GoodTilBlock（当前块 + 1 至 当前块 + ShortBlockWindow）
- 销毁：区块结束后或完全成交后从MemClob移除
- 不持久化：仅存在于当前区块的内存中

**填充量持久化**：
虽然订单本身不持久化，但成交量会写入KVStore：
```go
// 存储前缀: OrderAmountFilledKeyPrefix = "Fill:"
keeper.SetOrderFillAmount(ctx, orderId, fillAmount, prunableBlockHeight)
```

##### 2. 长期订单（Long-Term Orders）

**存储位置**（多层）：

**层1：主存储（KVStore）**
- 前缀：`LongTermOrderPlacementKeyPrefix = "SO/P/L:"`
- 数据：`LongTermOrderPlacement`消息
- 持久化：在DeliverTx阶段写入
- 文件：[protocol/x/clob/keeper/stateful_order_state.go](protocol/x/clob/keeper/stateful_order_state.go)

```go
func (k Keeper) SetLongTermOrderPlacement(
    ctx sdk.Context,
    order types.Order,
    blockHeight uint32,
) {
    placement := types.LongTermOrderPlacement{
        Order:            order,
        PlacementIndex:   types.TransactionOrdering{...},
    }
    store := ctx.KVStore(k.storeKey)
    key := order.OrderId.ToStateKey()
    store.Set(key, k.cdc.MustMarshal(&placement))
}
```

**层2：内存缓存（MemClob）**
- 从KVStore加载到MemClob（在PrepareCheckState）
- 参与实时撮合
- 每个区块重新加载

**层3：临时存储（TransientStore）**
- 前缀：`UncommittedStatefulOrderPlacementTransientStore`
- 用途：追踪CheckTx期间未提交的订单
- 生命周期：仅在区块边界内有效

**层4：过期索引**
- 前缀：`StatefulOrdersExpirationsKeyPrefix = "SOExp:"`
- 结构：`{GoodTilBlockTime} -> []OrderId`
- 用途：EndBlocker中快速查找过期订单

**完整生命周期**：
```
CheckTx阶段:
├─ 写入UncommittedStatefulOrderPlacementTransientStore
└─ 生成操作提议

DeliverTx阶段:
├─ SetLongTermOrderPlacement() → 主KVStore
├─ AddStatefulOrderIdExpiration() → 过期索引
└─ 增加订单计数

PrepareCheckState阶段:
├─ 从KVStore加载所有长期订单
├─ PlaceStatefulOrdersFromLastBlock() → MemClob
└─ 参与撮合

EndBlocker阶段:
├─ RemoveExpiredStatefulOrders() → 检查过期
├─ 完全成交的订单被移除
└─ 从所有存储层删除
```

##### 3. 条件订单（Conditional Orders）

**状态机模型**：

```
┌─────────────────────────────────────────────────┐
│                未触发状态                        │
│  存储: UntriggeredConditionalOrderKeyPrefix     │
│  前缀: "SO/U:"                                  │
│  检查: EndBlocker中每个区块检查触发条件         │
└─────────────────────────────────────────────────┘
                    ↓
          [触发条件满足]
                    ↓
┌─────────────────────────────────────────────────┐
│                已触发状态                        │
│  存储: TriggeredConditionalOrderKeyPrefix       │
│  前缀: "SO/P/T:"                                │
│  操作: 在PrepareCheckState放置到MemClob         │
└─────────────────────────────────────────────────┘
                    ↓
          [放置到订单簿]
                    ↓
┌─────────────────────────────────────────────────┐
│              在MemClob上活跃                     │
│  行为: 与普通长期订单相同                        │
└─────────────────────────────────────────────────┘
```

**触发检查**：
```go
// EndBlocker中执行
func (k Keeper) MaybeTriggerConditionalOrders(
    ctx sdk.Context,
) (triggeredConditionalOrderIds []types.OrderId) {
    // 遍历所有未触发的条件订单
    // 检查触发条件（价格、时间等）
    // 将满足条件的订单移动到已触发状态
}
```

#### D.1.3 账户和子账户状态

##### 1. Subaccount状态

**存储位置**：
- **KVStore前缀**：`SubaccountKeyPrefix = "SA:"`
- **Keeper**：SubaccountsKeeper
- **文件**：[protocol/x/subaccounts/keeper/subaccount.go](protocol/x/subaccounts/keeper/subaccount.go)

**数据结构**：
```go
type Subaccount struct {
    Id                  SubaccountId
    AssetPositions      []AssetPosition      // USDC等资产余额
    PerpetualPositions  []PerpetualPosition  // 永续合约头寸
}

type AssetPosition struct {
    AssetId  uint32  // 资产ID
    Quantums uint64  // 数量（最小单位）
}

type PerpetualPosition struct {
    PerpetualId      uint32  // 永续合约ID
    Quantums         int64   // 持仓数量（正为多，负为空）
    FundingIndex     int64   // 资金费率索引
}
```

**关键操作**：
```go
// 读取（默认返回空结构体）
func (k Keeper) GetSubaccount(ctx sdk.Context, id SubaccountId) Subaccount

// 写入（空子账户会被删除）
func (k Keeper) SetSubaccount(ctx sdk.Context, subaccount Subaccount)

// 更新头寸
func (k Keeper) UpdateSubaccountPositionWithFundingPayment(
    ctx sdk.Context,
    subaccountId SubaccountId,
    perpetualId uint32,
    fundingPayment *big.Int,
)
```

##### 2. 安全堆（Safety Heap）

**用途**：按安全等级排序维护子账户，用于快速识别需要清算的账户

**存储结构**：
```
SafetyHeapStorePrefix = "SH"
├─ Heap/           → []SubaccountId（堆数组）
├─ Idx/{saId}      → HeapIndex（子账户→堆索引映射）
└─ Len/            → 堆长度
```

**操作**：
- 插入：新子账户加入堆
- 更新：头寸变化后重新排序
- 提取：获取最不安全的子账户用于清算

##### 3. 负TNC子账户追踪

**存储前缀**：
```
NegativeTncSubaccountForCollateralPoolSeenAtBlockKeyPrefix = "NegSA:"
```

**用途**：
- 追踪最后一次见到负总净抵押品（Total Net Collateral）的区块高度
- 用于提款限制（Gate Withdrawals）
- 按抵押池隔离（支持跨抵押和隔离市场）

#### D.1.4 订单填充量状态

**存储位置**：
- **前缀**：`OrderAmountFilledKeyPrefix = "Fill:"`
- **键格式**：`orderId.ToStateKey()`
- **文件**：[protocol/x/clob/keeper/order_state.go](protocol/x/clob/keeper/order_state.go)

**数据结构**：
```go
type OrderFillState struct {
    FillAmount          uint64  // 已填充的基础数量
    PrunableBlockHeight uint32  // 可以修剪此状态的区块高度
}
```

**操作流程**：
```go
// 写入填充量
func (k Keeper) SetOrderFillAmount(
    ctx sdk.Context,
    orderId types.OrderId,
    fillAmount satypes.BaseQuantums,
    prunableBlockHeight uint32,
) {
    store := ctx.KVStore(k.storeKey)
    fillState := types.OrderFillState{
        FillAmount:          fillAmount.ToUint64(),
        PrunableBlockHeight: prunableBlockHeight,
    }
    store.Set(orderId.ToStateKey(), k.cdc.MustMarshal(&fillState))
}

// 读取填充量
func (k Keeper) GetOrderFillAmount(
    ctx sdk.Context,
    orderId types.OrderId,
) satypes.BaseQuantums {
    store := ctx.KVStore(k.storeKey)
    fillStateBytes := store.Get(orderId.ToStateKey())
    if fillStateBytes == nil {
        return 0
    }
    var fillState types.OrderFillState
    k.cdc.MustUnmarshal(fillStateBytes, &fillState)
    return satypes.BaseQuantums(fillState.FillAmount)
}
```

**修剪机制**：
```go
// EndBlocker中执行
func (k Keeper) PruneStateFillAmountsForShortTermOrders(ctx sdk.Context) {
    currentBlock := ctx.BlockHeight()
    // 遍历所有可修剪的订单
    k.PruneOrdersForBlockHeight(ctx, uint32(currentBlock))
}
```

#### D.1.5 CLOB配置状态

##### 1. Clob Pair配置

**存储前缀**：`ClobPairKeyPrefix = "Clob:"`

**数据结构**：
```go
type ClobPair struct {
    Id                  uint32
    Metadata            ClobMetadata  // Perpetual或Spot
    StepBaseQuantums    uint64        // 最小订单数量增量
    SubticksPerTick     uint32        // 价格精度
    QuantumConversionExponent int32   // 数量转换指数
    Status              ClobPairStatus // ACTIVE, PAUSED, CANCEL_ONLY等
}
```

##### 2. 清算配置

**存储键**：`LiquidationsConfigKey = "LiqCfg"`

**内容**：
- 清算保险基金费用PPM
- 验证者清算费用PPM
- 流动性清算费用PPM
- 填充价格上限PPM

##### 3. 区块速率限制

**存储键**：`BlockRateLimitConfigKey = "RateLimCfg"`

**用途**：限制每个区块的订单/取消操作数量

##### 4. 权益级别限制

**存储键**：`EquityTierLimitConfigKey = "EqTierCfg"`

**用途**：基于账户权益的订单数量和规模限制

#### D.1.6 ProcessProposerMatchesEvents状态

**存储位置**：MemStore（块内临时）

**存储键**：`ProcessProposerMatchesEventsKey`

**数据结构**：
```go
type ProcessProposerMatchesEvents struct {
    BlockHeight                            uint32
    OrderIdsFilledInLastBlock              []OrderId
    ExpiredStatefulOrderIds                []OrderId
    ConditionalOrderIdsTriggeredInLastBlock []OrderId
    RemovedStatefulOrderIds                []OrderId
}
```

**用途**：
- 在BeginBlocker中生成
- 在EndBlocker中更新
- 在PrepareCheckState中使用
- 用于同步MemClob状态

**生命周期**：
```
BeginBlock:
├─ 初始化为空
└─ BlockHeight设置为当前高度

DeliverTx:
├─ 记录处理的订单ID
└─ 累积成交信息

EndBlocker:
├─ 添加过期订单ID
├─ 添加触发的条件订单ID
└─ MustSetProcessProposerMatchesEvents()

PrepareCheckState:
├─ GetProcessProposerMatchesEvents()
├─ 使用这些ID同步MemClob
└─ 清理已完成的订单
```

#### D.1.7 已交付订单ID（MemStore）

**长期订单**：
- **前缀**：`OrderedDeliveredLongTermOrderKeyPrefix = "DLTO:"`
- **用途**：追踪在DeliverTx中已处理的长期订单

**条件订单**：
- **前缀**：`OrderedDeliveredConditionalOrderKeyPrefix = "DCIdx:"`
- **用途**：追踪在DeliverTx中已处理的条件订单

**取消订单**：
- **前缀**：`DeliveredCancelKeyPrefix = "DCancel:"`
- **用途**：追踪已取消的订单

**特性**：
- 存储在MemStore（块内有效）
- 按顺序追踪
- PrepareCheckState使用这些ID重新加载订单

#### D.1.8 状态存储层次总结

```
┌─────────────────────────────────────────────────────────┐
│                   存储层次架构                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Layer 1: 持久化存储（KVStore - Disk）                  │
│  ├─ 长期订单                                            │
│  ├─ 条件订单（未触发/已触发）                            │
│  ├─ 订单填充量                                          │
│  ├─ Subaccount状态                                      │
│  ├─ CLOB配置                                            │
│  └─ 过期索引                                            │
│                                                         │
│  Layer 2: 内存存储（MemStore - RAM，块内）              │
│  ├─ ProcessProposerMatchesEvents                        │
│  ├─ 已交付订单ID列表                                    │
│  ├─ 订单计数                                            │
│  └─ 块内临时数据                                        │
│                                                         │
│  Layer 3: 瞬态存储（TransientStore - 块边界重置）       │
│  ├─ 未提交的长期订单                                    │
│  ├─ 未提交的取消                                        │
│  └─ 未提交的订单计数                                    │
│                                                         │
│  Layer 4: MemClob（纯内存 - 应用层）                    │
│  ├─ 短期订单簿                                          │
│  ├─ 长期订单副本（从KVStore加载）                        │
│  ├─ 操作队列                                            │
│  └─ 订单哈希索引                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### D.2 CheckTx与共识后执行阶段的MemClob状态修改

这是一个关键问题，涉及状态一致性和共识安全性。

#### D.2.1 MemClob状态修改总览

**核心原则**：
- **CheckTx阶段**：对MemClob进行修改，但这些修改是**临时的**
- **DeliverTx阶段**：对持久化状态进行修改，MemClob仅作为缓存
- **PrepareCheckState**：同步两者的状态

#### D.2.2 CheckTx阶段的MemClob修改

**修改流程**：

```go
// 文件: protocol/x/clob/memclob/memclob.go
func (m *MemClobPriceTimePriority) PlaceOrder(
    ctx sdk.Context,
    order types.Order,
) (BaseQuantums, OrderStatus, *OffchainUpdates, error) {

    // 1. 验证订单
    if err := m.validateOrder(ctx, order); err != nil {
        return 0, OrderStatus{}, nil, err
    }

    // 2. 尝试匹配现有订单（使用分支上下文）
    matchedQuantums, matchedOrder, err := m.matchOrder(ctx, order)

    // 3. 如果有剩余数量，添加到订单簿
    if order.GetBaseQuantums() > matchedQuantums {
        m.mustAddOrderToOrderbook(ctx, order, isPostOnlyOrder)
        // ↑ 这会直接修改MemClob的orderbooks map
    }

    // 4. 添加到操作队列
    m.operationsToPropose.MustAddShortTermOrderPlacementToOperationsQueue(
        order,
        transactionIndex,
    )
    // ↑ 这会修改MemClob的operationsToPropose

    return matchedQuantums, orderStatus, offchainUpdates, nil
}
```

**具体修改的MemClob状态**：

1. **订单簿（Orderbooks）**：
```go
// 添加订单到订单簿
func (o *Orderbook) addOrderToOrderbook(
    order Order,
    levelOrder *types.LevelOrder,
) {
    // 修改Bids或Asks map
    if order.IsBuy() {
        o.addLevelOrderToLevel(order.GetOrderSubticks(), levelOrder, o.Bids)
        // 更新BestBid
        if order.GetOrderSubticks() > o.BestBid {
            o.BestBid = order.GetOrderSubticks()
        }
    } else {
        o.addLevelOrderToLevel(order.GetOrderSubticks(), levelOrder, o.Asks)
        // 更新BestAsk
        if order.GetOrderSubticks() < o.BestAsk {
            o.BestAsk = order.GetOrderSubticks()
        }
    }

    // 添加到各种索引
    o.orderIdToLevelOrder[order.OrderId] = levelOrder
    o.blockExpirationsForOrders[order.GoodTilBlock][order.OrderId] = true
    // ...
}
```

2. **操作队列（OperationsToPropose）**：
```go
type OperationsToPropose struct {
    OperationsQueue                []InternalOperation  // ← 追加操作
    OrderHashesInOperationsQueue   map[OrderHash]bool   // ← 添加哈希
    ShortTermOrderHashToTxBytes    map[OrderHash][]byte // ← 保存TX字节
    MatchedOrderIdToOrder          map[OrderId]Order    // ← 记录匹配
    // ...
}

func (o *OperationsToPropose) MustAddShortTermOrderPlacementToOperationsQueue(
    order Order,
) {
    operation := InternalOperation_ShortTermOrderPlacement{
        ShortTermOrderPlacement: &MsgPlaceOrder{Order: order},
    }
    o.OperationsQueue = append(o.OperationsQueue, operation)
    // ↑ 直接修改队列

    orderHash := order.GetOrderHash()
    o.OrderHashesInOperationsQueue[orderHash] = true
    // ↑ 修改哈希集合
}
```

3. **子账户开放订单追踪**：
```go
// 在订单簿中追踪
o.SubaccountOpenClobOrders[subaccountId][order.Side][order.OrderId] = true

// 如果是减仓订单
if order.IsReduceOnly() {
    o.SubaccountOpenReduceOnlyOrders[subaccountId][order.OrderId] = true
}
```

**关键点**：
- ✅ **确实修改了MemClob**：订单被添加到内存订单簿
- ✅ **操作被记录**：添加到`OperationsToPropose`队列
- ⚠️ **状态不持久化**：这些修改仅在CheckState中
- ⚠️ **可能被丢弃**：如果交易未被包含在区块中

#### D.2.3 DeliverTx阶段的状态处理

**重要区别**：在DeliverTx阶段，订单不是通过`PlaceOrder`再次添加到MemClob！

**实际流程**：

```go
// 文件: protocol/x/clob/keeper/msg_server_proposed_operations.go
func (k msgServer) ProposedOperations(
    ctx sdk.Context,
    msg *types.MsgProposedOperations,
) (*types.MsgProposedOperationsResponse, error) {

    lib.AssertDeliverTxMode(ctx)  // 断言在DeliverTx模式

    // 1. 验证和转换操作
    processedOperations := k.ProcessOperations(ctx, msg.OperationsQueue)

    // 2. 处理内部操作
    for _, operation := range processedOperations {
        switch op := operation.(type) {
        case *InternalOperation_Match:
            // 处理匹配
            k.PersistMatchToState(ctx, op.Match)
            // ↑ 这会更新KVStore，不是MemClob！

        case *InternalOperation_ShortTermOrderPlacement:
            // 短期订单：仅更新填充量
            // MemClob已经在CheckTx时处理过了

        case *InternalOperation_PreexistingStatefulOrder:
            // 长期订单：标记为已交付
            k.AddDeliveredLongTermOrderId(ctx, op.OrderId)
            // ↑ 写入MemStore，不是MemClob！
        }
    }

    // 3. 生成事件
    k.GenerateProcessProposerMatchesEvents(ctx)

    return &types.MsgProposedOperationsResponse{}, nil
}
```

**持久化操作**：

```go
// 持久化匹配到状态
func (k Keeper) PersistMatchToState(
    ctx sdk.Context,
    match *types.ClobMatch,
) {
    lib.AssertDeliverTxMode(ctx)

    // 更新订单填充量（KVStore）
    k.SetOrderFillAmount(ctx, takerOrderId, newFillAmount, prunableBlockHeight)

    // 更新子账户头寸（KVStore）
    k.subaccountsKeeper.UpdateSubaccount(ctx, takerSubaccount)
    k.subaccountsKeeper.UpdateSubaccount(ctx, makerSubaccount)

    // 生成事件
    ctx.EventManager().EmitEvent(orderFillEvent)

    // ↑ 注意：没有修改MemClob！
}
```

**关键观察**：
- ❌ **不修改MemClob**：DeliverTx阶段不会调用`MemClob.PlaceOrder()`
- ✅ **仅持久化状态**：更新KVStore中的订单填充量
- ✅ **记录已交付订单**：添加到MemStore中的已交付列表
- ✅ **生成事件**：为Indexer生成事件

**为什么不修改MemClob？**

因为MemClob的目的是为**下一个区块**准备操作，而DeliverTx处理的是**当前区块**的确定结果。当前区块的MemClob状态会在PrepareCheckState被丢弃和重建。

#### D.2.4 PrepareCheckState的状态同步

这是关键的同步点，确保CheckState的MemClob与DeliverState的持久化状态一致。

**完整流程**：

```go
// 文件: protocol/x/clob/abci.go
func (k Keeper) PrepareCheckState(ctx sdk.Context) {

    ctx.Logger().Info("CLOB PrepareCheckState Start")

    // ═══════════════════════════════════════════════════
    // 阶段1: 获取要重放的操作
    // ═══════════════════════════════════════════════════
    localValidatorOperationsQueue, shortTermOrderTxBytes :=
        k.MemClob.GetOperationsToReplay(ctx)
    // ↑ 从当前MemClob提取本地验证器的操作

    // ═══════════════════════════════════════════════════
    // 阶段2: 清除MemClob中的本地操作
    // ═══════════════════════════════════════════════════
    k.MemClob.RemoveAndClearOperationsQueue(
        ctx,
        localValidatorOperationsQueue,
    )
    // ↑ 删除MemClob中所有本地放置的订单
    // ↑ 清空operationsToPropose队列

    // ═══════════════════════════════════════════════════
    // 阶段3: 获取上一块的匹配事件
    // ═══════════════════════════════════════════════════
    processProposerMatchesEvents :=
        k.GetProcessProposerMatchesEvents(ctx)

    // ═══════════════════════════════════════════════════
    // 阶段4: 清除无效的MemClob状态
    // ═══════════════════════════════════════════════════
    offchainUpdates = k.MemClob.PurgeInvalidMemclobState(
        ctx,
        processProposerMatchesEvents.OrderIdsFilledInLastBlock,
        // ↑ 移除完全填充的订单

        processProposerMatchesEvents.ExpiredStatefulOrderIds,
        // ↑ 移除过期的长期订单

        k.GetDeliveredCancelledOrderIds(ctx),
        // ↑ 移除已取消的订单

        processProposerMatchesEvents.RemovedStatefulOrderIds,
        // ↑ 移除其他被移除的订单

        offchainUpdates,
    )

    // ═══════════════════════════════════════════════════
    // 阶段5: 重新加载长期订单（Post-Only通过）
    // ═══════════════════════════════════════════════════
    longTermOrderIds := k.GetDeliveredLongTermOrderIds(ctx)
    offchainUpdates = k.PlaceStatefulOrdersFromLastBlock(
        ctx,
        longTermOrderIds,
        offchainUpdates,
        true,  // postOnlyFilter = true
    )
    // ↑ 从KVStore读取订单
    // ↑ 调用MemClob.PlaceOrder()添加到订单簿（仅非立即成交）

    // ═══════════════════════════════════════════════════
    // 阶段6: 重新加载条件订单（Post-Only通过）
    // ═══════════════════════════════════════════════════
    conditionalOrderIds :=
        processProposerMatchesEvents.ConditionalOrderIdsTriggeredInLastBlock
    offchainUpdates = k.PlaceConditionalOrdersTriggeredInLastBlock(
        ctx,
        conditionalOrderIds,
        offchainUpdates,
        true,  // postOnlyFilter = true
    )

    // ═══════════════════════════════════════════════════
    // 阶段7: 重放本地操作（Post-Only通过）
    // ═══════════════════════════════════════════════════
    replayUpdates := k.MemClob.ReplayOperations(
        ctx,
        localValidatorOperationsQueue,
        shortTermOrderTxBytes,
        offchainUpdates,
        true,  // postOnlyFilter = true
    )
    // ↑ 重新执行本地验证器在上一块放置的操作

    // ═══════════════════════════════════════════════════
    // 阶段8: 清算和去杠杆化
    // ═══════════════════════════════════════════════════
    liquidatableSubaccountIds := k.GetSubaccountLiquidationInfo(ctx)
    subaccountsToDeleverage, err := k.LiquidateSubaccountsAgainstOrderbook(
        ctx,
        liquidatableSubaccountIds,
    )
    if err := k.DeleverageSubaccounts(ctx, subaccountsToDeleverage); err != nil {
        panic(err)
    }

    // ═══════════════════════════════════════════════════
    // 阶段9: 重新加载所有订单（完整通过）
    // ═══════════════════════════════════════════════════
    offchainUpdates = k.PlaceStatefulOrdersFromLastBlock(
        ctx,
        longTermOrderIds,
        offchainUpdates,
        false,  // postOnlyFilter = false
    )
    // ↑ 允许立即成交

    offchainUpdates = k.PlaceConditionalOrdersTriggeredInLastBlock(
        ctx,
        conditionalOrderIds,
        offchainUpdates,
        false,  // postOnlyFilter = false
    )

    // ═══════════════════════════════════════════════════
    // 阶段10: 重放本地操作（完整通过）
    // ═══════════════════════════════════════════════════
    replayUpdates = k.MemClob.ReplayOperations(
        ctx,
        localValidatorOperationsQueue,
        shortTermOrderTxBytes,
        offchainUpdates,
        false,  // postOnlyFilter = false
    )

    // ═══════════════════════════════════════════════════
    // 阶段11: 初始化新流
    // ═══════════════════════════════════════════════════
    k.MemClob.InitializeNewStreams(ctx)

    ctx.Logger().Info("CLOB PrepareCheckState Complete")
}
```

**两轮放置的原因**：

1. **Post-Only通过**（第一轮）：
   - 仅放置不会立即成交的订单
   - 避免自成交
   - 建立订单簿流动性

2. **完整通过**（第二轮）：
   - 允许订单与现有订单交叉成交
   - 生成撮合操作
   - 完整重建MemClob状态

#### D.2.5 状态一致性保证

**关键机制**：

```
区块N结束:
├─ EndBlocker更新ProcessProposerMatchesEvents
├─ 记录所有已填充、过期、取消的订单ID
└─ 持久化到MemStore

PrepareCheckState:
├─ 读取ProcessProposerMatchesEvents
├─ 清除MemClob中的这些订单
├─ 从KVStore重新加载长期订单
├─ 重放本地操作
└─ MemClob状态 = DeliverState + 本地未确认操作

CheckTx (区块N+1):
├─ 基于同步后的MemClob
├─ 新订单可以正确验证
└─ 不会与已完成的订单冲突

区块N+1被提议:
├─ PrepareProposal使用当前MemClob
├─ 生成新的MsgProposedOperations
└─ 包含区块N+1的所有操作

DeliverTx (区块N+1):
├─ 持久化新的成交结果
├─ 不修改MemClob
└─ 准备下一轮同步
```

#### D.2.6 修改总结表

| 阶段 | MemClob订单簿 | MemClob操作队列 | KVStore | MemStore | TransientStore |
|------|-------------|---------------|---------|----------|---------------|
| **CheckTx** | ✅ 添加订单 | ✅ 记录操作 | ❌ | ❌ | ✅ 未提交订单 |
| **DeliverTx** | ❌ 不修改 | ❌ 不修改 | ✅ 持久化填充量 | ✅ 记录已交付 | ❌ |
| **PrepareCheckState** | ✅ 清除+重建 | ✅ 清空+重放 | 🔍 读取 | 🔍 读取 | ❌ |
| **EndBlocker** | ❌ | ❌ | ✅ 删除过期 | ✅ 更新事件 | ❌ |

**图示说明**：
- ✅：写入/修改
- ❌：不操作
- 🔍：只读

#### D.2.7 常见误解澄清

**误解1**："DeliverTx也修改MemClob"
- ❌ **错误**：DeliverTx不调用`MemClob.PlaceOrder()`
- ✅ **正确**：DeliverTx只更新KVStore中的持久化状态
- 📍 **证据**：[msg_server_proposed_operations.go](protocol/x/clob/keeper/msg_server_proposed_operations.go) 中只有`PersistMatchToState()`

**误解2**："CheckTx的MemClob修改会影响共识"
- ❌ **错误**：CheckTx修改的是CheckState，是临时的
- ✅ **正确**：只有DeliverState的持久化状态参与共识
- 📍 **机制**：PrepareCheckState每个区块都重新同步

**误解3**："MemClob在所有节点上都相同"
- ❌ **错误**：不同节点的MemClob可能不同
- ✅ **正确**：
  - DeliverState中的KVStore是一致的（共识保证）
  - CheckState中的MemClob可能包含本地未确认订单
  - PrepareCheckState同步核心状态，但保留本地操作

**误解4**："短期订单不持久化"
- ⚠️ **部分正确**：订单本身不持久化
- ✅ **更准确**：订单的**填充量**会持久化到KVStore
- 📍 **位置**：`OrderAmountFilledKeyPrefix = "Fill:"`

#### D.2.8 实际代码验证

让我们看一些关键代码片段来验证上述分析：

**CheckTx确实修改MemClob**：
```go
// protocol/x/clob/memclob/memclob.go:PlaceOrder()
func (m *MemClobPriceTimePriority) PlaceOrder(...) {
    // ...验证...

    // 添加到订单簿 - 这是对MemClob的修改！
    m.mustAddOrderToOrderbook(ctx, order, isPostOnlyOrder)

    // 添加到操作队列 - 这也是对MemClob的修改！
    m.operationsToPropose.MustAddShortTermOrderPlacementToOperationsQueue(...)
}
```

**DeliverTx不修改MemClob**：
```go
// protocol/x/clob/keeper/msg_server_proposed_operations.go
func (k msgServer) ProposedOperations(...) {
    lib.AssertDeliverTxMode(ctx)  // 确保在DeliverTx

    // 处理操作
    for _, operation := range operations {
        k.PersistMatchToState(ctx, match)  // 写KVStore
        k.AddDeliveredLongTermOrderId(ctx, orderId)  // 写MemStore
        // 注意：没有调用MemClob.PlaceOrder()！
    }
}
```

**PrepareCheckState同步状态**：
```go
// protocol/x/clob/abci.go
func (k Keeper) PrepareCheckState(ctx sdk.Context) {
    // 1. 清除MemClob
    k.MemClob.RemoveAndClearOperationsQueue(ctx, operations)

    // 2. 清除无效状态
    k.MemClob.PurgeInvalidMemclobState(ctx, filledOrders, ...)

    // 3. 从KVStore重新加载
    k.PlaceStatefulOrdersFromLastBlock(ctx, orderIds, ...)
    // ↑ 内部调用 MemClob.PlaceOrder()

    // 4. 重放本地操作
    k.MemClob.ReplayOperations(ctx, operations, ...)
    // ↑ 也调用 MemClob.PlaceOrder()
}
```

#### D.2.9 结论

**问题1答案**：**是的，CheckTx阶段会修改MemClob状态**
- 添加订单到订单簿
- 记录操作到操作队列
- 这些修改在CheckState中，不影响DeliverState

**问题2答案**：**不，DeliverTx（共识后执行）不修改MemClob**
- 仅更新KVStore中的持久化状态
- 记录已交付订单到MemStore
- MemClob在PrepareCheckState时同步

**关键洞察**：
- **CheckState的MemClob**：可变的、本地的、临时的
- **DeliverState的KVStore**：不可变的、全局的、持久化的
- **PrepareCheckState**：桥接两者，确保一致性

这种设计允许：
1. 快速的CheckTx验证（使用MemClob）
2. 确定性的共识（基于KVStore）
3. 本地优化（保留未确认操作）
4. 最终一致性（通过PrepareCheckState同步）

---

### D.3 深入解答：CheckState的分布式执行与状态一致性

#### D.3.1 关键疑问

在理解D.2的内容后，会产生两个重要的疑问：

**疑问1**：CheckTx阶段对MemClob状态进行修改是所有的验证者和全节点吗？还是只是出块节点？

**疑问2**：没有进行共识就在CheckTx阶段进行MemClob状态修改，那么多节点执行时多笔交易的顺序不一样会导致修改后状态不一致的问题，如何解释？

这两个问题触及了分布式共识系统的核心设计原则。

#### D.3.2 疑问1：哪些节点执行CheckTx并修改MemClob？

**答案：所有节点都执行CheckTx并修改各自本地的MemClob**

这包括：
- ✅ 所有验证者节点（包括当前的出块节点和其他验证者）
- ✅ 所有全节点

**代码证据1：CheckTx在所有节点上执行**

```go
// 文件: protocol/x/clob/ante/clob.go (第201-209行)
func (cd ClobDecorator) AnteHandle(
    ctx sdk.Context,
    tx sdk.Tx,
    simulate bool,
    next sdk.AnteHandler,
) (newCtx sdk.Context, err error) {
    // 检查是否包含CLOB消息
    if clobante.HasClobMsg(tx) {
        return h.clobAnteHandle(ctx, tx, simulate)
        // ↑ 所有节点都执行此逻辑
    }
    return next(ctx, tx, simulate)
}
```

**代码证据2：时序图中的明确表示**

在本文档第3.1节的时序图中明确显示：
```
T1: CheckTx阶段 (并发)
Val1->>Val1: CheckTx流程
Val2->>Val2: 同样的CheckTx流程  ← 其他验证者
FullNode->>FullNode: 同样的CheckTx流程  ← 全节点
```

**代码证据3：短期订单处理**

```go
// 文件: protocol/x/clob/keeper/process_operations.go (第175-237行)
func (k Keeper) PlaceShortTermOrder(
    ctx sdk.Context,
    msg *types.MsgPlaceOrder,
) (satypes.BaseQuantums, types.OrderStatus, error) {
    lib.AssertCheckTxMode(ctx)  // 断言在CheckTx模式

    // 所有节点都执行以下操作
    orderSizeOptimisticallyFilledFromMatchingQuantums, orderStatus, offchainUpdates, err :=
        k.MemClob.PlaceOrder(ctx, msg.Order)
    // ↑ 修改本地MemClob

    return orderSizeOptimisticallyFilledFromMatchingQuantums, orderStatus, err
}
```

**关键点**：
- 每个节点维护**独立的MemClob实例**
- 每个节点的CheckState是**独立的**
- 所有节点都修改**各自本地的**MemClob

#### D.3.3 疑问2：如何处理不同节点MemClob状态不一致？

这是一个核心问题！让我们深入分析。

##### D.3.3.1 确实会出现不一致！

**是的，不同节点的CheckState MemClob确实可能不同，这是设计如此！**

**原因1：交易到达顺序不同**

```
场景：用户A和用户B几乎同时提交订单

验证者1的视角：
├─ T=0ms: 收到订单A
├─ T=5ms: CheckTx处理订单A，添加到MemClob
├─ T=10ms: 收到订单B
└─ T=15ms: CheckTx处理订单B，可能与A撮合

验证者2的视角：
├─ T=0ms: 收到订单B
├─ T=5ms: CheckTx处理订单B，添加到MemClob
├─ T=12ms: 收到订单A
└─ T=17ms: CheckTx处理订单A，可能与B撮合

结果：两个验证者的MemClob状态不同！
```

**原因2：网络传播延迟**

```
┌─────────────────────────────────────────────────┐
│         P2P网络中的交易传播                      │
├─────────────────────────────────────────────────┤
│                                                 │
│  User                                           │
│   │                                             │
│   ├──Tx1──> Val1 (10ms)                        │
│   │         │                                   │
│   │         └──Tx1──> Val2 (50ms via gossip)   │
│   │                   │                         │
│   └──Tx2──> Val3 (15ms)                        │
│             │                                   │
│             └──Tx2──> Val2 (30ms via gossip)   │
│                                                 │
│  Val2的接收顺序: Tx2 (30ms) → Tx1 (50ms)       │
│  其他节点的顺序可能不同                          │
│                                                 │
└─────────────────────────────────────────────────┘
```

**代码证据：使用NoOpMempool**

```go
// 文件: protocol/mempool/noop.go
type noOpMempool struct{}

func (noOpMempool) Insert(context.Context, sdk.Tx) error {
    return nil  // 完全丢弃交易，不排序
}

func (noOpMempool) Select(context.Context, [][]byte) mempool.Iterator {
    return nil  // 不从Mempool选择交易
}

// 注释说明：
// Note: When this mempool is used, it assumed that an application will rely
// on Tendermint's transaction ordering defined in `RequestPrepareProposal`
// ↑ 交易顺序由Tendermint的PrepareProposal决定，不是Mempool
```

##### D.3.3.2 为什么这种不一致是可接受的？

**核心原则：CheckState不参与共识！**

```
┌──────────────────────────────────────────────────────┐
│              状态的两个独立世界                       │
├──────────────────────────────────────────────────────┤
│                                                      │
│  CheckState（检查状态）                               │
│  ├─ 用途：验证交易、提供快速反馈                      │
│  ├─ 存储：CacheMultiStore（内存缓存）                │
│  ├─ 一致性：不要求、可以不同                          │
│  ├─ 生命周期：临时的，每个区块后重建                  │
│  └─ MemClob：本地的、乐观的撮合                      │
│                                                      │
│  DeliverState（交付状态）                            │
│  ├─ 用途：执行共识后的交易、更新链状态                │
│  ├─ 存储：KVStore（持久化磁盘）                      │
│  ├─ 一致性：必须相同、通过共识保证                    │
│  ├─ 生命周期：永久的，区块提交后持久化                │
│  └─ MemClob：仅在PrepareCheckState时使用            │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**代码证据：CacheMultiStore的隔离**

```go
// 文件: protocol/x/clob/ante/clob.go (第241-256行)
func (cd ClobDecorator) clobAnteHandle(
    ctx sdk.Context,
    tx sdk.Tx,
    simulate bool,
) (newCtx sdk.Context, err error) {
    var cacheMs storetypes.CacheMultiStore

    if !simulate && (ctx.IsCheckTx() || ctx.IsReCheckTx()) {
        // 创建缓存存储，完全隔离于持久状态
        cacheMs = ctx.MultiStore().(cachemulti.Store).CacheMultiStoreWithLocking(
            map[storetypes.StoreKey][][]byte{
                h.authStoreKey: signers,
            },
        )
        defer cacheMs.(storetypes.LockingStore).Unlock()

        // 使用缓存存储
        ctx = ctx.WithMultiStore(cacheMs)
    }

    // ... 执行CheckTx逻辑 ...

    if err == nil && !simulate && (ctx.IsCheckTx() || ctx.IsReCheckTx()) {
        // 写入缓存（但不提交到持久化存储）
        cacheMs.Write()
        // ↑ 这只是写入内存缓存，不是KVStore！
    }

    return ctx, err
}
```

##### D.3.3.3 共识如何保证最终一致性？

**关键机制：PrepareProposal决定最终顺序**

```
区块N的共识流程：

1. 提议阶段（仅提议者）：
   ┌─────────────────────────────────────────────┐
   │ PrepareProposal (提议者 = Val1)             │
   ├─────────────────────────────────────────────┤
   │ 1. 从Mempool收集交易（本地顺序）             │
   │ 2. Val1的MemClob生成MsgProposedOperations  │
   │ 3. 确定最终的交易顺序：                      │
   │    [Tx1, Tx2, Tx3, ..., MsgProposedOps]    │
   │ 4. 广播区块提议                             │
   └─────────────────────────────────────────────┘

2. 验证阶段（所有验证者）：
   ┌─────────────────────────────────────────────┐
   │ ProcessProposal (所有验证者)                │
   ├─────────────────────────────────────────────┤
   │ 1. 接收提议的区块                            │
   │ 2. 验证交易顺序和内容                        │
   │ 3. 投票ACCEPT或REJECT                       │
   │ 4. 不执行交易（还没到DeliverTx）             │
   └─────────────────────────────────────────────┘

3. 共识阶段：
   ┌─────────────────────────────────────────────┐
   │ Tendermint共识                              │
   ├─────────────────────────────────────────────┤
   │ 1. Prevote投票                              │
   │ 2. Precommit投票                            │
   │ 3. 达成共识（>2/3投票）                      │
   │ 4. 区块被确认                               │
   └─────────────────────────────────────────────┘

4. 执行阶段（所有节点）：
   ┌─────────────────────────────────────────────┐
   │ DeliverTx (所有节点)                        │
   ├─────────────────────────────────────────────┤
   │ 1. 按提议者确定的顺序执行交易：              │
   │    - Tx1 → DeliverTx                       │
   │    - Tx2 → DeliverTx                       │
   │    - Tx3 → DeliverTx                       │
   │    - MsgProposedOps → DeliverTx            │
   │                                             │
   │ 2. 所有节点执行相同的交易顺序                │
   │ 3. 更新KVStore（持久化状态）                │
   │ 4. 所有节点的DeliverState相同！             │
   └─────────────────────────────────────────────┘
```

**代码证据：PrepareProposal决定顺序**

```go
// 文件: protocol/app/prepare/prepare_proposal.go (第55-170行)
func PrepareProposalHandler(...) sdk.PrepareProposalHandler {
    return func(ctx sdk.Context, req *abci.RequestPrepareProposal)
        (*abci.ResponsePrepareProposal, error) {

        // 提议者从本地Mempool收集交易
        txsFromMempool := req.Txs

        // 提议者的MemClob生成操作
        operations := keeper.MemClob.GetOperationsToPropose(ctx)

        // 提议者决定最终顺序
        txs := [][]byte{
            priceUpdateTx,
            premiumVotesTx,
            bridgesTx,
            ...txsFromMempool,  // 其他交易
            operationsTx,       // MsgProposedOperations
        }

        // 这个顺序对所有节点都是权威的！
        return &abci.ResponsePrepareProposal{Txs: txs}, nil
    }
}
```

#### D.3.4 完整的状态流转图

```
┌────────────────────────────────────────────────────────────────┐
│                    区块N-1提交                                  │
│  所有节点的DeliverState相同（共识保证）                         │
└────────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│                 PrepareCheckState                              │
│  ├─ 从DeliverState读取共识状态                                 │
│  ├─ 清除本地MemClob                                            │
│  ├─ 从KVStore重新加载订单 → MemClob                            │
│  └─ 重放本地操作                                               │
│  结果: MemClob = 共识状态 + 本地操作                            │
└────────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│                 CheckTx阶段（区块N）                            │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  验证者1     │  │  验证者2     │  │  全节点      │         │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤         │
│  │ CheckState   │  │ CheckState   │  │ CheckState   │         │
│  │ MemClob: A,B │  │ MemClob: B,A │  │ MemClob: A,C │         │
│  │ (顺序不同)   │  │ (顺序不同)   │  │ (顺序不同)   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ↑ 每个节点的CheckState可以不同！                              │
│  ↑ 这是正常的、预期的行为                                      │
└────────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│            PrepareProposal（仅提议者 = 验证者1）                │
│  ├─ 使用验证者1的MemClob                                       │
│  ├─ 生成MsgProposedOperations                                  │
│  ├─ 确定交易顺序: [Tx1, Tx2, ..., Operations]                 │
│  └─ 广播区块提议                                               │
└────────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│            ProcessProposal（所有验证者）                        │
│  所有验证者验证提议，投票ACCEPT或REJECT                         │
└────────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│                 Tendermint共识                                 │
│  达成共识（>2/3验证者投票）                                     │
└────────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│            DeliverTx阶段（所有节点）                            │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  验证者1     │  │  验证者2     │  │  全节点      │         │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤         │
│  │ DeliverState │  │ DeliverState │  │ DeliverState │         │
│  │ 执行: Tx1    │  │ 执行: Tx1    │  │ 执行: Tx1    │         │
│  │ 执行: Tx2    │  │ 执行: Tx2    │  │ 执行: Tx2    │         │
│  │ 执行: Ops    │  │ 执行: Ops    │  │ 执行: Ops    │         │
│  │ 更新KVStore  │  │ 更新KVStore  │  │ 更新KVStore  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ↑ 所有节点执行相同顺序的相同交易                               │
│  ↑ DeliverState必须相同（共识保证）                            │
└────────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│                    区块N提交                                    │
│  所有节点的DeliverState相同（共识保证）                         │
└────────────────────────────────────────────────────────────────┘
                          ↓
                   (循环到区块N+1)
```

#### D.3.5 关键设计原则总结

**1. 状态分离原则**

| 状态类型 | CheckState | DeliverState |
|---------|-----------|-------------|
| **用途** | 交易验证、快速反馈 | 共识执行、状态更新 |
| **存储** | CacheMultiStore（内存） | KVStore（磁盘） |
| **一致性要求** | 不要求 | 必须相同 |
| **生命周期** | 临时（每块重建） | 永久（持久化） |
| **MemClob角色** | 乐观撮合 | 权威结果 |
| **修改来源** | 本地CheckTx | 共识DeliverTx |
| **节点间差异** | 允许不同 | 必须相同 |

**2. 两阶段一致性模型**

```
阶段1: CheckTx（本地的、乐观的）
├─ 目标：快速验证、提供反馈
├─ 方法：使用本地MemClob
├─ 结果：可能不同
└─ 不影响共识

阶段2: DeliverTx（全局的、确定的）
├─ 目标：执行共识后的交易
├─ 方法：按提议者确定的顺序
├─ 结果：必须相同
└─ 通过共识保证

桥接: PrepareCheckState（同步点）
├─ 清除本地CheckState
├─ 从DeliverState重建
└─ 确保下个区块从相同基准开始
```

**3. 为什么这种设计是优越的**

**优势1：性能**
- CheckTx可以并发执行
- 不需要等待共识即可验证
- 用户获得快速反馈（10-100ms）

**优势2：安全**
- CheckState不影响共识
- 恶意节点无法通过CheckTx攻击
- DeliverState通过共识保证安全

**优势3：灵活性**
- 节点可以有本地优化
- 验证者可以保留未确认操作
- 不影响全局一致性

**优势4：可扩展性**
- CheckTx可以轻量级处理
- 不需要在所有节点间同步CheckState
- 减少网络开销

#### D.3.6 最终答案

**疑问1答案**：
CheckTx在**所有节点**（验证者和全节点）上执行，每个节点都修改**各自本地的MemClob**。

**疑问2答案**：
不同节点的CheckState MemClob**确实可能不同**，这是**设计如此**：
- CheckState是临时的、本地的，使用CacheMultiStore隔离
- 只有DeliverState参与共识，通过共识保证一致
- PrepareCheckState在每个区块后从DeliverState重新同步
- 最终一致性由共识机制保证

**核心洞察**：
dYdX v4巧妙地利用了Cosmos SDK的状态分离机制，在保证共识安全的同时，允许本地优化和快速反馈。这是分布式系统设计中的经典权衡：**牺牲临时状态的一致性，换取性能和用户体验**。

---

### D.4 长期订单 vs 短期订单：CEX与DEX的根本差异

#### D.4.1 问题的本质：为什么DEX需要区分订单类型？

在中心化交易所（CEX）中，确实没有"长期订单"这个概念。这是因为CEX和DEX面临的技术约束完全不同。

#### D.4.2 CEX的订单生命周期

**在币安、Coinbase等CEX中**：

```
订单提交后的存储和处理：

用户提交订单
    ↓
写入交易所数据库（MySQL/PostgreSQL等）
    ↓
订单簿引擎处理
    ├─ GTC (Good Till Cancelled)：一直有效直到取消
    ├─ GTD (Good Till Date)：有效到指定日期
    ├─ IOC (Immediate or Cancel)：立即成交或取消
    ├─ FOK (Fill or Kill)：全部成交或取消
    └─ POST ONLY：只做maker，不立即成交

关键特点：
✅ 数据库写入成本极低（几微秒）
✅ 可以存储数百万个活跃订单
✅ 查询速度快（内存缓存 + 索引）
✅ 订单可以长期有效（几天、几周、甚至永久）
✅ 不需要考虑"存储成本"
```

**CEX的优势**：
- 中心化数据库，存储成本几乎为零
- 内存中的订单簿引擎，处理速度极快
- 订单生命周期完全由交易所控制

#### D.4.3 DEX的技术约束

**在dYdX v4这样的链上DEX中，面临完全不同的挑战**：

```
┌─────────────────────────────────────────────────────────┐
│              区块链的技术约束                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 约束1: 区块空间有限                                     │
│ ├─ 每个区块最大 ~2MB                                   │
│ ├─ 需要包含所有类型的交易                               │
│ └─ 订单数据占用宝贵的区块空间                           │
│                                                         │
│ 约束2: 存储成本高昂                                     │
│ ├─ 每个验证者都需要存储完整状态                         │
│ ├─ 状态增长 → 硬件成本增加                             │
│ └─ 需要控制链上状态的大小                               │
│                                                         │
│ 约束3: 共识开销                                         │
│ ├─ 每次状态修改都需要共识                               │
│ ├─ 共识需要时间（~1秒）                                │
│ └─ 不能频繁修改订单状态                                 │
│                                                         │
│ 约束4: Gas费用                                          │
│ ├─ 每个链上操作都需要Gas                               │
│ ├─ 存储写入的Gas成本最高                               │
│ └─ 用户需要为订单存储付费                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**如果把所有订单都存储在链上会怎样？**

```
假设场景：像CEX一样处理所有订单

问题1：区块空间爆炸
├─ 用户A提交1000个限价单
├─ 每个订单 ~200 bytes
├─ 总大小：200KB
├─ 占用单个区块10%的空间！
└─ 其他交易无法处理

问题2：状态膨胀
├─ 10万个活跃订单
├─ 每个订单存储：~500 bytes（包含索引）
├─ 总状态：50MB
├─ 每个验证者都要存储
└─ 状态越大，同步越慢

问题3：Gas成本
├─ 每个订单写入：~100,000 gas
├─ 如果用户提交100个订单
├─ 总成本：10,000,000 gas
└─ 在以太坊上可能要花费几百美元！

结论：不可行！
```

#### D.4.4 dYdX v4的解决方案：订单类型分层

**核心思想**：根据订单的生命周期需求，采用不同的存储策略

```
┌─────────────────────────────────────────────────────────┐
│           短期订单 (Short-Term Orders)                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 定义：                                                   │
│ ├─ GoodTilBlock：当前区块 + 1 到 当前区块 + 30         │
│ ├─ 最大生命周期：~30秒（假设1秒/块）                    │
│ └─ 类似于CEX的IOC订单，但时间单位是区块                 │
│                                                         │
│ 存储位置：                                               │
│ ├─ ✅ MemClob（纯内存）                                │
│ ├─ ❌ 不写入KVStore                                    │
│ ├─ ✅ 填充量持久化（仅记录成交部分）                    │
│ └─ ⚡ 每个区块后自动清理过期订单                        │
│                                                         │
│ 适用场景：                                               │
│ ├─ 高频交易                                             │
│ ├─ 快速成交                                             │
│ ├─ 市价单（IOC）                                        │
│ ├─ 套利交易                                             │
│ └─ 不需要长时间挂单的订单                               │
│                                                         │
│ 优势：                                                   │
│ ├─ 💰 零Gas成本（不占用链上存储）                      │
│ ├─ ⚡ 处理速度快（纯内存操作）                          │
│ ├─ 🔄 自动清理（不增加状态大小）                        │
│ └─ 🚀 适合高频场景                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│           长期订单 (Long-Term Orders)                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 定义：                                                   │
│ ├─ GoodTilBlockTime：使用Unix时间戳                     │
│ ├─ 可以有效几分钟、几小时、甚至几天                     │
│ └─ 类似于CEX的GTC或GTD订单                              │
│                                                         │
│ 存储位置：                                               │
│ ├─ ✅ KVStore（持久化磁盘）                            │
│ ├─ ✅ MemClob（运行时缓存）                            │
│ ├─ ✅ 过期索引（快速查找）                              │
│ └─ 💾 每个区块后从KVStore重新加载                       │
│                                                         │
│ 适用场景：                                               │
│ ├─ 限价挂单                                             │
│ ├─ 止损/止盈单                                          │
│ ├─ 长期策略                                             │
│ ├─ 流动性提供                                           │
│ └─ 需要跨区块有效的订单                                 │
│                                                         │
│ 优势：                                                   │
│ ├─ 📅 长期有效（不受区块数限制）                        │
│ ├─ 🔒 共识保证（所有节点存储）                          │
│ ├─ 🔄 持久化（节点重启后仍然存在）                      │
│ └─ 💪 适合做市商和流动性提供者                          │
│                                                         │
│ 成本：                                                   │
│ ├─ 💸 需要Gas费用（写入链上存储）                      │
│ ├─ 📦 占用区块空间                                      │
│ └─ 💾 增加链状态大小                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### D.4.5 对比表：CEX vs DEX订单类型

| 维度 | CEX (币安/Coinbase) | dYdX v4 短期订单 | dYdX v4 长期订单 |
|------|-------------------|----------------|----------------|
| **时间单位** | 时间（秒/分/天） | 区块数 | 时间戳 |
| **最大有效期** | 无限制（GTC） | ~30秒 | 几天 |
| **存储位置** | 中心化数据库 | 内存 | 链上KVStore |
| **存储成本** | 几乎为零 | 零 | 需要Gas |
| **持久化** | 永久 | 临时 | 永久 |
| **共识要求** | 无 | 无 | 需要 |
| **节点重启** | 无影响 | 丢失 | 保留 |
| **修改/取消成本** | 免费 | 免费 | 需要Gas |
| **适用场景** | 所有类型 | 高频交易 | 限价挂单 |
| **类比** | GTC/GTD/IOC | IOC with blocks | GTC/GTD |

#### D.4.6 实际代码实现的差异

**短期订单的提交**：

```go
// 文件: protocol/x/clob/keeper/process_operations.go
func (k Keeper) PlaceShortTermOrder(
    ctx sdk.Context,
    msg *types.MsgPlaceOrder,
) (satypes.BaseQuantums, types.OrderStatus, error) {
    lib.AssertCheckTxMode(ctx)  // 只在CheckTx执行

    // 直接添加到MemClob，不写入KVStore
    orderSizeOptimisticallyFilledFromMatchingQuantums, orderStatus, offchainUpdates, err :=
        k.MemClob.PlaceOrder(ctx, msg.Order)
    // ↑ 纯内存操作，零链上成本

    // 只有成交部分才写入KVStore
    if orderSizeOptimisticallyFilledFromMatchingQuantums > 0 {
        k.SetOrderFillAmount(ctx, orderId, fillAmount, ...)
        // ↑ 这是在DeliverTx时才执行
    }

    return orderSizeOptimisticallyFilledFromMatchingQuantums, orderStatus, err
}
```

**长期订单的提交**：

```go
// 文件: protocol/x/clob/keeper/msg_server_place_order.go
func (k msgServer) PlaceOrder(
    ctx sdk.Context,
    msg *types.MsgPlaceOrder,
) (*types.MsgPlaceOrderResponse, error) {
    lib.AssertDeliverTxMode(ctx)  // 在DeliverTx执行

    // 写入KVStore - 持久化！
    k.SetLongTermOrderPlacement(ctx, msg.Order, ctx.BlockHeight())
    // ↑ 写入磁盘，需要Gas，占用区块空间

    // 添加过期索引
    k.AddStatefulOrderIdExpiration(ctx, msg.Order.GoodTilBlockTime, msg.Order.OrderId)

    // 增加订单计数
    k.CheckAndIncrementStatefulOrderCount(ctx, msg.Order.OrderId)

    return &types.MsgPlaceOrderResponse{}, nil
}
```

#### D.4.7 生命周期对比

**短期订单的一生**：

```
T=0s: 用户提交短期订单（GTB = 当前块 + 20）
├─ CheckTx阶段
├─ 添加到MemClob
└─ 用户收到反馈（~100ms）

T=0-20s: 订单在MemClob中活跃
├─ 参与撮合
├─ 可能部分成交
└─ 每个区块都重新验证

T=20s: 到达GoodTilBlock
├─ EndBlocker自动清理
├─ 从MemClob移除
└─ 生命周期结束

如果节点重启：
└─ ❌ 订单丢失（因为只在内存中）
```

**长期订单的一生**：

```
T=0s: 用户提交长期订单（GTT = 现在 + 24小时）
├─ CheckTx阶段验证
├─ 写入UncommittedStatefulOrderPlacementTransientStore
└─ 用户收到反馈（~100ms）

T=1s: 区块确认
├─ DeliverTx阶段
├─ SetLongTermOrderPlacement() → KVStore
├─ 持久化到磁盘
└─ 订单已确认

T=1s-24h: 订单跨区块有效
├─ 每个PrepareCheckState从KVStore加载
├─ 放置到MemClob参与撮合
├─ 可能部分成交
└─ 填充量持续更新

T=24h: 到达GoodTilBlockTime
├─ EndBlocker检查过期
├─ RemoveExpiredStatefulOrders()
├─ 从KVStore删除
└─ 生命周期结束

如果节点重启：
└─ ✅ 订单保留（从KVStore恢复）
```

#### D.4.8 为什么不能都用长期订单？

**如果所有订单都作为长期订单处理**：

```
问题1：Gas费用暴涨
├─ 每个订单都需要写入KVStore
├─ 高频交易者每秒提交10个订单
├─ 每个订单 ~100,000 gas
├─ 每秒需要 1,000,000 gas
└─ Gas成本会让高频交易不可行

问题2：区块空间浪费
├─ 90%的订单在几秒内成交或取消
├─ 这些订单不需要持久化
├─ 但占用了宝贵的区块空间
└─ 导致其他交易无法处理

问题3：状态膨胀
├─ 链状态快速增长
├─ 验证者硬件要求提高
├─ 新节点同步时间增加
└─ 网络去中心化程度降低

问题4：PrepareCheckState开销
├─ 每个区块都要重新加载所有订单
├─ 订单越多，加载时间越长
├─ 影响区块生成速度
└─ 系统性能下降
```

#### D.4.9 为什么不能都用短期订单？

**如果所有订单都作为短期订单处理**：

```
问题1：用户体验差
├─ 限价单需要长期有效
├─ 用户不想每30秒重新提交
├─ 做市商需要持续挂单
└─ 无法提供流动性

问题2：节点重启风险
├─ 验证者节点重启
├─ 所有短期订单丢失
├─ 用户订单消失
└─ 不可接受的风险

问题3：不支持复杂策略
├─ 止损单需要长期监控
├─ 条件单需要持久化
├─ TWAP订单需要跨多个区块
└─ 策略交易无法实现

问题4：流动性不足
├─ 订单簿深度不够
├─ 缺少长期挂单
├─ 价差较大
└─ 交易体验差
```

#### D.4.10 最佳实践：如何选择订单类型

**使用短期订单的场景**：

```
✅ 市价单（想要立即成交）
✅ IOC订单（Immediate or Cancel）
✅ 高频交易（秒级交易）
✅ 套利交易（时间敏感）
✅ 快速止损（短期内执行）
✅ 测试订单（不想持久化）

示例代码：
order := types.Order{
    OrderId: types.OrderId{
        OrderFlags: types.ShortTermOrderFlags,  // 短期订单
        ...
    },
    GoodTilBlock: currentBlock + 20,  // 20个区块后过期
    Side: types.Order_SIDE_BUY,
    Quantums: 1000000,  // 1 BTC
    Subticks: 50000000, // 价格
    TimeInForce: types.Order_TIME_IN_FORCE_IOC,  // 立即成交或取消
}
```

**使用长期订单的场景**：

```
✅ 限价挂单（等待价格到达）
✅ 止损单（长期监控）
✅ 做市商策略（持续提供流动性）
✅ 网格交易（多个价格级别）
✅ 条件单（触发后执行）
✅ TWAP订单（时间加权平均价格）

示例代码：
order := types.Order{
    OrderId: types.OrderId{
        OrderFlags: types.LongTermOrderFlags,  // 长期订单
        ...
    },
    GoodTilBlockTime: currentTime + 86400,  // 24小时后过期
    Side: types.Order_SIDE_SELL,
    Quantums: 1000000,  // 1 BTC
    Subticks: 55000000, // 限价
    TimeInForce: types.Order_TIME_IN_FORCE_POST_ONLY,  // 只做maker
}
```

#### D.4.11 与CEX订单类型的对应关系

**CEX → dYdX v4 映射**：

| CEX订单类型 | dYdX v4对应 | 说明 |
|-----------|-----------|------|
| **Market** | 短期订单 + IOC | 立即成交，不持久化 |
| **Limit (GTC)** | 长期订单 + GTC | 持久化，长期有效 |
| **Limit (GTD)** | 长期订单 + GTT | 指定过期时间 |
| **IOC** | 短期订单 + IOC | 立即成交或取消 |
| **FOK** | 短期订单 + FOK | 全部成交或取消 |
| **Post Only** | 长期订单 + POST_ONLY | 只做maker |
| **Stop Loss** | 长期条件订单 | 触发条件 + 执行订单 |
| **Stop Limit** | 长期条件订单 | 触发 + 限价 |
| **TWAP** | 长期TWAP订单 | 分批执行 |

**实际使用示例**：

```
场景1：用户想以市价买入1 BTC
CEX方式：提交Market订单
dYdX方式：提交短期订单 + IOC
├─ GoodTilBlock: 当前 + 1（只在下个区块有效）
└─ TimeInForce: IOC

场景2：用户想在$55,000挂单卖出1 BTC
CEX方式：提交Limit订单 (GTC)
dYdX方式：提交长期订单 + POST_ONLY
├─ GoodTilBlockTime: 现在 + 7天
├─ Subticks: 55000 * priceExponent
└─ TimeInForce: POST_ONLY

场景3：用户想在BTC跌破$50,000时止损
CEX方式：提交Stop Loss订单
dYdX方式：提交长期条件订单
├─ TriggerPrice: $50,000
├─ ConditionalOrderType: STOP_LOSS
└─ GoodTilBlockTime: 现在 + 30天
```

#### D.4.12 性能对比

**处理1000个订单的成本**：

| 操作 | CEX | dYdX短期 | dYdX长期 |
|------|-----|---------|---------|
| **提交时延** | <1ms | 100ms | 1-3s |
| **存储成本** | $0 | $0 | ~$10-50 (Gas) |
| **区块空间** | N/A | 0 bytes | ~200KB |
| **内存占用** | ~500KB | ~500KB | ~1MB |
| **持久化** | Yes | No | Yes |
| **撮合速度** | <1ms | ~1s | ~1s |
| **取消成本** | $0 | $0 | ~$1-5 (Gas) |

#### D.4.13 总结

**长期订单的本质**：

```
长期订单 = 传统CEX订单 - 中心化优势 + 链上约束
```

**为什么DEX需要这个概念**：

1. **区块链的技术约束**：
   - 存储成本高
   - 区块空间有限
   - 共识需要时间

2. **性能优化需求**：
   - 高频交易不能等待共识
   - 短期订单零Gas成本
   - 减少链上状态膨胀

3. **用户体验平衡**：
   - 短期订单：快速、免费、适合高频
   - 长期订单：持久、安全、适合挂单

**关键洞察**：

dYdX v4通过引入"短期"和"长期"订单的概念，巧妙地在**链上DEX的技术约束**和**CEX般的用户体验**之间找到了平衡点：

- 短期订单：牺牲持久化，换取零成本和高性能
- 长期订单：付出Gas成本，获得持久性和长期有效性

这是区块链订单簿DEX的一个创新性设计，是DEX走向成熟的重要一步。