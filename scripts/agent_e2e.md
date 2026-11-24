# 本地外部 Agent 端到端测试

需要完全在 GitHub 的临时 macOS / Windows 虚拟机上安装和运行时，请阅读 [托管 CI 配置说明](agent_e2e_hosted.md)。本页说明如何在本机通过命令行执行。

使用真实 Codex CLI、WorkBuddy 内置执行引擎、DeepSeek Harness headless，通过仓库 Go CLI 的真实 MCP 服务连接 LazyMind。业务步骤由 Agent 执行并发布产物；图片辅助程序仅在 Agent 调用时请求配置的真实生图服务。工作流状态与审核通过正常用户/MCP 接口操作。

当前范围是 CLI/MCP 执行链路，不覆盖桌面插件的面板唤醒、停止、恢复。脚本复用已运行的本地 LazyMind 服务，不自动替换服务或证明服务构建与当前 Git SHA 相同。

## 首次运行：请配置自己的模型和密钥

> **拉取代码后不会获得其他人的 Key 或本机登录。** WorkBuddy 自定义模型模式需要自己的 LLM API Key，普通生图用例还需要生图 API Key。实际密钥保存在仓库外，GitHub CI 使用 Secrets。不要把填过密钥的配置、`.env`、登录凭证或整个测试输出目录提交到 Git。

无密钥模板是 [`agent_e2e.providers.example.json`](agent_e2e.providers.example.json)。在仓库根目录执行以下命令；已有私有配置会保留：

```sh
mkdir -p "$HOME/.config/lazymind-ci"
chmod 700 "$HOME/.config/lazymind-ci"
if [ ! -e "$HOME/.config/lazymind-ci/providers.json" ]; then
  (umask 077; cp scripts/agent_e2e.providers.example.json "$HOME/.config/lazymind-ci/providers.json")
fi
chmod 600 "$HOME/.config/lazymind-ci/providers.json"
```

用本地编辑器打开 `~/.config/lazymind-ci/providers.json`，分别填写 `llm`、`image` 的 `api_key`、`base_url`、`model` 和 `api`。两家服务使用各自的 Key。模板中的模型名和地址只是示例，需要账号具备对应模型的权限和额度。

| 字段 | LLM 示例 | 生图示例 |
| --- | --- | --- |
| `base_url` | `https://api.openai.com/v1` | `https://api.siliconflow.cn/v1` |
| `model` | `gpt-6.1-sol` | `Kwai-Kolors/Kolors` |
| `api` | `openai-responses`；Chat Completions 服务使用 `openai-compatible` | `siliconflow-images`；其他兼容服务使用 `openai-images` |
| `api_key` | 自己的 LLM Key | 自己的生图 Key |

先检查配置，再做只读连接探测：

```sh
uv run scripts/agent_e2e.py config-check
uv run scripts/agent_e2e.py run --agent workbuddy --case meme --probe --timeout 240
```

`config-check` 只检查字段、地址和文件权限，不调用模型、不显示密钥；它通过后仍需要真实探测和业务测试。缺少配置或字段时，脚本会提示模板位置和补充方法。通过 `--providers-config /仓库外/路径/providers.json` 可更换配置。

只跑 WorkBuddy PPT 或论文时可将生图 Key 留空，并使用 `config-check --case ppt` 或 `config-check --case paper`；普通生图和完整矩阵使用 `config-check --case meme` 或默认命令，必须配置生图 Key。

本机的 WorkBuddy 自定义模型和生图调用使用此配置。Codex、DSH 使用各自的模型配置与认证。三个 Agent 都需要当前本机的 LazyMind 登录。

## 前置条件

- 安装 `uv`，由脚本内声明安装 Python 3.11+、Pillow、PyYAML。
- 本机 LazyMind 已启动并登录；服务地址默认读取当前 `credentials.json`，仅接受回环地址。
- 默认使用测试目录下的专用 CLI，并在源码或二进制哈希变化时自动重建；需安装 `local/lazymind-cli/go.mod` 指定版本的 Go。显式传入 `--cli` 时保留指定路径。
- 凭证位于对应 `--home` 下的正常 `credentials.json`，本机身份位于 `connector-host-id`。
- Codex 已登录；WorkBuddy 已安装且其 CLI 能正常使用账号；DSH 已有可用模型配置。
- 三个工作流已发布：`ppt-workflow`、`paper-writing`、`image-workflow`。
- `--case meme` 保留旧参数名，现测试白色小猫、纯蓝背景、无指定动作和字幕的普通静态图片；不要求字幕适配器。更新测试脚本不会更新已安装应用，运行前应确认实际工作流 revision。

