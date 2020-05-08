### 背压

背压是使用流处理系统中经常会面对的问题之一(之所以说是之一，是因为我觉得数据倾斜也时常遇到^-^)，背压的场景非常之多，比如在京东或淘宝的618、双十一
购物节中，当时钟指向0点时，大量的用户开始将喜欢的商品加入购物车并进行结算，此时系统的流量将是平时流量的几倍甚至是十几倍，在这段时间内系统接收到的
数据将远高于其所能处理的数据量，这就是通常所说的"背压"。如果背压不能得到有效的处理，将会耗尽系统资源甚至导致系统崩溃。

我们知道目前主要的流处理引擎其实都提供了背压功能，但是具体到每个引擎其实现背压的方式却各不相同。新版的Storm采用的是引入高性能无锁缓冲队列Disruptor，
当这个队列已满时则停止发送数据，最终将背压一级级的向上传导直到Spout，于是Spout停止从kafka拉取数据。当Disruptor缓冲队列不再满时，Spout重新从kafka
拉取数据，从而实现了整个背压的处理过程。由于Disruptor使用的是环形数组RingBuffer实现，且在读取写入时无锁所以性能较高。但是需要注意的时一定要设置该缓
冲队列的长度，否则该队列的长度将会无限长，依然会有背压问题，但是该长度的设置就非常难以确定，设置的太短，会导致频繁的背压，影响数据的处理，设置得过长又
起不到良好的背压效果。

Flink没有继续使用Disruptor这种数据结构，但是其背压的实现依然还是有点类似。具体来讲，Flink中每一个Task都会有一个InputGate和ResultPartition，InputGate
负责接收数据，ResultPartition负责发送数据(当然Source Task没有InputGate，Sink Task没有ResultPartition)，而这两个组件都会有一个对应的LocalBufferPool
(缓冲池)，LocalBufferPool中会有一定量的Buffer(其实就是Flink内存管理的单位MemorySegment的包装类)。当缓冲池中已申请的数量达到了上限或没有内存块时，
Task就会暂停读取Netty Channel，因此上游发送端就会立即响应停止发送并进入背压状态，于是上游的写入也会停止，从而将背压逐级向上传递。这就是Flink基于TCP
的背压实现的主要思路。

基于TCP的背压有两个明显的弊端：一个是每个TaskManager中可能要执行多个Task，如果多个Task的数据最终都要传输到下游的同一个TaskManager时就会复用同一个
Socket进行传输，此时候如果单个Task产生背压，则会导致复用的Socket整个发生阻塞，其余的Task的数据也无法进行传输，包括Checkpoint Barrier也无法发出导
致下游执行checkpoint的延迟也增大；二是它依赖于最底层的TCP去做流控，会导致背压传播路径过长，生效的延迟也比较大。因此在Flink 1.5版本开始就引入了新的
基于Credit的背压机制。它的原理很类似于TCP的Window机制，相信对于熟悉TCP的我们应该很容易就能理清其原理。



以上是一些理论上的讲解，概念上的阐述，主要作用是帮助分析，但是我们知道"talk is cheap"，所以还是来从源码上进行分析会来的更清楚。