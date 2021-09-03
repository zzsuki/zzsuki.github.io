---
title: 软件授权
date: 2021-07-02 11:05:40
tags: [授权]
---
# Suricata #

## 关于安全产品 ##

- IDS类  
Intrusion Detection System的意思，即入侵检测系统，按照预设的安全策略，通过软硬件，对网络和系统的运行情况进行监视，尽可能多的发现各种攻击的企图、行为或结果，以保证网络和系统的机密性、完整性和可用性。IDS本身相当于监视器，主要负责“看到”入侵。实际部署中由于只负责检测，所以采用旁路部署为佳，这样不会阻断网络，主要负责报告和事后回溯监督
- IPS类  
Intrusion Prevention System的意思，即入侵防御系统，能够监视网络或网络设备的流量传输行为，在发现问题时能及时的采取预定的措施，中断、调整或隔离不正常或是有危害性的流量传输。实际工作时一般在线模式，即接入网络中，但表现为透明模式，即本身不对业务逻辑产生影响，因此肯定有多个网络端口，可以分析到数据包的内容，核心与IDS类一致，都要定义已知的攻击行为和模式，然后通过匹配的方式触发阻断
- 防火墙类  
主要实现基本的包过滤策略，以前的防火墙大多工作在4层以下，实现策略上大多是关闭所有都通过的策略，只开放允许访问的策略，即造墙后有选择的凿洞
- 主动安全类  
主动安全类的产品有很强的协议针对性，例如WAF专门负责HTTP协议的安全处理，DAF专门负责数据库SQL查询类的安全处理，这类产品基本都触碰到了应用层的内容，而对于不认识的业务访问采取的是强措施，即全部隔离

其实总的来说安全类产品在做的事都是一致的，即先关闭所有的通路，然后开放想开放的。还有一些观点可以参考：

> - 基础防火墙  
> 传统防火墙是典型的主动安全类，关闭后再有选择的开启，但早期的防火墙不能检测到数据包的具体内容级别，因此如果针对高层协议进行攻击，遍很难处理
> - IPS  
> 几乎所有的IPS产品都说可以检查数据包的内容，但有一个问题是，安全的原则反置了，将所有访问开放，然后阻断自己已知的攻击行为，非旁路部署又带来了性能的问题，移位置IPS永远不可能达到全面而细致的安全审计，除非砸钱怼设备或者本身流量就不大，也因为这个原因，大部分的IPS在实际运行中都形同虚设，一般只作为防DDOS的设备存在，因为IPS对未知的手段是无能为力的
> - 主动安全类  
> 工作在协议层，对协议进行彻底的分析和Proxy代理工作模式，同时结合对应用的访问流程进行分析，只通过已知的访问，阻断所有未知访问。

## Suricata原理 ##

Suricata是一个基于规则的入侵检测和防护引擎，利用外部开发的规则集进行网络流量的检测，可以处理多个千兆字节的流量，并提供电子邮件警报系统.

### Suricata主要特点 ###

- 支持从nfqueue中读取流量
- 支持分析离线pcap文件和pcap文件方式存储流量数据
- 支持ipv6
- 支持pcap、af_packet、pfring、硬件卡抓包
- 多线程
- 支持内嵌lua脚本，以实现自定义检测和脚本输出
- 支持IP信用等级
- 支持文件还原
- 兼容snort
- 支持常见数据包解码：IPv4, IPv6, TCP, UDP, SCTP, ICMPv4, ICMPv6, GRE, Ethernet, PPP, PPPoE, Raw, SLL, VLAN, QINQ, MPLS, ERSPAN, VXLAN
- 支持常见应用层协议解码：HTTP, SSL, TLS, SMB, DCERPC, SMTP, FTP, SSH, DNS, Modbus, ENIP/CIP, DNP3, NFS, NTP, DHCP, TFTP, KRB5, IKEv2, SIP, SNMP, RDP

### Suricata 基本架构 ###

**运行模式**  
包括三种运行模式，分别为single，workers，autofp。官方推荐性能最佳的运行模式为workers模式。

- single模式： 只有一个包处理线程，一般开发模式下使用
- workers模式： 多个包处理线程，每个线程包含完整的处理逻辑
- autofp模式： 多个包捕获线程和多个包处理线程，一般适用于nfqueue场景，从多个queue中消费流量来处理

**四种线程模块**  
包捕获：负责捕获数据包  
解码：对数据包和应用层协议解码  
检测：通过规则或者自定义脚本对数据包进行检测  
输出：输出检测结果和常规协议的日志  

---

### Suricata性能调优 ###

#### 抓包性能对比 ####

硬件捕获>pfring zc>pfring>af-packet>pcap  

#### 调优 ####

1. 关闭网卡多队列功能  
原因：一般使用流量镜像方式把流量镜像到服务器网卡，如果多队列的话，同一个tcp连接的数据有可能会被分散到不同的队列，由于时间的延迟可能导致有乱序可能。例如先收到了syn/ack，再收到syn，suricata会认为此流量无效而丢弃。如果做检测，则需要加缓冲和排序，代价较大
2. 关闭网卡lro，grp特性  
原因：lro/gro导致将各种较小的包合并成大的“超级包”，从而破坏suricata对tcp连接的跟踪  
3. 使用pfring zc模式捕获包  
原因：pfring+zero copy提升性能，但是zero copy需要网卡驱动支持；使用pfring模式抓包，只需要kernel支持即可。
4. 调整配置文件中内存相关配置，调大flow.memcap,stream.memcap,stream.reassembly.memcap
5. 使用workers模式运行
6. 调整配置文件中max-pending-packets为8192
7. suricata编译需要支持luajit（用于替换原始lua），Hyperscan高性能正则库，PF_RING高性能包捕获库

#### Suricata 规则 ####

1. 兼容snort规则
2. 通过规则和内置的关键字实现对数据包的过滤和处理
3. Suricata4.x之后有自带的规则管理工具

#### Suricata 自定义检测 ####

支持通过lua脚本对数据包进行自定义检测，例如识别协议和异常流量

#### Suricata http log自定义输出 ####

支持通过lua脚本获取http协议request和response的相关信息，从而可以输出http协议中的所有数据，如header，request body，response body等

#### Suricata 单进程同时监听两个网口 ####

通过修改suricata.yml文件实现，以pfring捕获方式为例，如下配置可同时捕获两个网口的流量配置：

```shell
pfring:
  - interface: em2
    threads: auto
    cluster-id: 81
    cluster-type: cluster_flow
  - interface: em4
    threads: auto
    cluster-id: 82
    cluster-type: cluster_flow
```

## 结果 ##

Suricata分析流量后，输出一般包括检测出的告警事件和解码后的所有应用协议的日志
