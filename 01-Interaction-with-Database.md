## 比特币源码分析-01-db.h-db.cpp

本文主要分析db.h与db.cpp中的疑难点。

### 1. 引入的外部依赖 db_cxx.h

[db.h](src/db.h) 中，通过引入`db_cxx.h`，导入了 Berkeley DB 的相关操作类。

> Berkeley DB（简称BDB）是一个高效的嵌入式数据库编程库，C语言、C++、Java、Perl、Python、Tcl以及其他很多语言都有其对应的API。Berkeley DB可以保存任意类型的键/值对（Key/Value Pair），而且可以为一个键保存多个数据。  
> 来源：https://zh.wikipedia.org/wiki/Berkeley_DB

#### class Db

可以创建数据库对象指针，例如`Db* pdb`。该对象指针常用的操作方法有`get`、`put`、`del`、`exists`。

#### class DbTxn

每一个`DbTxn`对象都是 BDB 中对某项事务(Transaction)的句柄(handle)。

> The DbTxn object is the handle for a transaction. Methods off the DbTxn handle are used to configure, abort and commit the transaction. DbTxn handles are provided to Db methods in order to transactionally protect those database operations.  
> 来源：https://web.stanford.edu/class/cs276a/projects/docs/berkeleydb/api_java/txn_class.html

#### class DbEnv

`DbEnv`对象是对BDB中环境配置(environment)的句柄。这些配置包括caching, locking, logging and transaction subsystems 等子系统以及数据库文件(Data Files)和日志文件(Log Files)的位置。

> The DbEnv object is the handle for a Berkeley DB environment -- a collection including support for some or all of caching, locking, logging and transaction subsystems, as well as databases and log files. Methods off the DbEnv handle are used to configure the environment as well as to operate on subsystems and databases in the environment.  
> 来源：https://web.stanford.edu/class/cs276a/projects/docs/berkeleydb/api_java/env_class.html

#### BDB 小结

> 写一个BDB程序的一般步骤：  
> a. 创建、设置和打开Environment；b. 创建、设置和打开Database；c. 访问Database；d.关闭Database；e. 关闭Environment。  
> 来源：https://blog.csdn.net/heiyeshuwu/article/details/51519459

### 2. class CDB

`CDB`是比特币源码中封装的数据库操作类,也是大多数涉及数据库操作类的公共父类。

该类封装得到以下函数：
```
protected:
    bool Read(const K& key, T& value);
    bool Write(const K& key, const T& value, bool fOverwrite=true);
    bool Erase(const K& key);
    bool Exists(const K& key);
    Dbc* GetCursor();
    int ReadAtCursor(Dbc* pcursor, CDataStream& ssKey, CDataStream& ssValue, unsigned int fFlags=DB_NEXT);
    DbTxn* GetTxn();

public:
    bool TxnBegin();
    bool TxnCommit();
    bool TxnAbort();
    bool ReadVersion(int& nVersion);
    bool WriteVersion(int nVersion);
```

### 3. class CTxDB

`CTxDB`以 public 方式继承自`CDB`，用于处理比特币中的交易数据。

封装得到以下函数：

```
public:
    bool ReadTxIndex(uint256 hash, CTxIndex& txindex);
    bool UpdateTxIndex(uint256 hash, const CTxIndex& txindex);
    bool AddTxIndex(const CTransaction& tx, const CDiskTxPos& pos, int nHeight);
    bool EraseTxIndex(const CTransaction& tx);
    bool ContainsTx(uint256 hash);
    bool ReadOwnerTxes(uint160 hash160, int nHeight, vector<CTransaction>& vtx);
    bool ReadDiskTx(uint256 hash, CTransaction& tx, CTxIndex& txindex);
    bool ReadDiskTx(uint256 hash, CTransaction& tx);
    bool ReadDiskTx(COutPoint outpoint, CTransaction& tx, CTxIndex& txindex);
    bool ReadDiskTx(COutPoint outpoint, CTransaction& tx);
    bool WriteBlockIndex(const CDiskBlockIndex& blockindex);
    bool EraseBlockIndex(uint256 hash);
    bool ReadHashBestChain(uint256& hashBestChain);
    bool WriteHashBestChain(uint256 hashBestChain);
    bool LoadBlockIndex();
```

### 4. class CReviewDB

`CReviewDB`以 public 方式继承自`CDB`，用于记录比特币客户端中的用户数据。

### 5. class CMarketDB

`CMarketDB`以 public 方式继承自`CDB`，用途暂时不详。

### 6. class CAddrDB

`CAddrDB`以 public 方式继承自`CDB`，用于处理比特币客户端中的地址信息。

### 7. class CWalletDB

`CWalletDB`以 public 方式继承自`CDB`，用于处理比特币客户端中与钱包相关的数据。

具有相关函数如下：

```
public:
bool ReadName(const string& strAddress, string& strName);
bool WriteName(const string& strAddress, const string& strName);
bool EraseName(const string& strAddress);
bool ReadTx(uint256 hash, CWalletTx& wtx);
bool WriteTx(uint256 hash, const CWalletTx& wtx);
bool EraseTx(uint256 hash);
bool ReadKey(const vector<unsigned char>& vchPubKey, CPrivKey& vchPrivKey);
bool WriteKey(const vector<unsigned char>& vchPubKey, const CPrivKey& vchPrivKey);
bool ReadDefaultKey(vector<unsigned char>& vchPubKey);
bool WriteDefaultKey(const vector<unsigned char>& vchPubKey);
bool ReadSetting(const string& strKey, T& value);
bool WriteSetting(const string& strKey, const T& value);
bool LoadWallet(vector<unsigned char>& vchDefaultKeyRet);
```