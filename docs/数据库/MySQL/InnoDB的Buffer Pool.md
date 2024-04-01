# 1. 缓存

对于InnoDB存储引擎的表来说，索引（聚簇索引、二级索引）、系统数据，都是以页的形式存放在表空间。

表空间实际上就是InnoDB对一个或几个实际文件的抽象，即数据最终仍然是存储在磁盘上。

直接访问磁盘和在内存中访问速度上肯定不在一个量级上，所以InnoDB存储引擎在处理客户端请求时，就会把对应需要访问的页加载到内存。

即访问某个页中的一条记录，也需要把整个页的数据加载到内存中，同时访问后也不会立即被释放，而是缓存起来，这样再有请求访问该页时，就可以省去I/O的开销。

# 2.InnoDB的Buffer Pool

## 2.1 Buffer Pool

Buffer Pool（缓冲区）：用于缓存磁盘中页 的一片连续内存空间，MySQL启动时就向操作系统申请。

默认128M，最小5M，可以在配置文件中通过`innodb_buffer_pool_size`进行配置，单位：字节

## 2.2 Buffer Pool 内部组成

- `缓冲页`：Buffer Pool对应的一片连续内存被划分成若干个和InnoDB表空间页大小一样的页，默认：16KB，内存中的页称为缓冲页。

- `控制块`：每个缓冲页都有一个对应的控制块，包含：页所属的表空间编号、页号、缓冲页在Buffer Pool中的地址、链表节点信息等

> PS：
>
> 控制块和缓冲页一一对应，都在Buffer Pool中
>
> 控制块在缓冲页前面
>
> 控制块占用大小是固定的，DEBUG模式下大约占用缓冲页大小的5%，但是我们设置的`innodb_buffer_pool_size_`是不包含控制块的大小的，
> 所以InnoDB会多申请这部分的内存空间

Buffer Pool内存空间如下图所示：

<img src="111" title="Buffer Pool对应的内存空间" alt="1111"/>

## 2.3 free链表管理

free链表（空闲链表）：把空闲的缓冲页对应的控制块作为一个节点放到一个链表中，该链表就称为free链表。初始时每个缓冲页对应的控制块都会加入到free链表中。

- 基节点：包含链表的头节点、尾节点地址，以及当前链表中节点的数量等信息。
    - 该节点不包含在缓冲区内存空间中，需要额外申请内存空间
    - MySQL5.7.22版本中只占用40字节

从磁盘加载一个页到缓冲区流程：

- 从free链表中获取一个空闲的缓冲页，并填充该缓冲页对应的控制块，即该页所在的表空间、页号等信息
- 将该缓冲页对应的free链表节点（控制块）从链表中移除，表示该缓冲页已被使用

## 2.4 缓冲页的哈希处理

如何判断一个页已经在Buffer Pool中：

- key：表空间号+页号， value：缓冲页对应的控制块地址， key、value构成一个哈希表
- 先从哈希表中根据`表空间号+页号`查看是否有对应的缓冲页
    - 有：直接使用该缓冲页
    - 没有：从free链表中选一个空闲的缓冲页，然后从磁盘将该页加载到该缓冲页

## 5.5 flush链表的管理

脏页（dirty page）：修改了Buffer Pool中缓冲页的数据，但是没有刷新到磁盘，该缓冲页就称为脏页。

flush链表：一个存储脏页的链表，修改过的缓冲页对应的控制块会作为一个节点加入到该链表中。

- 该链表对应的缓冲页需要被刷新到磁盘上，所以也称为flush链表
- flush链表和free链表构造差不多

> 一个缓冲页对应的控制块不可能既是free链表的节点，也是flush链表的节点

## 2.6 LRU链表的管理

预读（read ahead）：InnoDB从磁盘加载页时，考虑到可能也会在读取后面的页，所以把这些页面也加载到了Buffer Pool。

根据触发方式不同，预读可以分为以下两种：

