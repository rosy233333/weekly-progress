# io_uring的性能测试调研

## 测试程序

用于测试io_uring的性能，最常用的工具是[fio](https://github.com/axboe/fio)（[介绍](https://www.cnblogs.com/505donkey/p/18742275)）。它是一款磁盘性能测试工具，可以根据需要编写不同的测例，测量磁盘的性能指标（吞吐量（MB/s）、响应速度（IOPS/每秒操作数）、延迟（毫秒级精度））。它支持同步io、Linux aio、io_uring等io方式（[本文](https://dl.acm.org/doi/10.1145/3534056.3534945)中说其还支持SPDK）。由C语言编写。

[https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/](https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/)本链接中的性能测试代码和测例仍未找到。

## io_uring和SPDK的性能比较

在几个位置都将io_uring与SPDK进行了性能比较：

[https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/](https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/)本链接中，启用了轮询的io_uring性能与SPDK相近。

[https://asynciobench.github.io/](https://asynciobench.github.io/)此处的测试结果显示SPDK的性能远高于io_uring。因为文中没有介绍各框架的配置，不知是否是由于io_uring未开启轮询导致。

[https://dl.acm.org/doi/10.1145/3534056.3534945](https://dl.acm.org/doi/10.1145/3534056.3534945)本文的测试结果也说明启用了轮询的io_uring性能在一些条件下与SPDK相近，而其它条件下则低于SPDK。文章建议使用io_uring时，开启轮询且配置两倍于job数量的核心数。
