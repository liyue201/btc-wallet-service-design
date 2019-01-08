# 如何设计一个比特币钱包服务

## 概述
总所周知，比特币全节点的实现目前有两个版本，一个是中本聪c++写的原版bitcoin core，一个是go语言写的btcd。这两个版本从功能上并没有多大差异，都实现了比特币协议。从理论上讲，比特币网络中的每个全节点只要遵循相同的比特币协议，至于用什么语言实现，用什么方式存储都是无所谓的。bitcoin core和btcd都没有实现通用的钱包服务，这里说的钱包服务指的是一个可以给钱包app提供所数据的服务，私钥保存在钱包app中，用户交易在钱包app中签名，再将签名数据发给钱包服务，再由钱包服务转发到全节点上链，如图：
 ![](https://raw.githubusercontent.com/liyue201/btc-wallet-service-design/master/btc-wallet.jpg)
 
所幸的是全节点提供了一些JSON RPC接口，通过这些接口可以读取区块数据和发起转账交易。但是能读到的数据还是太原始了，就连获取某个地址的余额都做不到。因为比特币使用的是UTXO账号体系，并没有一个统一的地方存储地址余额，只有UTXO。而UTXO是分散在每个区块中的，要想知道某个地址的UTXO，需要从第一个区块开始遍历。有些人说这样设计可以更好的并发执行，这个鄙人真的不敢苟同。说可以更好的溯源倒是真的，因为每一笔交易都可以追溯到上一笔交易。

## 钱包服务功能
好了，废话不多说，我们看一下一个钱包服务应该具备什么样的功能。

* （1）查询任意一个地址的余额，钱包需要知道自己地址的余额。
* （2）查询任意一个地址的UTXO，钱包转账时需要UTXO来签名打包。
* （3）发起转账交易。钱包需要调服务发起转账交易。
* （4）查询任意一个地址的历史交易记录。
* （5）查询最新区块高度，钱包转账后通过区块高度来计算交易是否已经确认，比特币是6个区块确认。

第(1)(2)点实际上是一个问题，知道UTXO自然就知道余额了。第(3)(5)可以直接调全节点接口。所以钱包服务的难点只剩下(2)(4)这两个功能。钱包服务要做的就是从全节点从第一个区块开始遍历读取所有的交易记录，并推导出每个地址当前区块高度的UTXO，再将这个信息保存到数据库中方便钱包使用。

## 全节点选择
bitcoin core 虽然是原版的比特币全节点，但是他只提供了http接口，没有websocket接口，不能实时推送未上链的交易。一个体验好的钱包不仅要获取已经上链的交易记录，也要知道未上链的交易。btcd不仅提供了http接口，还有websocket，所以btcd应该是更好的选择。

## 数据库选型
到目前为止（2018-12-26）比特币的交易量是3.6亿，因为一笔交易涉及到多个输入和多个输出地址（如果你研究过区块链浏览器上的数据，会发现一笔交易几千个输入和几千个输出都是很正常的，这样可以省矿工费，可能是交易所转账），解析出来后大约是16.6亿条（这个我自己跑的数据）。若是用传统的关系型数据库肯定扛不住的，所以一般人都会往nosql或者列式数据库方向考虑，比如用mongo或cassandra等。虽然能够解决问题，但是面对这么大的数据，这类型的数据库都需要部署分片集群。对于一个小创业公司来说，如果用户不是很多，从运维成本上来看，是不是都点杀鸡用牛刀大感觉呢。有没有经济实惠的方案？当然有，比如leveldb和rocksdb这类kv嵌入式数据库，只需要单个机器就可以满足要求。在几亿数据量情况下，leveldb在普通硬盘上读写速度可以达到几万每秒，使用ssd硬盘更快，而且内存占用极少。当然使用leveldb或rocksdb，也有其缺点，就是单机数据安全的问题，如果硬盘坏了怎么办。恰好我们这个业务对数据的安全性要求并不高，因为所有数据都是从全节点来的，要是硬盘坏了，大不了重新跑一遍数据。rocksdb是facebook的开源产品，据说是对leveldb做了优化，性能更强。rocksdb是c++写的，而leveldb除了c++版本之外，还有go语言版本，比特币全节点btcd和以太坊全节点geth都是用的go版本的leveldb。最终我选择了leveldb，因为我的服务是用go写的，虽然go也可以通过cgo调用C++，但是代码写起来没有纯go那么舒服。

## 数据结构
leveldb是kv数据库，没有表的概念，所以数据结构的设计非常关键。比如地址的历史交易记录，在关系型数据库中可以建一个以地址为索引的历史交易记录表。而kv数据库则没有索引的概念。好在level的key是有序的，可以根据key的顺序读取内容。根据这个特性，我们可以这样设计索引，每个地址的交易是一组key，这组key在数据库的所有key中是紧挨着的。这样只要知道某个地址的第一个索引key就可以按顺序读到这个地址的其他交易。因为一个交易涉及到多个输入地址和输出地址所以需要索引跟数据分开，对于utxo只关联到一个地址，就不需要单独设计索引了，索引跟数据可以放在一起。

 ![](https://raw.githubusercontent.com/liyue201/btc-wallet-service-design/master/struct.jpg)
 
数据结构设计好了，该怎么存。这个就是序列化的问题了，可以是json，或者go自带的gob，也可以自己写。最后从编码难度，执行效率和压缩率等方面综合考虑采用了protobuf。这是我的proto文件
 
 ```
 syntax = "proto3";

package models;

//key:  utxo-{addr}-{Transaction.id}-{Transaction.vout}
message Utxo {
    string  txId            = 1;
    uint32  vout            = 2;
    int64   amount          = 3;
    string  scriptPubKey    = 4;
}

//context
//key:  context
message Context {
    int64               blockHeight      = 1;   // 当前处理的区块高度
    string              blockHash        = 2;   // 当前处理的区块hash
    int32               blockTotalTx     = 3;   // 当前区块的交易数
    int32               BlockProcessedTx = 4;   // 当前区块已经处理的交易数
    int64               totalTx          = 5;   // 总共的交易数
    int64               totalTxNode      = 6;   //
}

message TxInput {
    string  txId    = 1;    //上一笔交易id
    uint32  vout    = 2;    //上一笔交易输出的idx
    string  addr    = 3;    //通过查询获得
    int64   amount  = 4;    //通过查询获得
}

message TxOutput {
    string  addr         = 1;
    int64   amount       = 2;
    uint32  vout         = 3;
    string  scriptPubKey = 4;
}

//key:  tx-{Transaction.txId}
message Transaction {
    int64   id                  = 1; //自增di
    string  txId                = 2;
    int64   blockTime           = 3;
    string  blockHash           = 4;
    int64   blockHeight         = 5;
    repeated TxInput inputs     = 6;
    repeated TxOutput outputs   = 7;
}

//key: txnode-{addr}-{TxNode.id}
message TxNode{
    int64  id           = 1;    //自增di
    string txId         = 2;
    uint32 type         = 3;    //1-intput 2-output  3-intput|output
}

 ```

### 总结
这个项目的难点不在编码而是技术选型，当然还有调试。代码写完后经过漫长的同步，大概半个月才同步完成。 只有同步完成才能进行完整的功能测试，若果有数据的bug还得从头开始同步。需要注意的是这里只是设计了已经确认的区块，对于未确认的区块和内存池的交易要单独考虑。

