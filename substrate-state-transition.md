# State transitions and storage

Substrate使用一个简单的键值数据存储，实现为基于数据库的修改Merkle树。所有Substrate的更高级别的存储抽象都是建立在这个简单的键值存储之上的。

## Key-Value database

Substrate使用RocksDB作为其存储数据库，这是一种持久化键值存储，用于快速存储环境。它还支持实验性的Parity DB。

该数据库用于所有需要持久性存储的Substrate组件，例如：

Substrate客户端
Substrate轻客户端
离线工作者

### Trie abstraction

使用简单的键值存储的一个优点是，您可以轻松地在其上抽象存储结构。

Substrate使用来自paritytech/trie的Base-16 Modified Merkle Patricia树（“trie”）提供一个trie结构，其内容可以被修改，其根哈希可以高效地重新计算。

Tries允许高效地存储和共享历史块状态。trie根是trie内部数据的表示；也就是说，具有不同数据的两个tries将始终具有不同的根。因此，两个区块链节点可以通过比较它们的trie根轻松验证它们是否具有相同的状态。

访问trie数据是昂贵的。每个读取操作需要O(log N)时间，其中N是存储在trie中的元素数量。为了缓解这种情况，我们使用了键值缓存。

所有trie节点都存储在数据库中，并且trie状态的一部分可以被修剪，即当键值对对于非归档节点超出修剪范围时，可以从存储中删除。出于性能原因，我们不使用引用计数。

### State trie

Substrate-based链有一个单一的主trie，称为状态trie，其根哈希放置在每个块头中。这用于轻松验证区块链的状态并为轻客户端提供验证证明的基础。

这个trie只存储规范链的内容，而不是分叉。有一个单独的state_db层，用于维护具有内存中引用计数的trie状态，其中所有内容都是非规范的。

### Child trie

Substrate还提供了一个API来生成具有自己根哈希的新子trie，这些trie可以在运行时使用。

子trie与主状态trie相同，只是子trie的根存储并更新为主trie中的一个节点，而不是块头。由于它们的头是主状态trie的一部分，因此当它包括子trie时仍然很容易验证完整的节点状态。

当您想要自己独立的trie并具有单独的根哈希以验证该trie中的特定内容时，子trie非常有用。Trie的子部分没有类似于根哈希的表示形式，可以自动满足这些需求；因此使用子trie。

## Querying storage

使用Substrate构建的区块链会公开一个远程过程调用（RPC）服务器，可用于查询运行时存储。当您使用Substrate RPC访问存储项时，只需要提供与该项关联的键即可。Substrate的运行时存储API公开了许多存储项类型；继续阅读以了解如何计算不同类型存储项的存储键


### Storage value keys

要计算简单存储值的键，请获取包含存储值的模块名称的TwoX 128哈希，并将其附加到存储值本身名称的TwoX 128哈希。例如，Sudo模块公开了一个名为Key的存储值项。

```
twox_128("Sudo")                   = "0x5c0d1176a568c1f92944340dbfed9e9c"
twox_128("Key")                    = "0x530ebca703c85910e7164cb7d1c9e47b"
twox_128("Sudo") + twox_128("Key") = "0x5c0d1176a568c1f92944340dbfed9e9c530ebca703c85910e7164cb7d1c9e47b"
```

如果熟悉的Alice账户是sudo用户，则读取Sudo模块的Key存储值的RPC请求和响应可以表示为：


```
state_getStorage("0x5c0d1176a568c1f92944340dbfed9e9c530ebca703c85910e7164cb7d1c9e47b") = "0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d"
```

在这种情况下，返回的值（“0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d”）是Alice的SCALE编码帐户ID（5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY）。

您可能已经注意到，非加密的TwoX 128哈希算法用于生成存储值键。这是因为不必支付与加密哈希函数相关的性能成本，因为哈希函数的输入（模块和存储项的名称）由运行时开发人员确定，而不是由潜在的恶意用户确定。

### Storage map keys

与存储值一样，存储映射的键等于包含映射的模块名称的TwoX 128哈希，该哈希前缀为存储映射本身名称的TwoX 128哈希。要从映射中检索元素，请将所需映射键的哈希附加到存储映射的存储键上。对于具有两个键（存储双重映射）的映射，请将第一个映射键的哈希后跟第二个映射键的哈希附加到存储双重映射的存储键上。

与存储值一样，Substrate使用TwoX 128哈希算法来处理模块和存储映射名称，但您需要确保在确定地图中元素的散列键时使用正确的散列算法（在#[pallet::storage]宏中声明的算法）。

以下是一个示例，说明了如何查询名为Balances的模块中名为FreeBalance的存储映射以获取Alice账户余额。在此示例中，FreeBalance映射使用透明Blake2 128 Concat散列算法：

