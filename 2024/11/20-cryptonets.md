# 20-cryptonets
Source: CryptoNets: Applying Neural Networks to Encrypted Data with High Throughput and Accuracy, ICML 2016

将同态加密应用于神经网络推理的早期工作。

本文使用的是 Bos et al. 2013. 的 HE 方案，是一个 leveled HE 方案，支持加法和乘法操作，但要求事先知道应用于数据的算术电路的复杂性（即加密数据上计算的多项式函数的最大度是有限的）。

该加密方案不支持浮点数，本文使用 scaling 的方式来转换为整数。



max-pooling 是 non-polynomial 的，不能直接计算，本文用 $\sum x_i$ 这个 scaled mean-pooling 替代；

sigmoid 和 ReLU 激活也是非多项式的，本文用 $z^2$​ 这个最低度的非线性多项式函数替代。

* 训练阶段最后分类层还是需要 sigmoid 函数的，因为要去计算 loss；但是由于其时单调的，预测阶段只需要最大值 index，sigmoid 直接拆掉就可以



数据加密：
$$
c = [\lfloor q/t \rfloor m+e+hs]_q
$$
其中 $q, e, s$ 是随机的 polynomials， $h=tgf^{-1}, f=tf'+1$ 其中 $f'$ 也是随机的 polynomials， $[a]_q = a \mod q$ 

我们将 $h, f$ 分别作为 public 和 secret keys，t 由明文的 ring 决定

数据解密：
$$
m=[\lfloor \frac{t}{q} fc \rceil]_t
$$
其中 $\lfloor \cdot \rceil$ 是“向最临近取整”操作。



计算是在云端进行的，其并不需要对参数 w 做完整的加密操作，即可完成加法和乘法运算：
$$
Enc(w) = \lfloor q/t \rfloor w,~~~~~c+w=\lfloor q/t \rfloor (m+w)+e+hs \\
Enc(w) = w,~~~~~cw=\lfloor q/t \rfloor (mw)+e'+hs'.
$$
这里就可以理解前面 intro 里写的：

> 加密算法涉及 mod t 操作，要关注计算过程中数值 size 的增长，确保 reduction modulo t 不会发生

即由于在解密时实际上最后还要做 mod t 操作，如果 c+w、cw 的数值超过了，就会发生所谓“reduction modulo”的情况，导致结果偏差。

这一点实际上是由于神经网络计算和 HE 的 atomic 构造不匹配导致的：神经网络计算涉及的是 real numbers，但 HE 使用的是 ring 里的 polynomials



文章里利用 Chinese Remainder Theorem,CRT 做了两个优化：1. 编码较大数值（将一个大数分解成多个小数，这些小数并行处理，并在最后连接在一起） 2. SIMD 并行计算（取多个标量连接在一起，形成单个多项式。这个多项式被作为一个单元处理，只有在完成计算后，它才会被分解成其组成部分）

