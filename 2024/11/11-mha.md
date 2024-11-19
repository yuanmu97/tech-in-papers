# 11-mha
MHA: mult-head attention



> 相同的权重和输入，使用不同数量的 heads 时，计算结果不同。

这个事实我直到自己代码遇到 bug 才意识到，我之前一直认为多头机制只是为了提高效率和训练效果。

下面用一个具体的例子来说明：

当隐层维度为2，用一个头时：
$$
QK^T = \begin{pmatrix} q_1 & q_2 \\ q_3 & q_4 \end{pmatrix} \begin{pmatrix} k_1 & k_2 \\ k_3 & k_4 \end{pmatrix}^T  =\begin{pmatrix} q_1k_1+q_2k_2 & q_1k_3+q_2k_4 \\ q_3k_1+q_4k_2 & q_3k_3+q_4k_4 \end{pmatrix} \\

QK^TV=\begin{pmatrix} q_1k_1+q_2k_2 & q_1k_3+q_2k_4 \\ q_3k_1+q_4k_2 & q_3k_3+q_4k_4 \end{pmatrix}\begin{pmatrix} v_1 & v_2 \\ v_3 & v_4 \end{pmatrix}= \begin{pmatrix} (q_1k_1+q_2k_2)v_1+  (q_1k_3+q_2k_4)v_3& /\\ /& / \end{pmatrix}
$$
而用两个头时：
$$
Q_1K_1^T V_1= \begin{pmatrix} q_1 \\ q_3 \end{pmatrix} \begin{pmatrix} k_1 \\ k_3 \end{pmatrix}^T \begin{pmatrix} v_1 \\ v_3 \end{pmatrix}=\begin{pmatrix} q_1k_1 & q_1k_3 \\ q_3k_1 & q_3k_3 \end{pmatrix}\begin{pmatrix} v_1 \\ v_3 \end{pmatrix}=\begin{pmatrix} q_1k_1v_1+q_1k_3v_3 \\ / \end{pmatrix}\\
Q_2K_2^TV_2 = \begin{pmatrix} q_2 \\ q_4 \end{pmatrix} \begin{pmatrix} k_2 \\ k_4 \end{pmatrix}^T\begin{pmatrix} v_2 \\ v_4 \end{pmatrix}=\begin{pmatrix} q_2k_2 & q_2k_4 \\ q_4k_2 & q_4k_4 \end{pmatrix}\begin{pmatrix} v_2 \\ v_4 \end{pmatrix}=\begin{pmatrix} q_2k_2v_2+q_2k_4v_4  \\ /  \end{pmatrix}
$$
两个头合并之后：
$$
\begin{pmatrix} q_1k_1v_1+q_1k_3v_3 & q_2k_2v_2+q_2k_4v_4 \\ / & /  \end{pmatrix}
$$
可见，即使不考虑 softmax 对于值的影响，仅仅是 QKV 矩阵运算，当拆分为两个头时，也会导致计算结果完全不同。
