# Semantic Cache Pool

> OpenAI-compatible semantic cache proxy

开包即用的语义缓存代理层。克隆下来一行命令就能跑。

对 API 请求做精确匹配 + 语义相似度搜索，命中时 **0 tokens 返回**，不命中时透明转发上游 API 并自动缓存。

## 快速开始

```bash
git clone https://github.com/rebinelee/semantic-cache-pool
cd semantic-cache-pool
bash start-cache.sh
```

启动脚本会自动：

1. 安装 Python 依赖（Flask + requests）
2. 检测 NVIDIA GPU/CUDA，下载对应架构的 llama.cpp（CPU 或 CUDA 版）
3. 下载 embeddinggemma Q8_0 模型（~314MB，或使用自定义 GGUF）
4. 启动 embedding server（端口 8080）
5. 启动缓存代理（端口 18791）

全部一行搞定，无需手动配置。

## 架构

```
请求 → cache_proxy (localhost:18791)
           ↓
    语义缓存 (embeddinggemma + llama.cpp + SQLite)
           ↓
    命中？ → 是 → 0 tokens 返回
           ↓ 否
    转发 LLM API → 自动缓存 → 返回
```

## 手动操作缓存

```bash
bash cache.sh stats           # 查看统计
bash cache.sh search "xxx"    # 搜索缓存
bash cache.sh demo            # 跑 Demo
bash cache.sh put "Q" "A"     # 手动存入
bash cache.sh clean           # 清理过期
```

## 项目文件

| 文件 | 说明 |
|:----|:-----|
| `start-cache.sh` | 一键启动：安装依赖、检测 CUDA、下载 llama.cpp 与模型、启动 embedding 服务和代理 |
| `enable-proxy.sh` | 仅启动代理（embedding 服务需已在运行） |
| `cache.sh` | 手动操作 CLI：stats / search / put / clean / demo |
| `cache_pool.py` | 核心库：embedding、SQLite 读写、精确/语义/工具缓存 |
| `cache_proxy.py` | OpenAI 兼容 HTTP 代理（Flask） |
| `config.yaml` | 运行配置 |
| `requirements.txt` | Python 依赖 |
| `CLAUDE.md` | 项目说明与开发规范 |
| `AGENTS.md` | 快速入口，指向 `CLAUDE.md` |
| `models/` | 自动下载的 GGUF 模型（运行时生成，不提交） |
| `bin/` | 自动下载的 llama.cpp 二进制（运行时生成，不提交） |
| `cache.db` | SQLite 缓存数据库（运行时生成，不提交） |
| `cache_hit.log` | 命中/未命中日志，超过 5MB 自动轮转（运行时生成，不提交） |

## 配置

编辑 `config.yaml` 即可，不用碰 Python 代码：

```yaml
embedding:
  backend: "llama-server"        # 或 "ollama"
  llama_url: "http://localhost:8080/v1/embeddings"

cache:
  similarity_threshold: 0.92     # 闲聊0.85 / 通用0.92 / 技术0.94
  ttl_seconds: 604800            # 7天

proxy:
  port: 18791

upstream:
  base_url: "https://api.deepseek.com"
```

## 自定义 embedding 模型

默认自动下载 `embeddinggemma-300M-Q8_0`。想换别的 GGUF（比如中文更强的
Qwen3-Embedding）时，在 `config.yaml` 里二选一：

### 本地 GGUF 文件

```yaml
embedding:
  model_path: "D:/models/Qwen3-Embedding-0.6B-Q8_0.gguf"
  pooling: "last"                # Qwen3-Embedding 系列需要 last 池化
```

`model_path` 支持绝对路径、`~/` 开头，以及相对仓库根目录的路径；
Windows 下 `D:\models\xxx.gguf` 或 `D:/models/xxx.gguf` 都可以。

### HuggingFace 仓库（自动下载）

```yaml
embedding:
  hf_repo: "Qwen/Qwen3-Embedding-0.6B-GGUF"
  pooling: "last"
  # hf_file: "Qwen3-Embedding-0.6B-Q8_0.gguf"   # 可选，留空自动挑 Q8_0
  # hf_endpoint: "https://hf-mirror.com"        # 国内镜像（可选）
  # hf_token: "hf_xxx"                          # 私有/受限仓库才需要
```

