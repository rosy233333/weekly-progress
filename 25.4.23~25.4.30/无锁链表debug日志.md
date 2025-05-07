# 无锁链表debug日志

无锁链表的测试函数的执行过程出现了两个问题：SIGSEGV和卡死

SIGSEGV只有在delete函数中、只有设定了地址偏移量才会出现。

卡死就算只有pop函数、就算未设定地址偏移量也会出现。

## SIGSEGV问题

因为已经确定了问题在delete函数中，所以我列举了delete函数的各种情况（包括链表节点的标记情况），写了一个单线程测试。单线程调试起来就简单很多了，因此就定位到了错误。

错误原因如下：

1. MarkedPtr在转换成指针时，只去了标记，没有调用其内层的转换函数；
2. 位置无关指针在进行转换时，对空指针的判断方式是完全相等，因此判断不出带标记的空指针。解决方案是在进行地址转换时，先去掉标记，调用位置无关指针的转换函数后，再加上标记。

对应的commit：[6a1316c](https://github.com/AsyncModules/pilf_buddy_alloc/commit/6a1316c665edf495c77dcfc300b7590afadf2865)

## 卡死问题

我首先通过barrier等方式确定了问题由pop函数引起，且是在pop链表的最后几个元素时产生卡死，并且被卡死的只有一个线程。之后着重检查pop函数，发现了导致卡死的原因是一次设计失误导致一次pop可能删除多个元素。由于测试函数中需要成功执行指定次数的pop后才能退出（从而刚好删完链表的元素），如果出现了“一次pop删除多个元素”的情况，就会导致卡死。

pop函数具体的设计失误如下：

原本的设计是，给一个节点打上标记后，如果没能成功删除，则重新获取一个节点，打标记并删除。但这样就会造成一次pop调用标记了多个节点（相当于删除了多个节点），因此导致测试代码中某些线程一直pop失败而卡死。

解决方案是取消了“重新获取节点”的过程，而是改成和delete类似的“等待被标记的节点被删除”，而后直接返回标记的节点。

对应的commit：[fbbb178](https://github.com/AsyncModules/pilf_buddy_alloc/commit/fbbb178bddb88b0b4a2dd3f3546b482819336273)

## 访存异常问题

zfl学长反馈，在进行堆分配器测试时，依然出现了访存异常。

在debug时，发现存储的地址都具有不应出现的标记，之后发现是因为地址未对齐。

该bug由于分配的空间的首地址未对齐引起（采用`[u8; 512]`分配空间），从而导致正常地址在转换后被错误地识别为带标记地地址。因此我将分配的空间改为了`[usize; 64]`，就解决了此bug。同时，还为`GetDataBase`接口增加了安全使用说明。

对应的commit：[pilf_buddy_alloc:0a796f9](https://github.com/AsyncModules/pilf_buddy_alloc/commit/0a796f90f4442cb0b6b88fb749305c2e9b6bb842)、[pi_pointer:9f566a2](https://github.com/AsyncModules/pi_pointer/commit/9f566a2b166271e10f826c6aa5406b1abbf7c6da)
