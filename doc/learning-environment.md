# Whn6666 的学习环境

## 当前状态

- Fork：<https://github.com/Whn6666/nanovllm2deepinfer>
- 本地：`/workspace/nanovllm2deepinfer`
- `upstream` 是原仓库，`origin` 是自己的 fork。
- `Whn6666-main` 是累计进度分支，本节在 `learn/1.1`。
- Python 3.12；本地使用 PyTorch 2.6.0 CPU 版，适合 CPU 工具开发和测试。
- 参考仓库及 DeepInfer 的第一层 vendor 子模块已按固定提交初始化，仅供阅读，不参与安装。
- 本机没有 CUDA GPU。未安装 `triton`、`flash-attn`；不能在本机运行 `nanovllm` 推理。editable 安装使用了 `--no-deps`，不代表完整 GPU 依赖已满足。

## 激活及重建

```bash
cd /workspace/nanovllm2deepinfer
source .venv/bin/activate
export UV_CACHE_DIR=/workspace/.cache/uv
export MODAL_CONFIG_PATH=/workspace/nanovllm2deepinfer/out/modal.toml
```

在新的同类 CPU 环境重建：

```bash
UV_CACHE_DIR=/workspace/.cache/uv uv venv .venv --python 3.12
export UV_CACHE_DIR=/workspace/.cache/uv
uv pip install --python .venv/bin/python torch==2.6.0 --index-url https://download.pytorch.org/whl/cpu
uv pip install --python .venv/bin/python 'transformers==4.57.6' xxhash pytest 'modal[api-proxy-support]==1.6.4' 'pip-system-certs==5.3' ruff numpy
uv pip install --python .venv/bin/python --no-deps -e .
bash scripts/setup_reference.sh
```

本次解析的完整依赖版本记录在 `out/learning-environment.freeze.txt`。

## GPU 实验：Modal

Modal CLI 已安装，包含当前代理环境需要的 `api-proxy-support` 可选依赖。第一次运行前需要浏览器登录；凭据只保存在已被 Git 忽略的 `out/modal.toml`，不要提交，也不要贴进聊天。

初次登录返回 `Could not connect to the Modal server`。进一步绕过外层重试诊断后，确认根因为 `SSLCertVerificationError: self-signed certificate in certificate chain`：grpclib 显式使用 certifi 的 CA 包，缺少云环境代理的受信任 CA。通过在项目虚拟环境安装 `pip-system-certs==5.3`，让新启动的 Python 进程使用系统证书信任，保留 TLS 验证。修复后 Modal SDK 的 TLS/HTTP2 连接探测已通过。

浏览器认证已完成，Modal 返回 `Token verified successfully!`。当前 profile 为 `nanovllm-learning`，工作区为 `holland-w-1998`，token 申请有效期为 7 天。凭据文件权限为 `600`，位于被 Git 忽略的 `out/modal.toml`。新终端仍需按上面的命令导出 `MODAL_CONFIG_PATH`；未启动 GPU 实例。

```bash
mkdir -p out
export MODAL_CONFIG_PATH=/workspace/nanovllm2deepinfer/out/modal.toml
modal token new --profile nanovllm-learning --expires-in 7d
```

依照 `AGENTS.md` 的第一次使用要求，先在 Modal Usage & Billing 设置 budget $30、spend limit $0。教程中的 L4 价格和整节费用只是原作者估算，以账户实时价格为准。

登录并完成账户限额设置后，首个实验为：

```bash
GPU=L4 modal run scripts/modal_run.py --cmd 'python bench.py'
```

该命令会创建远端 GPU 容器并下载 Qwen3-0.6B；本次尚未运行。GPU 镜像由 `scripts/modal_run.py` 配置，使用 Python 3.11、CUDA 版 torch 2.6.0 和匹配的 FlashAttention wheel，独立于本地 CPU 环境。

## 1.1 的学习闭环

阅读 `doc/1.1-观测与正确性.md`，按发现 → 分析 → 解决 → 验收 → 对照推进。

1. **发现**：先预测 `bench.py` 的输出能回答哪些问题，再跑基线。记录硬件和真实输出；文档数字不当作自己的实测。
2. **分析**：沿 `bench.main → LLMEngine.generate → step → ModelRunner.run` 阅读。用临时计时观察前向发射和采样等待的差异，由学习者解释原因。
3. **解决**：问题确认后才建立脚手架。任务 A 拆分每步耗时；任务 B 记录请求时间戳、构造负载和报告；任务 C 引入 HF 正确性参照。导师准备样板和测试，学习者实现核心逻辑。
4. **验收**：重跑相同实验、四类负载和正确性对照；做可还原的破坏实验，确认测试能抓到错误。
5. **对照**：读本节 diff，解释结构变化，写学习卡片；本节 PR 从 `learn/1.1` 提到自己 fork 的 `Whn6666-main`。

当前只完成环境准备，尚未进行 GPU 复现和本节实现。`main` 上没有 `bench/` 和 `tests/test_1_1.py`；现阶段 pytest 收集不到测试是起点状态，不能算通过本节验收。
