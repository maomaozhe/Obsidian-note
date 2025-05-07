重要
#mydb #mvcc

涉及了数据的隔离级别




>VM 基于两段锁协议实现了调度序列的可串行化，并实现了 MVCC 以消除读写阻塞。同时实现了两种隔离级别。


Data Manager 是 MYDB 的数据管理核心， Version Manager 是MYDB的事务和数据版本的管理核心。

### 2PL  与 MVCC

#### 冲突与 2PL

首先来定义数据库的冲突，暂时不考虑插入， 只看更新和读操作，两个操作满足下面三个条件，就可以说明操作互相冲突：

1. 这两个操作是由不同的事务执行的
2. 这两个操作操作的是同一个数据项
3. 这两个操作至少有一个是更新操作

那么冲突或者不冲突，意义何在？作用在于，**交换两个互不冲突的操作的顺序，不会对最终的结果造成影响**，而交换两个冲突操作的顺序，则是会有影响的。


VM 一个重要职责就是 实现**调度序列的可串行化**。MYDB采用两段锁协议（2PL）实现。
当采用 2PL 时，如果某个事务 i 已经对 x 加锁，且另一个事务 j 也想操作 x，但是这个操作与事务 i 之前的操作相互冲突的话，事务 j 就会被阻塞。譬如，T1 已经因为 U1(x) 锁定了 x，那么 T<u>2 对 x 的读或者写操作都会被阻塞</u>，T2 必须等待 T1 释放掉对 x 的锁。



### MVCC
MVCC 的核心思想是：

> **对同一份数据的不同版本进行管理，不同事务读取数据时可以“看到”不同的版本。**

具体来说：

- **读操作不会阻塞写操作，写操作也不会阻塞读操作。**
    
- 每当事务对某条记录进行修改时，数据库会**创建一份新的版本副本**，而不是直接覆盖原记录。
    
- 不同事务根据其隔离级别和启动时间，会读取**某个历史版本**，从而实现**一致性视图**。



在介绍 MVCC 之前，首先明确记录和版本的概念。

DM 层向上层提供了数据项（Data Item）的概念，VM 通过管理所有的数据项，向上层提供了记录（Entry）的概念。上层模块通过 VM 操作数据的最小单位，就是记录。VM 则在其内部，为每个记录，维护了多个版本（Version）。每当上层模块对某个记录进行修改时，VM 就会为这个记录创建一个新的版本。

MYDB 通过 MVCC，降低了事务的阻塞概率。譬如，T1 想要更新记录 X 的值，于是 T1 需要首先获取 X 的锁，接着更新，也就是创建了一个新的 X 的版本，假设为 x3。假设 T1 还没有释放 X 的锁时，T2 想要读取 X 的值，这时候就不会阻塞，MYDB 会返回一个较老版本的 X，例如 x2。这样最后执行的结果，就**等价于**，T2 先执行，T1 后执行，调度序列依然是可串行化的。如果 X 没有一个**更老的版本**，那只能等待 T1 释放锁了。所以只是降低了概率。

还记得我们在第四章中，为了保证数据的可恢复，VM 层传递到 DM 的操作序列需要满足以下两个规则：

>规定1：正在进行的事务，不会读取其他任何未提交的事务产生的数据。  
规定2：正在进行的事务，不会修改其他任何未提交的事务修改或产生的数据

由于2PL和MVCC两个原则轻易的满足了

[[MYDB 4. 日志文件以及恢复策略]]






### 记录的实现

entry 封装了事务元信息的数据结构, MVCC的数据单位

对于一条记录,MYDB使用Entry类维护了其结构,理论上,MVCC实现了多版本,但实现中, VM未提供Update操作,对字段的操作更新由后面的表和字段管理(TBM)实现. 所以VM实现中, 一条记录只有一个版本.

一条记录存储在一条 Data Item 中，所以 Entry 中保存一个 DataItem 的引用即可


`Entry` 储存结构格式: `` [XMIN] [XMAX] [DATA]


XMIN 是创建该条记录（版本）的事务编号，而 XMAX 则是删除该条记录（版本）的事务编号。


DataItem 的 before() after()方法












### 事务隔离级别

#### 读提交 以及实现

之前的例子提到过,对于一个数据,T1 获取锁,修改的过程中,其他线程直接返回旧版本(有的话),这就体现了 读提交.

版本的可见性与事物的隔离性是相关的. MYDB支持的最低的事务隔离程度是读提交,(Read committed),
其好处  (防止级联回归 commit语义冲突.....)[[MYDB 4. 日志文件以及恢复策略]]



MYDB 实现读提交，为每个版本维护了两个变量，就是上面提到的 XMIN 和 XMAX：

- XMIN：创建该版本的事务编号
- XMAX：删除该版本的事务编号