- 线性预读：如果顺序访问的某个区(extent)的页面超过阈值，就会触发一次异步读取下一个区中全部的页面到Buffer Pool中的请求。
    - `innodb_read_ahead_thresold`：阈值，默认56，是一个全局变量，可通过`SET GLOBAL`命令修改
    - 异步：从磁盘加载这些被预读的页面时，不会影响当前工作线程的执行
- 随机预读：如果某个区的13个连续的页面都被加载到了Buffer Pool，无论这些页面是不是顺序，都会触发异步读取本区中其他页面到Buffer Pool中。
    - `innodb_random_read_ahead`：随机预读开关，默认关闭：OFF，是一个全局变量，可通过`SET GLOBAL`命令修改
    - 注意这里的13个页面还要求是在整个`LRU链表的 young区域的前1/4处`

降低Buffer Pool命中率的情况：

- 加载到Buffer Pool中页未被用到，例如`预读机制`
- 大量不常使用的页被同时加载到Buffer Pool中时，可能会把使用频率高的页从Buffer Pool中淘汰掉，例如`对大表进行全表扫描`

Buffer Pool的LRU链表：根据一定的比例将LRU链表分为两部分

- young区域 ：存储使用频率高的缓冲页，也称为热数据
- old区域：存储使用频率低的缓冲页，也称为冷数据；默认old区域占链表比例为37%

该比例可通过`innodb_old_blocks_pct`属性查看，同时该属性是一个全局变量，可通过`SET GLOBAL`命令修改

```bash
mysql> show variables like 'innodb_old_blocks_pct';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+
```

针对可能降低Buffer Pool命中率情况的解决方案：

- 针对预读页面可能不进行后续访问的优化
    - 磁盘上页首次加载到Buffer Pool中某个缓冲页时，该缓冲页对应的控制块会放到old区域的头部
        - 这样预读Buffer Pool却不进行后续访问的页面就会逐步从old区域删除，从而不影响young区域中使用比较频繁的缓冲页
- 针对全表扫描，短时间内访问大量使用频率非常低的页面的优化
    - 在对某个处于old区域的缓冲页进行第一次访问时，在对应控制块记录访问时间
    - 如果后续访问与第一次访问的时间在某个时间间隔内，那么该页面不会从old区域移动到young区域的头部，否则将它移动至young区域的头部
    - 时间间隔由系统变量控制：`innodb_old_blocks_time`：默认1000ms，
  ```bash
  mysql> show variables like 'innodb_old_blocks_time';
  +------------------------+-------+
  | Variable_name          | Value |
  +------------------------+-------+
  | innodb_old_blocks_time | 1000  |
  +------------------------+-------+
  ```

### 2.6.4 进一步优化LRU链表

- 策略一：降低调整LRU链表的频率，从而提升性能：只有被访问的缓冲页位于`young区域1/4的后面`时，才会被移动到LRU链表头部
- 略

## 2.7 其他一些链表

为了更好地管理Buffer Pool中的缓冲页，除了前面的措施，还引入其他的一些链表：

- unzip LRU链表：管理解压页的链表
- zip clean链表：管理压缩页的链表
- zip free数组：数组中每个元素都是一个链表，组成伙伴系统来为压缩页提供内存空间等
- 略

## 2.8 刷新脏页到磁盘

后台有专门的线程每隔一段时间把脏页刷新到磁盘，不影响用户正常的请求。主要有以下几种方式：

- BUF_FLUSH_LRU：从LRU链表的冷数据中刷新一部分页面到磁盘
    - 后台线程定时从LRU链表尾部开始扫描一些页，如果在LRU链表中发现脏页，则把他们刷新到磁盘
    - 注：
        - 扫描页面的数量通过系统变量`innodb_lru_scan_depth`设置
        - 缓冲页是否是脏页：缓冲页对应的控制块中记录了缓冲页是否被修改