```
twox_128("Balances")                                             = "0xc2261276cc9d1f8598ea4b6a74b15c2f"
twox_128("FreeBalance")                                          = "0x6482b9ade7bc6657aaca787ba1add3b4"
scale_encode("5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY") = "0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d"

blake2_128_concat("0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d") = "0xde1e86a9a8c739864cf3cc5ec2bea59fd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d"

state_getStorage("0xc2261276cc9d1f8598ea4b6a74b15c2f6482b9ade7bc6657aaca787ba1add3b4de1e86a9a8c739864cf3cc5ec2bea59fd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d") = "0x0000a0dec5adc9353600000000000000"
```

从存储查询返回的值（在上面的示例中为“0x0000a0dec5adc9353600000000000000”）是Alice帐户余额的SCALE编码值（在此示例中为“1000000000000000000000”）。请注意，在哈希Alice的帐户ID之前，它必须进行SCALE编码。还要注意，blake2_128_concat函数的输出由32个十六进制字符和函数的输入组成。这是因为Blake2 128 Concat是一种透明的哈希算法。

尽管上面的示例可能使这种特性看起来多余，但当目标是迭代映射中的键（而不是检索与单个键关联的值）时，其实用性变得更加明显。能够迭代映射中的键是一种常见要求，以便允许人们以看似自然的方式使用映射（例如UI）：首先，用户会看到映射中元素的列表，然后，该用户可以选择他们感兴趣的元素，并查询有关该特定元素的更多详细信息。

以下是另一个示例，它使用相同的示例存储映射（名为FreeBalances的使用Blake2 128 Concat散列算法的映射，在名为Balances的模块中），演示了使用Substrate RPC通过state_getKeys RPC端点查询存储映射以获取其键列表：


```
twox_128("Balances")                                      = "0xc2261276cc9d1f8598ea4b6a74b15c2f"
twox_128("FreeBalance")                                   = "0x6482b9ade7bc6657aaca787ba1add3b4"

state_getKeys("0xc2261276cc9d1f8598ea4b6a74b15c2f6482b9ade7bc6657aaca787ba1add3b4") = [
 "0xc2261276cc9d1f8598ea4b6a74b15c2f6482b9ade7bc6657aaca787ba1add3b4de1e86a9a8c739864cf3cc5ec2bea59fd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d",
 "0xc2261276cc9d1f8598ea4b6a74b15c2f6482b9ade7bc6657aaca787ba1add3b432a5935f6edc617ae178fef9eb1e211fbe5ddb1579b72e84524fc29e78609e3caf42e85aa118ebfe0b0ad404b5bdd25f",
 ...
]
```

Substrate RPC的state_getKeys端点返回的列表中的每个元素都可以直接用作RPC的state_getStorage端点的输入。实际上，上面示例列表中的第一个元素等于先前示例中用于state_getStorage查询（用于查找Alice余额）的输入。因为这些键所属的映射使用透明哈希算法生成其键，所以可以确定列表中第二个元素关联的帐户。请注意，列表中的每个元素都是以相同的64个字符开头的十六进制值；这是因为每个列表元素表示同一映射中的一个键，并且该映射由连接两个TwoX 128哈希（每个哈希均为128位或32个十六进制字符）而标识。在丢弃列表中第二个元素的此部分后，您将得到0x32a5935f6edc617ae178fef9eb1e211fbe5ddb1579b72e84524fc29e78609e3caf42e85aa118ebfe0b0ad404b5bdd25f。

您在先前的示例中看到，这代表某些SCALE编码帐户ID的Blake2 128 Concat哈希。Blake 128 Concat哈希算法由将哈希算法的输入附加（连接）到其Blake 128哈希组成。这意味着Blake2 128 Concat哈希的前128位（或32个十六进制字符）表示Blake2 128哈希，其余部分表示传递给Blake 2 128哈希算法的值。在此示例中，在删除表示Blake2 128哈希（即0x32a5935f6edc617ae178fef9eb1e211f）的前32个十六进制字符后，剩下的是十六进制值0xbe5ddb1579b72e84524fc29e78609e3caf42e85aa118ebfe0b0ad404b5bdd25f，这是一个SCALE编码帐户ID。解码此值将产生结果5GNJqTPyNqANBkUVMN1LPPrxXnFouWXoe2wNSmmEoLctxiZY，这是熟悉的Alice_Stash账户的帐户ID。

## Runtime storage API

Substrate的FRAME Support crate提供了用于为运行时存储项生成唯一确定性键的实用程序。这些存储项放置在状态trie中，并可通过按键查询trie来访问。


## Where to go next

- Runtime storage
- Type encoding (SCALE)



