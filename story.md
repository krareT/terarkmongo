# mongo on TerarkDB

# 趟过的那些坑
# rocksdb 的坑
关于 rocksdb 的坑，我们在 [这篇文章](https://github.com/Terark/terarkdb/wiki/rocksdb-bugfix-by-terark) 中有一些总结。

# mongo-rocks 的坑

# mongodb 自身的坑
# 从 initial sync 开始
如果我们的 mongodb 设置了主从（master-slave），当一个 slave 从空库初始化时，
它需要把 master 上的所有数据“同步”过来，在这个过程中，主库的读压力很大，从库的写压力很大，
虽然如此，mongo 官方 wiredtiger 引擎和 mongo-rocks 引擎都工作得不错。

我们的 mongodb 主库有 3.7TB 的数据，我们需要用这 3.7TB 的数据进行测试，
官方的 wiredtiger 引擎和 mongo-rocks 自不必说，它们本身经过了大量的测试。

当使用 TerarkDB 引擎进行测试时，很多问题就暴露出来了……

我们都知道，TerarkDB 默认使用的是 universal compaction，
而 universal compaction 在官方的 mongo-rocks 中根本没进行过任何测试。

第一个坑，在 initial sync 的过程中，持续不断的写入，自然会有频繁的 compact。
我们不难理解，initial sync 时，数据是根据 key 按序写入的，
并且我们知道，对于有序（按 key 有序）的输入，rocksdb 的 compact 实际上只需要
把 SST “移动”到目标 level 上，因为不同 SST 的 key 不会有重叠。
但是，这仅仅是针对 level compaction 的优化，对于 universal compaction，
完全没有类似的优化……

问题很快出现：level0 的 SST 达到数目上限了，写入停顿了！————可是，我们已经把
level0 SST 的上限设成 100 了呢！
所以，干脆直接去掉所有对 level0 SST 的限制…… 

结果，很快，又出问题了：内存超限，OOM 了！解释一下，为了测试在有限内存下的系统表现，我们使用
cgroup 对进程的内存做了限制，虽然机器有 512G 的内存，但是我们限制进程只能使用 8G，
还好，这个机器的 linux 内核版本，使得 cgroup 只能 anonymous 的 mmap 进行限制，
不能对 shared mmap 的内存进行限制，也就是说，它（基本上）只能限制 malloc 出来的内存。
所以，也就是说，这个 OOM 是因为 malloc 的内存达到了 8G 的上限！分析了一下 LOG 发现：
rocksdb 把超过 200G 的数据，往一个 level0 的 SST 进行 compact！
<table><tr><td>
TerarkDB 使用的是全局压缩，对于 Index，需要把 一个 SST 的所有 Key 放入内存进行压缩（创建压缩的索引），为此，TerarkDB 从一开始就有对内存用量的限制，有个 soft limit 和 hard limit，这两个 limit 主要是为了在并发 compact 时，限制 compact 的内存用量，因为单个TerarkDB SST 的生成也分为好几个阶段，多个阶段之间还可以存在一定程度的并发，同时还有不同 SST 的并发，所以，TerarkDB 有一套调度机制，在内存不超限的前提下，尽可能公平、高效地执行计算任务。为了尽最大努力工作，TerarkDB 仍允许单个计算任务（例如 Index 压缩）的内存用量超过hard limit（TerarkDB option 中的设置），只是此刻 TerarkDB 中就只能有这一个计算任务。<br/><br/>
然而，SST 终究还是太大，创建索引时 Index Key 的总和超出了 cgroup 内存限制，最终引发 OOM。
</td></tr></table>

这里就引出 rocksdb 的一个问题：**level0 到 level0 的 compact 中，SST 尺寸不受限制！**
为了暂时缓解这个问题，我们把 max_merge_width 改成了 20，同时也把 cgroup 内存限制改成
了 64G，同时增大并发线程数，期望 level0 能及时 compact 到下层……，然而事情并未依照我们的愿望发展，又 OOM 了，观察 LOG 发现，有个输入数据 2.8T 的 SST 创建企图，自然而然地失败了……

最终，我们对 mongo 本身做了一个修改，在 initial sync 开始时，关闭“**自动 compact**”，initial sync 结束时，强制执行一个 full compact，然后再开启自动 compact，恢复正常配置。

此刻，问题貌似完美解决，然而，新一轮的浩劫才刚刚开始……

我们很快发现，第一个 collection 同步完以后，mongo 显示正在创建索引，但 rocksdb 长时间没有任何日志输出，CPU 负载也一直是单核 100%，索引创建的速度竟然比数据本身的同步慢了一个数量级！然后我们发现，mongo 创建索引时是先把数据扫描一遍，期间把索引的 Key 抽出来，并排序成多个 Sorted Run 写入文件，然后对多个 Sorted Run 进行多路归并，多路归并的结果批量写入存储引擎（例如 wiredtiger 或 rocksdb）。

这个思路虽然不完美（例如没有使用著名的 replace-select-sort 外排算法），代码实现上也不够优化，但整体上没有瑕疵。可是，**为什么就这么慢呢**？我们很快发现，每次 pstack 时，经常在堆栈中看到`rocksdb::MergingIterator::status()`，原来，mongo-rocks cursor 的每次 `next` 操作，都会调用到这个函数，在启用 **自动 compact** 时，`MergingIterator` 没有那么大的输入路数（`MergingIterator`本质上是使用一个最小堆，对多路输入进行多路归并，其中 level0 的每个 SST 是一路输入，其他每个 level 是整个 level 作为一路输入），而禁用自动 compact 时，`MergingIterator` 的输入**路数**暴增，在我们这个 case 中达到 600 以上（所有 SST 都在 level0）。问题找到了，就很容易解决，把 mongo-rocks 中不必要的 `status()` 调用去掉即可。

然而，问题依旧……填了这个坑，又掉进另一个坑，这两个坑挨得很近，但是，是不同的人挖的，`status()` 这个坑是 mongo-rocks 挖的，这个新的坑，是 mongo 挖的：在 build index 过程中，mongo 每隔 10 毫秒，或者每遍历 128 条数据，会进行 `yield`，即**保存当前状态**，中断当前工作，给其他线程一个调度的机会，`yield` 结束之后，**恢复状态**，继续运行！这么周全的处理，能有有什么问题呢？其实跟前面那个坑一样：mongo 每次**恢复状态**时，会调用 `iterator->Seek`，`MergingIterator` 路数很少的时候，问题不大，路数一多，问题就严重暴露出来了。这个问题修复也很简单，在 mongo-rocks 中做个 workaround，如果 seek 的 key 跟 iterator 当前的 key 相同，就跳过 seek 的执行。


