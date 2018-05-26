## 比特币源码分析-01-db.h-db.cpp

本文主要分析db.h与db.cpp中的疑难点。

### 1. 引入的外部依赖 db_cxx.h

[db.h](../src/db.h) 中，通过引入`db_cxx.h`，导入了 Berkeley DB 的相关操作类。

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

#### 基本成员

首先来看该类有哪些基本成员：

``` cpp
protected:
    Db* pdb;
    string strFile;
    vector<DbTxn*> vTxn;
    explicit CDB(const char* pszFile, const char* pszMode="r+", bool fTxn=false);
    ~CDB() { Close(); }

public:
    void Close();

private:
    CDB(const CDB&);
    void operator=(const CDB&);
```

可见，一个CDB类首先包含 (1) BDB数据库的操作对象`pdb`，(2) 数据文件路径`strFile`，以及 (3)  BDB数据库事务句柄组成的指针数组`vTxn`。

`Close()`与析构函数`~CDB()`密切相关，完成内存的释放、对象的析构。

此外，初版代码中，并未对操作符`=`的重载进行实现，只进行了声明。

**重点来看CDB类的构造函数：**

首先，对传入的数据读取模式`pszMode`进行处理。并且若数据文件路径`pszFile`为空，则函数直接返回。

``` cpp
CDB::CDB(const char* pszFile, const char* pszMode, bool fTxn) : pdb(NULL)
{
    int ret;
    if (pszFile == NULL)
        return;

    bool fCreate = strchr(pszMode, 'c');
    bool fReadOnly = (!strchr(pszMode, '+') && !strchr(pszMode, 'w'));
    unsigned int nFlags = DB_THREAD;
    if (fCreate)
        nFlags |= DB_CREATE;
    else if (fReadOnly)
        nFlags |= DB_RDONLY;
    if (!fReadOnly || fTxn)
        nFlags |= DB_AUTO_COMMIT;
```

调用[main.cpp](../src/main.cpp)中的`GetAppDir()`，获取客户端数据所在的目录`strAppDir`。将`strAppDir/database/`作为数据库日志目录，若该目录不存在，则立即创建。

接着，设置BDB数据库环境`dbenv`。

``` cpp
    CRITICAL_BLOCK(cs_db)
    {
        if (!fDbEnvInit)
        {
            string strAppDir = GetAppDir();
            string strLogDir = strAppDir + "\\database";
            _mkdir(strLogDir.c_str());
            printf("dbenv.open strAppDir=%s\n", strAppDir.c_str());

            dbenv.set_lg_dir(strLogDir.c_str());
            dbenv.set_lg_max(10000000);
            dbenv.set_lk_max_locks(10000);
            dbenv.set_lk_max_objects(10000);
            dbenv.set_errfile(fopen("db.log", "a")); /// debug
            ///dbenv.log_set_config(DB_LOG_AUTO_REMOVE, 1); /// causes corruption
            ret = dbenv.open(strAppDir.c_str(),
                             DB_CREATE     |
                             DB_INIT_LOCK  |
                             DB_INIT_LOG   |
                             DB_INIT_MPOOL |
                             DB_INIT_TXN   |
                             DB_THREAD     |
                             DB_PRIVATE    |
                             DB_RECOVER,
                             0);
            if (ret > 0)
                throw runtime_error(strprintf("CDB() : error %d opening database environment\n", ret));
            fDbEnvInit = true;
        }

        strFile = pszFile;
        ++mapFileUseCount[strFile];
    }
```

创建新的BDB数据库操作对象`pdb`，确认数据库文件能够正常打开。

``` cpp
    pdb = new Db(&dbenv, 0);

    ret = pdb->open(NULL,      // Txn pointer
                    pszFile,   // Filename
                    "main",    // Logical db name
                    DB_BTREE,  // Database type
                    nFlags,    // Flags
                    0);

    if (ret > 0)
    {
        delete pdb;
        pdb = NULL;
        CRITICAL_BLOCK(cs_db)
            --mapFileUseCount[strFile];
        strFile = "";
        throw runtime_error(strprintf("CDB() : can't open database file %s, error %d\n", pszFile, ret));
    }

    if (fCreate && !Exists(string("version")))
        WriteVersion(VERSION);

    RandAddSeed();
}
```

至此，CDB类的构造函数就结束了。可见，在构造函数中，实现了在指定路径下创建有效的数据库操作对象`pdb`，并保存该路径。

#### 其他功能函数

除上面介绍的基本成员，该类还封装得到了以下函数：

``` cpp
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

从函数名上就可以大致看出每个函数的用途，以后有机会再详解这些功能函数。

### 3. class CTxDB

`CTxDB`以 public 方式继承自`CDB`，用于处理比特币中的交易数据。

其构造函数如下：

``` cpp
CTxDB(const char* pszMode="r+", bool fTxn=false) : CDB(!fClient ? "blkindex.dat" : NULL, pszMode, fTxn) { }
```

即对`blkindex.dat`文件建立`pdb`对象，从而实现对交易数据的处理。

`CTxDB`类封装得到了以下函数：

``` cpp
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

``` cpp
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