脚本为每个用例生成独立 MCP 配置，保留原有桌面配置。测试来源、MCP 调用记录、工作流 revision、运行 ID 和宿主会话 ID 分别记录。

## 使用

在仓库根目录运行：

```sh
uv run scripts/agent_e2e.py build-cli
uv run scripts/agent_e2e.py doctor

# 独立 DSH profile；保留 web/desktop/headless 原配置。
# 此选项通过正常 credentials 服务复用 web 的模型选择和已有登录。
uv run scripts/agent_e2e.py setup-dsh --dsh-use-stored-model

# 先验证三个真实 Agent 能调用 MCP 的发现与契约读取工具。
uv run scripts/agent_e2e.py run --agent all --case ppt --probe --timeout 180

# 单项真实生成，不传审核选项时停在人工审核边界。
uv run scripts/agent_e2e.py run --agent codex --case ppt

# 明确授权测试用户确认本次新建运行的固定测试材料。
# 只确认收到的 review_id、version、manifest_hash；不回答未知问题。
uv run scripts/agent_e2e.py run --agent all --case all --approve-test-reviews --timeout 2700

# 只在原测试监督进程已经结束后接续；保留原运行和宿主会话。
uv run scripts/agent_e2e.py resume --result outputs/agent-e2e/<本次用例目录>/result.json --approve-test-reviews

# 汇总连接探测和业务运行，保留全部历史尝试。
uv run scripts/agent_e2e.py report
```

`--cli`、`--home`、`--server`、`--output`、`--dsh-profile` 可覆盖默认值。为避免凭证误传，本地版仅接受回环地址。`--timeout` 为每个用例的执行预算；`--max-turns` 限制接续次数。宿主接续复用真实会话 ID，终态业务失败不会自动重试。

构建后默认优先使用测试输出目录下的专用 CLI。PPT 允许 Agent 使用本机已安装的确定性导出器和 Chromium；这属于宿主导出能力，不会将服务端导出失败改成通过。只有实际文件已发布并校验后才通过。不会自动安装渲染依赖或把 HTML 当作 PPTX 交付。

## 三个 Agent 共用的 CI 配置恢复

每个业务用例先启动独立只读会话，实际调用 `workflow.list` 和指定版本的 `workflow.get`。只有本次配置代次、实际工作流 ID 和 revision 都匹配才进入业务执行。首次探测遇到旧配置或连接异常时，重建本次专用配置并启动新进程再试一次；认证缺失直接阻塞，不靠重复登录或切换账号掩盖。

Codex 忽略日常 MCP 配置；WorkBuddy 使用内置正式启动入口、强制其轻量运行包和本次严格 MCP 配置；DSH 通过本次临时 patch 禁用继承的旧 MCP client / LazyMind Workflow host，再注入唯一的新连接。用户原有桌面配置和 DSH profile 文件保持原样。本次测试目录内被替换的配置保留 `before-*` 备份。

`connection.json` 只记录本次账号命名空间的哈希、当前服务地址、宿主 ID、连接器路径和哈希，不保存令牌。MCP 转发器在每次工具调用前检查这些绑定；账号、服务地址、连接器或配置代次变化时拒绝继续。只读探测禁止启动工作流，旧 revision 请求被拒绝。业务运行创建后，不因连接失败自动新建运行或重试业务步骤。

显式 `--server` 必须与当前凭据的服务地址一致；不一致时阻塞并报告 `credential_server_mismatch`。本次恢复只覆盖 CI 注入配置，不重连、重启或改写三个 Agent 的日常桌面连接器。

### WorkBuddy 独立 CLI 的认证与自定义模型

优先按上方模板配置自己的模型服务。CI 在仓库外生成临时 WorkBuddy profile，模型配置只引用环境变量；真实供应商 Key 由本机代理持有，WorkBuddy 只收到本次有效的回环地址和临时令牌。LLM 请求固定到配置的模型；普通生图通过同一代理调用配置的生图服务，并发布最终图片。日常 WorkBuddy 配置保持原样。

以下平台认证方式仅用于未提供私有模型配置的场景：

桌面已登录不等于 CI 独立 CLI 已获得认证。当前内置 CLI 有 API Key 认证模块，可通过进程环境注入有相应模型权限的 `CODEBUDDY_API_KEY`；脚本也透传正常的 `CODEBUDDY_AUTH_TOKEN` 和 `CODEBUDDY_INTERNET_ENVIRONMENT`。密钥和 Token 不写入本次生成的 MCP 配置或连接绑定，也不透传给 Codex、DSH 或监督进程的 MCP 子进程。

