# TiKV与TiDB间TCP小包优化 (Draft)

## 问题

在sysbench oltp_update_index负载下，观察到网络上充斥着小数据包，几个特定尺寸的包占绝大多数，且基本都在100B~300B，一般来说，这种网络环境时很差的。

## 状态

暂停。因为TiKV方面会有超过100个线程，仅仅处理这一块并不能一定带来整体的性能提升，这个只能靠猜和实验验证，这个方法是不那么好的，因为发现了coz。近来一直在尝试将coz应用到TiDB和TiKV上，但初步研究表明coz还很脆弱，和go/rust的程序交互上，并不能顺利进行。在考虑将coz移植到go和rust上，libcoz的代码量在2500+行。

## 笔记

- 之前做RPC时间追踪时，处理网络部分的时间时，观察到绝大多数GRPC请求基本都是1个req/resp占用一个TCP数据包。之前还想着这种的好，方便我处理，因为发现bench开始时会有大批量的GRPC请求/响应占用同一个TCP包，为了编号配对，我还得专门处理这类数据包的每一个GRPC请求，没注意到这里还有小包问题
- 鉴于tikv的release编译一次得差不多十分钟，本周我尝使用echo server为例，了解TPS极限，再来看tikv的处理差距
- 作为对照，我在程序里依次请求grpc-go的echo-server和rust版的echo-server
- 测试结果显示，同样请求数，go版本需要4ms完成的任务，rust版需要10ms
- 10w条req测试结果是go版本296ms，rust 1.0s
- 对比两者的网络包，发现巨大差异，数据包数量有25倍之差。至此我才意识到小包问题，此时收集了sysbench测试的网络包，小包问题确实严重，回顾之前整理网络包中的GRPC经历，明白过来一直有之。
- 所以tikv的ResponseBatcher可能没有正常工作
- 尝试在rust版的echo-server上使用`send_all`，无效
- 找到grpcio的0.7版中开始有一个`enhance_batch()`方法，启用后，`send_all`工作如预期。（enhance_batch好像是10月还是11月才合到master的）
- 作为对比，我的echo-server在无batch下，从server响应了19k+个包，开启batch后，只有500+，原来的数据包量是batch后的34倍，而且批量请求完成时间也从秒级降到毫秒级
- 虽然使用的loopback接口测试，MTU为65536，对比显示batch到9000字节（JumboFrame）附近并不如1500（一般以太网MTU）附近好，1500左右的还是最优的
- 全部batch的话，对于低并发情况，如果不加超时，没法继续前进，因为是请求/响应模型，客户端没收到响应的情况下，会一直等着，服务端没攒够也在等着
- 但加入超时还是不行，如果并发线程数不足以填满server缓冲区，server还是会不断等到超时，出现断流，损害吞吐
- 为解决这个这个问题，其实希望server可以知道有多少并发，这样控制batch最大个数，但考虑如果有闲置的并发连接，还是会有同样的问题
- 想计算一下数据速率，只在高速率下启用batch。但想到tikv的响应有一个transport_layer_load字段，用于反馈grpc线程的cpu使用率，考虑使用这个做判断条件，低使用率下不启用batch，不会引入延迟
- 考虑响应数据大小不一定是固定的，response个数作为batch依据应该不是很好，主要影响点在数据包的尺寸上
- 参看tikv中其他代码，得到计算message方法，加入按累积尺寸控制batch flush的时机
- 最终实现上，引入`启用Batch的CPU负载阈值`、`最大batch字节数`、`最大batch响应个数`和`batch超时阈值`4个可配置变量（不这么做，编译起来太耗时。通过配置的方式，好该配置对比）
- 测试验证，在500threads下跑oltp_update_index，有些配置下可以比master分支的tikv有不超过10%的TPS提升和平均延迟降低，但多个其他threads方案，性能下降。暂未能理解
- 可以确认的是，tcpdump的全部数据包数量能做到超过50%的减少。从perf拿到的火焰图对比中，grpc线程在master分支下，sendmsg系统调用占很大一块，启用我的batch后，火焰图上基本上看不到这一块了。
- 其实我更想使用系统自带的缓冲机制，比如TCP_CORK，利用rr查了sendmsg的调用链，看到[这个地方][1]，上面的注释也写着`TODO...CORK...`，之前看grpc repo上对这个事情的讨论，也是说还没实现
- Nagle算法（TCP_NODELAY）也在grpc连接初始化时以`grpc_set_socket_low_latency`的方式[禁掉了][2]
- 腾讯开源了他们内部使用Tars RPC框架，在他们的bench报告中展示了他们的grpc的TPS对比，包越小优势越明显，我倒想拿来换掉grpc对比测试下，但恐怕工作量太大，估计tikv也可能不会采用
- 其实TiDB作为RPC请求的发送端，也存在小包问题，tidb那边的batch似乎工作得也不是很好，这些都可以在Wireshark的TCP统计中观察到，除了Packet Length统计外，最直观的是TCP-Throughput展示
- TiDB往TiKV方向有TCP重传

[1]: https://github.com/pingcap/grpc/blob/ef43304e2ce673f3a0d134814ee4fc663f8c262b/src/core/lib/iomgr/tcp_posix.cc#L941
[2]: https://github.com/pingcap/grpc/blob/ef43304e2ce673f3a0d134814ee4fc663f8c262b/src/core/lib/iomgr/socket_utils_common_posix.cc#L234

