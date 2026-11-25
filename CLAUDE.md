# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

Semantic Cache Pool：一个 OpenAI 兼容的**语义缓存代理**。请求打到本地代理（默认 18791），命中缓存直接返回（usage 计 0 token），未命中透传到上游 LLM API 并自动缓存。仓库没有测试、没有 linter，验证靠手动跑 CLI 和 curl。

## 常用命令

```bash
bash start-cache.sh                 # 一键启动：装依赖 → 检测 CUDA → 下载 llama.cpp + GGUF → 起 embedding(8080) + 代理(18791)
bash cache.sh stats                 # 缓存统计
bash cache.sh demo                  # 端到端 demo（推荐改动后用它冒烟）
bash cache.sh search "xxx"          # 语义搜索缓存
bash cache.sh put "Q" "A"           # 手动写入（会计算 embedding）
bash cache.sh clean                 # 清理过期条目并 VACUUM
bash enable-proxy.sh                # 只起代理（embedding server 需已在跑）

python3 cache_pool.py search|store|tool-search|tool-store|stats|vacuum
python3 cache_proxy.py              # 只起代理进程

curl -s http://localhost:18791/health | python3 -m json.tool   # 健康检查 + 统计
tail -f /tmp/cache_proxy.log        # 代理日志
```

验证缓存行为（非流式）：对同一 query 连发两次 `POST /v1/chat/completions`，第一次响应头 `X-Cache: miss`，第二次应为 `exact`（或改写措辞后的 `semantic` / 命中工具时为 `tool`）。流式未命中的响应头是 `miss-stream`。

平台注意：所有 `*.sh` 在 Windows 上通过 Git Bash 运行。`start-cache.sh` 已支持 Windows：自动下载 `llama-<tag>-bin-win-cpu-x64.zip`（解压出 `llama-server.exe` 及 DLL），代理 host/port 从 `config.yaml` 读取；Linux/macOS 仍用 `tar.gz`。llama.cpp 新版 release 的 `latest` 指向 `v1.0.0`，具体 build 号在 `nightly-tag.txt` 里，脚本已处理；`--no-mmap` 在新版已移除，脚本会探测后再决定是否传入。Windows 下绑定 `0.0.0.0` 可能被系统/安全软件拒绝（WinError 10013），本地测试建议用 `127.0.0.1`；若端口被系统服务占用，换一个端口并在配置里同步。

## 架构

### 三个层次（按顺序尝试）

`cache_proxy.py: chat_completions` 是唯一入口逻辑：

1. **Tool cache**（`cache_pool.search_tool`）：按 `sha256(model||query)` 精确匹配 `tool_cache` 表，返回缓存的 `tool_calls`，不走向量。带循环保护：同一 `model||query` 在 `TOOL_CACHE_LOOP_WINDOW`(10s) 内命中超过 `TOOL_CACHE_MAX_LOOP`(3) 次就放行到下一层，避免 agent 被同一个工具调用卡死。
2. **语义缓存**（`cache_pool.search`）：先 exact hash 精确匹配 `cache_entries`，未命中再对本地 embedding 服务算向量，扫描所有未过期条目做 cosine 相似度，`>= similarity_threshold` 即命中。
3. **未命中**：`forward_to_deepseek` 透传上游，`_try_cache_response` 决定怎么存：
   - 响应含 `tool_calls` → `store_tool`（TTL 3600s，空数组会被 `store_tool`/`search_tool` 拒绝）
   - 普通 content → 算 embedding 存入 `cache_entries`（默认 TTL 7 天）
   - 流式响应在转发过程中按 SSE 增量重组出完整回复后再缓存（`collected_tool_calls` 按 `index` 合并 fragments），客户端仍实时收到原始 chunk。

### 缓存键与可缓存性

