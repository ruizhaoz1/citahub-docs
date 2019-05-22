---
id: version-0.20-faq
title: FAQ
original_id: faq
---

## CITA 是如何防止双花的？

首先简单解释下双花问题。双花简单来讲就是一笔钱进行了两次消费。对于传统的区块链，如果交易者使用同一笔余额，发送两个不同的交易到全网，虽然短暂时间内可能会造成分叉，但是因为区块链的共识的一致性，最终最多只有一笔成功，一笔资金不能同时被使用两次。用户交易在被打包入块后，被确认的交易所在的链存在一定概率被更长的链取代，会导致原来已经被确认的交易被回滚掉。在公网中，使用 POW 的算法时，随着区块被确认的次数增加，交易完全被确认的概率逐渐逼近 100%，但是永远到达不了 100%。以比特币网络为例大约 6 个块，以太坊大约 13 个块，即可在工程上认为交易被完全确认，如果想要更高的确定性，可以等待更多的块确认。

在 CITA 中，共识默认采用了 CITA-BFT 共识，一种经过了区块链适应性改造和调优的 BFT 算法。CITA-BFT 使用 Rust 实现，是一种确定性的共识算法，一旦当前块共识处理成功，就表示这个块被完全确认，不需要像比特币或者以太坊那样去担心因为更长的链出现而导致交易回滚。所以当能从链上查询到交易被成功处理，即表示交易被确认，无需再等待。所以如果用户广播两个有冲突的交易，交易经过共识后按照一定顺序来处理，最终在前面交易成功处理后，后面的交易因为条件不满足，肯定会处理失败。

在任何系统中，确定性都是基于概率的。对于 POW，存在 51%攻击。但是实际情况中，假设一个矿工有 25%的算力，在 6 个块被确认后，这个矿工通过构造一条超过主链长度的链来进行攻击，需要的概率为(0.25/0.75)^6，约等于 0.00137，所以可以认为足够安全。但是实际上的确定性并没有这么高，因为存在以下几种考虑：1）不能认为其他矿工是诚实的，2）由于网络原因，其余矿工并不是完全在挖主链，存在挖陈腐块，以及算力在分叉链的情况，3）矿工对其他其他矿池进行攻击，结盟，运行零费用矿池等等手段，可以以非常低的成本来构造 51%攻击。所以对于基于 POW 的公链来讲，确定性并不能满足所有的商业场景，并且确定时间也比较长。

而对于 CITA，由于采用 BFT 风格的算法，被确认的区块中至少要包含`2N/3 + 1`个共识节点的签名，伪造 ECDSA 签名的难度要远远高于挖矿算力计算难度 nonce 的。在 2011 年，比特币网络算力最高时，每秒大约可以进行 15 万亿次哈希计算，而对于 256 位的私钥，存在 2^256 种可能，所以使用当时情况所有的算力来暴力破解私钥，需要 pow(2,128) / (15 \* pow(2,40)) / 3600 / 24 / 365.25 / 1e9 / 1e9， 大约需要 0.6537992112229596 万兆年的时间。并且破解需要大于等于`2N/3 + 1`的节点的签名才可以。所以可以认为交易一旦共识入块之后，即可认为是确定的，链被回滚的概率可以忽略不计，实际上造成双花的可能性几乎为零。CITA 较公链的 POW 有更好的确定性，更短的确定时间。

## 交易中的 valid_until_block 是作用是什么？

在公网上，用户发送交易到节点处理时，首先链会返回一个交易的哈希作为交易的 ID。实际交易处理的时间，会因为节点的处理能力，以及节点选择交易的算法而受影响，可能出现长时间不能被打包入块（入块指打包成 Block 并共识成功）的情况。此时，用户不能确认交易依然是在某个节点交易池中排队，还是交易已经被完全丢弃，用户没有办法针对这种情况作出正确的判断。

CITA 采用的是先共识后处理的方式对交易进行处理。交易发送成功后，会返回交易哈希给用户。此时只是表明交易格式，签名等验证正确，并成功进入交易池，至于何时打包入块，同样取决于链的处理能力以及交易的选择算法。在交易中的 valid_until_block 表示交易最终的超时时间。举例来讲，用户在高度 100 时发送交易，且 valid_until_block 填写的 200，则在 201 块之前交易如果能成功打包入块都可以。如果到了出 201 块时，交易依然未打包入块，此时无论交易是否在交易池中，在出块阶段都会把此交易当作非法交易。由此，valid_until_block 起到一个超时的作用。在一定时间交易未打包，用户就可以完全确定交易不会再打包。在 CITA 中默认的 valid_until_block 最大只能比当前高度大 100，这个参数用户可以根据实际情况来调整，最大值的配置也可以根据实际情况来调整。

