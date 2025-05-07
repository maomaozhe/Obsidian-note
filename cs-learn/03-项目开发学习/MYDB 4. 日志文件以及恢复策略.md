
//读代码的时候，就像递归一样，不应该在不同层次切换，费时费力  
// 应该在同一层面看待，先不去看封装的具体细节
这也启发我们，先要知道各个方法是做什么的，工作流程是什么样子的，之后才去思考怎么做的。
### 前言

MYDB 提供了崩溃后的数据恢复功能。DM 层在每次对底层数据操作时，都会记录一条日志到磁盘上。在数据库奔溃之后，再次启动时，可以根据日志的内容，恢复数据文件，保证其一致性。



### 日志读写

日志的二进制文件按照以下格式排布：

```
[XChecksum][Log1][Log2][Log3]...[LogN][BadTail]
```

XChecksum 是四字节的整数，是对后续所有日志的**校验和**，Log1 ~ LogN是常规的日志数据，BadTail是在数据库崩溃的时候，没来得及写完的日志数据，不一定存在。


**校验和（Checksum）** 是一种用于**检测数据在存储或传输过程中是否被破坏**的简单技术。  
本质上，它是把一段数据通过某种算法“压缩”成一个小型的数字（通常是固定大小，比如32位int）。（数据改变，重新计算的校验和几乎不可能相同）

代码中，初始化时候，比较头部的XChecksum和计算出来的


每条日志的格式如下：

```
[Size][Checksum][Data]
```

其中，Size是一个四字节整数，标识了Data段的字节数。Checksum则是该条日志的校验和。

单条日志的校验和是通过一个指定的种子实现的：


```java
private int calCheacksum(int xCheck, byte[] log){
	for(byte b : log){
		xCheck = xCheck * SEED + b;	
	}
	return xCheck;
}
```


我们来看看Logger接口sad

```java
public interface Logger {  
    void log(byte[] data);  
  
    //截短  
    void truncate(long x) throws Exception;  
    byte[] next();  
    void rewind();  
    void close();
    public static Logger create(String path){...};
    public static Logger open(String path){...};

}
```

它定义了日志系统的核心操作：`log` (写入)、`truncate` (截断)、`next` (顺序读取下一条)、`rewind` (重置读取位置)、`close` (关闭资源)。


`LoggerImpl` 是一个**日志组件**（Logger）的实现类，
其主要功能是**顺序写入日志数据**，**校验日志完整性**，并**读取有效日志**，同时支持**异常恢复（Bad Tail处理）**。














Logger 被实现成迭代器模式，通过 `next()` 方法，不断地从文件中读取下一条日志，并将其中的 Data 解析出来并返回。`next()` 方法的实现主要依靠 `internNext()`，大致如下，其中 position 是当前日志文件读到的位置偏移：


```java
private byte[] internNext() {  
    if(position + OF_DATA >= fileSize) {    
        return null;  
    }  
    //读取size  
    ByteBuffer tmp = ByteBuffer.allocate(4);  
    try {  
        fc.position(position);  
        fc.read(tmp);  
    } catch(IOException e) {  
        Panic.panic(e);  
    }  
    int size = Parser.parseInt(tmp.array());  
    if(position + size + OF_DATA > fileSize) {  
        //说明是最后一个log了  
        return null;  
    }  
    //读取checksum + data  
    ByteBuffer buf = ByteBuffer.allocate(OF_DATA + size);  
    try {  
        fc.position(position);  
        fc.read(buf);  
    } catch(IOException e) {  
        Panic.panic(e);  
    }  
      
    //校验  
  
    byte[] log = buf.array();  
    int checkSum1 = calChecksum(0, Arrays.copyOfRange(log, OF_DATA, log.length));  
    int checkSum2 = Parser.parseInt(Arrays.copyOfRange(log, OF_CHECKSUM, OF_DATA));  
    //校验一下看看是否有修改  
    if(checkSum1 != checkSum2) {  
        return null;  
    }  
    position += log.length;  
    return log;  
}
```




在打开一个日志文件时，需要首先校验日志文件的 XChecksum，并移除文件尾部可能存在的 BadTail，由于 BadTail 该条日志尚*未写入完成*，文件的校验和也就不会包含该日志的校验和，去掉 BadTail 即可保证日志文件的一致性。


为什么？

>因为日志写入通常是**先写数据**，再**更新校验和**，如果程序突然崩溃，最后一条日志可能只写了一半，系统重启后就会残留坏数据（BadTail）


```java
private void checkAndRemoveTail() {  
    rewind();  
  
    int xCheck = 0;  
    while(true) {  
    
        byte[] log = internNext();  
        if(log == null) break;  
        xCheck = calChecksum(xCheck, log);  
    }  
    if(xCheck != xChecksum) {  
        Panic.panic(Error.BadLogFileException);  
    }  
  
    try {  
        //log末尾截断  
        truncate(position);  
    } catch (Exception e) {  
        Panic.panic(e);  
    }  
    try {  
        //返回位置  
        file.seek(position);  
    } catch (IOException e) {  
        Panic.panic(e);  
    }  
    //处理完毕，重置读取指针  
    rewind();  
}
```



向日志文件写入日志时，也是**首先**将数据包裹成日志格式，写入文件后，再更新文件的校验和，更新校验和时，会刷新缓冲区，保证内容写入磁盘。

(刷新缓冲区体现在？)