- 缓存键只用**最后一条 user message 文本 + model**，不看对话历史（system prompt 仅截 500 字符存进 `metadata` 供排查）。改了 prompt 或前文不会让缓存失效。
- 写入向量时 `store` 会在 `metadata.embedding_model` 记录来源模型；语义扫描会跳过来源不同的条目（换 embedding 模型后不会误用旧向量）。
- `body.cache.disable == true` 可让单次请求跳过缓存；没有 user message 的请求不缓存。
- `config.yaml` 中 `proxy.cache_streaming` / `cache_non_streaming` 被读取进模块变量，但当前代码路径并未用它们做开关——流式响应实际总是会被缓存。

### 配置加载

`cache_pool.load_config()` 把 `config.yaml` 深度合并进默认值（只合并已知 section），然后在**模块导入时**展开成全局变量（`SIMILARITY_THRESHOLD`、`DB_PATH`、`LLAMA_SERVER_URL` 等），`cache_proxy.py` 再从中取一份。因此改配置后必须重启代理；`start-cache.sh` 用自带的 awk 函数 `cfg_get <section> <key>` 读取 `embedding.backend`、代理 host/port、自定义模型等配置（不用 `grep -P`，兼容 Git Bash）。它要求 section 标题顶格、key 缩进两级，并且只读 section 下的一级 key——改 yaml 结构时保持这个格式，`backend:` 那行的写法也不要破坏。

### Embedding 后端

两种，由 `embedding.backend` 切换：
- `llama-server`：`POST /v1/embeddings`（OpenAI 格式，返回 `data[0].embedding`），由 `start-cache.sh` 用 `--embedding` 启动；池化/归一化来自 `embedding.pooling`（`auto` 时不传 `--pooling`，用模型自带默认）和 `embedding.embd_normalize`。
- `ollama`：`POST /api/embeddings`（返回 `embedding`），模型名来自 `embedding.ollama_model`。

模型来源三选一，优先级 `model_path` > `hf_repo` > 默认 `embeddinggemma-300M-Q8_0`：
- `model_path`：本地 GGUF，支持 `~/`、Windows 绝对路径（`D:\...` / `D:/...`）和相对仓库根目录的路径；脚本用 `resolve_model_path` 转成 Git Bash 路径。
- `hf_repo`：HF 仓库 ID（如 `Qwen/Qwen3-Embedding-0.6B-GGUF`）。脚本查 `/api/models/{repo}`，按 Q8_0 > Q6_K > Q5_K_M > Q4_K_M 挑文件，下载到 `models/`（先写 `.part` 并校验 GGUF 魔数再改名）；`hf_file` / `hf_token` / `hf_endpoint` 可覆盖。
- 换模型时 `cache_pool.EMBED_ID` 会变化；语义搜索只比较 `metadata.embedding_model` 相同的条目，老条目（无该字段）按旧默认模型处理。精确匹配不受影响。`start-cache.sh` 还会用 `/props` 检查 8080 上已运行的服务是否加载了配置中的模型，不一致时直接报错退出。

### GPU / CUDA 检测

`start-cache.sh` 在非 ollama 路径下每次启动都会用 `nvidia-smi` 探测 GPU（名称、显存、驱动 CUDA 版本、`compute_cap`），由 `embedding.accelerator`（`auto`/`cpu`/`cuda`）和 `embedding.cuda_build`（`auto`/`12`/`13`）控制：
- `auto`：Blackwell（CC ≥ 10）且驱动 ≥ 13.0，或 CC ≥ 7.5 且驱动 ≥ 13.5 → CUDA 13.5；否则驱动 ≥ 12.4（Linux 为 12.8）→ CUDA 12.x；驱动太旧 → CPU。Pascal 及更老（CC < 7.5）绝不选 CUDA 13（CUDA 13 已移除这些架构）。arm64 目前只有 CUDA 13.5 包，会自动覆盖 `cuda_build`。
- CUDA 主包不含运行库，下载完主包还要下载配套 `cudart-*` 包（Windows 资产名不含 build tag，如 `cudart-llama-bin-win-cuda-12.4-x64.zip`；Linux 含：`cudart-llama-<tag>-bin-ubuntu-cuda-...`）。
- 若 `bin/` 已有二进制则跳过下载；会检查 `ggml-cuda.dll` / `libggml-cuda.so*`，没有只提示不覆盖。macOS 或 AMD 走 CPU；`--n-gpu-layers` 默认 `auto`，无需额外传。

