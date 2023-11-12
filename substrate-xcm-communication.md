ref: https://docs.substrate.io/learn/xcm-communication/

# 跨共识消息

跨共识通信依赖于一种消息格式——XCM，该格式旨在提供一套通用且可扩展的指令集，用于完成跨不同的共识系统、交易格式和传输协议所创建的边界的交易。

XCM格式表达了消息的内容。每条消息由一组由**发送者**请求的**指令**组成，消息**接收者**可以接受或拒绝这些指令。消息格式与用于发送和接收消息的**消息协议**完全独立。

## 消息协议

在Polkadot生态系统中，有三种主要的通信渠道——消息协议——用于在链之间传输消息：

- 向上消息传递（UMP）使得平行链能够将消息传递给其中继链。
- 向下消息传递（DMP）使得中继链能够将消息传递给平行链。
- 跨共识消息传递（XCMP）使得平行链能够与连接到同一中继链的其他平行链交换消息。

向上和向下的消息传递协议提供了一个垂直的消息传递通道。跨共识消息传递可以被视为一个水平的——平行链到平行链的——传输协议。由于完整的跨共识消息传递（XCMP）协议仍在开发中，水平中继路由消息传递（HRMP）为通过中继链路由到平行链的消息提供了一个临时解决方案。水平中继路由消息传递（HRMP）旨在作为一个临时解决方案，当XCMP发布到生产环境时将被弃用。

尽管这些消息传递协议是Polkadot生态系统中链之间通信的主要手段，但XCM本身并不受这些传输协议的限制。相反，你可以使用XCM来表达许多常见类型的交易，无论消息的来源和目的地在哪里。例如，你可以构造从智能合约或pallet路由的消息，通过桥，或者使用不属于Polkadot生态系统的一部分的传输协议。

