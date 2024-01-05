ref: https://docs.substrate.io/build/runtime-storage/
Runtime storage structures

# Runtime存储结构

在你开发Runtime逻辑的过程中，你需要对存储的信息做出重要的决策，以尽可能提高存储信息的效率。如在[状态转换和存储](https://docs.substrate.io/learn/state-transitions-and-storage/)中所讨论的，读取和写入数据到存储是昂贵的。并且存储不必要的大数据集会拖慢你的网络并消耗系统资源。

Substrate被设计为提供一个灵活的框架，允许你构建适合你需求的区块链。然而，在设计Runtime存储时，你应该记住一些基本的指导性原则，以确保你构建的区块链在长期内是安全的、高性能的、可维护的。

## 决定存储什么

区块链Runtime存储的基本原则是尽量减少你存储的条目的数量和大小。例如，你应该只在Runtime存储关键的共识信息。你不应该在Runtime存储中间或临时数据，或者在操作失败时产生的数据。

## 使用哈希后的数据

尽可能使用像哈希这样的技术来减少你必须存储的数据量。例如，许多治理能力——如Democracy pallet中的`propose`函数——允许网络参与者对可分派的call的哈希进行投票，而不是call本身。call的哈希大小总是有界的，而call本身长度没有限制。

在Runtime升级的情况下，使用call的哈希尤其重要，其中可分派的call将整个Runtime Wasm blob作为其参数。因为这些治理机制是在链上实现的，所有需要对给定提案状态达成共识的信息也必须存储在链上 - 这包括正在投票的内容。然而，通过将链上提案绑定到其哈希，Substrate的治理机制允许实现这样的机制，即在提案被批准后再将该提案相关的原始数据放到链上，这意味着存储不会浪费在失败的提案上。

一旦提案通过，某人可以发起实际的可分派call（包括所有参数），它将被哈希并与提案中的哈希值进行比较。

使用哈希来最小化存储在链上的数据的另一个常见模式是将与对象相关的预映像(pre-image)存储在[IPFS](https://docs.ipfs.io/)中；这意味着只需要将IPFS位置（一个大小有界的哈希）存储在链上。

### 避免存储临时数据

不应该使用Runtime存储来存储中间或临时数据（这些数据来自逻辑上原子操作的上下文），或者在操作失败时产生的不需要的数据。这并不意味着在需要多个原子操作的行动的状态跟踪（就像[Utility pallet的多签功能](https://paritytech.github.io/substrate/master/pallet_utility/pallet/enum.Call.html#variant.as_multi)的情况一样）上，不应该使用Runtime存储。在这种情况下，Runtime存储被用来跟踪一个可分派call的签名者，即使一个给定的call可能永远不会收到足够的签名来实际被调用。在这种情况下，每一个签名被认为是正在进行的多签操作中的一个原子事件。单个签名所需的数据在与该签名相关的所有前提条件都被满足后才会被存储。

### 创建边界

创建存储项大小的边界是一种非常有效的控制Runtime存储使用的方法，这在Substrate代码库中被反复使用。一般来说，任何由用户操作的确定大小的存储项都应该有一个边界。上面描述的[Multisig pallet](https://paritytech.github.io/substrate/master/pallet_multisig/pallet/trait.Config.html#associatedtype.MaxSignatories)多签功能就是一个例子。在这种情况下，与多签操作相关的签名者列表由多签参与者提供。因为这个签名者列表是达成对多签操作状态的[共识](https://docs.substrate.io/build/runtime-storage/#what-to-store)所必需的，所以它必须存储在Runtime。然而，为了控制签名者列表可以使用的空间大小，Utility pallet要求用户在写入存储之前必须配置这个大小的边界。

## 交易存储

如在[状态转换和存储](https://docs.substrate.io/learn/state-transitions-and-storage/)中所解释的，Runtime存储涉及到一个底层的键值数据库和内存存储上层Overlay抽象，这些抽象跟踪键和状态变化，直到值被提交到底层数据库。默认情况下，Runtime的函数在提交它们到主存储Overlay之前，将更改写入到一个单一的内存**交易存储层**。如果一个错误阻止了交易的完成，那么交易存储层中的更改将被丢弃，而不是被传递到主存储Overlay，底层数据库中的状态保持不变。

### 添加交易存储层

你可以使用`#[transactional]`宏来扩展交易存储层，生成额外的内存存储Overlay。通过生成额外的内存交易存储Overlay，你可以选择是否要将特定的更改提交到主存储Overlay。额外的交易存储层为你提供了灵活性，可以隔离对特定函数调用的更改，并在任何时候选择要提交的更改。

你还可以嵌套交易存储层，最多可以嵌套十层交易存储层。对于你创建的每一层嵌套交易存储层，你可以选择是否要将更改提交到下面的交易层，这让你可以控制要提交到底层数据库的内容。限制嵌套交易存储层的总数可以限制要提交的更改的计算开销。

### 分派交易存储层调用

如果你想在自己的交易层中分派一个函数调用，你可以使用`dispatch_with_transactional(call)`函数来明确地为调用生成一个新的交易层，并使用该交易层上下文来处理结果。

### 在不使用交易存储层的情况下提交更改

如果你想在不使用默认交易存储层的情况下提交更改到主存储Overlay，你可以使用`#[without_transactional]`宏。`#[without_transactional]`宏使你能够识别一个可以在没有自己的交易层的情况下安全执行的函数。

例如，你可能会定义一个像这样的函数：

```rust
/// This function is safe to execute without an additional transactional storage layer.
#[without_transactional]
fn set_value(x: u32) -> DispatchResult {
    Self::check_value(x)?;
    MyStorage::set(x);
    Ok(())
}
```

调用这个函数不会生成一个交易存储层。

然而，如果你使用`#[without_transactional]`宏，要记住，存储的更改会影响主内存存储Overlay中的值。如果在你修改存储后发生错误，那些更改将会持久化，可能会导致你的数据库处于不一致的状态。

## 访问Runtime存储

在[状态转换和存储](https://docs.substrate.io/learn/state-transitions-and-storage/)中，你了解到Substrate如何使用存储抽象来提供对底层键值数据库的读写访问。FRAME [`Storage`](https://paritytech.github.io/substrate/master/frame_support/storage) 模块简化了对这些分层存储抽象的访问。你可以使用FRAME存储类数据结构来读取或写入任何可以由[SCALE编解码器](https://docs.substrate.io/reference/scale-codec/)编码的值。存储模块提供了以下类型的存储结构：

- [StorageValue](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageValue.html)用于存储任何单一值，如`u64`。
- [StorageMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageMap.html)用于存储单一键值Map，如特定账户键到特定余额值的Map。
- [StorageDoubleMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageDoubleMap.html)用于在具有两个键的存储Map中存储值，作为一种优化，以高效地删除具有共同第一键的条目。
- [StorageNMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageNMap.html)用于在具有任意数量键的Map中存储值。

你可以在pallets中包含这些存储结构，这些新的存储项将成为区块链状态的一部分。你选择实现的存储项类型完全取决于你在Runtime逻辑的上下文中要存储的数据类型。

## 简单存储值

你可以使用 `StorageValue` 存储项来存储Runtime 中的单个值。例如，你应该使用这种类型的存储来处理以下常见的情况：

- 单一的原始值
- 单一的struct数据类型对象
- 单一的集合相关条目

如果你使用这种类型的存储来存储条目列表，你应该注意你存储的列表的大小。大型列表和`struct`会产生存储成本，而在Runtime遍历大型列表或`struct`可能会影响网络性能，甚至完全停止区块的生产。如果遍历存储超过了区块生产时间，并且你的项目是一个[平行链](https://docs.substrate.io/reference/glossary/#parachain)，那么区块链将停止生产区块和运行。

请参阅[StorageValue](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageValue.html#required-methods)文档，以获取StorageValue暴露出来的全部方法的列表。

## 单层键值对Map

Map数据结构非常适合管理随机访问而不是按顺序迭代访问的条目集合。Substrate中的单层键值对Map类似于传统的[哈希Map](https://en.wikipedia.org/wiki/Hash_table)，可以执行随机查找。为了提供灵活性和可控制性，Substrate允许你选择不同的哈希算法来生成key。例如，如果一个Map存储敏感数据，你可能想要使用一个具有更强加密性能的哈希算法来生成键，而不是一个性能更好但加密属性较弱的哈希算法。有关选择Map使用的哈希算法的更多信息，请参阅[哈希算法](https://docs.substrate.io/build/runtime-storage/#hashing-algorithms)。

请参阅[StorageMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageMap.html#required-methods)文档，以获取StorageMap暴露出来的全部方法的列表。


## 双层键值对Map

[DoubleStorageMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageDoubleMap.html)存储项与StorageMap类似，只是它包含两个键。使用这种类型的存储结构对于查询具有公共键的值非常有用。

## 多层键值对Map

[StorageNMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageNMap.html)存储结构也类似于单层和双层键值对Map，但它允许你定义任意数量的键。要在`StorageNMap`结构中指定键，你必须在声明`StorageNMap`时，将包含`NMapKey`结构的元组作为类型提供给Key类型参数。

有关这种类型的存储结构的更多详细信息，请参阅[StorageNMap](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.StorageNMap.html)文档。

## 对存储Map进行迭代

你可以使用键和值来遍历Substrate Runtime 中的 map。然而，需要记住的是，Map通常用于跟踪无界或非常大的数据集，如账户和余额。遍历大型数据集可能会消耗你用于生产区块的有限资源。例如，如果遍历数据集所需的时间超过了生产区块的最大分配时间，Runtime可能会停止生产新的区块，从而阻止链的进展。此外，访问存储Map中的元素所需的数据库读取远超过访问列表中的元素所需的数据库读取。因此，从性能和执行时间的角度来看，遍历存储Map中的元素比读取列表中的元素要昂贵得多。

考虑到相对成本，通常最好避免在Runtime遍历存储Map。然而，关于如何使用Substrate存储能力，并没有固定的规则，最终，你需要决定访问Runtime存储的最佳方式。

Substrate提供了以下方法来使你能够遍历存储Map：

- `iter()`：在没有特定顺序的情况下枚举Map中的所有元素。如果你在此过程中更改Map，你将得到未定义的结果。有关更多信息，请参阅[IterableStorageMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageMap.html#tymethod.iter)，[IterableStorageDoubleMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageDoubleMap.html#tymethod.iter)或[IterableStorageNMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageNMap.html#tymethod.iter)。
- `drain()`：从Map中移除所有元素，并在没有特定顺序的情况下遍历它们。如果你在此过程中向Map中添加元素，你将得到未定义的结果。有关更多信息，请参阅[IterableStorageMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageMap.html#tymethod.iter)，[IterableStorageDoubleMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageDoubleMap.html#tymethod.iter)或[IterableStorageNMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageNMap.html#tymethod.iter)。
- `translate()`：在没有特定顺序的情况下转换Map中的所有元素。要从Map中移除一个元素，从转换函数中返回`None`。有关更多信息，请参阅[IterableStorageMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageMap.html#tymethod.iter)，[IterableStorageDoubleMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageDoubleMap.html#tymethod.iter)或[IterableStorageNMap](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageNMap.html#tymethod.iter)。

## 声明存储项

你可以在任何基于FRAME的pallet中使用`#[pallet::storage]`属性宏来创建Runtime存储项。以下是一些声明不同类型存储项的例子：

### 单值

```
#[pallet::storage]
type SomePrivateValue<T> = StorageValue<
    _,
    u32,
    ValueQuery
>;

#[pallet::storage]
#[pallet::getter(fn some_primitive_value)]
pub(super) type SomePrimitiveValue<T> = StorageValue<_, u32, ValueQuery>;

#[pallet::storage]
pub(super) type SomeComplexValue<T: Config> = StorageValue<_, T::AccountId, ValueQuery>;
```

### 单层键Map

```
#[pallet::storage]
#[pallet::getter(fn some_map)]
pub(super) type SomeMap<T: Config> = StorageMap<
    _,
    Blake2_128Concat, T::AccountId,
    u32,
    ValueQuery
>;
```

### 双层键Map
```
#[pallet::storage]
pub(super) type SomeDoubleMap<T: Config> = StorageDoubleMap<
    _,
    Blake2_128Concat, u32,
    Blake2_128Concat, T::AccountId,
    u32,
    ValueQuery
>;
```


### 多层键Map

```
#[pallet::storage]
#[pallet::getter(fn some_nmap)]
pub(super) type SomeNMap<T: Config> = StorageNMap<
    _,
    (
        NMapKey<Blake2_128Concat, u32>,
        NMapKey<Blake2_128Concat, T::AccountId>,
        NMapKey<Twox64Concat, u32>,
    ),
    u32,
    ValueQuery,
>;
```
注意，Map的存储项指定了将要使用的哈希算法。

### 处理查询返回值

当你声明一个存储项时，你可以指定如果存储中没有指定键的值，查询应如何处理返回值。在存储声明中，你可以指定以下内容：

- `[OptionQuery](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.OptionQuery.html)`：从存储中查询一个可选值，如果存储中包含一个值，则返回`Some`，如果存储中没有值，则返回`None`。
- `[ResultQuery](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.ResultQuery.html)`：从存储中查询一个结果值，如果存储中没有值，则返回一个错误。
- `[ValueQuery](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.ValueQuery.html)`：从存储中查询一个值并返回该值。你也可以使用`ValueQuery`来返回默认值，如果你为存储项配置了特定的默认值，或者返回与`OnEmpty`通用配置的值。

### 可见性

在上述示例中，除了`SomePrivateValue`之外，所有的存储项都通过pub关键字公开。区块链存储始终可以从Runtime外部公开查看。Substrate存储项的可见性只影响Runtime内的其他pallet是否能够访问存储项。

### Getter 方法

`#[pallet::getter(..)]`宏提供了一个可选的get扩展，可以用来在包含该存储项的模块上实现一个getter方法。该扩展以getter函数的期望名称作为参数。如果你省略了这个可选扩展，你可以访问存储项的值，但你将无法通过在模块上实现的getter方法来做到这一点；相反，你需要使用存储项的[get方法](https://docs.substrate.io/build/runtime-storage/#methods)。

可选的getter扩展只影响从Substrate代码内部访问存储项的方式——你总是能够在外部[查询](https://docs.substrate.io/build/runtime-storage/#Querying-Storage)你的Runtime存储来获取存储项的值。

下面是一个实现了名为`some_value`的getter方法的例子，该方法对应于名为`SomeValue`的Storage Value。这个pallet现在可以访问`Self::some_value()`方法，除此之外还可以访问`SomeValue::get()`方法：

```rust
#[pallet::storage]
#[pallet::getter(fn some_value)]
pub(super) type SomeValue = StorageValue<_, u64, ValueQuery>;
```

### 默认值

Substrate允许你指定一个默认值，当存储项的值未设置时返回该默认值。尽管默认值实际上并不占用Runtime存储，但在执行过程中，Runtime逻辑会看到这个值。

下面是在存储中指定默认值的一个例子：

```rust
#[pallet::type_value]
pub(super) fn MyDefault<T: Config>() -> T::Balance { 3.into() }
#[pallet::storage]
pub(super) type MyStorageValue<T: Config> =
    StorageValue<Value = T::Balance, QueryKind = ValueQuery, OnEmpty = MyDefault<T>>;
```

请注意，为了增加每个存储字段的清晰度，上述语法是声明存储项的非缩写版本。

## 访问存储项

使用Substrate构建的区块链暴露了一个远程过程调用（RPC）服务器，可以用来查询Runtime存储。你可以使用像[Polkadot JS](https://polkadot.js.org/)这样的软件库轻松地从你的代码中与RPC服务器进行交互并访问存储项。Polkadot JS团队还维护了[Polkadot Apps UI](https://polkadot.js.org/apps)，这是一个功能齐全的Web应用程序，用于与基于Substrate的区块链进行交互，包括查询存储。

## 哈希算法

在Substrate中，Storage Maps的一个新颖特性是它们允许开发者指定用于生成Map键的哈希算法。用于封装哈希逻辑的Rust对象被称为"hasher"。从广义上讲，对Substrate开发者可用的hashers可以用两种方式描述：(1) 它们是否是加密的；(2) 它们是否产生透明的输出。

为了完整性，下面描述了非透明哈希算法的特性，但请注意，任何不产生透明输出的hasher都已经在基于FRAME的区块链中被弃用。

### 密码学哈希算法

密码学哈希算法使我们能够构建工具，使得极难通过操纵哈希算法的输入来影响其输出。例如，即使输入是1到10的数字，密码学哈希算法也会产生广泛的输出分布。当用户能够影响Storage Map的键时，使用密码学哈希算法是至关重要的。如果不这样做，就会创建一个攻击向量，使恶意行为者可以轻易地降低你的区块链网络的性能。一个应该使用密码学哈希算法来生成其键的Map的例子是用于跟踪账户余额的Map。在这种情况下，使用密码学哈希算法是重要的，这样攻击者就不能用许多小额转账到顺序账户号来轰炸你的系统。如果没有适当的密码学哈希算法，那么存储结构会不平衡且性能受损。在[Common Substrate hashers](https://docs.substrate.io/build/runtime-storage/#common-substrate-hashers)中阅读更多关于Substrate中常见的hashers的信息。

密码学哈希算法比它们的非加密对应物更复杂和消耗更多资源，这就是为什么对于Runtime工程师来说，理解它们的适当用法以便最好地利用Substrate提供的灵活性是重要的。

### 透明哈希算法

一个透明的哈希算法是一种使人们容易发现和验证用于生成给定输出的输入的算法。在Substrate中，通过将算法的输入与其输出连接起来，使哈希算法变得透明。这使得用户可以轻易地检索键的原始未哈希值，并且如果他们愿意的话，可以通过重新哈希它的方式来验证它。Substrate的创建者已经在基于FRAME的Runtime中弃用了使用非透明hashers，所以这个信息主要是为了完整性而提供的。事实上，如果你想要访问[可迭代的Map](https://docs.substrate.io/build/runtime-storage/#iterable-storage-maps)功能，就必须使用透明的哈希算法。

### 常用Substrate hashers

这个表格列出了在Substrate中使用的一些常见的hashers，并标注了哪些是加密的，哪些是透明的：

| Hasher | Cryptographic | Transparent |
| --- | --- | --- |
| [Blake2 128 Concat](https://paritytech.github.io/substrate/master/frame_support/struct.Blake2_128Concat.html) | X | X |
| [TwoX 64 Concat](https://paritytech.github.io/substrate/master/frame_support/struct.Twox64Concat.html) |  | X |
| [Identity](https://paritytech.github.io/substrate/master/frame_support/struct.Identity.html) |  | X |

Identity hasher封装了一个输出等于其输入（恒等函数）的哈希算法。只有当起始键已经是一个加密哈希时，才应该使用这种类型的hasher。

## 下一步

查看一些涵盖各种存储主题的指南。

- [How-to: Create a storage structure](https://docs.substrate.io/reference/how-to-guides/pallet-design/create-a-storage-structure/)
- [StorageValue](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.StorageValue.html)
- [StorageMap](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.StorageMap.html)
- [StorageDoubleMap](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.StorageDoubleMap.html)
- [StorageNMap](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.StorageNMap.html)

