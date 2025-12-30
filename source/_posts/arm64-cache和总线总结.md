---
title: arm64 cache和总线总结
author: Joy Xu
date: 2025-12-30 02:57:07
tags: [arm64, CSU, CHI, Cache]
---

# 目的

本文主要作为自己的一个总结，用于记录一些arm64 cache和总线相关的知识。
 
## CSU

Cache System Unit，缓存系统单元。
功能：管理缓存，包括缓存的命中检测、数据访问、缺失处理，以及根据缓存策略进行数据更新和替换。

## CHI

Coherent Hub Interface，一种协议接口
功能：维护系统的缓存一致性，在多个组件（如处理器、缓存、内存控制器等）之间传输数据，确保数据操作的顺序一致性。

CHI 协议通常定义了请求（REQ）、数据（DAT）和响应（RSP）三大通道，实现极致并发+高带宽+不阻塞。

* 请求通道（REQ）：声明我要干什么，操作类型+地址+事务ID(TxnID)，注意REQ不带真正的数据。用于发送读 / 写请求、缓存维护请求和DVM请求等，是发起数据操作的起始通道。例如，CPU 需要从内存读取数据时，就会通过 REQ 通道发送相应的读请求。
* 数据通道（DAT）：真正的数据搬运，会带上DBID。包括写数据通道（WDAT）和读数据通道（RDAT）。WDAT 用于传输写数据、原子数据、监听数据和转发数据等；RDAT 用于接收读数据和原子数据等，负责实际数据的传输。
* 响应通道（RSP）：状态与完成。包括发送响应通道（SRSP）和接收响应通道（CRSP）。SRSP 用于发送监听响应和完成确认等；CRSP 用于接收来自完成者（Completer）的响应，用于反馈请求的处理结果，确保数据操作的完整性和一致性。

```mermaid
sequenceDiagram
	participant Core as CPU Core
	participant CSU as CSU
	participant CHI as CHI 互连
	participant RC as PCIe RC
	participant EP as PCIe EP

	%% 第一阶段：发起写请求并等待就绪
	Core->>CSU: 1. 发起写请求<br/>(REQ, TxnID, BAR_ADDR)
	CSU->>CHI: 2. 非缓存判断→透传
	CHI->>RC: 3. REQ通道路由
	RC->>EP: 4. CHI→PCIe TLP转换
	EP-->>RC: 5. 接收请求→生成就绪响应
	RC-->>CHI: 6. PCIe→CHI RSP转换 (带DBID)
	CHI-->>CSU: 7. RSP通道回传DBID
	CSU-->>Core: 8. 透传RSP响应给Core
	Note over Core: 9. 接收DBID<br/>准备写数据

	%% 第二阶段：发送写数据并确认完成
	Core->>CSU: 10. 发送写数据<br/>(DAT, TxnID, DBID, W_DATA)
	CSU->>CHI: 11. 透传DAT数据到CHI
	CHI->>RC: 12. DAT通道路由到PCIe RC
	RC->>EP: 13. CHI→PCIe TLP转换
	EP-->>RC: 14. 写入MMIO寄存器→生成完成响应
	RC-->>CHI: 15. PCIe→CHI RSP转换 (带TxnID)
	CHI-->>CSU: 16. RSP通道回传完成确认
	CSU-->>Core: 17. 透传RSP完成确认给Core
	Note over Core: 18. 接收确认<br/>释放TxnID/DBID
```

![flash scp sample](/images/arm_server_flash_scp.png)

# 参考

* [Arm Neoverse N2 reference design Technical Overview](https://developer.arm.com/documentation/102337/0000/Software-stack/About-the-software?lang=en)