两套响应格式不同，改 `get_embedding` 时注意分支。

### 代理与鉴权

- 上游只有 `upstream.base_url` 一个配置，`DEEPSEEK_BASE_URL` 等变量名是历史遗留，实际可指向任何 OpenAI 兼容服务。
- **代理不保存 API key**：每次请求从 `Authorization: Bearer` 头提取并透传；未提供时用内存中最近一次的 key。
- 转发时会透传客户端的 `User-Agent` 和所有 `x-opencode-*` 头（OpenCode Go 要求稳定的 `x-opencode-session`，否则返回 `MissingSessionID`）；缺失 session 时按 query 哈希生成一个稳定值。新增上游时注意不要把这些头过滤掉。
- 除 `/health`、`/v1/models`、`/chat/completions`、`/v1/chat/completions`、`/v1/embeddings` 外，所有路径走 `catch_all` 原样转发到上游。
- 命中响应的 usage 被强制写 0（`completion_tokens_details.cached_tokens`），客户端据此判断本次是缓存。

### 数据与日志

- SQLite `cache.db`（WAL 模式），两张表：`cache_entries`（response 是完整 JSON 字符串，embedding 以 JSON 文本存 BLOB 列）、`tool_cache`（tool_calls JSON）。每次操作 `get_db()` 都新建连接并执行一遍建表语句，没有连接池。
- `cache_hit.log`：HIT/MISS 结构化行，超过 5MB 自动改名为 `cache_hit.log.old`（覆盖旧的）。`log_hit` 里 query 截 120 字符，超长会附完整 repr。
- 过期判断：exact/tool 路径用每条自己的 `ttl_seconds`；语义搜索的扫描窗口用全局 `DEFAULT_TTL`，两者不一致（个别条目可能在自己 TTL 过期后仍被语义命中），改动前先确认这是不是预期。

## 代码与文件规范

### 文本风格

- 注释、日志、CLI 输出、文档一律使用中文（代码标识符和协议字段保持英文）。
- 不使用装饰性 emoji。需要表示状态时用纯文本标记，例如 `[OK]`、`[WARN]`、`[ERROR]`。
- 历史代码中的 emoji 可逐步清理；新增或修改的代码不要继续添加。
- 面向用户的响应/日志保持简洁，不堆砌修饰语。

### 文件命名与组织

- Python 模块用 snake_case（`cache_pool.py`、`cache_proxy.py`）；Shell 脚本用 kebab-case（`start-cache.sh`、`enable-proxy.sh`）；配置用 `config.yaml`；文档用 `README.md`、`CLAUDE.md`、`AGENTS.md`。
- 不在仓库根目录新增临时脚本或一次性文件。确需保留的辅助脚本要放进对应目录，并在 `README.md` 的“项目文件”表中登记。
- `README.md` 的“项目文件”表必须覆盖仓库根目录下所有脚本、模块和配置；增删文件时同步更新。
- 运行时生成物不提交：`cache.db`、`cache_hit.log`、`models/*.gguf`、`bin/`、`__pycache__/`。新增生成物时同步补充 `.gitignore`。
- 命令、配置项、端口、文件结构发生变化时，同步更新 `README.md` 和本文件的对应说明。
- 提交信息用中文，格式为 `类型: 摘要`（例如 `修复: 流式响应缓存拼接`），不使用 emoji。

### 实现约定

- 缓存写入失败永远不能影响请求转发——`_try_cache_response` 与调用处都吞异常（当前 `store_tool` 抛 `ValueError` 也在上层被捕获）。新增缓存逻辑时保持这个边界。
- `cache_proxy.py` 的 Flask 以 `threaded=True` 运行，模块级 `_tool_cache_loop_guard` 字典没有加锁，改动并发相关逻辑时要考虑这一点。
