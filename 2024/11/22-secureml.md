# 22-secureml
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



## Protocols

### 1. Linear Regression

准备阶段，client 将训练数据 SS 到两个 servers 上，权重也是 SS 在两个 servers 之间，可以直接随机而不需要任何通信。

训练阶段，由于权重的更新：
$$
w_j := w_j - \alpha (\sum_{k=1}^d x_{ik}w_k - y_i)x_{ij}
$$
仅包含乘法和加法运算，可以直接用上面介绍的 multiplication triplet 方法，两个 servers 在本地完成计算后，其权重的 shares 仍满足： $\langle w\rangle_0+\langle w\rangle_1 = w$ 性质。

接着文章讨论了矩阵情况下以及 mini-batch 下的具体操作，总的来说，还是 multiplication triplet 思路，让整体协议支持并行，提高效率。



上述的性质都是在 real number 情况下的，而真实实现的时候需要用二进制表示。文章在使用 fixed-point 表示的情况下，使用了一种 truncation 的操作：

* 先将 x 和 y 乘 $2^{l_D}$ （其中 $l_D$ 是小数部分的位长），该操作实际上等价于将整体左移 $l_D$​ ，即将全部小数部分都移动到整数部分，实现了对 x 和 y 的整数转化操作，得到 $x',y'$ ；
* 然后计算 $z=x'y'$ 并将其分解为： $z=z_1 2^{l_D} + z_2$ 
* 之后截取出 $z_1$ 部分

正确性分析：

*  $z=x'y'=(xy2^{l_D}) 2^{l_D}$​
* 因此，当 x 和 y 的 fixed-point 表示没有误差时，$z_1=xy 2^{l_D}$ 即为 xy 乘积的左移 $l_D$ 结果；如果 xy 乘积无法用 $l_D$ 位准确表达小数部分，则会产生误差，即 $z_2$ 部分

具体例子，给定 $l_D=2$ ：
$$
x=1.25=1.01_2, ~~y=2.25=10.01_2, ~~xy=2.8125=10.1101_2\\
x'= 101_2=5, y'=1001_2=9,z=x'y'=45=101101_2=1011_2*2^{l_D}+01_2\\
z_1 = 1011_2=10.11_2*2^{l_D} = 2.75 *2^{l_D}, z_2=01_2
$$
可见结果 2.75 与准确结果 2.8125 存在误差，即 $z_2$ 所表示的 $0.0001_2$ 的误差。

关于这个误差，文章证明了截断后与真实结果偏差，在很大概率下不超过 1：

<img src="/Users/yuanmu/Desktop/tech-in-papers/2024/11/image-20241124161413251.png" alt="image-20241124161413251" style="zoom:50%;" />

文章证明了协议的 security，思路是描述 ideal world 情况下的 adversary，然后声明其安全性 follows from the security of arithmetic secret sharing，所以我感觉实际上还是只是把 SS 应用过来，并没有证明什么别的。

* 我大概明白了，这类工作都是用 HE、OT、SS 等作为 building blocks 构建自己的协议，然后安全性自然可以基于这些 primitives 的安全性推出



离线阶段：

注意到上面 u 和 v 很容易在两个 servers 之间进行 SS，但是 z=uv 如何进行 SS？文章分别使用 linearly HE 和 OT 协议提出两种方法实现：

* 该问题形式化为：给定 shares A 和 B，想要计算 shares C = A x B
* 文章利用一个简单性质： $C=A_0 \times B_0 + A_0 \times B_1 + A_1 \times B_0 + A_1 \times B_1.$ 则只需要计算 $A_0 \times B_1$ 和 $A_1 \times B_0$ 即可，其他两项可以本地完成计算。
* LHE 方法具体流程，以 A0 x B1 为例：
  * S1 将 B1 加密后发给 S0
  * S0 在密文上进行矩阵乘法（+替换为x，x替换为exp）
  * S0 用随机值对密文计算结果进行 masking（相当于构造 shares），发给 S1
  * S1 解密，得到其 shares
* OT 方法具体流程，以 A0 x B1 为例，先计算 aij x bj ：
  * S1 根据 bj 每一位的值来从 S0 进行二选一
  * 当 bj[k]=0 时，S1 得到 rk
  * 当 bj[k]=1 时，S1 得到 $a_{ij} 2^k + r_k \mod 2^l$ 
  * 最后，S1 设定 $\langle a_{ij} b_j \rangle_1 = \sum_{k=1}^l (b_j[k] a_{ij} 2^k +r_k)=a_{ij}b_j + \sum_{k=1}^l r_k \mod 2^l$ 
  * S0 设定 $\langle a_{ij} b_j \rangle_0 = \sum_{k=1}^l (-r_k) \mod 2^l$ 
  * 显然，符合 additive SS 的要求

文章后面还讨论了一些优化 OT 效率的细节。实验表明，**OT 整体运行时间比 LHE 更快**。



### 2. Logistic Regression

把非线性 logistic 激活，改成了一个 MPC-friendly 的复合函数：

<img src="/Users/yuanmu/Desktop/tech-in-papers/2024/11/image-20241125145140570.png" alt="image-20241125145140570" style="zoom:70%;" />

这种改动只能适用于训练新模型，已经完成训练的模型不能改激活。

文章只是通过实验说明，这样改动之后训练出来的模型精度没有明显损失。



具体协议：

* 简单地应用 Yao's garbled circuit protocol 来计算完整的 logistic regression 很低效
* 所以文章对于前面的 xW 部分还是用 arithmetic sharing 来算，只对最后的激活用 Yao sharing 的 garbled circuit 来计算，之后再切换回 arithmetic sharing 来完成后向传播



其安全性证明：

> The proof is omitted due to lack of space but we note that it is implied by the security of the secret sharing scheme, the garbling scheme, and OT.

一样，还是依赖 ss、garbling、OT 这些 primitives 的安全性。



### 3. Neural Networks

对于 ReLU，类似于文章提出的新的激活，同样采用切换到 Yao sharing 的方式进行计算；

对于 softmax，同样是做了替换，把分子换成用 ReLU 计算，然后计算全部 ReLU 输出的和，最后用一个 division garbled circuit 计算除法。



### client-aided offline protocol

如果让 client 对 multiplication triplet 进行构造和分发，能够避免高开销的 OT 和 LHE 操作；

但同时，也会改变 threat model：

* 如果 server 还能和 client 共谋，则有可能恢复出 coefficient vector，导致信息泄露
* 因此，文章改为不允许 server 和 client 之间共谋。

然后，文章声明了协议在这个新的 threat model 下是安全的。



## 总结

整体来说，文章是用 additive arithmetic SS 构建了在两个互补共谋的 servers 上进行安全模型训练的协议，然后对非线性激活，设计了专门的替换函数，并基于 garbled circuit 进行计算。安全性分析方面，核心还是依赖其 building blocks：SS, LHE, OT, garbled circuit 这些 primitives 的安全性上，没有什么新的证明过程。
