# 快速上手：用 SiliconFlow DeepSeek V3 跑一个完整案例

本文档带你用最少步骤在本地跑通一次完整流程：选一个基准 (benchmark) -> 生成 fuzz harness -> 自动修复编译 -> 短时 fuzz 与覆盖率 -> 生成可视化报告。

适用读者：
- 初次上手 OSS-Fuzz-gen
- 需要使用 OpenAI 兼容接口（SiliconFlow DeepSeek V3）

---

## 1. 前置条件

- 系统工具
  - Python 3.11
  - Git
  - Docker（可拉镜像并正常运行容器）
  - c++filt 命令（GNU binutils）
- 本仓库代码已就绪

可选：如使用 Vertex AI，请参考仓库 `USAGE.md` 中 Vertex 章节；本篇聚焦 SiliconFlow。

---

## 2. 安装依赖

```zsh
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

## 3. 配置 LLM（SiliconFlow DeepSeek V3）

导出密钥与可选参数（不要把密钥写入代码库或日志）：

```zsh
# 必需：API Key（请替换为你的真实密钥）
export SILICONFLOW_API_KEY="<your-siliconflow-api-key>"

# 可选：不设置则用默认
export SILICONFLOW_BASE_URL="https://api.siliconflow.cn/v1"
export SILICONFLOW_MODEL="deepseek-ai/DeepSeek-V3"
```

模型名在框架中对应：`siliconflow_deepseek-v3`。

---

## 4. 运行单基准：cJSON 示例

选择仓库自带的大基准集中的一个单 YAML 来跑（更快定位问题）：

```zsh
python run_all_experiments.py \
  --model siliconflow_deepseek-v3 \
  --benchmark-yaml ./benchmark-sets/from-test-large/cjson.yaml \
  --work-dir ./results-cjson \
  --num-samples 1 \
  --temperature 0.4 \
  --run-timeout 30
```

说明：
- `--benchmark-yaml` 指定只跑一个基准，避免一次跑太多；
- `--num-samples 1` 与 `--temperature 0.4`：快速烟测设置；
- `--run-timeout 30`：每个成功构建的 harness fuzz 30 秒，平衡速度与效果。

---

## 5. 生成并查看可视化报告

```zsh
python -m report.web -r ./results-cjson -o ./results-cjson-report
python -m http.server 8012 -d ./results-cjson-report
```

浏览器打开：http://localhost:8012

可以看到：
- 每个基准的构建成功率、崩溃率、覆盖率
- 详细日志、prompt、LLM 输出、工具调用等

---

## 6. 期望产物在什么位置

以 `--work-dir ./results-cjson` 为例：
- `output-<benchmark-id>/prompt.txt` 最终拼装的 Prompt
- 若干 `*.rawoutput` LLM 原始输出
- `fixed_targets/` 修复后可编译的 fuzz harness 源码
- `build/` 编译日志与产物
- `run/` fuzz 运行日志与崩溃样本（若有）
- `coverage/` 行覆盖率与对比

---

## 7. 常见问题与建议

- 授权问题：确保 `SILICONFLOW_API_KEY` 已导出到当前 shell 会话；
- 限流或网络问题：降低并发、重试、或稍后再试；
- 构建失败：查看 `build/` 日志，框架会把错误回馈给 LLM 做多轮修复；
- 覆盖率低：增大 `--run-timeout` 或 `--num-samples`，或切换其它函数/项目；
- 批量评估：把 `--benchmark-yaml` 换成 `--benchmarks-directory <目录>` 批量跑；
- Vertex AI：参考 `USAGE.md` 的 Vertex 章节，并把 `--model` 换成 `vertex_ai_*` 模型名。

---

## 8. 进阶

- 替换/自定义 Prompt 模板：用 `--template-directory` 指向你的模板目录；
- 使用工具调用的智能代理：项目内基于 Responses API 的工具调用路径已与 SiliconFlow 适配，无需额外设置；
- 本地 Fuzz Introspector：参考 `USAGE.md` 末尾“Using Local Fuzz Introspector Instance”。

---

祝你运行顺利！如需把这份流程集成到 CI 或需要一个更短的 smoke 脚本示例，请在 issue 或 PR 中说明你的环境与期望时长，我可以给出精简版命令组合。