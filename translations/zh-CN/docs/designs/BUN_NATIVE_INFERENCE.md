# Bun 原生提示注入分类器 — 研究计划

**状态:** P3 研究 / 早期原型
**分支:** `garrytan/prompt-injection-guard`
**骨架:** `browse/src/security-bunnative.ts`
**待办事项锚点:** "Bun-native 5ms DeBERTa inference (XL, P3 / research)"

## 解决的问题

编译后的 `browse/dist/browse` 二进制文件无法链接 `onnxruntime-node`,
因为 Bun 的 `--compile` 会生成一个单文件可执行程序,该程序从临时解压目录
动态加载依赖项,而原生 .dylib 加载从该目录失败(已在 oven-sh/bun#3574、
#18079 中记录 + 在 CEO 计划 §Pre-Impl Gate 1 中验证)。

当前的缓解方案(branch-2 架构):ML 分类器仅在 `sidebar-agent.ts`
(非编译的 bun 脚本)中通过 `@huggingface/transformers` 运行。Server.ts
(已编译)没有 ML — 依赖于金丝雀 + 架构控制(XML 框架 + 命令白名单)。

branch-2 的问题:分类器只能扫描 sidebar-agent 看到的内容。任何停留在
编译二进制文件内部的内容路径(直接用户输入在输出时,仅金丝雀检查)都会
错过 ML 层。

从零开始的 Bun 原生分类器 — 无原生模块,无 onnxruntime — 将让编译后的
二进制文件在任何地方都能运行完整的 ML 防御。

## 目标指标

| 指标 | 当前(非编译 Bun 中的 WASM) | 目标(Bun 原生) |
|---|---|---|
| 冷启动 | ~500ms (WASM 初始化) | <100ms (嵌入向量 mmap) |
| 稳态 p50 | ~10ms | ~5ms |
| 稳态 p95 | ~30ms | ~15ms |
| 在编译二进制中工作 | 否 | 是(主要目标) |
| macOS arm64 | 可以(WASM) | 首要目标 |
| macOS x64 | 可以(WASM) | 延伸目标 |
| Linux amd64 | 可以(WASM) | 延伸目标 |

## 架构

三个构建块,按影响力排序:

### 1. 分词器(已完成 — 已在 security-bunnative.ts 中发布)

纯 TS 的 WordPiece 编码器,直接读取 HuggingFace 的 `tokenizer.json`
并为 BERT-small 词汇表生成与 transformers.js 相同的 `input_ids` 序列。

**为什么原生分词器本身很重要:** 分词在 transformers.js 路径中会分配
大量小数组。我们的纯 TS 版本跳过了 Tensor 分配开销。适度的加速
(~5x 仅分词器),但更重要的是:移除了异步边界,因此冷路径从零动态导入开始。

**测试覆盖:** `browse/test/security-bunnative.test.ts` 断言我们的
`input_ids` 在 20 个固定字符串上与 transformers.js 输出匹配。

### 2. 前向传播(研究中 — 需要数周)

困难的部分。BERT-small 有:
  * 12 个 transformer 层
  * 隐藏层大小 512,注意力头 8
  * 总共约 30M 参数

每次前向传播是:
  1. 嵌入查找(ids → 512 维向量)
  2. 位置编码相加
  3. 12 × (自注意力 + FFN + LayerNorm)
  4. 池化器(CLS token 投影)
  5. 分类器头(2 路 sigmoid)

热路径是每个 transformer 层的 12 次矩阵乘法。每次约为 ~512×512×{seq_len}。
在 seq_len=128 时,这是约 100 次形状为 (128, 512) @ (512, 512) 的矩阵乘法。

**两种可行的方法:**

**方法 A: 纯 TS 使用 Float32Array + SIMD**
  * 使用 Bun 的类型化数组支持 + SIMD 内联函数(当它们在 Bun 稳定版中
    落地时 — 目前仅限 wasm)
  * 实现:约 2000 行精心设计的数值计算代码。LayerNorm、GELU、
    softmax、缩放点积注意力都是手写的。
  * 延迟估计:在 M 系列上约 30-50ms(明显慢于使用 WebAssembly SIMD 的 WASM)
  * 结论:单独来看不值得。纯 TS 在矩阵乘法上无法击败 WASM。

**方法 B: Bun FFI + Apple Accelerate**
  * 使用 `bun:ffi` 调用 Apple 的 Accelerate 框架(cblas_sgemm)。
    在 M 系列上,768×768 矩阵乘法的 cblas_sgemm 约为 ~0.5ms。
  * 权重存储为 Float32Array(启动时从 ONNX 初始化器张量加载),
    分词器用 TS,矩阵乘法通过 FFI,激活函数用纯 TS。
  * 实现:约 1000 行代码。数值计算相同,但大量工作卸载到 BLAS。
  * 延迟估计:3-6ms p50(达到目标)。
  * 风险:仅限 macOS。Linux 需要通过 FFI 使用 OpenBLAS(不同的
    符号布局)。Windows 是完全不同的情况。
  * 结论:对于 macOS 优先的 gstack 可行。符合我们现有的发布
    策略(编译二进制文件仅用于 Darwin arm64)。

**方法 C: Bun 中的 WebGPU**
  * Bun 在 1.1.x 中获得了 WebGPU 支持。transformers.js 已经有
    WebGPU 后端。我们能否通过它路由原生 Bun?
  * 风险:macOS 上无头服务器上下文中的 WebGPU 需要适当的
    显示上下文。不清楚它是否能从编译的 bun 二进制文件中工作。
  * 状态:未探索。可能是获胜路径 — 值得尝试。

### 3. 权重加载(简单 — 已发布)

ONNX 初始化器张量可以在构建时一次性提取到一个扁平的二进制 blob 中,
`bun:ffi` 可以 `mmap()` 它。最终结果:运行时零解压缩。骨架还没有这样做
(它通过 transformers.js 加载),但一旦选择方法 B,计划就足够简单,
权重加载器是首先要构建的东西。

## 里程碑

1. **分词器 + 基准测试工具**(已发布)
   分词器通过正确性测试。基准测试记录当前 WASM
   基线为 10ms p50。

2. **Bun FFI 概念验证** — 从 Apple Accelerate 调用 `cblas_sgemm`,
   计时 768×768 矩阵乘法。确认 <1ms 延迟。

3. **FFI 中的单个 transformer 层** — 为 Q/K/V
   投影调用 cblas_sgemm,在 TS 中实现 LayerNorm + softmax。将输出
   与相同 input_ids 上的 onnxruntime 进行比较。必须在 1e-4
   绝对误差内匹配。

4. **完整前向传播** — 连接所有 12 层 + 池化器 + 分类器。
   在 100 个固定字符串上与 onnxruntime 进行正确性对比。

5. **生产切换** — 替换 security-bunnative.ts 中的 `classify()` 主体。
   删除 WASM 回退。

6. **量化** — 通过 Accelerate 的 cblas_sgemv_u8s8 进行 int8 矩阵乘法
   (如果可用)或回退到 onnxruntime-extensions。约 50% 内存
   减少,边际速度提升。

## 为什么不在 v1 中直接发布?

正确性是问题所在。预训练 transformer 的浮点重新实现是一项需要数周的
工程工作,其中每个操作都需要与参考实现达到 epsilon 级别的一致。
LayerNorm epsilon 设置错误,准确性会悄悄漂移。softmax 溢出
处理错误,分类器在长输入上会产生垃圾。

在 P0 安全功能的 PR 下发布这个是错误的风险分配。现在发布 WASM 路径
(已完成),证明接口(通过 `classify()` 发布),将原生版本作为后续
PR 逐步落地,并配有自己的正确性回归测试套件。

## 基准测试

当前基线(来自 `browse/test/security-bunnative.test.ts`
基准测试模式,在 Apple M 系列上测量 — 其他硬件上可能有所不同):

| 后端 | p50 | p95 | p99 | 备注 |
|---|---|---|---|---|
| transformers.js (WASM) | ~10ms | ~30ms | ~80ms | 预热后 |
| bun-native (存根 — 委托) | 与 WASM 相同 | | | 按设计匹配 |

当方法 B(Accelerate FFI)落地时,此行将使用新数字刷新,
并在提交消息中标记差异。