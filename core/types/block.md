这部分代码主要是有关区块数据结构的。

**BlockNonce**是一个长度为8的byte数组。这是一个64比特位的散列，用来证明在一个块上进行了足够的运算。又定义了**EncodeNonce**和**Uint64**两个函数进行uint64和BlockNonce之间的转换.

**MarshalText**和**UnmarshalText**调用的是commmon/hexutil.json中的函数，作用是字符转成0x开头的十六进制字符串。举个例子。比如['a','b'],转化结果就是：48 120 54 49 54 50 48 48 48 48 48 48 48 48 48 48 48 48。对应ascii就是0x6162000000000000 这个结果不言而喻。

**Header**定义了区块头的数据结构。具体包括：

<table>
        <tr>
            <th>变量名</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
            <th>ParentHash</th>
            <th>common.Hash</th>
            <th>父区块的Hash</th>
        </tr>
        <tr>
            <th>UncleHash</th>
            <th>common.Hash</th>
            <th>叔区块的Hash</th>
        </tr>
        <tr>
            <th>Coinbase</th>
            <th>common.Address</th>
            <th>矿工地址</th>
        </tr>
        <tr>
            <th>Root</th>
            <th>common.Hash</th>
            <th>根的Hash</th>
        </tr>
        <tr>
            <th>ReceiptHash</th>
            <th>common.Hash</th>
            <th>Receipt的Hash</th>
        </tr>
        <tr>
            <th>Bloom</th>
            <th>Bloom</th>
            <th>日志用的布隆过滤器</th>
        </tr>
        <tr>
            <th>Difficulty</th>
            <th>*big.Int</th>
            <th>区块难度</th>
        </tr>
        <tr>
            <th>Number</th>
            <th>*big.Int</th>
            <th>区块编号</th>
        </tr>
        <tr>
            <th>GasLimit</th>
            <th>uint64</th>
            <th>gas消耗上限</th>
        </tr>
        <tr>
            <th>GasUsed</th>
            <th>uint64</th>
            <th>实际消耗Gas</th>
        </tr>
        <tr>
            <th>Time</th>
            <th>*big.Int</th>
            <th>时间戳</th>
        </tr>
        <tr>
            <th>Extra</th>
            <th>[]byte </th>
            <th>额外数据</th>
        </tr>
        <tr>
            <th>MixDigest</th>
            <th>common.Hash</th>
            <th>不知道什么东西的hash</th>
        </tr>
        <tr>
            <th>Nonce</th>
            <th>BlockNonce</th>
            <th>区块Nonce,随机数</th>
        </tr>
</table>

**headerMarshaling**定义了headerMarshaling的数据结构。应该是一个生成headerjson的东西。grep了一下只在gen_header_json中用了一次具体包括：
<table>
        <tr>
            <th>变量名</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
            <th>Difficulty</th>
            <th>*hexutil.Big</th>
            <th>区块难度</th>
        </tr>
        <tr>
            <th>Number</th>
            <th>*hexutil.Big</th>
            <th>区块编号</th>
        </tr>
        <tr>
            <th>GasLimit</th>
            <th>hexutil.Uint64</th>
            <th>gas消耗上限</th>
        </tr>
        <tr>
            <th>GasUsed</th>
            <th>hexutil.Uint64</th>
            <th>实际消耗Gas</th>
        </tr>
         <tr>
            <th>Time</th>
            <th>*hexutil.Big</th>
            <th>时间戳</th>
        </tr>
        <tr>
            <th>Extra</th>
            <th>hexutil.Bytes </th>
            <th>额外数据</th>
        </tr>
        <tr>
            <th>Hash</th>
            <th>common.Hash </th>
            <th>用来调用MarshalJSON中的Hash</th>
        </tr>
</table>
有一些数据类型和Head的不一样。（定义在hexutil/json.go)
hexutil.Big和big.Int是一样的。
hexutil.Uint64和uint64是一样。
hexutil.Big和big.Int是一样的。
hexutil.Bytes和byte是一样的。


和**head**有关的一些函数：

    **Hash**返回了头的hash的rlp编码。(rlp参见RLP编码.md)

    **HashNoNonce**返回头的hash，这个hash是pow的输入。

    **Size**用来估算内存消耗

    **rlpHash** 计算hash的rlp编码

和**body**与**block**有关的一些结构体和函数：
**body**定义了区块体的数据结构：

<table>
        <tr>
            <th>变量名</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
            <th>Transactions</th>
            <th>[]*Transaction</th>
            <th>交易数组</th>
        </tr>
        <tr>
            <th>Uncles</th>
            <th>[]*Header</th>
            <th>叔区块头数组</th>
        </tr>
</table>

**body**定义了区块的数据结构：
<table>
        <tr>
            <th>变量名</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
            <th>header</th>
            <th>*Header</th>
            <th>头</th>
        </tr>
        <tr>
            <th>Uncles</th>
            <th>[]*Header</th>
            <th>叔区块头数组</th>
        </tr>
        <tr>
            <th>transactions</th>
            <th>Transactions</th>
            <th>交易（是一个数组）</th>
        </tr>
        <tr>
            <th>hash</th>
            <th>atomic.Value</th>
            <th></th>
        </tr>
        <tr>
            <th>size</th>
            <th>atomic.Value</th>
            <th></th>
        </tr>
        <tr>
            <th>td</th>
            <th>*big.Int</th>
            <th>从创世块到现在的total difficulty</th>
        </tr>
        <tr>
            <th>ReceivedAt</th>
            <th>time.Time</th>
            <th></th>
        </tr>
        <tr>
            <th>ReceivedFrom</th>
            <th>interface{}</th>
            <th>似乎是用来追踪不同peer的区块？</th>
        </tr>
</table>

**extblock** 用来给eth协议做区块编码。
<table>
        <tr>
            <th>变量名</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
            <th>Header</th>
            <th>*Header</th>
            <th></th>
        </tr>
        <tr>
            <th>Txs</th>
            <th>[]*Transaction</th>
            <th></th>
        </tr>
        <tr>
            <th>Txs</th>
            <th>[]*Header</th>
            <th></th>
        </tr>
</table>

**storageblock** 用来给数据库做区块编码。比extBlock多一个TD
<table>
        <tr>
            <th>变量名</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
            <th>Header</th>
            <th>*Header</th>
            <th></th>
        </tr>
        <tr>
            <th>Txs</th>
            <th>[]*Transaction</th>
            <th></th>
        </tr>
        <tr>
            <th>Txs</th>
            <th>[]*Header</th>
            <th></th>
        </tr>
        <tr>
            <th>TD</th>
            <th>*big.Int</th>
            <th></th>
        </tr>
</table>

函数**NewBlock**用来生成新的区块，需要的参数有：*header，[]*Transaction，[]*Header,[]*Receipt.有一些参数是不需要的， 因为通过tx,uncles等可以推断出来。

**NewBlockWithHeader**通过给定的头创建一个区块。

**CopyHeader**对头进行深拷贝

**DecodeRLP** **EncodeRLP**解码/编码RLP

接下来是一堆获取头属性的函数。不赘述。

**writeCounter** 实际上是一个float64。

**WithSeal**传输一个Block指针b，返回一个新区块，头是b的头。**WithBody**同理。

**Hash**返回头的keccak256hash。第一次的时候计算然后缓存下来。


最后是关于block排序的一组函数。

