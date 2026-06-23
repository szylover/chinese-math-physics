# PyTorch 面试速通：最快上手 + 高频面试题

> 不啃大部头，只学面试真考的。面向有 Python 基础、需要在短时间内冲刺 ML/DL/算法工程师面试的读者。

一份**尽量简单、精炼、高频**的 PyTorch 面试速通指南：快速入门到能应付面试。深度取舍上砍掉了论文复现、手写 autograd 引擎等深水区，但保留了 LLM 时代**面试高频的分布式训练**（DDP / FSDP / 并行策略）。

## 章节

| 章 | 主题 |
|----|------|
| Ch1 快速上手 | 环境安装 + 张量 + autograd 速通 |
| Ch2 一章学会训练 | `nn.Module` + 损失 + 优化器 + 训练循环 + `DataLoader` |
| Ch3 必会网络速查 | MLP / CNN / RNN / Transformer 要点 + 最小可跑代码 |
| Ch4 高频理论面试题 | 反向传播、BN/Dropout、过拟合、优化器、梯度消失/爆炸… |
| Ch5 高频编码面试题 | 手写 attention / 常见层 / 训练循环 / 张量操作 |
| Ch6 分布式训练面试速通 | DataParallel vs DDP、FSDP/ZeRO、数据/张量/流水线并行、AMP、显存优化 |
| Ch7 速查表 + 常见坑 | 速查表 + API 对照 + 临场速记 |
| Ch8 面试打法 | 项目怎么讲、简版系统设计、冲刺学习路线 |

## 编译

使用 **XeLaTeX**（需 TeX Live + CJK 字体）：

```bash
make          # 编译两遍出 PDF
make clean    # 清理中间文件
```

> 想深入 AI Agent 应用层，可参阅同仓库的 [《AI Agent 工程师转行与面试完全指南》](../ai-agent-book/README.md)。
