# 21-mpc
Source: A Pragmatic Introduction to Secure Multi-Party Computation

这是一本关于安全多方计算的书，决心从头学一下 secure computation 相关知识。



> 提出一种新的 secure protocol 必须要进行 formal security analysis

* 之所以很多基于 MPC 和 HE 的方法没有做这件事，原因在于它们并没有提出新的安全协议，只是在现有协议之下给出了具体的实现，正如 BOLT, S&P 的作者所言："As we do not propose new protocols, but combine them in novel ways, the security of BOLT follows naturally from the security of these building blocks."
* 根据本书的分类，现有的 MPC 协议包括：Yao’s Garbled Circuits Protocol, Goldreich-Micali-Wigderson (GMW) Protocol, BGW protocol, MPC From Preprocessed Multiplication Triples, Constant-Round Multi-Party Computation: BMR, Information-Theoretic Garbled Circuits, Oblivious Transfer
* 如何进行新的 formal security analysis？我找到了一篇文章 Formal Security Analysis of Neural Networks using Symbolic Intervals, Security 2018. 其并非研究推理协议，而是尝试为神经网络对给定的输入范围提供可证明的安全性分析。具体而言，他们用到了一种称为 interval arithmetic 的数学工具，来证明 adversarial examples 的不存在性。
  * 即使是这种工作，实际上都是有很多现有工作的。之前的方法基于 Satisfiability Modulo Theory，局限性在于 solver 的高开销。
  * 所以我在想可能 STIP 理应能找到相符合的现有的理论协议框架。
    * MPC 中的确存在不同的 roles：data provider, function owner, computational node
    * 我想应该可以找到相关的理论工作。



> 当参与方不遵守 threat model 的假设（例如 semi-honest）时的安全问题，称为 malicious security

* 原本 STIP 中讨论什么 social engineering attacks 实际上就是 malicious security 问题，但当时我不知道这个概念