```java
@Override  
public void log(byte[] data) {  
    byte[] log = wrapLog(data);  
    ByteBuffer buf = ByteBuffer.wrap(log);  
    lock.lock();  
    try {  
        fc.position(fc.size());  
        fc.write(buf);  
    } catch(IOException e) {  
        Panic.panic(e);  
    } finally {  
        lock.unlock();  
    }  
    updateXChecksum(log);  
}  
  
private void updateXChecksum(byte[] log) {  
    this.xChecksum = calChecksum(this.xChecksum, log);  
    try {  
        fc.position(0);  
        fc.write(ByteBuffer.wrap(Parser.int2Byte(xChecksum)));  
        fc.force(false);  
    } catch(IOException e) {  
        Panic.panic(e);  
    }  
}  
  
private byte[] wrapLog(byte[] data) {  
    byte[] checksum = Parser.int2Byte(calChecksum(0, data));  
    byte[] size = Parser.int2Byte(data.length);  
    return Bytes.concat(size, checksum, data);  
}
```




### 恢复策略

当数据库意外崩溃，比如断电、系统崩溃时：

- 可能有些事务已经完成提交（Commit），但数据尚未持久到磁盘；
    
- 有些事务可能正在执行中，但尚未完成；
    
- 日志恢复机制要保证：**提交的事务被正确重做（Redo）**，**未提交的事务被正确回滚（Undo）**。

提交到磁盘之后，日志会删除吗？


DM为上层模块，提供了两种操作，分别是插入新数据 I 和更新数据 U  。 嘶， 似乎没有删除

后面 VM 会讲到

DM 的日志策略很简单，一句话就是：

> 在进行 I 和 U 操作之前，必须先进行对应的日志操作，在保证日志写入磁盘后，才进行数据操作。

**核心原则：Write-Ahead Logging (WAL)**

所有设计都应遵循 WAL 原则：**在数据页（Data Page）写入磁盘之前，必须先将对应的日志记录（Log Record）写入持久化的日志文件（Log File）中。** 这是确保即使在写入数据页过程中发生崩溃，也能通过日志恢复数据的基础。



对于这两种操作，DM记录为： 

- (Ti, I, A, x)，表示事务 Ti 在 A 位置插入了一条数据 x
- (Ti, U, A, oldx, newx)，表示事务 Ti 将 A 位置的数据，从 oldx 更新成 newx
### 单线程情况

在单线程情况下，只会有一个事务在操作数据库，日志类似如下：
`(Ti, x, x), ..., (Ti, x, x), (Tj, x, x), ..., (Tj, x, x), (Tk, x, x), ..., (Tk, x, x)`

Ti Tj Tk 等等永远不会相交，这种情况利用日志恢复比较简单，假设最后一个日志是Ti：

1. 对Ti之前所有的事务的日志进行重做 （redo）
2. 接着检查Ti的状态（XID文件）*，若Ti的状态已经完成，（committed or aborted）,就将Ti重做，否则撤销undo*

接着，是如何对事务 T 进行 redo：

1. 正序扫描事务 T 的所有日志
2. 如果日志是插入操作 (Ti, I, A, x)，就将 x 重新插入 A 位置
3. 如果日志是更新操作 (Ti, U, A, oldx, newx)，就将 A 位置的值设置为 newx

undo 也很好理解：

1. 倒序扫描事务 T 的所有日志
2. 如果日志是插入操作 (Ti, I, A, x)，就将 A 位置的数据删除
3. 如果日志是更新操作 (Ti, U, A, oldx, newx)，就将 A 位置的值设置为 oldx

注意，MYDB 中其实没有真正的删除操作，对于插入操作的 undo，只是将其中的标志位设置为 invalid。对于删除的探讨将在 VM 一节中进行。


### 多线程


其实这里涉及了蛮多关于 线程安全的，隔离级别的思想

多线程情况下



两个线程运行，崩溃时，线程1提交，线程2正在执行。数据库重新启动之后，会撤销T2，他对数据库影响会删除，但是T1读取了T2更新的值，T1也应该撤销? 这称为**级联回滚**，但是，所有的commit应该提交，应该持久化，这里就有矛盾了，所以我们规定：


>规定：正在进行的事务，不会读取/修改其他任何未提交的事务产生的数据。
这是读提交吗哈哈哈


这种情况下，日志的恢复操作就简单些了

1. 重做所有崩溃时已完成（committed 或 aborted）的事务
2. 撤销所有崩溃时未完成（active）的事务









### 实现


首先规定两种日志的格式



```java
private static final byte LOG_TYPE_INSERT = 0; 
private static final byte LOG_TYPE_UPDATE = 1; 



// updateLog: 
// [LogType] [XID] [UID] [OldRaw] [NewRaw] 

// insertLog: 
// [LogType] [XID] [Pgno] [Offset] [Raw]

//有点没理解为什么他们的结构是这样的
```


recover**例程**也是两步： 重做已完成，撤销未完成事务。（静态方法）
























#### 疑问： 重复日志不是耗时吗？
如果**不清理日志**，**一直Redo历史操作**，确实会导致恢复变慢，存储浪费。

所以实际系统（如InnoDB，PostgreSQL）是这样做的：

- 定期Checkpoint；
    
- 在Checkpoint之后，清理不再需要的日志段；
    
- 保留一段必要的历史日志用于备份、复制、延迟恢复等需求；
    
- 新建日志文件（Log Rotation）以继续写入。