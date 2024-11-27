# 27-tee
Source: Trusted Execution Environment: What it is and what it is not

这个文章尝试给 TEE 一个准确的、一致的定义，并分析其核心的性质。



文章首先分析了现有工作中对于类似 TEE 的描述存在不一致性。为了给出一个明确定义，先介绍了 separation kernel 的概念：

​	将系统分出隔离的 partitions，并控制 inter-partition communication，主要有四个安全 polices：

* data (spatial) separation：一个 partition 里的数据，不能够被其他 partitions 进行 read 和 write
* sanitization (temporal separation)：共享的 resources 不会泄露信息
* Control of information flow：partitions 之间除非允许，不会进行 communication 
* Fault isolation：一个 partition 的安全 breach 不会扩散到其他 partitions

文章给出 TEE 的定义：

> Trusted Execution Environment (TEE) is a tamper-resistant processing environment that runs on a separation kernel.

即一个运行在 separation kernel 里的、可以防篡改的计算环境。

要求所有在 TEE 里的 code 都是 trusted（但是没有说明对哪一方是可信的？）

文章强调，主要问题在于 trust 是一个主观概念，是 non-measurable 的

> In the real world, an entity is trusted if it has behaved and/will behave as expected.

按照这个说法，是否可以将其视为 semi-honest party？然后我们考虑 cloud 和 user 是 malicious 的。



![image-20241127191251493](/Users/yuanmu/Desktop/tech-in-papers/2024/11/image-20241127191251493.png)

我的感觉是 TEE 似乎是保护 application 免受其他运行在 main memory 上其他程序攻击的一个硬件。其内部运行的代码可以验证，且数据对于 main memory 而言是隔离的，以此保护安全性。

* TEE 与 normal OS 之间的通信称为 inter-environment communication
* secure storage: 只有 authorized entities 可以 access data
  * 那么，我认为 cloud 自己肯定有办法是 authorized
  * 然后用户是否 authorized 由 cloud 管理；那么一旦用户 authorized，其 storage 内容是否就算是暴露给用户了？
* trusted io path: 指的是 TEE 可以直接和外设通信，比如键盘等 UI 设备。
  * 现在也可以实现远程设备直接和 TEE 通信：用户和TEE之间采用标准的密钥协商协议，可以防止中间人攻击。



> For the best of our knowledge, there is no TEE that is formally veriﬁed.

这个结论我觉得有点用，TEE 安全性不完备。



应用方面，大多数 TEE 都是为智能手机应用设计的，用于在线支付等安全性要求比较高的应用，防止这些敏感应用被其他应用、超级用户和管理员等能够访问 main memory 的恶意攻击。
