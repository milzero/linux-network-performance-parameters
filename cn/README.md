#  Linux 网络性能参数

# TOC

* [开门见山](#开门见山)
* [Linux网络协议栈一览](#Linux网络协议栈一览)
* [Fitting the sysctl variables into the Linux network flow](#fitting-the-sysctl-variables-into-the-linux-network-flow)
  * Ingress - 他来了
  * Egress - 他又走了
  * 为什么你是对的？ - perf
* [What, Why and How - 网络和sysctl参数](#what-why-and-how---network-and-sysctl-parameters)
  * 环形缓冲 - rx,tx
  * 中断合并 (IC) - rx-usecs, tx-usecs, rx-frames, tx-frames (hardware IRQ)
  * 中断合并 (soft IRQ) and Ingress QDisc
  * Egress QDisc - txqueuelen and default_qdisc
  * TCP Read and Write Buffers/Queues
  * Honorable mentions - TCP FSM and congestion algorithm
* [Network tools](#network-tools-for-testing-and-monitoring)
* [References](#references)

# 开门见山

我们有时希望在 [sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) 中找到秘籍: 无需在各种场景下做出取舍，仅修改这些值可以提高吞吐量和降低低延迟。这是不现实的，**较新的内核版本在默认情况下已经调整的很好了**。事实上，反而可能会 [如果你乱用默认设置]会损害性能(https://medium.com /@duhroach/the-bandwidth-delay-problem-c6a2a578b211)。
 
这个简短的教程展示**sysctl/network在网络流中的作用**，它受到了[Linux网络栈指南](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/)和[ Marek Majkowski's 的帖子](https://blog.cloudflare.com/how-to-achieve-low-latency/).)的许多启发

> #### 有什么问题请尽管提吧 :)

# Linux网络协议栈一览

![linux network queues](/img/linux_network_flow.png "A graphic representation of linux/kernel network main buffer / queues")

# 将系统变量拟合到Linux网络流中

## 入口 - 他们来了
1. 数据包到达 NIC
1. NIC校验 `MAC`(非混杂模式)和`FCS`,决定丢掉这个包或者继续处理
1. NIC由驱动[RAM中的DMA数据包](https://en.wikipedia.org/wiki/Direct_memory_access), 映射到一个事先开辟的内存区域
1. NIC查询接收处数据包的引用[环形缓冲区](https://en.wikipedia.org/wiki/Circular_buffer)队列“rx”直到“rx-usecs”超时或“rx-frames”。
1. NIC触发硬中断。
1. CPU运行驱动程序代码的“IRQ Handler”
1. 驱动“调度一个NAPI”，清除“硬中断”并返回
1. 驱动触发一个 `软中断(NET_RX_SOFTIRQ)`
1. NAPI从接收环缓冲区中轮询数据，直到“netdev_Budget_usecs”超时或“netdev_Budget”和“des_ weight”数据包
1. Linux会为 `sk_buff`开辟内存
1. Linux对元数据进行填充：协议、接口、设置MAC header，并删除以太网头部
1. Linux 将skb传递至内核栈 (`netif_receive_skb`)
1. 将设置网络header，克隆`skb'到抽头(即tcpdump )并将其传递到tc入口
1. 数据包被处理为qdisc大小的`netdev_max_backlog '，其算法由`default _ qdisc'定义。
1. 调用 `ip_rcv` ，数据包交给IP层处理
1. 调用netfilter (`PREROUTING`)
1. 查询路由表，本地包或转发包
1. 如果是本地包调用 netfilter (`LOCAL_IN`)
1. 调用L4协议(例如`tcp_v4_rcv`)
1. 找到正确的socket
1. 进入tcp有限状态机
1. 将数据包按“tcp_rmem”规则插入到接收缓冲区
1. 如果“tcp_mid_rcvbuf”被启用内核将自动调整接收缓冲区
1. 内核使用信号通知有数据可供应用程序读取(epoll其他poll模型)
1. 唤醒应用读取数据

## Egress - 他们又走了
1. 应用发送数据 (`sendmsg` 或其他api)
1. TCP 层开辟 skb_buff 内存
1. 写入socket `write buff`(占用 `tcp_wmem` 大小)
1. 创建TCP头 (src and dst port, checksum)
1. 调用三层实现 (in this case `ipv4` on `tcp_write_xmit` and `tcp_transmit_skb`)
1. 三层 (`ip_queue_xmit`) 开始干活: 创建IP头，调用netfilter (`LOCAL_OUT`)
1. 调用输出路由
1. 调用 netfilter (`POST_ROUTING`)
1. 分包 (`ip_output`)
1. 调用二层函数 (`dev_queue_xmit`)
1. 填充输出队列，队列长度`txqueuelen`根据`default_qdisc`默认算法计算
1. 驱动代码将把加入`ring buffer tx`
1. 在 `tx-usecs` 超时 或 `tx-frames`后，驱动触发一个软中断
1. 重新启用到 NIC 的硬IRQ
1. 驱动程序会将所有数据包（要发送）映射到某个 DMA 区域
1. NIC 从 RAM 中获取数据包（通过 DMA）进行传输
1. 传输完成后 NIC 将触发一个“hard IRQ”，表示其完成
1. 驱动处理这次 IRQ，关闭它
1. 并调度（`soft IRQ`）NAPI 轮询系统
1. NAPI 将处理接收数据包信号并释放 RAM

## 怎么检查 - perf

如果你想在 Linux 中查看网络跟踪，你可以使用 [perf](https://man7.org/linux/man-pages/man1/perf-trace.1.html).

```
docker run -it --rm --cap-add SYS_ADMIN --entrypoint bash ljishen/perf
apt-get update
apt-get install iputils-ping

# 这将在执行 ping 时将所有事件（不是系统调用）跟踪到子系统 net:*
perf trace --no-syscalls --event 'net:*' ping globo.com -c1 > /dev/null
```
![perf trace network](https://user-images.githubusercontent.com/55913/147019725-69624e67-b3ca-48b4-a823-10521d2bed83.png)


# What, Why and How - 网络和sysctl参数

## Ring Buffer - rx,tx
* **What** - 驱动程序接收/发送队列包含一个或多个具有固定大小的列表，队列通常在内存中，实现方式是队列。
* **Why** - 缓冲区作用平滑地接受突发请求，避免丢掉他们，当发现缓冲区调包或溢出，当您看到丢弃或溢出，也就是进来的数据包比内核能够消耗它们多时，您可能需要增加这些队列，这个操作的副作用可能是延迟增加。
* **How:**
  * **查看:** `ethtool -g eth0`
  * **修改:** `ethtool -G eth0 rx value tx value`
  * **监控:** `ethtool -S eth0 | grep -e "err" -e "drop" -e "over" -e "miss" -e "timeout" -e "reset" -e "restar" -e "collis" -e "over" | grep -v "\: 0"`
 
## 中断合并 (IC) - rx-usecs, tx-usecs, rx-frames, tx-frames (hardware IRQ)
* **What** - 在提高 hard IRQ之前等待的微秒/帧数，从NIC的角度来看，它将DMA数据包直到超时/凑齐帧数
* **Why** - 减少 CPU 使用率、hard IRQ 可能会以延迟为代价增加吞吐量。
* **How:**
  * **查看:** `ethtool -c eth0`
  * **修改:** `ethtool -C eth0 rx-usecs value tx-usecs value`
  * **监控:** `cat /proc/interrupts` 
  
## 中断合并 (soft IRQ) and Ingress QDisc
* **What** - 一个 [NAPI](https://en.wikipedia.org/wiki/New_API) 轮询周期中的最大微秒数。当轮询周期中的`netdev_budget_usecs` 结束或处理的数据包数量达到`netdev_budget`时，轮询将退出。
* **Why** -驱动程序没有对大量的 soft IRQ 做出反应，而是不断地轮询数据；重点关注 `dropped`（因为超过 `netdev_max_backlog` 而被丢弃的数据包数）和 `squeezed`（ksoftirq 用完 `netdev_budget` 的次数或剩余工作的时间片）。
* **How:**
  * **查看:** `sysctl net.core.netdev_budget_usecs`
  * **修改:** `sysctl -w net.core.netdev_budget_usecs value`
  * **监控:** `cat /proc/net/softnet_stat`; 或 [better tool](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)

* **What** -* `netdev_budget` 是轮询周期（NAPI 轮询）中从所有网卡获取的最大数据包数。轮询周期内，注册到轮询的接口以循环方式进行探测。此外，轮询周期不得超过 `netdev_budget_usecs` ，即使 `netdev_budget` 尚未用尽。
* **How:**
  * **查看:** `sysctl net.core.netdev_budget`
  * **修改:** `sysctl -w net.core.netdev_budget value`
  * **监控:** `cat /proc/net/softnet_stat`; 或 [better tool](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)

* **What** -`dev_weight` 是内核在 NAPI 中断上可以处理的最大数据包数，它是一个 Per-CPU 变量。对于支持 LRO 或 GRO_HW 的驱动程序，硬件聚合数据包在此计为一个数据包。
* **How:**
  * **查看:** `sysctl net.core.dev_weight`
  * **修改:** `sysctl -w net.core.dev_weight value`
  * **监控:** `cat /proc/net/softnet_stat`; 或 [better tool](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)

* **What** - `netdev_max_backlog` 是当接口接收数据包的速度超过内核处理它们的速度时 ，在输入端排队的最大数据包数（_the ingress qdisc_），
* **How:**
  * **查看:** `sysctl net.core.netdev_max_backlog`
  * **修改:** `sysctl -w net.core.netdev_max_backlog value`
  * **监控:** `cat /proc/net/softnet_stat`; 或 [better tool](https://raw.githubusercontent.com/majek/dump/master/how-to-receive-a-packet/softnet.sh)
  
## Egress QDisc - txqueuelen and default_qdisc
* **What** - `txqueuelen` 是在 输出 端排队的最大数据包数。
* **Why** - 同样适用于一个缓冲区/队列的突发连接 [tc (traffic control).](http://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html)
* **How:**
  * **查看:** `ifconfig eth0`
  * **修改:** `ifconfig eth0 txqueuelen value`
  * **监控:** `ip -s link` 
* **What** - `default_qdisc` 是网络设备的默认排队规则。
* **Why** - 每个应用程序都有不同的负载和流量控制方法，它也用于对抗 [bufferbloat](https://www.bufferbloat.net/projects/codel/wiki/)
* **How:**
  * **查看:** `sysctl net.core.default_qdisc`
  * **修改:** `sysctl -w net.core.default_qdisc value`
  * **监控:** `tc -s qdisc ls dev eth0`

## TCP 读写缓冲区/队列

> 定义什么是 [内存压力](https://wwwx.cs.unc.edu/~sparkst/howto/network_tuning.php) 的策略在 tcp_mem 和 tcp_moderate_rcvbuf 中指定。

* **What** - `tcp_rmem` - min (压力下的最小值), default (初始值), max (最大值) - TCP用于接受数据的buffer大小。
* **Why** - 应用程序写入数据的buffer大小, [understand its consequences can help a lot](https://blog.cloudflare.com/the-story-of-one-latency-spike/).
* **How:**
  * **查看:** `sysctl net.ipv4.tcp_rmem`
  * **修改:** `sysctl -w net.ipv4.tcp_rmem="min default max"`; when changing default value, remember to restart your user space app (i.e. your web server, nginx, etc)
  * **监控:** `cat /proc/net/sockstat`

* **What** - `tcp_wmem` - min (内存压力小的最小值), default (初始值), max 最大值) -TCP用于发送数据的buffer大小。
* **How:**
  * **查看:** `sysctl net.ipv4.tcp_wmem`
  * **修改:** `sysctl -w net.ipv4.tcp_wmem="min default max"`; when changing default value, remember to restart your user space app (i.e. your web server, nginx, etc)
  * **监控:** `cat /proc/net/sockstat`

* **What** `tcp_moderate_rcvbuf` - 如果设置，TCP 尝试自行调整接收缓冲区大小。
* **How:**
  * **查看:** `sysctl net.ipv4.tcp_moderate_rcvbuf`
  * **修改:** `sysctl -w net.ipv4.tcp_moderate_rcvbuf value`
  * **监控:** `cat /proc/net/sockstat`

## 进阶 - TCP FSM 和拥塞算法

> Accept 和 SYN 队列由 net.core.somaxconn 和 net.ipv4.tcp_max_syn_backlog 管理。[Nowadays net.core.somaxconn caps both queue sizes.](https://blog.cloudflare.com/syn-packet-handling-in-the-wild/#queuesizelimits)

* `sysctl net.core.somaxconn` - 传递 backlog 参数值上限[`listen()` function](https://eklitzke.org/how-tcp-sockets-work), 在用户空间中称为“SOMAXCONN”。 如果您更改此值，您还应该将您的应用程序更改为兼容的值(i.e. [nginx backlog](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen)).
* `cat /proc/sys/net/ipv4/tcp_fin_timeout` - 在套接字被强制关闭之前等待最终 FIN 数据包的秒数。 这严重违反了 TCP 规范，这个操作会让服务保留拒绝服务攻击中。
* `cat /proc/sys/net/ipv4/tcp_available_congestion_control` - 显示已注册可用拥塞控制选项。
* `cat /proc/sys/net/ipv4/tcp_congestion_control` -设置用于新的拥塞控制算法。
* `cat /proc/sys/net/ipv4/tcp_max_syn_backlog` -设置尚链接确认请求，排队连接请求的最大数； 如果超过这个数，内核将开始决绝请求。
* `cat /proc/sys/net/ipv4/tcp_syncookies` - 启用/关闭 [syn cookies](https://en.wikipedia.org/wiki/SYN_cookies), 可用于免受攻击 [syn flood attacks](https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/).
* `cat /proc/sys/net/ipv4/tcp_slow_start_after_idle` -启用/关闭 tcp 慢启动.

**监控:** 
* `netstat -atn | awk '/tcp/ {print $6}' | sort | uniq -c` - 统计状态
* `ss -neopt state time-wait | wc -l` - 统计指定状态的连接数: `established`, `syn-sent`, `syn-recv`, `fin-wait-1`, `fin-wait-2`, `time-wait`, `closed`, `close-wait`, `last-ack`, `listening`, `closing`
* `netstat -st` - tcp 统计状态
* `nstat -a` - 人性化 tcp 统计状态
* `cat /proc/net/sockstat` - socket 统计状态
* `cat /proc/net/tcp` - 详细的统计信息，请参阅每个字段的含义 [kernel docs](https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt)
* `cat /proc/net/netstat` - `ListenOverflows` 和 `ListenDrops` 是值得关注
  * `cat /proc/net/netstat | awk '(f==0) { i=1; while ( i<=NF) {n[i] = $i; i++ }; f=1; next} \
(f==1){ i=2; while ( i<=NF){ printf "%s = %d\n", n[i], $i; i++}; f=0} ' | grep -v "= 0`;  [肉眼可读] `/proc/net/netstat`](https://sa-chernomor.livejournal.com/9858.html)

![tcp finite state machine](https://upload.wikimedia.org/wikipedia/commons/a/a2/Tcp_state_diagram_fixed.svg "图形化的 tcp 状态机")
Source: https://commons.wikimedia.org/wiki/File:Tcp_state_diagram_fixed_new.svg

# 用于测试和监控的网络工具

* [iperf3](https://iperf.fr/) - 网络吞吐量
* [vegeta](https://github.com/tsenart/vegeta) - HTTP 压测工具
* [netdata](https://github.com/firehol/netdata) - 实时分布式系统性能和健康监控

# 参考文献

* https://www.kernel.org/doc/Documentation/sysctl/net.txt
* https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
* https://www.kernel.org/doc/Documentation/networking/scaling.txt
* https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt
* https://www.kernel.org/doc/Documentation/networking/multiqueue.txt
* http://man7.org/linux/man-pages/man7/tcp.7.html
* http://man7.org/linux/man-pages/man8/tc.8.html
* http://www.ece.virginia.edu/cheetah/documents/papers/TCPlinux.pdf
* https://netdevconf.org/1.2/papers/bbr-netdev-1.2.new.new.pdf
* https://blog.cloudflare.com/how-to-receive-a-million-packets/
* https://blog.cloudflare.com/how-to-achieve-low-latency/
* https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/
* https://www.youtube.com/watch?v=6Fl1rsxk4JQ
* https://oxnz.github.io/2016/05/03/performance-tuning-networking/
* https://www.intel.com/content/dam/www/public/us/en/documents/reference-guides/xl710-x710-performance-tuning-linux-guide.pdf
* https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf
* https://medium.com/@matteocroce/linux-and-freebsd-networking-cbadcdb15ddd
* https://blogs.technet.microsoft.com/networking/2009/08/12/where-do-resets-come-from-no-the-stork-does-not-bring-them/
* https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/multi-core-processor-based-linux-paper.pdf
* http://syuu.dokukino.com/2013/05/linux-kernel-features-for-high-speed.html
* https://www.bufferbloat.net/projects/codel/wiki/Best_practices_for_benchmarking_Codel_and_FQ_Codel/
* https://software.intel.com/en-us/articles/setting-up-intel-ethernet-flow-director
* https://courses.engr.illinois.edu/cs423/sp2014/Lectures/LinuxDriver.pdf
* https://www.coverfire.com/articles/queueing-in-the-linux-network-stack/
* http://vger.kernel.org/~davem/skb.html
* https://www.missoulapubliclibrary.org/ftp/LinuxJournal/LJ13-07.pdf
* https://opensourceforu.com/2016/10/network-performance-monitoring/
* https://www.yumpu.com/en/document/view/55400902/an-adventure-of-analysis-and-optimisation-of-the-linux-networking-stack
* https://lwn.net/Articles/616241/
* https://medium.com/@duhroach/tools-to-profile-networking-performance-3141870d5233
* https://www.lmax.com/blog/staff-blogs/2016/05/06/navigating-linux-kernel-network-stack-receive-path/
* https://es.net/host-tuning/100g-tuning/
* http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm
* http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html
* https://people.cs.clemson.edu/~westall/853/tcpperf.pdf
* http://tldp.org/HOWTO/Traffic-Control-HOWTO/classless-qdiscs.html
* https://es.net/assets/Papers-and-Publications/100G-Tuning-TechEx2016.tierney.pdf
* https://www.kernel.org/doc/ols/2009/ols2009-pages-169-184.pdf
* https://devcentral.f5.com/articles/the-send-buffer-in-depth-21845
* http://packetbomb.com/understanding-throughput-and-tcp-windows/
* https://www.speedguide.net/bdp.php
* https://www.switch.ch/network/tools/tcp_throughput/
* https://www.ibm.com/support/knowledgecenter/en/SSQPD3_2.6.0/com.ibm.wllm.doc/usingethtoolrates.html
* https://blog.tsunanet.net/2011/03/out-of-socket-memory.html
* https://unix.stackexchange.com/questions/12985/how-to-check-rx-ring-max-backlog-and-max-syn-backlog-size
* https://serverfault.com/questions/498245/how-to-reduce-number-of-time-wait-processes
* https://unix.stackexchange.com/questions/419518/how-to-tell-how-much-memory-tcp-buffers-are-actually-using
* https://eklitzke.org/how-tcp-sockets-work
* https://www.linux.com/learn/intro-to-linux/2017/7/introduction-ss-command
* https://staaldraad.github.io/2017/12/20/netstat-without-netstat/
* https://loicpefferkorn.net/2016/03/linux-network-metrics-why-you-should-use-nstat-instead-of-netstat/
* http://assimilationsystems.com/2015/12/29/bufferbloat-network-best-practice/
* https://wwwx.cs.unc.edu/~sparkst/howto/network_tuning.php