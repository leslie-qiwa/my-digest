# Fable Converted Pylint to Rust

一个用 Rust 重新实现的 pylint 错误检查工具，产生与 pylint 逐字节完全一致的输出——速度快 15-84 倍

## 项目描述

### prylint

一个用 Rust 重新实现的 pylint，产生逐字节完全一致的输出——速度快 15–2300 倍（中位数约 85 倍）。

prylint 不是"受 pylint 启发"的产物。它是一个连 bug 都完全复刻的移植版：相同的消息、相同的行号和列号、相同的文本、相同的顺序、相同的退出码，以及相同的 `Your code has been rated` 页脚——在 52 个生产级代码库（约 65,000 个 Python 文件）上与真实 pylint 进行了逐字节验证，包括 django、numpy、pandas、sympy、home-assistant、sqlalchemy、twisted、scikit-learn，以及 pylint 自身的功能测试套件。pylint 有 bug 的地方，prylint 也会复现。pylint 崩溃的地方，prylint 也会报告相同的崩溃信息。

### 安装

```
pip install prylint
```

要求：PATH 中有 `python3`（≥3.9）（仅用于镜像 pylint 的模块解析路径，以及为无法解析的文件复现 CPython 的精确语法错误信息）。不需要安装 pylint 和 astroid 本身。

### 用法

使用方式与 pylint 完全相同——默认为完整检查模式：

```
prylint .                      # 所有检查（等同于 `pylint .`）
prylint -E .                   # 仅错误（等同于 `pylint -E .`）
prylint --disable=C0114,... .  # 相同的 --disable / --enable / 内联指令
```

输出、消息顺序、退出码、评分页脚、`--rcfile` / `pyproject.toml` 发现、`init-hook`，以及 `# pylint:` 指令均与 pylint 4.0.5 保持一致。

### 基准测试

`prylint .` 对比 `pylint .`（均为完整检查模式），pylint 4.0.5，Apple M 系列芯片，单线程：

| 代码库 | pylint | prylint | 加速比 |
|---|---|---|---|
| black | 26.7 小时 | 41 秒 | 2328× |
| sentry | 3.7 小时 | 24 秒 | 546× |
| home-assistant（17,500 个文件） | 10.3 小时 | 82 秒 | 452× |
| airflow | 1.9 小时 | 17 秒 | 399× |
| salt | 1890 秒 | 8.8 秒 | 215× |
| zulip | 909 秒 | 5.3 秒 | 172× |
| django | 1524 秒 | 10.1 秒 | 150× |
| ansible | 419 秒 | 2.9 秒 | 143× |
| nova (OpenStack) | 1209 秒 | 10.3 秒 | 117× |
| fastapi | 116 秒 | 1.0 秒 | 120× |
| mypy | 367 秒 | 3.9 秒 | 95× |
| sqlalchemy | 614 秒 | 7.1 秒 | 87× |
| pandas | 1009 秒 | 14.2 秒 | 71× |
| scikit-learn | 613 秒 | 9.6 秒 | 64× |
| sympy | 1238 秒 | 26 秒 | 48× |
| ……以及另外 12 个，均 ≥30× | |||
| 汇总（以上 27 个） | 45.8 小时 | 4.9 分钟 | ~560× |

（以上 27 个是足够大、运行时间足够长以便有意义地计时的大型仓库；完整的准确性测试套件包含 52 个仓库——详见下文。）

每仓库中位加速比约 85 倍；汇总数据更高，因为 pylint 的重复代码检查（`R0801`）是 O(n²) 的，在测试文件较多的仓库（如 black）上占主导地位。这些是单核数据——推理引擎是单线程的，以精确复制 astroid 的顺序敏感全局缓存（详见 LIMITATIONS.md），而这条逐字节一致的路径已经比 pylint 快 15–2300 倍。

上表中的每一行同时也是一个准确性测试：每个仓库的完整输出与 pylint 逐字节一致（例外情况见 LIMITATIONS.md）。

### 准确性

prylint 通过与固定版本的 pylint 4.0.5 / astroid 4.0.4 / CPython 3.12 进行差异测试构建而成：

- **AST 保真度** — prylint 的解析树（基于 ruff 解析器构建）与 astroid 的解析树在所有语料库文件上逐节点比较（位置、作用域、局部变量、brain 转换）：零差异。
- **推理保真度** — astroid 的推理引擎被精确移植：惰性生成器语义、100 节点推理预算、有界 LRU 缓存（`lookup` 128 条、`_metaclass_lookup_attribute` 1024 条）及其精确的淘汰策略、64 条目推理提示 FIFO、`Uninferable` 传播。每个名称/属性/调用节点的推理结果都会导出并与 astroid 进行比较。
- **输出保真度** — 完整运行结果逐字节比较，包括消息顺序、模块头、评分页脚、`# pylint:` 指令处理（disable/enable 块、`disable-next`、`skip-file`）、配置文件发现，以及退出码位掩码。
- **盲测** — 在开发完成后分两批各添加了 10 个仓库进行冷测试；每一处差异都被追溯到根本原因并修复。

已知的、已记录的例外情况（一个冷门的 SQLAlchemy 类；被刻意排除的 `no-member` 系列；pylint 对自身也不确定的非确定性行为）均记录在 LIMITATIONS.md 中。

