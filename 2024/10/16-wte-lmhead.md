# 16-wte-lmhead
Source: [Why is the lm_head layer in GPT2LMHeadModel not a parameter?](https://discuss.huggingface.co/t/why-is-the-lm-head-layer-in-gpt2lmheadmodel-not-a-parameter/639)

其实不应该算是一个技术，而是一个发现：GPT2 模型里，lm_head （分类头线性层）和 wte （embedding层）使用的是相同的参数，相互呈转置关系，即 `lm_head.weight = wte.weight.T`

**Why 为什么需要做**

* 根据该问题里 sgugger 的说法，因为 wte 的功能是将在 vocabulary 空间的 tokens 映射到 embedding，而 lm_head 则是将 embedding 映射到 vocabulary 空间的 tokens，因此自然地希望这两个 embedding 共享相同的特征空间

**How 具体要怎么做**

* 在 Huggingface transformers 库中，这一操作通过 `PreTrainedModel.tie_weights()` 函数实现：

  ```python
  def tie_weights(self):
          """
          Tie the weights between the input embeddings and the output embeddings.
  
          If the `torchscript` flag is set in the configuration, can't handle parameter sharing so we are cloning the
          weights instead.
          """
          if getattr(self.config, "tie_word_embeddings", True):
              output_embeddings = self.get_output_embeddings()
              if output_embeddings is not None:
                  self._tie_or_clone_weights(output_embeddings, self.get_input_embeddings())
  
          if getattr(self.config, "is_encoder_decoder", False) and getattr(self.config, "tie_encoder_decoder", False):
              if hasattr(self, self.base_model_prefix):
                  self = getattr(self, self.base_model_prefix)
              self._tie_encoder_decoder_weights(self.encoder, self.decoder, self.base_model_prefix)
  
          for module in self.modules():
              if hasattr(module, "_tie_weights"):
                  module._tie_weights()
  ```

  其中用到的 `_tie_or_clone_weights()` 函数：

  ```python
  def _tie_or_clone_weights(self, output_embeddings, input_embeddings):
          """Tie or clone module weights depending of whether we are using TorchScript or not"""
          if self.config.torchscript:
              output_embeddings.weight = nn.Parameter(input_embeddings.weight.clone())
          else:
              output_embeddings.weight = input_embeddings.weight
  
          if getattr(output_embeddings, "bias", None) is not None:
              output_embeddings.bias.data = nn.functional.pad(
                  output_embeddings.bias.data,
                  (
                      0,
                      output_embeddings.weight.shape[0] - output_embeddings.bias.shape[0],
                  ),
                  "constant",
                  0,
              )
          if hasattr(output_embeddings, "out_features") and hasattr(input_embeddings, "num_embeddings"):
              output_embeddings.out_features = input_embeddings.num_embeddings
  ```

  可以看到，根据 config 里 `tie_word_embeddings` 的值，会决定是否将 `output_embeddings` 赋值为 `input_embeddings`

  而在 `GPT2LMHeadModel` 中：

  ```python
  def get_output_embeddings(self):
          return self.lm_head
  ```

  在 `GPT2Model` 中：

  ```python
  def get_input_embeddings(self):
          return self.wte
  ```

  可见其 output_embeddings 即是 lm_head，input_embeddings 即是 wte.

  那一个可能的问题是，lm_head 和 wte 不是互相为转置关系吗？为什么直接赋值？原因如下：

  * wte 的实现是 `self.wte = nn.Embedding(config.vocab_size, self.embed_dim)`
    * Embedding 类实现的是 $xW$​​ 
    * x 的 shape 是 (token_num, vocab_size)
    * W 的 shape 是 (vocab_size, embed_len)
  * Lm_head 的实现是 `self.lm_head = nn.Linear(config.n_embd, config.vocab_size, bias=False)`
    * Linear 类实现的是 $xW^T$​
    * x 的 shape 是 (token_num, embed_len)
    * W 的 shape 仍然是 (vocab_size, embed_len)

  因此，这俩个类的权重直接赋值之后，实现的计算效果仍然是互为转置关系的。

**What 做到什么效果**

* lm_head 与 wte 共享参数

**Comments**

* 今天是在做自己工作的实验的过程中发现，当修改 lm_head 参数时，我的中间 embedding 结果也发生了变化，就觉得非常奇怪。debug 最后还是 ChatGPT 说：

  > When using `model.named_parameters()` with `GPT2LMHeadModel`, it might seem like only the transformer weights are listed because the `lm_head` often shares its weights with the transformer’s embeddings to reduce the number of parameters.

  正是其中的“lm_head 通常和 embeddings 共享权重”这一句提醒了我，才找到了 huggingface 论坛里的这个问题的回答。
