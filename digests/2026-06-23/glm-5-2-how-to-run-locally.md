# GLM-5.2 – How to Run Locally

GLM-5.2 — 如何在本地运行

在本地硬件上运行 Z.ai 的全新 GLM-5.2 模型！

GLM-5.2 是 Z.ai 的新一代开源模型，在长周期编程、推理和智能体任务方面达到了 SOTA 性能。该模型拥有 7440 亿参数、400 亿活跃参数和 100 万上下文窗口，现可通过 Unsloth 动态 GGUF 量化在本地运行。GLM-5.2 是目前最强的开源模型，在 Artificial Analysis 及多项其他基准测试中与 Claude 4.8 Opus、GPT-5.5 和 Gemini 3.1 Pro 表现相当。

动态 1-bit 量化在体积缩小 86% 的情况下达到约 76.2% 的 top-1 准确率。动态 2-bit 量化在体积缩小 84% 的情况下达到约 82% 的准确率。这意味着模型并非因为体积缩小 86% 就变差 86%——相比完整的 1.5TB 模型，其准确率仅下降约 24%。感谢 Z.ai 为 Unsloth 提供了首日访问权限。GLM-5.2-GGUF

2-bit 动态量化 UD-IQ2_M 占用 239GB 磁盘空间——可直接在 256GB 统一内存的 Mac 上运行，也适用于 1 块 24GB 显存 GPU 加 256GB 内存并启用 MoE 卸载的配置。1-bit 量化需要 223GB 内存，8-bit 量化需要 810GB 内存。

表：推理硬件需求（单位 = 总内存：RAM + VRAM，或统一内存）

| 1-bit | 2-bit | 3-bit | 4-bit | 5-bit | 8-bit |
|-------|-------|-------|-------|-------|-------|
| 223 GB | 245 GB | 290-360 GB | 372-475 GB | 570 GB | 810 GB |

为获得最佳性能，请确保可用总内存（包括 VRAM 和系统 RAM）充分超过量化模型文件大小。

**推荐设置**

GLM-5.2 有 3 种思考模式：非思考模式和两种思考模式（High 和 Max）。复杂任务建议使用 Max 思考模式。在 Unsloth Studio 中，您可以通过界面轻松切换 High、Max 思考模式和非思考模式。

大多数场景推荐以下设置：

| 默认设置（大多数任务） | SWE-Bench Pro |
|---|---|
| temperature = 1.0 | temperature = 1.0 |
| top_p = 0.95 | top_p = 1.0 |

最大上下文窗口：1,048,576。

**禁用思考模式、调整推理力度**

GLM 5.2 默认启用推理。同时支持推理力度设置，reasoning_effort 可设为 "high"、"max" 或禁用。

要禁用思考模式，使用 `--chat-template-kwargs '{"enable_thinking":false}'`。如果使用 Windows Powershell，请使用：`--chat-template-kwargs "{\"enable_thinking\":false}"`

也可以在 llama.cpp 中使用 `--reasoning on` 或 `--reasoning off`！

如需自定义推理力度或禁用推理，请参考以下示例：

**📈 量化分析**

我们还运行了 KLD（KL 散度）基准测试来评估 GLM-5.2-GGUF 量化的准确性。动态 4-bit UD-Q4_K_XL 和动态 5-bit UD-Q5_K_XL 几乎无损，较小的量化方案也表现出色——通过动态保留重要层的高精度，将不重要的层降至低比特。

在纯 top-1% 准确率方面，动态 1-bit 在体积缩小 86% 的情况下达到约 76.2% 的准确率！动态 2-bit 在体积缩小 84% 的情况下达到约 82% 的准确率。这表明动态量化某些层到更高精度并不会使模型在缩小 86% 的同时变差 86%——而仅比完整 1.5TB 模型降低约 24% 的能力。

**那么"76% 准确率"实际意味着什么？**

76% 的 top-1% 准确率并不意味着"法国的首都是"这个问题会 76% 选巴黎、24% 选悉尼。模型并非"笨了 24%"。对于这类问题，巴黎始终是 100%，悉尼始终是 0%。76% 这个数字包含了整个语料库中的填充词和停用词，例如提问：

"写一篇小说"会因 LLM 采样得到：
- 我现在来写一篇小说……
- 小说如下：
- 您希望它是什么类型的？

每个示例都是正确的，但 [我、小说、您] 的分布发生了变化——基线可能 100% 使用 [我]，但现在 [我] 是 76%，[小说] 是 24%。

这并不意味着你会有 24% 的概率得到乱码或错误输出。

