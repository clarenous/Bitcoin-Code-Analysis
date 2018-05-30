## 比特币源码分析-02-main.h

本文主要分析 main.h 与 main.cpp 中与`UTXO`和`Transaction`有关的疑难点。

### 1. 区块(Block)和交易(Transaction)的存储格式

为方便下文的代码分析，首先简要介绍比特币的Block和Transaction是如何存储的。

> 本段内容参考：[各区块链底层数据存储分析](http://stor.51cto.com/art/201802/566928.htm?from=singlemessage)

在比特币客户端数据路径下，`blocks/`内存储着一些以`blkxxxxx.dat`的格式命名的数据文件，例如`blk00000.dat`、`blk00001.dat`等，是真正的区块存储所在地。

路径`blocks/index/`下用kv数据库存储着对于这些dat文件的索引文件。

对于某个区块，它的索引格式为`<blockHash,xxxxx+npos>`，xxxxx为blk*.dat文件的文件序号，npos是该区块在blk*.dat文件中的起始位置。

对于某个交易，它的索引格式为`<txHash, xxxxx+npos+nTxOffset>`,xxxxx与npos用于定位区块位置，nTxOffset是具体交易数据写入blk*.dat文件的起始位置(基于区块位置，即npos位置)。

于是，我们可以利用索引快速地找到某个区块或交易的序列化数据，并进一步进行解析。

在此基础之上，我们继续分析[main.h](../src/main.h)中的代码。

### 2. 交易索引在内存中的表示

#### 2.1 class CDiskTxPos

`CDiskTxPos`用于记录某笔存储在`blocks/blk*.dat`文件内的交易的数据位置。

它具有基础数据成员如下：

``` cpp
public:
    unsigned int nFile;
    unsigned int nBlockPos;
    unsigned int nTxPos;
```

nFile,nBlockPos,nTxPos 分别代表了上文(1.)所说的xxxxx,npos,nTxOffset。这些基本数据成员构成了唯一定位某笔交易数据所需的参数。

再来看看其他函数的处理过程：

``` cpp
public:
    CDiskTxPos()
    {
        SetNull();
    }

    CDiskTxPos(unsigned int nFileIn, unsigned int nBlockPosIn, unsigned int nTxPosIn)
    {
        nFile = nFileIn;
        nBlockPos = nBlockPosIn;
        nTxPos = nTxPosIn;
    }

    IMPLEMENT_SERIALIZE( READWRITE(FLATDATA(*this)); )
    void SetNull() { nFile = -1; nBlockPos = 0; nTxPos = 0; }
    bool IsNull() const { return (nFile == -1); }

    friend bool operator==(const CDiskTxPos& a, const CDiskTxPos& b)
    {
        return (a.nFile     == b.nFile &&
                a.nBlockPos == b.nBlockPos &&
                a.nTxPos    == b.nTxPos);
    }

    friend bool operator!=(const CDiskTxPos& a, const CDiskTxPos& b)
    {
        return !(a == b);
    }

    string ToString() const
    {
        if (IsNull())
            return strprintf("null");
        else
            return strprintf("(nFile=%d, nBlockPos=%d, nTxPos=%d)", nFile, nBlockPos, nTxPos);
    }

    void print() const
    {
        printf("%s", ToString().c_str());
    }
```

可见，在初始化时，nFile的值是为`-1`的。而一个`CDiskTxPos`对象是否有效，也正是通过判断nFile的值来实现的。

其他的函数则是以操作符的重载为主，可自行阅读。

#### 2.2 class CTxIndex

`CTxIndex`的作用在于，在内存中存储某笔交易引用了哪些交易的输出，从而方便调试。其基本数据成员如下：

``` cpp
public:
    CDiskTxPos pos;
    vector<CDiskTxPos> vSpent;
```

其他的成员函数也很容易理解，暂时不做介绍。

### 3. 交易的输出与输入

#### 3.1 class CTxOut

`CTxOut`用于记录某笔交易的一个输出，该输出的基础数据成员有：

``` cpp
public:
    int64 nValue;
    CScript scriptPubKey;
```

分别代表该输出的数值`nValue`以及该输出的锁定脚本`scriptPubKey`。

这里有一个细节，即`int64`实际上是指`long long`类型，其最大值大于`1×10^18`。而比特币的最小单位是聪(satoshi)，且`1 BTC = 1×10^8 satoshi`，则比特币的总量为`2.1×10^7 BTC = 2.1×10^15 satoshi`。可见，该数据类型足以用最小单位记录比特币的数量。

其他的成员函数如下：

``` cpp
public:
    CTxOut()
    {
        SetNull();
    }

    CTxOut(int64 nValueIn, CScript scriptPubKeyIn)
    {
        nValue = nValueIn;
        scriptPubKey = scriptPubKeyIn;
    }

    IMPLEMENT_SERIALIZE
    (
        READWRITE(nValue);
        READWRITE(scriptPubKey);
    )

    void SetNull()
    {
        nValue = -1;
        scriptPubKey.clear();
    }

    bool IsNull()
    {
        return (nValue == -1);
    }
```

`nValue`的值是否为`-1`作为一个`CTxOut`对象是否有效的标记。

``` cpp
    uint256 GetHash() const
    {
        return SerializeHash(*this);
    }

    bool IsMine() const
    {
        return ::IsMine(scriptPubKey);
    }

    int64 GetCredit() const
    {
        if (IsMine())
            return nValue;
        return 0;
    }
```

如上，通过调用类外部的`bool IsMine(const CScript& scriptPubKey)`([位于script.cpp](../src/script.cpp))函数，实现了对于某笔交易输出归属权的验证。

``` cpp
    friend bool operator==(const CTxOut& a, const CTxOut& b)
    {
        return (a.nValue       == b.nValue &&
                a.scriptPubKey == b.scriptPubKey);
    }

    friend bool operator!=(const CTxOut& a, const CTxOut& b)
    {
        return !(a == b);
    }

    string ToString() const
    {
        if (scriptPubKey.size() < 6)
            return "CTxOut(error)";
        return strprintf("CTxOut(nValue=%I64d.%08I64d, scriptPubKey=%s)", nValue / COIN, nValue % COIN, scriptPubKey.ToString().substr(0,24).c_str());
    }

    void print() const
    {
        printf("%s\n", ToString().c_str());
    }
```

其他的则是简单的操作符重载与输出函数，不做介绍。

#### 3.2 class COutPoint

`COutPoint`是对某个输出(CTxOut)的一种"指针"。它的基本数据成员有：

``` cpp
public:
    uint256 hash;
    unsigned int n;
```

`hash`是指产生该输出的交易TxPrev的256位哈希值，这个值是唯一的。需要注意的是，这里的类`uint256`在[uint256.h](../src/uint256.h)中被定义，类中包含一个数组`unsigned int pn[8]`。这个256位的哈希值正是存储在这个数组中。

`n`是指该输出在交易TxPrev中，位于输出数组的序号值。对于Coinbase交易，`n`取`unsigned int`类型的最大值，即`n = 2^32 - 1 = 4294967295`。

`COutPoint`的其他成员函数也很好理解，这里不做介绍。

#### 3.3 class CTxIn

`CTxIn`是某笔交易中的一个输入，它的基本数据成员有：

``` cpp
public:
    COutPoint prevout;
    CScript scriptSig;
    unsigned int nSequence;
```

这里的数据成员要结合比特币的设计思想来理解，即任何一笔非Coinbase交易中的任何一个输入，都应当来自于一个已存在且未被花费的输出。

所以，这里的`prevout`指出了本`CTxIn`的来源。而`scriptSig`的作用在于，用我的解锁脚本证明`prevout`这笔输出的解锁权归我所有，即证明这一笔钱归持有唯一私钥的我所有。

`nSequence`的类型是`unsigned int`，最大值`0xffffffff`。该数值常常在构造交易时与`CTransaction`类的`nLockTime`配合使用，将在下一部分详解。但我们可以先对这个成员函数`isFinal()`留下印象，即在`nSequence`值达到最大时，认为该交易输入达到了最终状态。

``` cpp
public:
    bool IsFinal() const
        {
            return (nSequence == UINT_MAX);
        }
```

还有两个比较重要的成员函数是：

``` cpp
public:
    bool IsMine() const;
    int64 GetDebit() const;
```

将在介绍之后的交易与钱包部分后，另行分析。

#### 3.4 class CInPoint

`CInPoint`是对交易中某个输入(CTxIn)的一种"指针"。它的构成比较简单：

``` cpp
class CInPoint
{
public:
    CTransaction* ptx;
    unsigned int n;

    CInPoint() { SetNull(); }
    CInPoint(CTransaction* ptxIn, unsigned int nIn) { ptx = ptxIn; n = nIn; }
    void SetNull() { ptx = NULL; n = -1; }
    bool IsNull() const { return (ptx == NULL && n == -1); }
};
```

其中，`ptx`是指向引入了该输入的交易的指针。`n`是该输入在`ptx`的输入数组中的序号数值。

### 4. class CTransaction 交易类

`CTransaction`是包含在区块数据中，以及被广播的基础交易类，每一笔交易可以有多个输入(CTxIn)和输出(CTxOut)。

类的基础数据成员如下：

``` cpp
public:
    int nVersion;
    vector<CTxIn> vin;
    vector<CTxOut> vout;
    int nLockTime;
```

`nVersion`是为以后新协议的扩展准备的，目前的值为1。`nlockTime`则是指该交易计划被延后打包的区块高度或时间戳。`vin`和`vout`分别是包含了本交易中所有输入和输出的数组。

#### 4.1 nLockTime 详解

下面重点介绍很多人感到困惑的`nLockTime`。

`CTransaction`中的`nLockTime`使得某个已经被签名的交易不会被立即打包，而是延后等待一段时间，具体的延后的方法是：

> | Value | Description |
> | - | - |
> | 0 | Not locked |
> | < 500000000 | Block number at which this transaction is unlocked |
> | >= 500000000 | UNIX timestamp at which this transaction is unlocked |
> 来源：https://en.bitcoin.it/wiki/Protocol_documentation#tx

即`nLockTime == 0`时，交易被立即打包。`0 < nLockTime < 500000000`时，若当前块高度大于nLockTime则进行打包。`nLockTime >= 500000000`时，nLockTime代表一个UNIX时间戳，当前时间大于改时间戳时，允许打包。

但是，并非为交易设置nLockTime就一定能够被延后打包。如果一笔交易中，所有输入(CTxIn)都达到了Final状态，即`nSequence`都被设置为了`0xffffffff`，那么该交易也会被立即打包。落实到代码，就是这样：

``` cpp
public:
    // 这里nLockTime并未做UNIX时间戳使用，是因为初版比特币代码发布时，协议中还只是用了区块高度用来做交易延迟
    bool IsFinal() const
        {
            if (nLockTime == 0 || nLockTime < nBestHeight)
                return true;
            foreach(const CTxIn& txin, vin)
                if (!txin.IsFinal())
                    return false;
            return true;
        }
```

比特币中引入这种设计，可以满足一些特殊的场景。例如在商业合作中，一笔交易将包含多个人的签名，他们预先构造并广播交易后，还有机会在延迟打包前用新的交易替换原交易。而在更为复杂的场景下，如果对单个输入(CTxIn)已经达成最终共识，就可以重新签名，将该输入的nSequence设置为`0xffffffff`；当一笔交易中的所有输入都已达成最终共识，则继续等待延迟打包就不再有意义，而可以直接将交易打包进区块了。

#### 4.2 从硬盘中读取并构造交易对象

