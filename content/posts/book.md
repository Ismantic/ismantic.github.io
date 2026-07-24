+++
date = '2026-03-18T19:00:00+08:00'
draft = false
title = '底层实现：语言模型技术泛谈'
url = '/1/'
+++

语言模型的技术栈通常由许多独立系统拼接而成：文本由 Tokenizer 处理，模型依赖训练框架执行，算子由 CPU 或 GPU 实现，上层再使用 Python 把这些组件组织起来。这样的分层便于使用，却也隐藏了各部分真正的连接方式。

曾经对这个问题做过不少思考，也做了一些尝试，逐渐整理出 Text、Zero 和 Matx 几个项目。它们以 C++ 为系统核心，分别探索文本处理、模型训练和语言编译，并尝试为用户提供一种接近 Python 的脚本语言。目标不是重新包装现有库，而是观察一套语言模型系统从文本输入到硬件计算究竟需要哪些基础结构。这些工作既包含对已有实现的重新整理，也是重新理解和学习相关技术的过程。至于能否在此基础上产生一些创新，还要看之后的积累和机遇。

## 想法

最终的系统以类 Python 语言作为用户界面。用户仍然可以用熟悉的方式组织 Tokenizer、Tensor、模型和训练循环，但程序不再依赖 Python 充当各个组件之间的胶水：

```python
tokenizer = Tokenizer.Load("BPE")
model = GPT("Gpt-2")
optimizer = AdamW(model.Parameters())

tokens = tokenizer.Encode(text)
loss = model(tokens)
loss.Backward()
optimizer.Step()
```

这段代码在底层会进入一套以 C++ 为主的技术栈：

```text
                 类 Python 脚本语言
                          │
                          ▼
                    Matx 语言编译器
                 语法、类型、对象与控制流
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
       Text 文本处理               Zero 训练框架
  Tokenizer / Regex / Trie    Tensor / Autograd / Module
             │                         │
             └────────────┬────────────┘
                          ▼
                    Tensor Operator
                          │
                          ▼
                    Tensor 编译器
               计算表示、调度、内存与并行化
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
             CPU         CUDA       其他硬件
```

这里需要两层不同的编译器。语言编译器处理函数、对象、容器、控制流和模块调用，把类 Python 程序转换成 C++ 或更低层的程序表示；Tensor 编译器处理 Tensor 的索引、布局、循环、并行化和内存调度，再将 Tensor IR 降低为不同硬件上的 Kernel。

两层编译器之间通过 Tensor Operator 或 Tensor IR 连接。上层语言不需要理解 CUDA Thread Block，Tensor 编译器也不需要处理 Python 字典和字符串。C++ Runtime 则位于整个系统中间，为文本组件、训练框架、语言对象和动态模块提供共同的对象生命周期与调用边界。

这里所说的“替代 Python”，不是复刻 CPython 的全部动态语义，而是替代 Python 在语言模型技术栈中的胶水角色。用户面对的是简洁的脚本语言，真正的数据结构、模型计算和本地执行则由统一的 C++ 系统完成。

## 已实现的部分

这个愿景还没有成为一个完整系统。目前已经完成的工作被整理为三本书，它们分别建立文本处理、训练框架和语言编译三块基础。

**[底层实现：文本处理](/text/)**

文本进入语言模型之前，需要从字符和字节变成稳定的 Token ID。这本书沿真实的数据处理顺序，介绍 Unicode 与 UTF-8、正则表达式引擎、Trie、中文分词和 SentencePiece，并通过 W2V 与 LDA 展示处理后的词语怎样进入经典统计模型。

中文分词并不是 Tokenizer 的必要条件。将它放入 PreTokenize 阶段，是为了让空格语言与非空格语言通过统一接口进入子词学习，同时缩短待处理片段，减少候选搜索，提高词表训练和编码效率。

**[底层实现：训练框架](/zero/)**

Zero 是一个以 C++ 实现的教学型训练框架。这本书从 Autograd、Tensor 和 Operator 开始，继续讨论 Module、Optimizer、Python Frontend 与 GPU 计算，最后通过 GPT 训练把注意力、反向传播和参数更新放回完整任务。

它目前使用手写的 CPU 与 CUDA Kernel。未来的 Tensor 编译器并不是另一套训练框架，而是要逐步接替这一层硬件相关实现，让 Zero 中的 Tensor Operator 可以通过编译生成不同设备上的执行代码。

**[底层实现：编译器](/matx/)**

Matx 目前实现了一个受限 Python 子集到 C++ 的提前编译流程。它读取 Python AST，构造自己的程序表示，通过 Visitor 与 Rewriter 生成 C++，编译成动态库，再由 FFI 回到 Python 中调用。

为了支撑这条链路，Matx 同时实现了 Runtime、动态值、对象生命周期、数据容器、统一函数调用和动态模块。它既是语言编译器的起点，也是在实践中重新理解 Python 对象、容器和函数的一种方式。

Matx 还不是愿景中的完整脚本语言。目前的重点是打通 `Python → AST → C++ → .so → Python`，后续才会逐步连接 Text 与 Zero，使 Tokenizer、Tensor 和模型成为语言中的一等对象。

## 接下来的连接

三本书目前可以独立阅读，但它们最终需要共享稳定的系统边界：

```text
Text Tokenizer ──┐
                 ├── Matx Runtime ── 类 Python 语言
Zero Tensor ─────┘
                         │
                         ▼
                  Tensor / Kernel IR
                         │
                         ▼
                    多硬件代码生成
```

之后比较关键的连接，是让 Matx 能够持有 Zero 的 Tensor 与 Module，并从脚本中执行前向计算、反向传播和参数更新。Text 中的 Tokenizer 则可以作为 Runtime 对象进入同一种语言接口。更远的一步，是建立独立的 Tensor IR 与 Tensor 编译器，将 Zero 中手写的 CUDA Kernel 转换成可以调度、优化并迁移到其他硬件的计算描述。

目前的三本书记录了已经完成的基础，后续工作会尝试让这些基础逐渐汇合。它们共同指向一个问题：如果不依赖现成的 Python 机器学习生态，从文本、训练、语言到硬件，一套语言模型系统应当怎样建立？更重要的问题是，这样做是否真的有价值？目前我的判断是，接下来可以把更多精力放在 Tensor 编译器上，PyTorch 生态其实还是很健康，搞一个新生态可能意义不大，只是做探索而已；Text、Zero 和 Matx 则更适合作为教学项目，用来展示一套系统在底层如何工作。