99.9% KLD 通常也很好——不过从 4-bit 开始有较大提升，因此对于大规模分布外任务，动态 4-bit 可能是最佳选择。

Top-1% 是 KLD 本身的一种"强制"二项分布。KLD 是基线（BF16 或 Q8_0）与量化版本之间概率的"距离"。量化的目标是最小化以下目标：

minimize n1∑DKL[f(q(W))∣∣f(W))]

其中 f 是语言模型的前向传播，q 是量化操作，W 是模型的参数或权重。目标是使基线 f(W) 的 logits 输出与量化模型输出之间的"距离"尽可能小。如果能实现 0 KLD，则意味着完美重建了模型！

我们使用如下的均值 KLD，因为在完整训练语料上运行 KLD 非常昂贵（例如 15T 个 token）——我们改用采样方式，获取训练语料/下游任务的一个小型代表性子集来进行优化。均值 KLD 通常与磁盘空间呈单调趋势，表明即使在 1-bit 下 GLM 5.2 也能良好工作！

Top-1% 准确率只是一个贪心解码算子，假设会选择 argmax 项，对于 1-bit 来说，76% 的情况下它与基线的 argmax 选择一致。

**运行 GLM-5.2 教程：**

现在可以在 llama.cpp 和 Unsloth Studio 中运行 GLM-5.2。我们将使用 239GB 的 UD-IQ2_M 量化，以获得可访问性和准确性的最佳平衡。

**🦥 在 Unsloth Studio 中运行 GLM-5.2**

GLM-5.2 可在 Unsloth Studio（本地 AI 的开源 Web UI）中运行。Unsloth Studio 自动卸载至 RAM 并检测多 GPU 配置。通过 Unsloth Studio，您可以在 macOS、Windows、Linux 上本地运行模型。

然后在浏览器中打开 http://127.0.0.1:8888（或您的特定 URL）。

**通过 HTTPS 和 Cloudflare 安全启动 Unsloth**

全新功能！Unsloth 现提供通过免费 Cloudflare 隧道以 HTTPS 安全启动 Studio 的方式。使用以下命令（适用于 Windows、Mac 和 Linux）：

**2. 搜索并下载 GLM-5.2**

Unsloth Studio 自动卸载至 RAM 并检测多 GPU 配置。首次启动时需要创建密码以保护账户并用于后续登录。

然后前往 Studio 聊天标签页，在搜索栏中搜索 GLM-5.2，下载所需的模型和量化版本。请确保有足够的计算资源来运行模型。

**3. 运行 GLM-5.2**

使用 Unsloth Studio 时推理参数会自动设置，但仍可手动更改。还可以编辑上下文长度、聊天模板及其他设置。

本指南将运行 UD-IQ2_M 量化，需要至少 245GB 内存。您可以自行更改量化类型。在这些教程中，我们将使用 llama.cpp 进行快速本地推理。GGUF：GLM-5.2-GGUF

**1.** 从 GitHub 获取最新版 llama.cpp。您也可以按照以下构建说明操作。如果没有 GPU 或只想使用 CPU 推理，将 `-DGGML_CUDA=ON` 改为 `-DGGML_CUDA=OFF`。对于 Apple Mac / Metal 设备，设置 `-DGGML_CUDA=OFF` 然后正常继续——Metal 支持默认开启。

**2.** 现在可以直接使用 llama.cpp 加载和下载模型，就像 `ollama run` 一样。首先选择所需的量化类型，如 UD-IQ2_M。还可以使用 `export LLAMA_CACHE="unsloth/GLM-5.2-GGUF"` 强制 llama.cpp 保存到指定位置。注意此下载过程可能非常慢，因此建议使用下一节的手动下载方式。

**3.** 如果想手动下载模型（速度更快！），可以通过以下代码下载（安装 `pip install huggingface_hub` 后）。如果下载卡住，请参阅：Hugging Face Hub、XET 调试。

如果想使用动态 1-bit 版本：

**4.** 然后以对话模式运行模型。2-bit 使用 `unsloth/GLM-5.2-GGUF/UD-IQ2_M/GLM-5.2-UD-IQ2_M-00001-of-00006.gguf`，1-bit 使用 `unsloth/GLM-5.2-GGUF/UD-IQ1_S/GLM-5.2-UD-IQ1_S-00001-of-00006.gguf`。

**5.** 启动 llama-cli 后，您将看到：

然后在提示它制作一个简易 Flappy Bird 游戏后，我们得到了：

完整的对话和游戏如下：

完整游戏（HTML）完整对话

游戏带有音效且运行良好！请注意这是 1-bit 量化，效果依然出色！

**📐 通过 KV 缓存实现长上下文**