启动脚本会查询仓库文件列表，优先挑 `Q8_0`（其次 `Q6_K` / `Q5_K_M` / `Q4_K_M`），
下载到 `models/` 后再启动 llama-server。同名文件已存在时直接复用。

可调参数：

| 参数 | 默认 | 说明 |
|:----|:----|:----|
| `model_path` | 空 | 本地 GGUF，优先级高于 `hf_repo` |
| `hf_repo` | 空 | HuggingFace 仓库 ID |
| `hf_file` | 空 | 仓库内文件名，留空自动选择 |
| `hf_token` | 空 | 也可用环境变量 `HF_TOKEN` |
| `hf_endpoint` | huggingface.co | 下载镜像地址 |
| `pooling` | `auto` | `mean` / `cls` / `last` / `rank` / `auto`（auto = 用模型自带默认） |
| `embd_normalize` | `2` | 2 = L2 归一化（余弦相似度用 2 即可） |

> **注意**：换模型后旧向量与新向量不可比较。代理会自动跳过由其他 embedding 模型生成的
> 语义缓存条目（精确匹配不受影响），无需手动删库。若想彻底清空，删除 `cache.db`。
>
> **注意**：如果 8080 端口已有 llama-server 在跑，脚本会检查它加载的模型是否与配置一致；
> 不一致时会提示你先停掉旧进程。

## GPU / CUDA 自动检测

`start-cache.sh` 每次启动都会检测系统环境：

- 如果 `nvidia-smi` 可用，读取显卡名、显存、驱动 CUDA 版本和计算能力；
- 自动选择匹配的 llama.cpp 构建：Blackwell（计算能力 ≥ 10）或较新架构 + 驱动 ≥ 13.5
  时用 CUDA 13；其余显卡用 CUDA 12.4（Windows）/ 12.8（Linux）；
- Pascal 及更老的显卡（计算能力 < 7.5）不会被选到 CUDA 13 构建（CUDA 13 已不再支持）；
- 没有 NVIDIA GPU、驱动过旧或 macOS 时自动使用 CPU 版。

想手动控制：

```yaml
embedding:
  accelerator: "auto"   # auto / cpu / cuda（cuda = 检测不到 GPU 时直接报错）
  cuda_build: "auto"    # auto / 12 / 13
```

CUDA 版 llama.cpp 的发布包**不含** CUDA 运行库，脚本会额外下载配套的 `cudart-*`
包（Windows 解压出 `cublas64_*` / `cublasLt64_*` / `cudart64_*`，Linux 解压出
`libcublas*` 等）。首次 CUDA 安装约 600–700MB，之后复用 `bin/`。

> 如果 `bin/` 里已经有 CPU 版 llama-server，脚本只会提示"未包含 CUDA 后端"，
> 不会自动覆盖。想切到 CUDA：停掉 8080 上的服务，删除 `bin/` 后重跑脚本。

## 命中日志示例

```
[2026-07-11 15:09:17] HIT  | sim=1.0000 | saved=~180 tokens | query=今天天气怎么样
[2026-07-11 15:09:41] MISS | sim=N/A    | saved=~0 tokens   | query=UV-K5的Flash怎么烧
```

## 协议

与 OpenAI-compatible API 完全兼容：

```bash
POST /chat/completions
Authorization: Bearer <your-api-key>
Content-Type: application/json

{
  "model": "deepseek-v4-flash",
  "messages": [{"role": "user", "content": "你好"}]
}
```

响应头 `X-Cache: exact` / `X-Cache: semantic` / `X-Cache: miss` 标识命中来源。

## 系统要求

- Linux x86_64 / arm64、macOS x64 / arm64、Windows x64 / arm64（Git Bash）
- Python 3.8+
- curl
- 约 500MB 磁盘空间（含模型文件；CUDA 模式额外约 1.5–2GB）
- 约 500MB 内存

> Windows 下脚本会自动下载 `win-cpu` 版 llama.cpp 并解压出 `llama-server.exe`，代理端口从 `config.yaml` 读取。

## License

MIT
