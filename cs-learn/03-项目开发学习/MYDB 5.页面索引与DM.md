

本节，实现简单的页面索引，同时实现DM层对上层的抽象： DataItem


读取缓存以页为单位，但是一个页包含多个数据


## 页面索引

>当我们要插入一条新的数据时，数据库系统需要寻找**有足够空间的页**来存放该数据，否则遍历查找，效率低

缓存了每一页的空闲空间，用于上传模块执行插入操作时，迅速找到合适空间的页面，从而无需从磁盘/缓存逐个查找。


MYDB 用一个比较粗略的算法实现了页面索引，将一页的空间划分成了 40 个区间。在启动时，就会遍历所有的页面信息，获取页面的空闲空间，安排到这 40 个区间中。insert 在请求一个页时，会首先将所需的空间向上取整，映射到某一个区间，随后取出这个区间的任何一页，都可以满足需求。


PageIndex 的实现也很简单，一个 List 类型的数组。





```java
public class PageIndex { 

// 将一页划成40个区间 
	private static final int INTERVALS_NO = 40; 
	private static final int THRESHOLD = PageCache.PAGE_SIZE / INTERVALS_NO;
	private List[] lists; 
}
```


从 PageIndex 中获取页面就是算出区间号，直接取即可：

```java
public PageInfo select(int spaceSize) {  
    lock.lock();  
    try {  
        int number = spaceSize / THRESHOLD;  
        if(number < INTERVALS_NO) number ++;  
        while(number <= INTERVALS_NO) {  
            if(lists[number].size() == 0) {  
                number ++;  
                continue;  
            }  
            //这个0是啥意思
            return lists[number].remove(0);  
        }  
        return null;  
    } finally {  
        lock.unlock();  
    }  
}
```


返回的 PageInfo 中包含页号和空闲空间大小的信息。


被选择的页，会直接从 PageIndex 中移除，这意味着，同一个页面是**不允许并发写的**。在上层模块使用完这个页面后，需要将其重新插入 PageIndex：

```java
public void add(int pgno, int freeSpace) {  
    lock.lock();  
    try {  
        int number = freeSpace / THRESHOLD;  
        lists[number].add(new PageInfo(pgno, freeSpace));  
    } finally {  
        lock.unlock();  
    }  
}
```




在 DataManager 被创建时，需要获取所有页面并填充 PageIndex：


使用完Page之后及时release, 否则容易撑爆缓存


```java
void fillPageIndex() {  
    int pageNumber = pc.getPageNumber();  
    for(int i = 2; i <= pageNumber; i ++) {  
        Page pg = null;  
        try {  
            pg = pc.getPage(i);  
        } catch (Exception e) {  
            Panic.panic(e);  
        }  
        pIndex.add(pg.getPageNumber(), PageX.getFreeSpace(pg));  
        pg.release();  
    }  
}
```


### DataItem
DataItem 是 DM***向上层提供的数据抽象***。上层模块通过地址，向DM请求到对应的DataItem，在获取到其中的数据。

保存dm的引用是因为其释放依赖dm的释放 （dm同时实现了缓存接口，用于缓存DataItem), 以及修改数据时落日志。

DataItem保存的数据结构如下：

`[ValidFlag] [DataSize] [Data]`

其中 ValidFlag 占用 1 字节，标识了该 DataItem 是否有效。删除一个 DataItem，只需要简单地将其有效位设置为 0。DataSize 占用 2 字节，标识了后面 Data 的长度。

上层模块在获取到 DataItem 后，可以通过 `data()` 方法，该方法返回的数组是数据共享的，而不是拷贝实现的，所以使用了 SubArray。


上层模块试图对DataItem进行修改时，需要遵循一定的流程：修改之前要调用before（）方法，想要撤销修改时，调用unBefore方法，修改完成后，调用after()方法，主要是为了保存前相数据，并且及时落日志。DM会保证对DataItem的修改是原子性的。



```java
@Override  
public void before() {  
    wLock.lock();  
    pg.setDirty(true);  
    System.arraycopy(raw.raw, raw.start, oldRaw, 0, oldRaw.length);  
}  
  
@Override  
public void unBefore() {  
    System.arraycopy(oldRaw, 0, raw.raw, raw.start, oldRaw.length);  
    wLock.unlock();  
}  
  
@Override  
public void after(long xid) {  
    dm.logDataItem(xid, this);  
    wLock.unlock();  
}
```


`after()` 方法，主要就是调用 dm 中的一个方法，对修改操作落日志，不赘述。

在使用完 DataItem 后，也应当及时调用 release() 方法，释放掉 DataItem 的缓存（由 DM 缓存 DataItem）。



### DM的实现




DataManager 是 DM 层直接对外提供方法的类，同时，也实现成 DataItem 对象的缓存。DataItem 存储的 **key，是由页号和页内偏移组成的一个 8 字节无符号整数**，页号和偏移各占 4 字节。

DataItem 缓存，`getForCache()`，只需要从 key 中解析出页号，从 pageCache 中获取到页面，再根据偏移，解析出 DataItem 即可


```java
@Override 
protected DataItem getForCache(long uid) throws Exception {
	short offset = (short)(uid & ((1L << 16) - 1));
	uid >>>= 32; int pgno = (int)(uid & ((1L << 32) - 1)); 
	Page pg = pc.getPage(pgno); 
	return DataItem.parseDataItem(pg, offset, this); 

}
```

DataItem缓存释放，需要将DataItem写回数据源，由于对文件读写以页为单位进行，把其所在页release


从已有文件创建 DataManager 和从空文件创建 DataManager 的流程稍有不同，除了 PageCache 和 Logger 的创建方式有所不同以外，从空文件创建首先需要对第一页进行初始化，而从已有文件创建，则是需要对第一页进行校验，来判断是否需要执行恢复流程。并重新对第一页生成随机字节。


```java
public static DataManager create(String path, long mem, TransactionManager tm) {  
    PageCache pc = PageCache.create(path, mem);  
    Logger lg = Logger.create(path);  
  
    DataManagerImpl dm = new DataManagerImpl(pc, lg, tm);  
    dm.initPageOne();  
    return dm;  
}  
  
public static DataManager open(String path, long mem, TransactionManager tm) {  
    PageCache pc = PageCache.open(path, mem);  
    Logger lg = Logger.open(path);  
    //dm对象提供DM的方法  
    DataManagerImpl dm = new DataManagerImpl(pc, lg, tm);  
    if(!dm.loadCheckPageOne()) {  
        Recover.recover(tm, lg, pc);  
    }  
    dm.fillPageIndex();  
    PageOne.setVcOpen(dm.pageOne);  
    dm.pc.flushPage(dm.pageOne);  
  
    return dm;  
}
```





DM 层提供了三个功能供上层使用，分别是读、插入和修改。修改是通过读出的 DataItem 实现的，于是 DataManager 只需要提供 `read()` 和 `insert()` 方法。

`read()` 根据 UID 从缓存中获取 DataItem，并校验有效位


`insert()` 方法，在 pageIndex 中获取一个足以存储插入内容的页面的页号，获取页面后，首先需要写入插入日志，接着才可以通过 pageX 插入数据，并返回插入位置的偏移。最后需要将页面信息重新插入 pageIndex。


DataManager 正常关闭时，需要执行缓存和日志的关闭流程，不要忘了设置第一页的字节校验：