# 8-rope
Source: https://github.com/huggingface/transformers/issues/25199

RoPE: rotary positional embedding

考虑处于序列中位置为 i 的特征 $x\in R^{1\times d}$  ，首先将其在特征维度按顺序两两组成一对，例如 $x=[x_1, x_2, x_3, x_4, x_5, x_6]$ 则会产生三对： $[x_1, x_2], [x_3, x_4], [x_5, x_6]$​

对于一对特征 $[x_{2k-1}, x_{2k}], k=\{0, ..., d/2\}$ 其用于变换的 encoding 如下计算：
$$
R(i\theta)=\begin{pmatrix} \cos(i\theta) & -\sin(i\theta) \\ \sin(i\theta) & \cos(i\theta) \end{pmatrix}
$$
其中的 $\theta=\frac{1}{10000^{\frac{2k}{d}}}$​  即：
$$
R(i, k)=\begin{pmatrix} \cos(\frac{i}{10000^{2k/d}}) & -\sin(\frac{i}{10000^{2k/d}}) \\ \sin(\frac{i}{10000^{2k/d}}) & \cos(\frac{i}{10000^{2k/d}}) \end{pmatrix}, k=\{0,1, ..., d/2\}.
$$
然后计算：
$$
\begin{pmatrix} x'_{2k-1} \\ x'_{2k} \end{pmatrix} =  \begin{pmatrix} \cos(\frac{i}{10000^{2k/d}}) & -\sin(\frac{i}{10000^{2k/d}}) \\ \sin(\frac{i}{10000^{2k/d}}) & \cos(\frac{i}{10000^{2k/d}}) \end{pmatrix} \begin{pmatrix} x_{2k-1} \\ x_{2k} \end{pmatrix}
$$
最后将分成对的特征再拼接回去即可。

具体地，对于 $[x_1, x_2]$ 而言：

$x_1' = cos*x_1 - sin*x_2$

$x_2' = sin*x_1 + cos * x_2$



在实现上，`huggingface transformers` 这样实现 LLaMA 的 RoPE 操作：

```python
def rotate_half(x): 
   """Rotates half the hidden dims of the input.""" 
   x1 = x[..., : x.shape[-1] // 2] 
   x2 = x[..., x.shape[-1] // 2 :] 
   return torch.cat((-x2, x1), dim=-1) 


def apply_rotary_pos_emb(q, k, cos, sin, unsqueeze_dim=1):
    """Applies Rotary Position Embedding to the query and key tensors."""
    cos = cos.unsqueeze(unsqueeze_dim)
    sin = sin.unsqueeze(unsqueeze_dim)
    q_embed = (q * cos) + (rotate_half(q) * sin)
    k_embed = (k * cos) + (rotate_half(k) * sin)
    return q_embed, k_embed
```

其计算中用到了一个 `rotate_half` 功能是将 x 切两半，但拼接的时候是拼成 `[-x2,x1]`

计算的时候，上半部结果是： $x_1 * cos - x_2 * sin$ ，下半部的结果是： $x_2*cos + x_1*sin$

可是，这只对于 d=2 的情况下成立，当 d 不是 2 的时候，`rotate_half` 会将整个特征从中间切分，而非按顺序两两为一对地切分，例如，会把`[x1, x2, x3, x4, x5, x6]` 切分为 `[x1, x2, x3], [x4, x5, x6]` 这样再计算 embedding 的时候，会得到： $x_1' = cos*x_1 - sin*x_4$ 这显然是错误的！

那么问题来了，huggingface transformers 的模型又是可以正常使用的，说明 RoPE 的实现并没有问题。原因在哪呢？

https://github.com/huggingface/transformers/issues/25199#issuecomment-1687720247 

这个人的回答帮我弄明白了：

原来，HF transformers 在加载 llama 参数的时候，会对 wq, wk 先进行一个置换操作：

```python
# permute for sliced rotary 
 def permute(w, n_heads=n_heads, dim1=dim, dim2=dim): 
     return w.view(n_heads, dim1 // n_heads // 2, 2, dim2).transpose(1, 2).reshape(dim1, dim2) 
```

直接看代码不一定好理解，我们通过了一个具体例子来看一下这个 permute 到底做了什么：

```python
>>> w = torch.randn(6,6)
>>> w
tensor([[ 0.0351, -1.8382, -0.4659, -0.6392, -1.4064,  2.5892],
        [ 0.1871, -1.6733, -0.1340,  0.1229, -0.0832,  0.8563],
        [-1.4261,  0.1210, -0.7404, -0.7363,  0.2171, -0.5006],
        [ 1.1344,  0.9882,  0.5771,  1.6343, -0.5803, -0.6329],
        [ 0.5153, -0.4251,  0.2446,  0.8374, -1.2831,  0.0325],
        [-0.5279, -0.5472, -0.2414,  0.1889,  1.3524, -0.7277]])
>>> w1 = w.view(1, 3, 2, 6)
>>> w2 = w1.transpose(1,2).reshape(6,6)
>>> w2
tensor([[ 0.0351, -1.8382, -0.4659, -0.6392, -1.4064,  2.5892],
        [-1.4261,  0.1210, -0.7404, -0.7363,  0.2171, -0.5006],
        [ 0.5153, -0.4251,  0.2446,  0.8374, -1.2831,  0.0325],
        [ 0.1871, -1.6733, -0.1340,  0.1229, -0.0832,  0.8563],
        [ 1.1344,  0.9882,  0.5771,  1.6343, -0.5803, -0.6329],
        [-0.5279, -0.5472, -0.2414,  0.1889,  1.3524, -0.7277]])
```