![](https://docs.substrate.io/static/b0d45f2ae0f2d3f4bc411ee44623bda3/cd5e4/xcm-channel-overview.webp)

因为XCM专门设计用来通知接收系统应该做什么，所以它可以为许多常见类型的交易提供一个灵活且中立的交易格式。

## XCM格式中的消息

使用XCM格式的消息有四个重要的原则你应该了解：

- 消息是**异步的**。在你发送一条消息后，不要指望系统会立即给一个响应，表明消息已经被传递或执行。
- 消息是**绝对的**，它们保证按**顺序**被传递和解释，并且**有效**地执行。
- 消息是**不对称的**，并且不会将任何结果返回给发送者。你只能使用额外的消息单独地将结果通知给发送者。
- 消息是**中立的**，并且不对消息传递的共识系统做任何假设。

记住这些基本原则，然后你可以开始使用XCM构造消息。在Rust中，消息被定义如下：

```rust
pub struct Xcm<Call>(pub Vec<Instruction<Call>>);
```

此定义表明，消息只是一个执行一组有序指令的调用。`Instruction`类型是一个枚举数据类型，变量的定义顺序通常反映了它们在构造消息时使用的顺序。例如，`WithdrawAsset`是第一个变量，因为它通常在其他指令——如`BuyExecution`或`DepositAsset`——之前在指令列表中执行。

大多数的XCM指令使你能够执行常见任务，如将资产转移到新的位置或在不同的账户中存入资产。执行这些类型任务的指令允许你构造一致的消息，无论你通信的共识系统如何配置，都能按照你的期望去执行。然而，你也可以灵活地定制指令的执行方式或使用`Transact`指令。

`Transact`指令允许你执行由消息接收者公开的任何可调用函数。通过使用`Transact`指令，你可以对接收系统上的任何函数进行调用，但它需要你了解该系统的配置情况。例如，如果你想调用另一个平行链的特定pallet，你必须知道接收方Runtime是如何配置的，以构造正确的消息并达到正确的pallet。这些信息会随着链而变化，因为每个Runtime都可以被不同地配置。


## 在虚拟机中执行


跨共识虚拟机（XCVM）是一个高级虚拟机，它有一个XCM执行器程序，该程序执行它接收到的XCM指令。该程序按顺序执行指令，直到运行结束或遇到错误并停止执行。

当XCM指令被执行时，XCVM通过使用几个专门的寄存器来维护其内部状态。XCVM还可以访问执行指令的底层共识系统的状态。根据执行的操作，XCM指令可能会改变寄存器或共识系统的状态，或者两者都改变。

例如，`TransferAsset`指令指定要转移的资产和资产要转移到的位置。当执行这个指令时，**源寄存器**会自动设置为反映消息来自哪里，并从这个信息中确定应该从哪里取出要转移的资产。执行XCM指令时被操作的另一个寄存器是**持有寄存器**。持有寄存器用于在等待对它们应该做什么的额外指令时临时存储资产。

XCVM中还有几个寄存器用于处理特定的任务。例如，有一个剩余权重寄存器用于存储费用的过高估计，还有一个退款权重寄存器用于存储已经退款的剩余权重的部分。通常，你不能直接修改存储在寄存器中的值。相反，当XCM执行器程序开始时，值会被设置，并且在特定的情况下，或者根据特定的规则，由特定的指令来操作。有关每个寄存器中包含的内容的更多信息，请参阅[XCM参考](https://docs.substrate.io/reference/xcm-reference/)。


## 配置

和Substrate以及基于FRAME的链中的其他组件一样，XCM执行器是模块化和可配置的。你可以使用`Config` trait来配置XCM执行器程序的许多方面，并定制实现来以不同的方式处理XCM指令。例如，`Config` trait提供了以下类型定义：

```rust
/// 用于参数化`XcmExecutor`的trait。
pub trait Config {
    /// 外部调用分发类型。
    type Call: Parameter + Dispatchable<PostInfo = PostDispatchInfo> + GetDispatchInfo;
    /// 如何发送一个后续的XCM消息。
    type XcmSender: SendXcm;
    /// 如何提取和存入一个资产。
    type AssetTransactor: TransactAsset;
    /// 如何从一个`OriginKind`值获取一个调用源。
    type OriginConverter: ConvertOrigin<<Self::Call as Dispatchable>::Origin>;
    /// 作为储备的（位置，资产）对的组合。
    type IsReserve: FilterAssetLocation;
    /// 作为传送器的（位置，资产）对的组合。
    type IsTeleporter: FilterAssetLocation;
    /// 用于反转位置的方法。
    type LocationInverter: InvertLocation;
    /// 是否执行给定的XCM。
    type Barrier: ShouldExecute;
    /// 用于估计XCM执行的权重的处理器。
    type Weigher: WeightBounds<Self::Call>;
    /// 用于购买XCM执行的权重信用的处理器。
    type Trader: WeightTrader;
    /// 用于处理查询的响应的处理器。
    type ResponseHandler: OnResponse;
    /// 用于处理执行后在Holding寄存器中的资产的处理器。
    type AssetTrap: DropAssets;
    /// 用于处理有索取资产的指令的处理器。
    type AssetClaims: ClaimAssets;
    /// 用于处理版本订阅请求的处理器。
    type SubscriptionService: VersionChangeNotifier;
}
```

配置设置和XCM指令集——消息，或者更准确地说，在接收系统上执行的程序——作为XCM执行器的输入。通过XCM构建模块提供的额外类型和函数，XCM执行器按照其提供的顺序逐个解释和执行指令中包含的操作。下图提供了一个简化的工作流程概述。

![](https://docs.substrate.io/static/2d81fb1433ac45ac31e641c4a2078390/b418f/xcvm-overview.webp)


## 位置

因为XCM是一种用于在不同共识系统之间通信的语言，它必须有一种抽象的方式来表达位置，使其具有一般性、灵活性和无歧义性。例如，XCM必须能够识别以下类型的活动的位置：

- 指令应该在哪里执行。
- 资产应该从哪里提取。
- 可以找到接收资产的账户的位置。

对于这些活动，位置可能是在中继链、平行链、外部链、特定链上的账户、特定的智能合约或单个pallet的上下文中。例如，XCM必须能够识别以下类型的位置：

- 一个层级为0的链，如Polkadot或Kusama中继链。
- 一个层级为1的链，如比特币或以太坊主网或平行链。
- 一个层级为2的智能合约，如以太坊上的ERC-20。
- 一个平行链或以太坊上的地址。
- 一个中继链或平行链上的账户。
- 一个基于Frame的Substrate链上的特定pallet。
- 一个基于Frame的Substrate链上的单个pallet实例。

为了在共识系统的上下文中描述位置，XCM使用`MultiLocation`类型。`MultiLocation`类型表达了一个相对于当前位置的位置，由两个参数组成：

- `parents: u8`，用于描述在解释`interior`参数之前，从当前共识位置向上移动的层级数。
- `interior: InteriorMultiLocation`，用于描述在按照`parents`参数指定的相对路径上升后，外部共识系统内部的位置。

`InteriorMultiLocation`使用**连接点**的概念来识别本地共识系统内部的共识系统，每个连接点指定一个相对于前一个更内部的位置。一个没有连接点的`InteriorMultiLocation`只是指本地共识系统（Here）。你可以使用连接点来为XCM指令指定一个平行链、账户或pallet实例相对于外部共识的内部上下文。

例如，以下参数从中继链的上下文中指向一个具有唯一标识符1000的平行链：

```
{ "parents": 1,
"interior": { "X1": [{ "Parachain": 1000 }]}
}
```

在这个例子中，`parents`参数上升一级到父链，`interior`指定一个内部位置，连接点类型为`Parachain`，索引为`1000`。

在文本中，MultiLocation遵循用于描述文件系统路径的惯例。例如，表示为`../PalletInstance(3)/GeneralIndex(42)`的MultiLocation描述了一个有一个父级（..）和两个连接点（`PalletInstance{index: 3}`）和（`GeneralIndex{index: 42}`）的MultiLocation。

有关指定位置和连接点的更多信息，请参阅[通用共识位置标识符](https://github.com/paritytech/xcm-format#7-universal-consensus-location-identifiers)。


## 资产


大多数区块链依赖于某种类型的数字资产来提供对网络安全至关重要的经济激励。一些区块链支持单一的本地资产，而有些其他区块链允许在链上管理多种资产，例如，作为智能合约中定义的资产或非本地代币。还有一些区块链支持非同质化的数字资产，用于一种收藏品。为了让XCM支持这些不同类型的资产，它必须能够以一种通用、灵活和无歧义的方式表达资产。

为了描述链上的资产，XCM使用`MultiAsset`类型。`MultiAsset`类型指定资产的身份以及资产是可替代的还是不可替代的。通常，资产的身份是使用具体位置来指定的。如果资产是可替代的，定义中包括一个数量。

虽然可以使用抽象标识符来识别资产，但具体标识符是一种无需全球资产名称注册表就能无歧义地识别资产的方式。

具体标识符通过其在相对于上下文的共识系统中的位置来特定地识别单一资产。然而，值得注意的是，具体的资产标识符不能只是在共识系统之间复制。相反，资产是使用每个共识系统的相对路径来移动的。我们必须构造相对路径，以便从接收系统的角度来阅读。

对于本地资产——如Polkadot中继链上的DOT——资产标识符通常是铸造资产的链或者从其一个平行链的上下文中向上一级（`..`）。如果一个资产是在一个pallet内管理的，资产标识符使用pallet实例标识符和该pallet内的索引来指定位置。例如，Karura平行链可能会引用Statemine平行链上位置为`../Parachain(1000)/PalletInstance(50)/GeneralIndex(42)`的资产。

有关指定位置和连接点的更多信息，请参阅[通用资产标识符](https://github.com/paritytech/xcm-format#6-universal-asset-identifiers)。


## 指令


大多数XCM指令使你能够构造一致的消息，无论你通信的共识系统如何配置，都能按照你的期望去执行。然而，你也有灵活性可以使用`Transact`指令来执行消息接收者公开的任何可调用函数。通过使用`Transact`指令，你可以对接收系统上的任何函数进行通用调用，但它需要你了解该系统的配置情况。例如，如果你想调用另一个平行链的特定pallet，你必须知道接收Runtime如何配置，以构造正确的消息并到达正确的pallet。这些信息可以从链到链变化，因为每个Runtime都可以被不同地配置。







