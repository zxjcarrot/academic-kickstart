> 应好友[胡神](http://conanhujinming.github.io)疯狂邀请(a.k.a. push)，写篇文章讲下[DPTree](http://www.vldb.org/pvldb/vol13/p421-zhou.pdf)。
- [背景](#背景)
- [灵感来源: Hybrid Index](#灵感来源-hybrid-index)
- [DPTree: 写优化NVM索引结构](#dptree-写优化nvm索引结构)
	- [In-Place Merge](#in-place-merge)
	- [读优化](#读优化)
- [总结](#总结)
- [感想](#感想)
- [参考文献](#参考文献)
## 背景
2019年4月，Intel的Optane DC PMM商用了，数据库领域研究了几十年的NVM终于有真实NVM硬件可以用了，研究者们也是喜大普奔。如学术界预测，实际NVM硬件存在着读写性能不对称的问题，即读带宽要高于写带宽。在Optane DC PMM上，单DIMM的读写带宽差距高达3倍<sup>[6]</sup>。同时这么多年的NVM相关研究中，索引结构对于数据系统来说是一个非常重要的组件，NVM研究领域中里其实已非常多的索引结构相关研究了，例如CDDS-Tree<sup>[1]</sup>, FPTree<sup>[2]</sup>, wB<sup>+</sup>-Trees<sup>[3]</sup>, BzTree<sup>[4]</sup>等等，这些工作提出了很多有用的NVM索引设计技巧，但是在NVM写入优化方面没有研究的非常透彻。不同于这些工作，DPTree是一种针对写优化的NVM索引结构，缓解现有NVM硬件下的写性能较差问题，同时也采用了很多前人的NVM索引设计技巧。

## 灵感来源: Hybrid Index
研一的时候哼哧哼哧读了CMU 15-721(强烈推荐)课程中不少论文，在[database compression](https://15721.courses.cs.cmu.edu/spring2017/slides/11-compression.pdf)一节中一篇关于Hybrid Index的论文<sup>[5]</sup>引起了我的注意(Hybrid Index的一作[Huanchen Zhang
](http://www.cs.cmu.edu/~huanche1)也是SuRF的一作, SuRF拿了SIGMOD'18 Best Paper Award, 这老哥太猛了orz)。

Hybrid Index的Motivation在于降低内存索引结构的空间开销，方法是采用dual-stage的索引框架，将一个索引结构分成dynamic和static两个stage。以B+Tree为例，Hybrid B+Tree的dynamic stage是一个write buffer，比如是普通的B+Tree，处理insert/delete/update/get/range操作；static stage是一个空间优化过的只读B+Tree结构。dynamic stage的buffer 满的时候与static stage进行归并合并，产生出一个新的static stage。由于static stage只处理get/range查询，同时更新的时候采用out-of-place 合并更新，那么传统B+Tree很多结构上的空间overhead就可以进行优化了：如节点间的指针、叶子节点内部用于处理写入的预留空间等等。

我注意到Hybrid Index这种批量更新static stage的方法不仅可以用来降低内存索引的空间开销，其实还可以用来帮助NVM索引降低写入开销。因为批量更新的思想其实能够很好地均摊NVM索引写入操作的元数据更新代价：比如说很常见的NVM索引叶子节点原子更新技巧，每次对叶子节点的写入一个KV数据后，首先需要保证KV数据刷入NVM，然后需要更新叶子节点的bitmap，通常bitmap不超过8B，可以使用8B failure-atomic update技巧 (bitmap通常用于记录节点中kv slots/array的分配情况，这是一种非常实用的NVM数据结构设计技巧)。为了保证索引结构的crash-consitency特性，这些诸如bitmap的元数据更新往往是需要单独的一次NVM写入的。因此，这类元数据往往会占据较多NVM索引写入开销。如果我们能**批量**地往一个叶子节点里写入多个KV，那么对于bitmap这类的元数据更新代价就能被均摊开了，这就是DPTree的核心insight。

## DPTree: 写优化NVM索引结构
![image](/img/dptree-arch.jpg)

如Figure 1所示，DPTree在NVM场景下采用了类似Hybrid Index的二级架构，引入buffer tree和base tree的结构。其中，buffer tree存储在DRAM中，其crash-consitency用一个NVM log来保证，这点类似于LSMT的memtable，处理来针对DPTree的insert/delete/update/get/range请求。base tree存储在NVM中，处理只读请求，并针对读和合并进行了优化。DPTree针对buffer tree和base tree的维护一个大小比例R，当buffer与base的大小比超过R时候，会将buffer合入base tree。这里我们类似前人的工作，假设插入的KV是固定长度的。

注意到这里DPTree的buffer tree虽然采用了logging，但是针对NVM做出了优化，能够做到单条cache-line大小日志的只需要接近一次的NVM顺序写入，对细节感兴趣的同学可以读下论文2.1部分。
### In-Place Merge
在处理buffer合入base的过程中，我们当然可以直接采用LSMT的out-of-place这种简单的合并方式，但是我们发现这种out-of-place合并方式可能会产生1/R的NVM写入放大，并不会比传统NVM索引结构有更好的写入改善。我们注意到这种out-of-place的合并方式适合block-storage，即HDD/SSD这类无法进行按字节寻址的设备，并没有利用上NVM可以按字节寻址这个特性。因此DPTree设计了一种in-place merge方式，来利用NVM这种独特的性质降低写入量。

![image](/img/base-leaf.jpg)
In-place merge的核心在于针对base tree的每个叶子节点，我们存储两份元数据信息，并由一个全局布尔变量gv来表示当前base tree的叶子节点使用哪一份分元数据。如Figure 3所示，base tree的叶子节点由两份元数据和一个KV数组组成。其中元数据中包含了诸如下一个叶子节点指针（用于range遍历)，bitmap（用于表示kv pool的kv分配状态），cnt(该元数据版本下节点中KV数量），以及一些用于加速读操作的元数据(下一节介绍)。

为了实现crash-consistency，处理合并时DPTree遵循一个合并准则：**不会覆盖当前gv变量所指的那一份元数据所表示的KV数据**。

举个例子，假设gv=0, 某个叶子节点L的元数据M[0].bitmap={0,1,1,0,1}，表示KV数组下标1,2,4上是有效数据，假设KV数组的内容为{x,2,4,x,5}(其中x表示未分配，简单起见，我们这里只展示key)。当我们需要合入更新{2',4'}时，DPTree在!gv表示的元数据上进行操作，在上面的例子中就是M[1]，根据准则，合并后L的kv数组为{2',2,4,4',5}, M[1].bitmap={1,0,0,1,1}，即对于版本0来说，有效数据是{x,2,4,x,5}，而对于版本1来说，有效数据是{2',x,x,4',5}. 在所有的叶子节点KV数组和元数据更新完毕后，我们只需要针对全局变量gv做一次flip，就能使所有的叶子节点更新生效，释放出老版本数据占据的空间。在这个过程的任何时刻出现crash，我们都能够利用合并准则恢复出原始数据，进而保证crash-consitency（回想下，合并成功前我们不会在覆盖老版本数据）。以此，我们就利用了NVM的按字节寻址特性，做到了大部分情况（叶子节点的free slots能容纳下更新)下叶子节点的in-place更新，降低了NVM写入代价。实验结果显示，DPTree的写入代价相比于现有索引结构降低了40%-90%。

当然了，这是一个极度简化的例子，只是帮助大家理解这个in-place合并过程的一些核心思想。
论文中详细讨论了许多复杂的合并情况，包括节点分裂/合并，恢复协议等等，感兴趣的同学可以读下论文。
### 读优化
因为引入了两级索引架构，一个请求可能需要查询两个数据结构才能完成，直觉上读延迟上有一些劣势。
但其实不然，两级结构在经过足够多的优化其实是能够与单级结构媲美的甚至有所超越的。
为了优化读，DPTree引入了如下优化技术：
1. **Bloom Filter for Buffer Tree** 注意到R取值一般较小(如0.1)，也就是数据和查询key均匀分布情况下，对buffer tree接近90%的查询是不会找到key的，使用bloom filter可以降低buffer tree get查询miss的代价。
2. **Selective Persistence** 我们知道DRAM的速度是要比NVM快的(Optane 随机读延迟是DRAM的3-4倍), 同时注意到base tree的内部结点占数据量的极小部分，但是查询每次都需要经过这些内部结点，将内部结点放在NVM中需要付出较高的读延迟代价。因此DPTree采用类似FPTree的设计，将base tree的内部结点存储在了DRAM中，以降低查询代价。
3. **Hash-based + Fingerprinting Leaf Design** 我们知道base tree叶子节点内进行顺序查询平均情况下需要读取一半的数据才能完成查询。为进一步加速叶子节点内的查询，我们将KV数组以hash table的形式进行组织，hash冲突解决采用linear probing的方式，对cache极其友好。同时DPTree借鉴了FPTree的fingerprinting技术。在Figure 3中，元数据信息中为每个key维护了1 byte大小的指纹信息，查询时先比对元数据的key的指纹信息，这样能够在key不存在的情况下进一步减少查询代价(大部分情况下不需要访问KV数组)。两种优化组合保证了大部分情况下successful lookup只需要两次cache-miss，unsuccessful lookup只需要一次cache-miss。同时linear probing的hash组织结构让fingerprint的检查也再次减少(FPTree中KV数组是无序的，需要线性扫描)。
4. **Accelerating Range Queries using Order Array** 现有NVM索引的leaf节点大多都采用无序存储(FPTree, BzTree)，这样执行range查询，即有序输出一个范围里的KV数据，需要首先执行sorting，因此导致range查询代价较高。DPTree吸纳wB<sup>+</sup>-Trees的设计思想，在元数据中加入了一个小索引数组(order array, O)，用于表示KV数组的内的数据顺序，以加速range查询。即KV[O[0]]表示了key最小的数据，KV[O[1]]表示了key第二小的数据，以此类推。

经过这一系列的读优化，实验表明，DPTree的点读延迟是最低的，同时range查询也只比叶节点有序存储的B树(fastfair<sup>[7]</sup>)略差。注意到在base tree的这些读优化技术引入了更多的元数据，但是NVM写入代价依旧能够做到很低，这还是要归功于批量更新和in-place合并，让DPTree能够以很低的NVM写入代价来引入更多对读有帮助的元数据，使得DPTree整体在读和写方面表现得都非常优秀。

## 总结
DPTree的核心思想是批量更新和原地合并来降低元数据的更新代价，使用两级数据结构来构造一个索引结构。DPTree为了弥补两级结构带来的读延迟增加，引入了许多读优化技术，将读延迟做到了现有索引结构中最低，同时保持最少的NVM写入次数，这归功于批量更新和原地合并技巧。当然本文只是DPTree一个简单的介绍，对于具体合并步骤实现和以及并发DPTree的等复杂设计，感兴趣的同学可以再看下原论文。这里分别再给出[代码](https://github.com/zxjcarrot/DPTree-code)和[VLDB会议录屏](https://www.bilibili.com/video/BV1sa4y1J76f/)，希望对感兴趣的同学有帮助，也欢迎大家多多指正不足，第一次写论文，自认为写作和presentation还是很烂的。

## 感想
读研前连research是啥都不知道，所以研一基本处于一个学习的状态，基本上围绕15445/15721(强烈推荐)来学习数据库系统的理论知识和前沿课题，连DPTree的idea也是来自这里2333333。研二在DPTree工作上总共花了快一年的时间，写作也比较烂，第一次review结果出来因为没有在真实硬件上跑实验给了weak reject，但是reviewer觉得idea害行，给了次revision机会，让我找个真实硬件跑下，后面找了阿里的师兄帮忙弄了个设备，最后能中VLDB也是纯属运气好2333。当然了，这个过程中收获也非常大，比如锻炼了我写作能力，科研能力以及系统编程能力(性能调优，并发编程）等等。同时也离不开老师同学们的帮助，比如帮忙提供NVM硬件的阿里师兄，在论文writing方面帮助我的导师和胡神，非常感激。

## 参考文献
[1] Venkataraman, Shivaram, et al. "Consistent and Durable Data Structures for Non-Volatile Byte-Addressable Memory." FAST. Vol. 11. 2011.<br>
[2] Oukid, Ismail, et al. "FPTree: A hybrid SCM-DRAM persistent and concurrent B-tree for storage class memory." Proceedings of the 2016 International Conference on Management of Data. 2016.<br>
[3] Chen, Shimin, and Qin Jin. "Persistent b+-trees in non-volatile main memory." Proceedings of the VLDB Endowment 8.7 (2015): 786-797.<br>
[4] Arulraj, Joy, et al. "Bztree: A high-performance latch-free range index for non-volatile memory." Proceedings of the VLDB Endowment 11.5 (2018): 553-565.<br>
[5] Zhang, Huanchen, et al. "Reducing the storage overhead of main-memory OLTP databases with hybrid indexes." Proceedings of the 2016 International Conference on Management of Data. 2016. <br>
[6] Lersch, Lucas, et al. "Evaluating persistent memory range indexes." Proceedings of the VLDB Endowment 13.4 (2019): 574-587.<br>
[7] Deukyeon Hwang, Wook-Hee Kim, Youjip Won, and Beomseok Nam. 2018. Endurable transient inconsistency in byte-addressable persistent B+-tree. In Proceedings of the 16th USENIX Conference on File and Storage Technologies (FAST'18). USENIX Association, USA, 187–200.
