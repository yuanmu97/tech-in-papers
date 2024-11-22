# 22-secureml-p1
Source: SecureML: A System for Scalable Privacy-Preserving Machine Learning, S&P 2017

第一个使用 MPC 实现 NNs 训练的工作。

三方参与：

* data owner，提供数据
* Server1，server2，互不共谋，使用 2PC 共同完成训练

在 vectorization 部分，有用到 linearly HE 和 oblivous transfer 



> Fixed-point arithmetic in secure MPC (such as garbled circuits)

fixed-point 是指用固定位数表达整数部分和小数部分的表达形式，例如对于 3.25 而言，4-bit 整数 4-bit 小数的 fixed-point 表达方法下为 0011.0100

与之相对的是 float-point 即浮点数，浮点数采用 $value=significand \times 2^{exponent}$ 的方式表示



关于 privacy-preserving neural networks 的相关工作：

* Privacy-preserving deep learning. CCS 2015. 这个工作实际上是 FL 的思路，提出 parameter server，client  上传梯度到云。本文的评价是：很高效（因为没有 cryptographic operation），但是梯度泄露的信息并没有很深入地研究，而且没有 formal security guarantee
* Cryptonets: Applying neural networks to encrypted data with high throughput and accuracy. ICML 2016. 本文的评价是：使用 FHE 支持 NNs prediction

关于差分隐私在 ML 里的工作，本文的定位是”**orthogonal line of work**“，可以互补。

* 这一点在我们之后的论文写作里要注意，避免对抗式的对比，尽量找到互补的点。



preliminaries 包含 ML 和 secure computation 两块：

* linear regression, SGD, logistic regression, neural networks

* Oblivious transfer：sender 拥有 $x_0, x_1$ ；receiver 选择一个 bit $b$ ；OT 旨在将 $x_b$ 传给 receiver，但不向 sender 暴露 b 且不向 receiver 暴露 $x_{1-b}$​

  * notation 是： $(\bot; x_b) \leftarrow OT(x_0, x_1;b)$
    * 我的理解是括号分号左右分别是 sender 和 receiver，即 $(Sender; Receiver)$​ 
    * OT 协议输入的情况是 sender 拥有 $x_0, x_1$ 而 receiver 拥有 $b$
    * OT 协议输出的情况是 sender 不获得任何信息，即 $\bot$ ，而 receiver 获得 $x_b$

  * 提到了 one-round OT, OT extension, correlated OT extension 等具体实现，本文选择了 correlated OT extension 来优化效率

* garbled circuit 2PC：Alice 有 x；Bob 有 y；他们想一起算 f(x,y)=z；在 NNs inference 背景下，x 是输入数据，y 是模型权重，f(x,y) 就是模型推理计算

  * Alice 运行 garbling 算法，对 x 和 f 加密；y 的加密（即 garbling）依赖 Alice 和 Bob 间对于 y 的每一个 bit 使用 OT，从而让 Bob 得到 garbled y；然后 Bob 用 garbled circuit 在 garbled x 和 garbled y 上做 evaluation 得到 garbled output $\hat{z}$
  * notation 是： $(z_a, z_b) \leftarrow GarbledCircuit(x;y,f)$

* secret sharing and multiplication triplets

  * 本文中涉及的中间结果都通过 SS 在两个 servers 之间共享，具体地，使用了三种方案：

    * additive sharing：很简单，就是把秘密 a 拆成随机数 $a_0$ 和 $a_1=a-a_0$​

      * 表示为： $\langle a\rangle^{A}_0=a_0, \langle a\rangle^{A}_1=a_1$​

      * 让这种 SS 方式支持乘法，需要用到一个称为 Beaver's multiplication triplet 的技术：

        * 目标是计算 $c=ab$ ，那么让两方除了 a、b 以外，再 SS 三个值：random u, random v, z=uv

        * 这样，第 i 方有： $a_i, b_i, u_i, v_i, z_i$

        * 然后两方各自计算： $e_i = a_i - u_i$ 和 $ f_i = b_i - v_i$​

        * 之后各方重建 e 和 f

        * 最后各自计算： $c_i = f a_i + e b_i +z_i -ief$​

          **注意这里论文原本写的是正的 ief，但我推算之后发现应该是负的 ief 才能正确恢复出 c**

      * 正确性验证：

      * $$
        c_0+c_1 &= f(a_0+a_1)+e(b_0+b_1)+z_0+z_1 - ef\\
        &= fa+eb+z-ef \\
        &= (b-v)a+(a-u)b+uv-(a-u)(b-v)\\
        &= ab - av + ab - bu + uv - ab +av + bu - uv\\
        &= ab
        $$

    * boolean sharing

      * 在二进制下使用 additive sharing，并用 XOR 替代 add，AND 替代 multiply

    * Yao sharing

      * 额外生成随机 permutation bit $r_w$ ，将 $r_w, 1-r_w$​  分别拼接到两方的 garbled strings 后



## Security Model

定义为 server-aided setting where clients outsource the computation to two untrusted but non-colluding servers

* server-aided MPC 这个概念我之前看到过，叫做 Outsourcing Multi-Party Computation，本文中也引用了，这个 setting 有一些工作：
  * Privacy-preserving ridge regression on hundreds of millions of records. 将两个 servers 分别视为 evaluator 和 cryptography service provider
  * Secure data exchange: A marketplace in the cloud. 将两个 servers 分别视为 evaluator 和 cloud service provider
  * **感觉和我们 STIP 是相关的，我们相当于也是在 data owner 之外，设计了两个角色，model developer &  model server**

安全性考虑的是 semi-honest adversary，最多 corrupt 一个 server；用 Universal Composition framework 定义什么是 security，我看了一下就是用 ideal-real world 模型定义的。