XMIN在版本创建的时候写


![[ReadView.drawio.webp]]


之所以DM不提供删除操作,无非就是实现多版本,删除时设置XMAX, 此版本对于XMAX之后的事务都不可见,相当于删除.

所以,在读提交下，版本对事务的可见性逻辑如下：

- 为自己创建并且未删除
  `(XMIN == Ti and XMAX == NULL )`
- 由已提交的事务创建 并且未删除 或者 被未提交事务删除
  `(XMIN is committed and (XMAX == NULL or (XMAX != Ti and XMAX is not committed)))

```java
private static boolean readCommitted(TransactionManager tm, Transaction t, Entry e) {  
    long xid = t.xid;  
    long xmin = e.getXmin();  
    long xmax = e.getXmax();  
    if(xmin == xid && xmax == 0) return true;  
  
    if(tm.isCommitted(xmin)) {  
        if(xmax == 0) return true;  
        if(xmax != xid) {  
            if(!tm.isCommitted(xmax)) {  
                return true;  
            }  
        }  
    }  
    return false;  
}
```

若条件为 true，则版本对 Ti 可见。那么获取 Ti 适合的版本，只需要从最新版本开始，依次向前检查可见性，如果为 true，就可以直接返回


#### 可重复读 以及实现


读提交肯会遇到 **不可重复读 以及 幻读**

- 一个事务内多次读同一个数据,前后两次不一致. (读的数据别别的并发线程修改了)

需要引入更为严格的隔离级别,可重复读,

**!!!!!! 重要** 

**事务只能读取它开始时, 就已经结束的那些事务产生的数据版本**


这条规定,事务需要忽略

1.本事务开始后开始的事务的数据;
2.本事务开始时还是active的事务的数据


对于第一条，只需要*比较事务 ID*，即可确定。而对于第二条，则需要在事务 Ti 开始时，<u>记录下当前活跃的所有事务 SP(Ti)</u>，如果记录的某个版本，XMIN 在 SP(Ti) 中，也应当对 Ti 不可见。



于是，需要提供一个结构，来抽象一个事务，以保存快照数据.

**快照是一份事务开始时可见的数据版本视图**。快照并不是真正的复制数据，而是逻辑上的一个**版本过滤规则**，告诉事务哪些版本的数据是“可见”的，哪些版本应该被忽略


```java
// vm对一个事务的抽象  
public class Transaction {  
    public long xid;  
    public int level;  
    public Map<Long, Boolean> snapshot;  
    public Exception err;  
    public boolean autoAborted;  
  
    public static Transaction newTransaction(long xid, int level, Map<Long, Transaction> active) {  
        Transaction t = new Transaction();  
        t.xid = xid;  
        t.level = level;  
        if(level != 0) {  
            t.snapshot = new HashMap<>();  
            for(Long x : active.keySet()) {  
                t.snapshot.put(x, true);  
            }  
        }  
        return t;  
    }  
  
    public boolean isInSnapshot(long xid) {  
        if(xid == TransactionManagerImpl.SUPER_XID) {  
            return false;  
        }  
        return snapshot.containsKey(xid);  
    }  
}
```

构造方法中的 active，保存着当前所有 active 的事务。于是，可重复读的隔离级别下，一个版本是否对事务可见的判断如下：



```java
private static boolean repeatableRead(TransactionManager tm, Transaction t, Entry e) {  
    long xid = t.xid;  
    long xmin = e.getXmin();  
    long xmax = e.getXmax();  
    if(xmin == xid && xmax == 0) return true;  
  
    if(tm.isCommitted(xmin) && xmin < xid && !t.isInSnapshot(xmin)) {  
        if(xmax == 0) return true;  
        if(xmax != xid) {  
            if(!tm.isCommitted(xmax) || xmax > xid || t.isInSnapshot(xmax)) {  
                return true;  
            }  
        }  
    }  
    return false;  
}
```


TODO: 完整的工作流程, 逻辑



















### Entry 与 MVCC 的配合逻辑简图

在事务执行过程中，MyDB 会如下运作：

1. 插入数据：
    
    - 构造 Entry，写入 XMIN = 当前事务 ID，XMAX = 0
        
    - 由 VM 调用 DM 写入 DataItem
        
    - 生成 UID 并返回
        
2. 读取数据：
    
    - VM 根据 Entry 的 XMIN 和 XMAX，判断是否可见
        
    - 若可见，则返回 `Entry.data()`
        
3. 删除数据：
    
    - 设置 XMAX = 当前事务 ID
        
    - 表示该条记录被当前事务“逻辑删除”
        
4. 回滚操作：
    
    - 利用 UndoLog 恢复旧版本 Entry
        
    - 由 VM 控制事务生命周期与数据恢复
