
> May you do good and not evil.
> May you find forgiveness for yourself and forgive others.
> May you share freely, never taking more than you give.

这段话在出现在sqlite每一个源码开头。觉得很有意思，所以跟大家分享一下。  

今天要看的是sqlite中VFS层的一个示例代码(test_onefile.c)。这个文件展示了如何通过sqlite的VFS层使其可以直接在嵌入式设备上进行操作，而无需中间的文件系统。  


这个实现基于以下假设：
- 该设备的只能被mediaRead()、 mediaWrite()以及mediaSync()三个操作访问

文件格式：  
![file](https://note.youdao.com/favicon.ico)  
基本原则是database file从前往后存，journal file从后往前存。实例中规定size为10MB，当写入量超过其90%的时候，返回SQLITE_FULL的错误。   
文件的头一个512字节的块用于存储数据库文件的大小，随着sync()操作进行更新。只有当journal file不存在时，此值正确，否则文件大小以journal file中记录的值为准。从第二个块起用于存储数据库文件内容。  
“日志文件”的大小不会永久存储在文件中。当系统运行时，日志文件的大小存储在易失性内存中。从崩溃中恢复时，此vfs报告日志文件的大小非常大。常规的日记标题和校验和机制用于防止SQLite处理任何超出日记逻辑端的数据。  
当SQLite调用OsDelete()删除日志文件时，blob的最后512个字节（包含第一个日记头的区域）将清零。  
锁：  
没有文件锁，单线程访问  

一个新的vfs需要实现 sqlite3_vfs, sqlite3_io_methods, sqlite3_file 三个对象  
sqlite3_vfs对象定义vfs的名字

```
/*
** Maximum pathname length supported by the fs backend.
*/
#define BLOCKSIZE 512
#define BLOBSIZE 10485760
```
首先定义了这个文件系统后端能支持的最大pathname length


```
/*
** Name used to identify this VFS.
*/
#define FS_VFS_NAME "fs"

// 内部文件的具体实现
typedef struct fs_real_file fs_real_file;
struct fs_real_file {
  sqlite3_file *pFile;
  const char *zName;
  int nDatabase;              /* Current size of database region 当前数据库的大小 */
  int nJournal;               /* Current size of journal region 日志的大小*/
  int nBlob;                  /* Total size of allocated blob 分配的总的blob（我理解是一块存储空间）的大小*/
  int nRef;                   /* Number of pointers to this structure 当前被引用量，为零时可以修改*/
  fs_real_file *pNext;        /* 指向下一个文件/数据库 */
  fs_real_file **ppThis;      /* 指向 */
};

// 继承sqlite3_file，一个文件对象，表示打开的文件。sqlite3_vfs的xOpen方法在打开文件时构建一个sqlite3_file对象，sqlite3_file跟踪文件打开时的状态
typedef struct fs_file fs_file;
struct fs_file {
  sqlite3_file base;
  int eType;
  fs_real_file *pReal;
};

// 存在内存中的临时文件
typedef struct tmp_file tmp_file;
struct tmp_file {
  sqlite3_file base;
  int nSize;
  int nAlloc;
  char *zAlloc;
};

// 文件类型的定义
/* Values for fs_file.eType. */
#define DATABASE_FILE   1
#define JOURNAL_FILE    2
```
这部分对文件的存储格式和对象进行了定义


```
/*
** Method declarations for fs_file.
*/
static int fsClose(sqlite3_file*);
static int fsRead(sqlite3_file*, void*, int iAmt, sqlite3_int64 iOfst);
static int fsWrite(sqlite3_file*, const void*, int iAmt, sqlite3_int64 iOfst);
static int fsTruncate(sqlite3_file*, sqlite3_int64 size);
static int fsSync(sqlite3_file*, int flags);
static int fsFileSize(sqlite3_file*, sqlite3_int64 *pSize);
static int fsLock(sqlite3_file*, int);
static int fsUnlock(sqlite3_file*, int);
static int fsCheckReservedLock(sqlite3_file*, int *pResOut);
static int fsFileControl(sqlite3_file*, int op, void *pArg);
static int fsSectorSize(sqlite3_file*);
static int fsDeviceCharacteristics(sqlite3_file*);

/*
** Method declarations for tmp_file.
*/
static int tmpClose(sqlite3_file*);
static int tmpRead(sqlite3_file*, void*, int iAmt, sqlite3_int64 iOfst);
static int tmpWrite(sqlite3_file*, const void*, int iAmt, sqlite3_int64 iOfst);
static int tmpTruncate(sqlite3_file*, sqlite3_int64 size);
static int tmpSync(sqlite3_file*, int flags);
static int tmpFileSize(sqlite3_file*, sqlite3_int64 *pSize);
static int tmpLock(sqlite3_file*, int);
static int tmpUnlock(sqlite3_file*, int);
static int tmpCheckReservedLock(sqlite3_file*, int *pResOut);
static int tmpFileControl(sqlite3_file*, int op, void *pArg);
static int tmpSectorSize(sqlite3_file*);
static int tmpDeviceCharacteristics(sqlite3_file*);

/*
** Method declarations for fs_vfs.
*/
static int fsOpen(sqlite3_vfs*, const char *, sqlite3_file*, int , int *);
static int fsDelete(sqlite3_vfs*, const char *zName, int syncDir);
static int fsAccess(sqlite3_vfs*, const char *zName, int flags, int *);
static int fsFullPathname(sqlite3_vfs*, const char *zName, int nOut,char *zOut);
static void *fsDlOpen(sqlite3_vfs*, const char *zFilename);
static void fsDlError(sqlite3_vfs*, int nByte, char *zErrMsg);
static void (*fsDlSym(sqlite3_vfs*,void*, const char *zSymbol))(void);
static void fsDlClose(sqlite3_vfs*, void*);
static int fsRandomness(sqlite3_vfs*, int nByte, char *zOut);
static int fsSleep(sqlite3_vfs*, int microseconds);
static int fsCurrentTime(sqlite3_vfs*, double*);
```
这部分是文件操作方法的声明

```
// 继承vfs，定义vfs的名称以及实现于操作系统的接口的核心方法
typedef struct fs_vfs_t fs_vfs_t;
struct fs_vfs_t {
  sqlite3_vfs base;
  fs_real_file *pFileList;                      /* 
  sqlite3_vfs *pParent;
};

// 一个初始化实例化的静态vfs
static fs_vfs_t fs_vfs = {
  {
    1,                                          /* iVersion */
    0,                                          /* szOsFile */
    0,                                          /* mxPathname */
    0,                                          /* pNext */
    FS_VFS_NAME,                                /* zName */
    0,                                          /* pAppData */
    fsOpen,                                     /* xOpen */
    fsDelete,                                   /* xDelete */
    fsAccess,                                   /* xAccess */
    fsFullPathname,                             /* xFullPathname */
    fsDlOpen,                                   /* xDlOpen */
    fsDlError,                                  /* xDlError */
    fsDlSym,                                    /* xDlSym */
    fsDlClose,                                  /* xDlClose */
    fsRandomness,                               /* xRandomness */
    fsSleep,                                    /* xSleep */
    fsCurrentTime,                              /* xCurrentTime */
    0                                           /* xCurrentTimeInt64 */
  }, 
  0,                                            /* pFileList */
  0                                             /* pParent */
};

// 一个静态的实例化的fs_io_methods
static sqlite3_io_methods fs_io_methods = {
  1,                            /* iVersion */
  fsClose,                      /* xClose */
  fsRead,                       /* xRead */
  fsWrite,                      /* xWrite */
  fsTruncate,                   /* xTruncate */
  fsSync,                       /* xSync */
  fsFileSize,                   /* xFileSize */
  fsLock,                       /* xLock */
  fsUnlock,                     /* xUnlock */
  fsCheckReservedLock,          /* xCheckReservedLock */
  fsFileControl,                /* xFileControl */
  fsSectorSize,                 /* xSectorSize */
  fsDeviceCharacteristics,      /* xDeviceCharacteristics */
  0,                            /* xShmMap */
  0,                            /* xShmLock */
  0,                            /* xShmBarrier */
  0                             /* xShmUnmap */
};

// 一个静态的实例化的tmp_io_methods
static sqlite3_io_methods tmp_io_methods = {
  1,                            /* iVersion */
  tmpClose,                     /* xClose */
  tmpRead,                      /* xRead */
  tmpWrite,                     /* xWrite */
  tmpTruncate,                  /* xTruncate */
  tmpSync,                      /* xSync */
  tmpFileSize,                  /* xFileSize */
  tmpLock,                      /* xLock */
  tmpUnlock,                    /* xUnlock */
  tmpCheckReservedLock,         /* xCheckReservedLock */
  tmpFileControl,               /* xFileControl */
  tmpSectorSize,                /* xSectorSize */
  tmpDeviceCharacteristics,     /* xDeviceCharacteristics */
  0,                            /* xShmMap */
  0,                            /* xShmLock */
  0,                            /* xShmBarrier */
  0                             /* xShmUnmap */
};
```
基础类的实例化  



tmp-file的操作函数
```
/*
** Close a tmp-file.
** 传入的参数是sqlite3_file *类型
** 在函数体中强制转换为子类
** 调用sqlite3_free()进行内存释放
*/
static int tmpClose(sqlite3_file *pFile){
  tmp_file *pTmp = (tmp_file *)pFile;
  sqlite3_free(pTmp->zAlloc);
  return SQLITE_OK;
}

/*
** Read data from a tmp-file.
*/
static int tmpRead(
  sqlite3_file *pFile, 
  void *zBuf, 
  int iAmt, 
  sqlite_int64 iOfst
){
  tmp_file *pTmp = (tmp_file *)pFile;
  // 如果读取的长度超出了文件大小返回错误
  if( (iAmt+iOfst)>pTmp->nSize ){
    return SQLITE_IOERR_SHORT_READ;
  }
  // memcpy，将从iOfst位置开始大小为iAmt的内容复制到zBuf中
  memcpy(zBuf, &pTmp->zAlloc[iOfst], iAmt);
  return SQLITE_OK;
}

/*
** Write data to a tmp-file.
*/
static int tmpWrite(
  sqlite3_file *pFile, 
  const void *zBuf, 
  int iAmt, 
  sqlite_int64 iOfst
){
  tmp_file *pTmp = (tmp_file *)pFile;
  // 如果写入的位置超出分配的内存大小，则重新分配大小扩大为原始大小加上写入值之后的两倍
  if( (iAmt+iOfst)>pTmp->nAlloc ){
    int nNew = (int)(2*(iAmt+iOfst+pTmp->nAlloc));
    char *zNew = sqlite3_realloc(pTmp->zAlloc, nNew);
    if( !zNew ){
      return SQLITE_NOMEM;
    }
    pTmp->zAlloc = zNew;
    pTmp->nAlloc = nNew;
  }
  // 复制写入内容到tmp_file
  memcpy(&pTmp->zAlloc[iOfst], zBuf, iAmt);
  pTmp->nSize = (int)MAX(pTmp->nSize, iOfst+iAmt);
  return SQLITE_OK;
}

/*
** Truncate a tmp-file.
*/
static int tmpTruncate(sqlite3_file *pFile, sqlite_int64 size){
  tmp_file *pTmp = (tmp_file *)pFile;
  // 截断到size和nSize中更小的那个位置
  pTmp->nSize = (int)MIN(pTmp->nSize, size);
  return SQLITE_OK;
}

/*
** Sync a tmp-file.
** 未实现
*/
static int tmpSync(sqlite3_file *pFile, int flags){
  return SQLITE_OK;
}

/*
** Return the current file-size of a tmp-file.
*/
static int tmpFileSize(sqlite3_file *pFile, sqlite_int64 *pSize){
  tmp_file *pTmp = (tmp_file *)pFile;
  *pSize = pTmp->nSize;
  return SQLITE_OK;
}

/*
** Lock a tmp-file.
*/
static int tmpLock(sqlite3_file *pFile, int eLock){
  return SQLITE_OK;
}

/*
** Unlock a tmp-file.
*/
static int tmpUnlock(sqlite3_file *pFile, int eLock){
  return SQLITE_OK;
}

/*
** Check if another file-handle holds a RESERVED lock on a tmp-file.
*/
static int tmpCheckReservedLock(sqlite3_file *pFile, int *pResOut){
  *pResOut = 0;
  return SQLITE_OK;
}

/*
** File control method. For custom operations on a tmp-file.
** 自定义操作，没理解具体作用
*/
static int tmpFileControl(sqlite3_file *pFile, int op, void *pArg){
  return SQLITE_OK;
}

/*
** Return the sector-size in bytes for a tmp-file.
*/
static int tmpSectorSize(sqlite3_file *pFile){
  return 0;
}

/*
** Return the device characteristic flags supported by a tmp-file.
*/
static int tmpDeviceCharacteristics(sqlite3_file *pFile){
  return 0;
}
```

```
/*
** Close an fs-file.
*/
static int fsClose(sqlite3_file *pFile){
  // 默认返回值
  int rc = SQLITE_OK;
  // 类型转换并指向realfile
  fs_file *p = (fs_file *)pFile;
  fs_real_file *pReal = p->pReal;

  /* Decrement the real_file ref-count. */
  // 减少被引数量
  pReal->nRef--;
  assert(pReal->nRef>=0);

  /* When the ref-count reaches 0, destroy the structure */
  if( pReal->nRef==0 ){
    *pReal->ppThis = pReal->pNext;
    if( pReal->pNext ){
      pReal->pNext->ppThis = pReal->ppThis;
    }
    rc = pReal->pFile->pMethods->xClose(pReal->pFile);
    sqlite3_free(pReal);
  }

  return rc;
}

/*
** Read data from an fs-file.
*/
static int fsRead(
  sqlite3_file *pFile, 
  void *zBuf, 
  int iAmt, 
  sqlite_int64 iOfst
){
  // 初始化各个指针
  int rc = SQLITE_OK;
  fs_file *p = (fs_file *)pFile;
  fs_real_file *pReal = p->pReal;
  sqlite3_file *pF = pReal->pFile;

  // 根据文件类型，计算偏移，读取对应值
  if( (p->eType==DATABASE_FILE && (iAmt+iOfst)>pReal->nDatabase)
   || (p->eType==JOURNAL_FILE && (iAmt+iOfst)>pReal->nJournal)
  ){
    rc = SQLITE_IOERR_SHORT_READ;
  }else if( p->eType==DATABASE_FILE ){
    rc = pF->pMethods->xRead(pF, zBuf, iAmt, iOfst+BLOCKSIZE);
  }else{
    /* Journal file. */
    int iRem = iAmt;
    int iBuf = 0;
    int ii = (int)iOfst;
    while( iRem>0 && rc==SQLITE_OK ){
      int iRealOff = pReal->nBlob - BLOCKSIZE*((ii/BLOCKSIZE)+1) + ii%BLOCKSIZE;
      int iRealAmt = MIN(iRem, BLOCKSIZE - (iRealOff%BLOCKSIZE));

      rc = pF->pMethods->xRead(pF, &((char *)zBuf)[iBuf], iRealAmt, iRealOff);
      ii += iRealAmt;
      iBuf += iRealAmt;
      iRem -= iRealAmt;
    }
  }

  return rc;
}

/*
** Write data to an fs-file.
*/
static int fsWrite(
  sqlite3_file *pFile, 
  const void *zBuf, 
  int iAmt, 
  sqlite_int64 iOfst
){

  // 初始化各个指针
  int rc = SQLITE_OK;
  fs_file *p = (fs_file *)pFile;
  fs_real_file *pReal = p->pReal;
  sqlite3_file *pF = pReal->pFile;

  if( p->eType==DATABASE_FILE ){
    if( (iAmt+iOfst+BLOCKSIZE)>(pReal->nBlob-pReal->nJournal) ){
      rc = SQLITE_FULL;
    }else{
      rc = pF->pMethods->xWrite(pF, zBuf, iAmt, iOfst+BLOCKSIZE);
      if( rc==SQLITE_OK ){
        pReal->nDatabase = (int)MAX(pReal->nDatabase, iAmt+iOfst);
      }
    }
  }else{
    /* Journal file. */
    int iRem = iAmt;
    int iBuf = 0;
    int ii = (int)iOfst;
    while( iRem>0 && rc==SQLITE_OK ){
      int iRealOff = pReal->nBlob - BLOCKSIZE*((ii/BLOCKSIZE)+1) + ii%BLOCKSIZE;
      int iRealAmt = MIN(iRem, BLOCKSIZE - (iRealOff%BLOCKSIZE));

      if( iRealOff<(pReal->nDatabase+BLOCKSIZE) ){
        rc = SQLITE_FULL;
      }else{
        rc = pF->pMethods->xWrite(pF, &((char *)zBuf)[iBuf], iRealAmt,iRealOff);
        ii += iRealAmt;
        iBuf += iRealAmt;
        iRem -= iRealAmt;
      }
    }
    if( rc==SQLITE_OK ){
      pReal->nJournal = (int)MAX(pReal->nJournal, iAmt+iOfst);
    }
  }

  return rc;
}

/*
** Truncate an fs-file.
*/
static int fsTruncate(sqlite3_file *pFile, sqlite_int64 size){
  fs_file *p = (fs_file *)pFile;
  fs_real_file *pReal = p->pReal;
  if( p->eType==DATABASE_FILE ){
    pReal->nDatabase = (int)MIN(pReal->nDatabase, size);
  }else{
    pReal->nJournal = (int)MIN(pReal->nJournal, size);
  }
  return SQLITE_OK;
}

/*
** Sync an fs-file.
*/
static int fsSync(sqlite3_file *pFile, int flags){
  fs_file *p = (fs_file *)pFile;
  fs_real_file *pReal = p->pReal;
  sqlite3_file *pRealFile = pReal->pFile;
  int rc = SQLITE_OK;

  if( p->eType==DATABASE_FILE ){
    unsigned char zSize[4];
    zSize[0] = (pReal->nDatabase&0xFF000000)>>24;
    zSize[1] = (unsigned char)((pReal->nDatabase&0x00FF0000)>>16);
    zSize[2] = (pReal->nDatabase&0x0000FF00)>>8;
    zSize[3] = (pReal->nDatabase&0x000000FF);
    rc = pRealFile->pMethods->xWrite(pRealFile, zSize, 4, 0);
  }
  if( rc==SQLITE_OK ){
    rc = pRealFile->pMethods->xSync(pRealFile, flags&(~SQLITE_SYNC_DATAONLY));
  }

  return rc;
}

/*
** Return the current file-size of an fs-file.
*/
static int fsFileSize(sqlite3_file *pFile, sqlite_int64 *pSize){
  fs_file *p = (fs_file *)pFile;
  fs_real_file *pReal = p->pReal;
  if( p->eType==DATABASE_FILE ){
    *pSize = pReal->nDatabase;
  }else{
    *pSize = pReal->nJournal;
  }
  return SQLITE_OK;
}

/*
** Lock an fs-file.
*/
static int fsLock(sqlite3_file *pFile, int eLock){
  return SQLITE_OK;
}

/*
** Unlock an fs-file.
*/
static int fsUnlock(sqlite3_file *pFile, int eLock){
  return SQLITE_OK;
}

/*
** Check if another file-handle holds a RESERVED lock on an fs-file.
*/
static int fsCheckReservedLock(sqlite3_file *pFile, int *pResOut){
  *pResOut = 0;
  return SQLITE_OK;
}

/*
** File control method. For custom operations on an fs-file.
*/
static int fsFileControl(sqlite3_file *pFile, int op, void *pArg){
  if( op==SQLITE_FCNTL_PRAGMA ) return SQLITE_NOTFOUND;
  return SQLITE_OK;
}

/*
** Return the sector-size in bytes for an fs-file.
*/
static int fsSectorSize(sqlite3_file *pFile){
  return BLOCKSIZE;
}

/*
** Return the device characteristic flags supported by an fs-file.
*/
static int fsDeviceCharacteristics(sqlite3_file *pFile){
  return 0;
}

```

最后看一下VFS的注册
```
/*
** This procedure registers the fs vfs with SQLite. If the argument is
** true, the fs vfs becomes the new default vfs. It is the only publicly
** available function in this file.
*/
int fs_register(void){
  if( fs_vfs.pParent ) return SQLITE_OK;
  fs_vfs.pParent = sqlite3_vfs_find(0);
  fs_vfs.base.mxPathname = fs_vfs.pParent->mxPathname;
  fs_vfs.base.szOsFile = MAX(sizeof(tmp_file), sizeof(fs_file));
  return sqlite3_vfs_register(&fs_vfs.base, 0);
}

#ifdef SQLITE_TEST
  int SqlitetestOnefile_Init() {return fs_register();}
#endif

```