使用国内 CodeBuddy 平台 Key 时，官方文档要求 `CODEBUDDY_INTERNET_ENVIRONMENT=internal`。这是平台 CLI 的配置规则；该 Key 是否具备当前 WorkBuddy 模型和端点权限仍需真实探测，不能将普通 CodeBuddy CLI 的通过算作 WorkBuddy 通过。认证入口见 [官方身份和访问管理](https://www.workbuddy.ai/docs/zh/cli/iam)。不要从桌面加密凭据中手工提取 Token。

在本机 zsh 中输入凭据（不回显、不写入命令历史），先只做读取探测：

```sh
read -s 'CODEBUDDY_API_KEY?API Key: '
export CODEBUDDY_API_KEY
export CODEBUDDY_INTERNET_ENVIRONMENT=internal
uv run scripts/agent_e2e.py run --agent workbuddy --case meme --probe --timeout 180
```

只读探测实际通过后，再执行业务用例。GitHub Actions 的真实 E2E 执行环境应从 Secrets 注入同名变量，不在工作流、报告或聊天中填写 Key。本仓库新增的脚本单元测试 job 不需要这些 Secrets，也不调用模型。

不要在同一个测试目录同时启动多个完整矩阵。不同用例的产物位于唯一子目录；`report` 应在本轮测试结束后执行。

## 输出与判定

默认保存到已被本地忽略的 `outputs/agent-e2e/`：

- `doctor.json`：安装版本、MCP 工具和已发布工作流 revision。
- 每次尝试的 `result.json`：真实状态、失败原因、宿主会话、运行 ID、审核回执和产物检查。
- `mcp.jsonl`：仅记录工具名、调用成功与否、运行/revision ID 和精简错误。不保存完整参数、工具结果或模型推理。
- `connection.json`、`connection-attempts.json`：本次绑定、两次以内的探测与恢复证据；每次探测的配置和调用记录保存在独立子目录。
- `artifacts/`：本次测试发布的实际文件及校验信息。
- `report.md`、`summary.json`、`history.json`、`junit.xml`：最新组合结果以及历史失败记录。

连接探测必须由真实 Agent 成功调用 `workflow.list` 和 `workflow.get`，且不发生写操作。真实业务通过要求工作流确认完成，并存在已发布的必需产物且文件检查通过。进程退出 0、Agent 自报完成、仅生成本地文件均不算业务通过。

PPT 检查 PPTX 结构、页数和主题；论文检查已发布大纲与正文、篇幅和章节；普通生图检查文件解码、尺寸和帧数。文件结构检查不证明视觉、字幕或科学质量，报告会单独声明未验证项。

图片产物通过 `workflow.artifact.get result_directory` 取回完整发布值，校验交接文件和图片字节的哈希；MCP 的图片预览不作为下载成功证据。普通生图验收要求发布 `generated_image_output`，不接受仅有底图回执。旧运行仍绑定原版本。

DSH 当前测试模型仅支持文本输入，图片用例明确禁止 `read_image` 向模型发送图片；使用真实生图和文件解码检查；控制器不提供自动视觉验收，须人工检查归档图片，不把结构检查当成目视检查。

失败和阻塞返回非零退出码。JUnit 中阻塞列为 skipped，但汇总命令仍失败，避免缺账号、缺模型或待审核被误报为通过。

原有 `skip-worktree` 兼容性修改不会因 Git 状态干净而被忽略：报告记录这些文件的内容哈希，同时说明当前运行的是已有服务。报告和私有 MCP 配置只保存在本机，不应直接上传整个测试目录。监督进程有互斥锁；不得同时对一个测试结果发起两次接续。

## 验证脚本

```sh
uvx ruff check scripts/agent_e2e.py scripts/agent_e2e_providers.py scripts/tests/test_agent_e2e*.py
uv run --python 3.11 --with pytest python -m pytest tests/doc_check/ -q
uv run --python 3.11 --with Pillow==12.0.0 --with PyYAML==6.0.3 python -m unittest discover -s scripts/tests -p 'test_agent_e2e*.py' -v
```

GitHub CI 的 `test-agent-e2e-script` job 收集同一套脚本回归测试，只使用临时目录及受控进程，不依赖账号、不调用模型；真实 Agent 连接和业务生成仍由上面的本地/self-hosted 执行命令验证。

GitHub Actions 手动工作流使用相同入口；请使用专用测试账号和 runner，不上传登录配置、私有连接配置或原始宿主日志。运行中的 LazyMind 服务需由 runner 管理者部署和更新；报告中的 checkout SHA 不证明后台服务来自同一个 SHA。
