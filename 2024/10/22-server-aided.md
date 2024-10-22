# 22-server-aided
Source: Outsourcing Multi-Party Computation



> Server-aided MPC

* 这个工作提出了一种 MPC in a server-aided setting：
  * 参与 MPC 的 parties 能够访问一个 server，其满足：
    * 不拥有任何计算相关的输入
    * 不收到任何计算相关的输出
    * 拥有大量但有限的计算资源

  * 这个 setting 符合我们的场景，即一种 server-aided 2PC 问题
    * 模型开发者、用户为两个参与的 parties，各拥有推理计算 $F(x)=y$ 相关的输入：模型开发者拥有函数 $F$ 而用户拥有数据 $x$​
      * 仅用户应该收到输出 $y$

    * 云服务器属于 server，具有大量且有限的计算资源，其不拥有任何计算相关的输入、且不应该得到计算结果 $y$ 

* 这个工作里设计的 MPC 协议，相较于不使用 server 而言，计算和通信效率显著提高
* 顺着这个逻辑，我们可以这样介绍：
  * 从理论上而言，正如 server-aided MPC 的相关研究发现，当引入一个只用于计算的辅助参与方时，我们有机会设计出计算和通信效率更高的协议。所以，我们研究如何在三方场景下，为 Transformer 推理任务优化计算和通信效率。
  * 也就是说，为我们 “为什么要研究三方场景” 提供一个理论支持。