## 交易中的 nonce 的作用是什么？

比特币是基于 UTXO 的账号，交易是由 UTXO 来做组成，因为 UTXO 被消费后即失效，所以交易可以认为是唯一的。对于基于状态机的账号模型，用户发送的交易，存在被其他用户获取并重新发送到链上的可能，由此造成交易的多次执行，这种攻击行为称为重放攻击。为了避免重放攻击，需要采取一定的策略。例如以太坊中，用户发送的每一笔交易都必须包含一个自增的 nonce 值，交易一旦被确认，该用户的合法 nonce 值会自增，含有同样 nonce 的交易被认为是非法交易，这样来防止重放。但是对于以太坊的这种设计有一个很大的缺陷，后一笔交易必须等待前一笔交易进交易池才可以，交易只能顺序处理，限制了交易的并行性。例如，对于账户 A 向同一节点发送某一笔交易 T0 之后，只能等待 T0 打包入块并处理完成后，才可以发送后续的交易，即便后续交易对 T0 没有任何依赖关系，否则可能存在 T0 交易打包入块失败，而导致后续交易成功打包但是验证 nonce 失败。另外，在现实世界中存在同一账户被多人使用，或者向多个节点发送交易的情况，由于交易的 nonce 自增的特性，导致这种情况下，账户向多个节点同时发送交易会比较困难。

在 CITA 中 nonce 使用的是一个随机的字符串（有一定的长度限制），来使交易生成不同的 hash，使用 hash 来作为交易的唯一性验证。但是仅仅用 nonce 来保证哈希的唯一性，还是远远不够的，因为同一个用户发送交易足够多，nonce 还是有很大概率重复，且在工程上去保证全局 nonce 的唯一也会严重影响性能。在 CITA 的交易中 nonce 和 valid_until_block 配合使用，由此只要保证在 max_valid_until_block 范围交易的 hash 没有重复，就可以保证交易永远不会重复。并且 max_valid_until_block 的值可以保证验证 nonce 的缓存池不会过大。在兼顾性能和并行处理的同时，非常完美的解决了交易的唯一性问题，防止了重放攻击。

## 在压力测试时，出现交易未被处理的情况，该如何处理？

前面已经提到，交易未被处理是因为交易的超时，确保交易不会出现“意外“打包入块的情况。CITA 的交易池在 Auth 模块，在 RPC 将交易转发给 Auth，Auth 进行交易的签名等信息验证成功后，将交易放入交易池。默认情况下，交易池的最大交易容量是无穷大，所以对于一般的个人用户在进行压力测试时，交易发送过快，由于机器性能限制，交易可能处理不过来，可能会出现交易累积在交易池，导致交易超时。所以普通用户可以根据机器性能选择将 auth.toml 中的 tx_pool_limit 参数由 0（0 表示没有限制）改为一个合适的值。（对于单节点 4c8g 的节点，建议 50000）。此时，如果发送交易超出交易池的容纳能力，RPC 会返回 BUSY，提示用户发送交易速度过快。

## 交易中的 quota 的作用？

CITA 当前支持以太坊的 EVM，由于 EVM 支持图灵完备的语言，所以就存在停机问题。在公链上，以太坊的 gas 是为了解决停机问题，以及因为计算需要消耗资源，配合 gasPrice 来解决交易的市场化问题。CITA 是面向企业的区块链管理平台，同时支持智能合约的虚拟机，所以 quota 也是起到类似解决停机问题的作用，另外因为联盟链内部一般不需要发行代币。所以这里的 quota 是系统定时分配，这样可以在一定程度上可以杜绝恶意用户对计算资源的滥用。

## 参考文献

- How long would it take a large computer to crack a private key?: <https://bitcoin.stackexchange.com/questions/2847/how-long-would-it-take-a-large-computer-to-crack-a-private-key>
- Design Rationale - ethereum/wiki: <https://github.com/ethereum/wiki/wiki/Design-Rationale#accounts-and-not-utxos>