- BUF_FLUSH_LIST：从flush链表中刷新一部分页面到磁盘
    - 后台线程定时从flush链表中刷新一部分页面到磁盘，刷新速率取决于系统是否繁忙
- BUF_FLUSH_SINGLE_PAGE：将单个页面刷新到磁盘
    - 后台线程有时刷新脏页进度较慢，导致用户线程在加载一个磁盘页到Buffer Pool时没有可用的缓冲页
    - 尝试查看LRU链表尾部是否可以直接释放到未修改的缓冲页，如果没有则将LRU链表尾部的一个脏页刷新到磁盘
    - 注：
        - 磁盘操作较慢，在请求时刷盘会降低用户请求速度
        - 系统繁忙时，也有可能出现用户线程从flush链表中刷新脏页的情况

## 2.9 多个Buffer Pool实例

Buffer Pool可以拆分成多个小的Buffer Pool，每个Buffer Pool称为一个实例。

`innodb_buffer_pool_instances`属性可以设置Buffer Pool实例个数

> 注：
>
> buffer pool实例 = innodb_buffer_pool_size / innodb_buffer_pool_instances
>
> 当innodb_buffer_pool_size小于1GB时，设置多个实例是无效的

## 2.10 innodb_buffer_pool_chunk_size

MySQL不是一次性为Buffer Pool实例申请一大片连续内存空间，而是以`chunk`为单位申请空间。

一个Buffer Pool实例由若干个`chunk`组成，一个`chunk`代表一片连续的内存空间，包含了若干缓冲页与其对应的控制块

MySQL在运行期间可以调整Buffer Pool的大小，可以通过`chunk`为单位增加或删除内存空间

`innodb_buffer_pool_chunk_size`：指定chunk大小，，默认值128M，该属性值在运行期间不可以修改。

- 由于该属性不包含缓冲页对应的控制块内存空间，所以实际InnoDB申请的内存空间会比该属性值大点

## 2.11 配置Buffer Pool时的注意事项

`innodb_buffer_pool_size`必须是 `innodb_buffer_pool_chunk_size` * `innodb_buffer_pool_instances`的倍数

- 保证Buffer Pool实例中的chunk数量相同

默认的chunk大小是128M，实例数默认是16，所以`innodb_buffer_pool_size`值必须是2GB或其整数倍，不是的话会被MySQL自动调整为整数倍

```bash
mysql> show variables like '%innodb_buffer_pool%';
+-------------------------------------+----------------+
| Variable_name                       | Value          |
+-------------------------------------+----------------+
| innodb_buffer_pool_chunk_size       | 134217728      |
| innodb_buffer_pool_dump_at_shutdown | ON             |
| innodb_buffer_pool_dump_now         | OFF            |
| innodb_buffer_pool_dump_pct         | 25             |
| innodb_buffer_pool_filename         | ib_buffer_pool |
| innodb_buffer_pool_instances        | 1              |
| innodb_buffer_pool_load_abort       | OFF            |
| innodb_buffer_pool_load_at_startup  | ON             |
| innodb_buffer_pool_load_now         | OFF            |
| innodb_buffer_pool_size             | 536870912      |
+-------------------------------------+----------------+
```

## 2.12 查看Buffer Pool的状态信息

查看Buffer Pool状态：`show engine innodb status;`

```bash
mysql> show engine innodb status\G;

BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 549715968
Dictionary memory allocated 441202
Buffer pool size   32764
Free buffers       31447
Database pages     1301
Old database pages 460
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 508, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 511, created 849, written 20160
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 1301, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```

参数说明：

- Total large memory allocated：表示Buffer Pool申请的连续内存空间大小，包括全部控制块、缓冲页、碎片大小
- Dictionary memory allocated：数据字典信息分配的内存空间大小，不包含在Total large memory allocated
- Buffer pool size：表示Buffer Pool可以容纳多少缓冲页，单位：页
- TODO