### 工作原理

- 文件发现、消息控制、配置解析和报告输出是 pylint 自身逻辑的直接移植（细致到 `os.walk` 的排序、`************* Module` 头部规则，以及评分报告页脚）。
- 解析使用 ruff 的 Rust 解析器，然后重建 astroid 的精确树形结构（文档字符串提取、装饰器位置、隐式类局部变量、元类处理、针对 dataclasses/enums/namedtuples/attrs/……的 brain 转换）。
- 完整移植了 astroid 的推理引擎，用于解析名称、调用、属性、MRO 和运算符协议，保持 astroid 精确的保守策略——包括其缓存及其怪癖，因为这些怪癖在输出中是可观测的。
- Rust 解析器无法解析的文件会交由 CPython 本身重新判定（一个内嵌的、仅使用标准库的辅助程序），使语法错误信息与 `ast.parse` 精确匹配。

### 复现测试套件

`scripts/setup_corpora.sh` 会在固定提交点克隆所有 52 个语料库，并构建固定版本的 pylint/astroid 基准虚拟环境。准确性契约：每次更改都必须保持语料库输出逐字节一致（`harness/` 目录下存放差异比较器）。

### 许可证

GPL-2.0-or-later，与 pylint 使用相同的许可证——prylint 逐字复现了 pylint 的消息文本和行为。

## 项目详情

### 下载文件

请下载适用于您平台的文件。如果不确定选择哪个，请了解更多关于安装包的信息。

源代码分发包

预构建分发包

按文件名、解释器、ABI 和平台筛选文件。

如果不确定文件名格式，请了解更多关于 wheel 文件名的信息。

复制当前筛选条件的直接链接

### 文件详情

`prylint-0.4.2.tar.gz` 的详细信息。

文件元数据

- 下载 URL：prylint-0.4.2.tar.gz
- 上传日期：
- 大小：1.9 MB
- 标签：Source
- 是否使用可信发布？否
- 上传工具：twine/6.2.0 CPython/3.12.12

文件哈希值

| 算法 | 哈希摘要 | |
|---|---|---|
| SHA256 | `0dea90b8bc79eb5e7412c180a9cf034a107cf6e1fea46f8cbe078775908db60f` | |
| MD5 | `e9f5cb1c4cd317474aa07f156fb72280` | |
| BLAKE2b-256 | `c7cf38edbebace645bb87f83909858382391fad6ca4596bd4aa9d3be5910e0d0` |

### 文件详情

`prylint-0.4.2-py3-none-win_amd64.whl` 的详细信息。

文件元数据

- 下载 URL：prylint-0.4.2-py3-none-win_amd64.whl
- 上传日期：
- 大小：4.2 MB
- 标签：Python 3, Windows x86-64
- 是否使用可信发布？否
- 上传工具：twine/6.1.0 CPython/3.13.12

文件哈希值

| 算法 | 哈希摘要 | |
|---|---|---|
| SHA256 | `55394ec3ead2d10652cbe29f820d14f0c03ec8b65d4430c89d57eb549e94d2f1` | |
| MD5 | `cc90eebcdc13d73091cf150c82942e5d` | |
| BLAKE2b-256 | `05031745da692b62eedc94dabb9fbc6572ff571e399ddb66b9bf6dc4530e36b6` |

### 文件详情

`prylint-0.4.2-py3-none-manylinux_2_28_aarch64.whl` 的详细信息。

文件元数据

- 下载 URL：prylint-0.4.2-py3-none-manylinux_2_28_aarch64.whl
- 上传日期：
- 大小：4.3 MB
- 标签：Python 3, manylinux: glibc 2.28+ ARM64
- 是否使用可信发布？否
- 上传工具：twine/6.1.0 CPython/3.13.12

文件哈希值

| 算法 | 哈希摘要 | |
|---|---|---|
| SHA256 | `16efe779edc96b8ae3228aef8d6f54c69a9d8114a862f5897e29cf6dc62c9917` | |
| MD5 | `ca5863550a15653390e4567225157536` | |
| BLAKE2b-256 | `041efe64feb4410656c601d83ef22615da8a9a634fbb4d9dd004062c119ea423` |

### 文件详情

`prylint-0.4.2-py3-none-manylinux_2_17_x86_64.manylinux2014_x86_64.whl` 的详细信息。

文件元数据

- 下载 URL：prylint-0.4.2-py3-none-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
- 上传日期：
- 大小：4.4 MB
- 标签：Python 3, manylinux: glibc 2.17+ x86-64
- 是否使用可信发布？否
- 上传工具：twine/6.1.0 CPython/3.13.12

文件哈希值

| 算法 | 哈希摘要 | |
|---|---|---|
| SHA256 | `c948478aa3bc14f0d38a25ebf34189d80d7918863ab1e2620d6727772b73dfe7` | |
| MD5 | `c89a2642c299eb1f9b7f744c39d53f37` | |
| BLAKE2b-256 | `aee59e1b9f44a0b70ab5285c89576ac001aa8b38ee1abf2c02c5b68790216ec8` |

### 文件详情

`prylint-0.4.2-py3-none-macosx_11_0_arm64.w` 的详细信息
