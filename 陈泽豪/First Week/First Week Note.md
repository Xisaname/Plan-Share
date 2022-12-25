#### 一些解释

对于Dedup的处理场景一共分为以下四个：

① 处理的场景为no hash && no lbn，指代产出了新的文件和新的内容。

 * 1- 计算hash，获得hash_pbn
 * 2- 通过hash查找对应的hash_pbn_value，没有找到
 * 3- 寻找bil是否具有lbn，也没找到
 * 4- 执行handle_write_no_hash
 * 处理流程为接来下就是需要把所需要的新的hash index和LBN都产生出来，并且记录。

~~~
static  int __handle_no_lbn_pbn(struct dedup_config *dc, struct bio *bio, uint64_t lbn, u8 *hash)
~~~

② 处理的场景为no hash && has lbn，一般指代chunk IO得到了更新

 * 1- 获得hash
 * 2- 没有找到pbn
 * 3- 找到了lbn
 * 4- 到达此处理函数
 * 处理流程：只要应用层修改的数据小于4k，比如我们的文档和代码工作。出现这种情况程序需要去将曾经的LBN-oldPBN给替换成新的LBN-newPBN，并且将hash index的refcount -1代表，我们少了一次映射在这个已经存在的hash-PBN上，这时这个hash-PBN很可能成为了孤儿，没有LBN与之map，但是程序并不着急释放掉它，更希望它能够被情况③所利用。

~~~
static int __handle_has_lbn_pbn(struct dedup_config *dc,struct bio *bio, uint64_t lbn, u8 *hash,u64 pbn_old)
~~~

③ 处理的场景是hash && no lbn，指代可以重复删除的数据

 * 1- 获得hash
 * 2- 找pbn找到
 * 3- 找lbn没找到
 * 4- 到达此执行函数
 * 处理的流程是在这种情况下只需要添加一条LBN-PBN的映射关系，并将PBN的引用+1，
 * 即可以节省一个BLOCK的空间。这就是最能够体现重删程序价值的地方，节省了实际空间。

~~~
static int __handle_no_lbn_pbn_with_hash(struct dedup_config *dc,struct bio *bio, uint64_t lbn,
u64 pbn_this,struct lbn_pbn_value lbnpbn_value)
~~~

④ 该处理场景为hash && lba，但是不一定代表数据没有改变。
因为hash-PBN的存在，仅仅是某个物理块和request的内容一样，但是不代表曾经的LBA里面存 在内存也是这个内容，所以还需要判断曾经的LBA里面是否也是这个内容，那么需要比对一下LBN-PBN ?= hash-PBN的PBN-number）， 如果不一样就把PBN-old refcount -1和把PBN-new refcount+1，并且更新LBN-PBN。通常这种情况是：在一个系统下的批量文件同步/覆盖的，
这种情况副本A和副本B的引用不变，逻辑映射关系改变。

~~~
static int __handle_has_lbn_pbn_with_hash(struct dedup_config *dc,struct bio *bio, uint64_t lbn,
u64 pbn_this,struct lbn_pbn_value lbnpbn_value)
~~~



#### Filebench

这个东西较为简单，我并未深入到源码级别去了解，大体看了看怎么用，其源码本身就包括了大量的工作负载，其本身还提供生成自定义工作负载的能力。