可以看到，这个 permute 操作实际上就是把原本的参数的第 [2k-1, 2k] 行，置换到第 [k, d/2+k] 行去了。那么，我们将得到特征 `[x1, x3, x5, x2, x4, x6]` ，这样的话，再进行 rotate_half 操作之后，我们会切分为 `[x1, x3, x5], [x2, x4, x6]` ，再计算 embedding，会得到： $x_1' = cos*x_1 - sin*x_2$  结果正确！

至此，我们弄明白了 RoPE 的原理和公式，以及 HF 的实现。



但是，Meta 的官方 LLaMA 实现是这样做的：

```python
def apply_rotary_emb(xq: torch.Tensor, xk: torch.Tensor, freqs_cis: torch.Tensor,) -> Tuple[torch.Tensor, torch.Tensor]:
    xq_ = torch.view_as_complex(xq.float().reshape(*xq.shape[:-1], -1, 2))
    xk_ = torch.view_as_complex(xk.float().reshape(*xk.shape[:-1], -1, 2))
    freqs_cis = reshape_for_broadcast(freqs_cis, xq_)
    xq_out = torch.view_as_real(xq_ * freqs_cis).flatten(3)
    xk_out = torch.view_as_real(xk_ * freqs_cis).flatten(3)
    return xq_out.type_as(xq), xk_out.type_as(xk)
```

首先，可以看到用到了 `view_as_complex` 函数，其作用是将输入的最后一维（必须为2）分别视作复数的实部和虚部，具体地，根据 torch 的例子：

```python
>>> x=torch.randn(4, 2)
>>> x
tensor([[ 1.6116, -0.5772],
        [-1.4606, -0.9120],
        [ 0.0786, -1.7497],
        [-0.6561, -1.6623]])
>>>torch.view_as_complex(x)
tensor([(1.6116-0.5772j), (-1.4606-0.9120j), (0.0786-1.7497j), (-0.6561-1.6623j)])
```

接下来解释 `reshape` 到底在做什么：

* 首先，假设 xq.shape 是 (batch_size, seq_len, hidden_dim)
* 那么 `xq.shape[:-1]` 即 (batch_size, seq_len) 这样一个 tuple
* `*` 操作表示将 tuple 展开，那么 reshape 的变量是：`reshape(batch_size, seq_len, -1, 2)`
* 这里的 -1 表达的意思是：不指定这一维的值，具体值可以推理得到。其实是这些库为了方便开发提供的自动化功能，实际上这一维的值就是 `hidden_dim/2`，即将原本 hidden_dim 那一维的值，按顺序分为两两一组

* 给一个具体例子：

* ```python
  >>> x = torch.randn(1,2,4)
  >>> x
  tensor([[[-0.9001,  0.5472, -0.2143,  1.7586],
           [ 0.3527, -2.4083,  0.0380, -2.0296]]])
  >>> x1 = x.reshape(1, 2, -1, 2)
  >>> x1
  tensor([[[[-0.9001,  0.5472],
            [-0.2143,  1.7586]],
  
           [[ 0.3527, -2.4083],
            [ 0.0380, -2.0296]]]])
  >>> x1.shape
  torch.Size([1, 2, 2, 2])
  ```

那么，这个 reshape 操作实际上就是将特征维度按顺序两两组成一对，之后再和每一对特征的encoding `freqs_cis` 相乘。

首先，要知道复数的乘法如何计算。对于 x1=a+bj, x2=c+dj 而言，x1*x2 = (ac-bd) + (ad+bc)j

那么，对于复数特征 $x_1 + x_2 j$ 而言，我们希望得到新的复数 embedding 应该是：

$(x_1 cos - x_2 sin)+(x_1 sin + x_2 cos)j$

即 $c=cos,d=sin$ ，即 freqs_cis = [cos, sin] 的复数时，即可正确完成 embedding 计算。

最后，再将复数形式转化为实数形式，完成 RoPE。



总结一下，Meta 是按照标准的 RoPE 方式来实现，即先将特征维度按顺序两两配对，与 [cos, sin] encoding 进行计算后再恢复为正常结构。只是在实现的时候，使用复数作为中间形式，简化了向量操作。实际上，从 RoFormer 也就是提出 RoPE 的工作来看，其设计的出发点应该就是一种复数的表达，所以使用复数反而是更自然的。只是这确实是我第一次看到在 DL 的实现中使用。
