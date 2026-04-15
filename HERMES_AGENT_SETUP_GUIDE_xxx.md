# Hermes Agent 配置指南

本指南详细说明如何安装和配置 Hermes Agent，特别是使用自定义 OpenAI 兼容端点（如 vLLM、Ollama、自托管服务等）的场景。

## 前提条件

- Python 3.10+
- Git
- 自定义 OpenAI 兼容模型端点信息：
  - **base_url**: API 端点地址（如 `https://your-server/v1`）
  - **api_key**: API 密钥（如果是本地服务可能不需要）
  - **model**: 模型名称（如 `minimax-m2.5`）

## 快速安装

```bash
# 1. 克隆仓库
cd ~/development
git clone https://github.com/nousresearch/hermes-agent.git
cd hermes-agent

# 2. 安装依赖
pip install -e .
pip install openai

# 3. 验证安装
python hermes --version
```

## 配置自定义模型端点

### 重要说明：配置文件格式

Hermes Agent 的配置文件 `~/.hermes/config.yaml` 使用 YAML 格式。**配置自定义端点时，必须使用正确的字段名称和结构**。

官方文档参考：https://hermes-agent.nousresearch.com/docs/integrations/providers

### 关键配置字段

| 字段 | 类型 | 说明 | 是否必需 |
|------|------|------|----------|
| `model.default` | string | 默认模型名称 | **必需** |
| `model.provider` | string | 提供商类型，自定义端点必须设为 `"custom"` | **必需** |
| `model.base_url` | string | API 端点地址（OpenAI 兼容格式，以 `/v1` 结尾） | **必需** |
| `model.api_key` | string | API 密钥，可直接写在这里或引用环境变量 | 可选（本地服务可能不需要） |
| `model.context_length` | int | 上下文窗口大小（tokens） | 可选（默认自动检测） |
| `model.max_tokens` | int | 单次响应最大输出 tokens | 可选 |

**注意事项：**
- 字段名使用 **snake_case**（如 `base_url`），不是 camelCase（如 `baseUrl`）
- `provider: custom` 是关键字段，告诉 Hermes 这是一个自定义 OpenAI 兼容端点
- 如果不设置 `provider: custom`，Hermes 会尝试通过 OpenRouter 访问，导致 401 错误

### 配置步骤

#### 1. 创建配置目录

```bash
mkdir -p ~/.hermes
```

#### 2. 创建主配置文件 (~/.hermes/config.yaml)

```yaml
# Hermes Agent 配置文件
# 用于自定义 OpenAI 兼容端点（vLLM、Ollama 等）

model:
  # 模型名称 - 与端点支持的模型名称一致
  default: "minimax-m2.5"

  # 提供商类型 - 自定义端点必须设置为 "custom"
  # 可选值: "auto", "openrouter", "nous", "anthropic", "custom", "gemini", "zai", "kimi-coding", "minimax" 等
  # "custom" = 任何 OpenAI 兼容端点（vLLM、Ollama、LM Studio、llama.cpp 等）
  provider: custom

  # API 端点地址 - OpenAI 兼容格式，通常以 /v1 结尾
  base_url: "https://your-endpoint.example.com/v1"

  # API 密钥 - 可直接写值，或引用环境变量 ${OPENAI_API_KEY}
  api_key: "YOUR_API_KEY_HERE"

  # 上下文窗口大小（可选，默认自动检测）
  context_length: 128000

  # 单次响应最大输出 tokens（可选）
  max_tokens: 16384

# 内存配置
memory:
  enabled: true
  search_enabled: true

# 工具配置
tools:
  web_access: true
  file_operations: true
  code_execution: true

# 上下文压缩配置（默认禁用）
# 注意：启用压缩需要配置辅助 LLM 提供商（见下文）
compression:
  enabled: false
```

#### 3. 设置环境变量（可选）

如果不想在 config.yaml 中直接写 API 密钥，可以使用环境变量：

```bash
# 创建 ~/.hermes/.env 文件
cat > ~/.hermes/.env << 'EOF'
# 主模型 API 密钥（可选，如果 config.yaml 中已配置则不需要）
OPENAI_API_KEY="YOUR_API_KEY_HERE"

# OpenRouter API 密钥（用于辅助功能，见下文）
# OPENROUTER_API_KEY="sk-or-..."
EOF

chmod 600 ~/.hermes/.env
```

然后在 config.yaml 中引用：

```yaml
model:
  api_key: "${OPENAI_API_KEY}"
```

#### 4. 验证配置

```bash
cd ~/development/hermes-agent
python hermes config

# 检查配置是否完整
python hermes doctor
```

## 启动 Hermes

```bash
# 方法1：直接运行（默认使用当前目录作为工作目录）
cd ~/development/hermes-agent
python hermes  # 终端工作目录 = ~/development/hermes-agent

# 方法2：更方便的方式
# 在任意目录启动 hermes，终端工作目录就是该目录
cd ~/development/xxx
python ~/development/hermes-agent/hermes  # 终端工作目录 = ~/development/xxx
```

### 配置终端工作目录为当前目录（推荐）

在 `~/.hermes/config.yaml` 中添加：

```yaml
terminal:
  backend: local
  cwd: "."  # 使用当前目录（即启动 hermes 时的 pwd）
  timeout: 180
```

这样无论你在哪个目录启动 hermes，终端工具都会使用该目录作为工作目录。

**示例：**

```bash
# 在项目目录启动
cd ~/development/xxx
python ~/development/hermes-agent/hermes

# hermes 启动后，终端命令会在 ~/development/xxx 下执行
# pwd 显示的就是 ~/development/xxx
```

## 辅助模型配置（可选但重要）

### 什么是辅助模型？

Hermes 使用辅助模型处理以下任务：
- **context compression**: 上下文压缩/摘要
- **vision**: 图像分析
- **web_extract**: 网页内容摘要
- **approval**: 危险命令审批判断

### 为什么需要配置？

如果只配置了自定义端点而没有配置辅助模型：
- 压缩功能会尝试使用 OpenRouter，但没有 `OPENROUTER_API_KEY` 会导致 401 错误
- 图像分析、网页摘要等功能无法使用

### 配置选项

#### 选项 1：禁用压缩（最简单）

```yaml
compression:
  enabled: false
```

这是默认设置，可以避免 401 错误，正常使用 Hermes 进行对话。

#### 选项 2：使用 OpenRouter 作为辅助模型（推荐）

```bash
# 1. 获取 OpenRouter API 密钥: https://openrouter.ai/keys

# 2. 添加到 ~/.hermes/.env
echo 'OPENROUTER_API_KEY="sk-or-your-key"' >> ~/.hermes/.env

# 3. 在 config.yaml 中配置
```

```yaml
compression:
  enabled: true
  threshold: 0.50  # 当上下文达到 50% 时触发压缩
  target_ratio: 0.20
  protect_last_n: 20

auxiliary:
  compression:
    provider: openrouter
    model: "google/gemini-3-flash-preview"  # 推荐，便宜且快速
  vision:
    provider: openrouter
    model: "openai/gpt-4o"  # 支持视觉的模型
  web_extract:
    provider: openrouter
    model: "google/gemini-3-flash-preview"
```

#### 选项 3：使用同一自定义端点作为辅助模型

如果你的自定义端点支持所有必要的功能：

```yaml
auxiliary:
  compression:
    provider: custom
    base_url: "${model.base_url}"  # 引用主模型的 base_url
    api_key: "${model.api_key}"
    model: "minimax-m2.5"  # 或其他适合摘要的模型
```

注意：辅助模型需要有足够大的上下文窗口来处理摘要任务。

#### 选项 4：使用 provider: "main"

`provider: "main"` 表示使用主模型的配置：

```yaml
auxiliary:
  compression:
    provider: main  # 使用主模型的端点和密钥
    model: "minimax-m2.5"
```

但注意：这要求你的主模型能胜任辅助任务。

## 支持的提供商列表

| provider 值 | 说明 | 环境变量 |
|-------------|------|----------|
| `auto` | 自动检测（默认） | - |
| `openrouter` | OpenRouter 多模型路由 | `OPENROUTER_API_KEY` |
| `nous` | Nous Portal OAuth | `hermes auth` |
| `anthropic` | Anthropic Claude API | `ANTHROPIC_API_KEY` |
| `gemini` | Google AI Studio | `GOOGLE_API_KEY` 或 `GEMINI_API_KEY` |
| `custom` | **自定义 OpenAI 兼容端点** | 在 config.yaml 配置 |
| `zai` | z.ai / ZhipuAI GLM | `GLM_API_KEY` |
| `kimi-coding` | Kimi / Moonshot | `KIMI_API_KEY` |
| `minimax` | MiniMax Global | `MINIMAX_API_KEY` |
| `minimax-cn` | MiniMax China | `MINIMAX_CN_API_KEY` |
| `huggingface` | HF Inference Providers | `HF_TOKEN` |
| `copilot` | GitHub Copilot | `COPILOT_GITHUB_TOKEN` 或 `gh auth token` |

**自定义端点的别名：** `lmstudio`, `ollama`, `vllm`, `llamacpp` 都映射到 `custom`。

## 常见错误及解决方案

### 错误 1: 401 Unauthorized (OpenRouter)

```
⚠️  API call failed (attempt 1/3): AuthenticationError [HTTP 401]
   🔌 Provider: openrouter  Model: xxx
   🌐 Endpoint: https://openrouter.ai/api/v1
```

**原因：**
- 配置文件中没有设置 `provider: custom`
- Hermes 将模型名称误认为 OpenRouter 模型，尝试通过 OpenRouter 访问

**解决方案：**
```yaml
model:
  provider: custom  # 必须明确指定！
  base_url: "https://your-endpoint/v1"
```

### 错误 2: 401 Unauthorized (压缩功能)

```
⚠ No auxiliary LLM provider configured — context compression will drop middle turns without a summary.
```

**原因：** 启用了压缩但没有配置辅助 LLM 提供商

**解决方案：**
```yaml
# 方案 A: 禁用压缩
compression:
  enabled: false

# 方案 B: 配置 OpenRouter 作为辅助
# 在 .env 中添加 OPENROUTER_API_KEY
auxiliary:
  compression:
    provider: openrouter
```

### 错误 3: 字段名大小写错误

```yaml
# 错误格式 ❌
model:
  providerName: "custom-openai"  # 错误字段名
  baseUrl: "..."                  # 错误，应该是 base_url
  contextWindow: 128000           # 错误，应该是 context_length

# 正确格式 ✅
model:
  provider: custom                # 正确
  base_url: "..."                 # 正确（snake_case）
  context_length: 128000          # 正确（snake_case）
```

### 错误 4: 模型名称不匹配

确保 `model.default` 的值与你的端点实际支持的模型名称完全一致。可以通过访问端点的 `/v1/models` 端点查看：

```bash
curl https://your-endpoint/v1/models -H "Authorization: Bearer YOUR_API_KEY"
```

### 错误 5: 上下文窗口太小

如果模型响应不连贯或忘记上下文：

```yaml
model:
  context_length: 32768  # 至少 16k-32k 用于 agent 使用
```

注意：Ollama 默认上下文很小（可能只有 4k），需要在服务端配置：
```bash
OLLAMA_CONTEXT_LENGTH=32768 ollama serve
```

## 配置文件完整示例

### 示例 1：使用自定义 vLLM 端点

```yaml
model:
  default: "Qwen/Qwen2.5-72B-Instruct"
  provider: custom
  base_url: "https://your-vllm-server.example.com/v1"
  api_key: "your-api-key"
  context_length: 131072
  max_tokens: 8192

compression:
  enabled: false

memory:
  enabled: true

terminal:
  backend: local
  timeout: 180
```

### 示例 2：使用本地 Ollama

```yaml
model:
  default: "qwen2.5-coder:32b"
  provider: custom
  base_url: "http://localhost:11434/v1"
  # Ollama 本地服务通常不需要 API 密钥
  context_length: 32768

compression:
  enabled: false

terminal:
  backend: local
```

### 示例 3：自定义端点 + OpenRouter 辅助

```yaml
model:
  default: "minimax-m2.5"
  provider: custom
  base_url: "https://your-endpoint.example.com/v1"
  api_key: "your-main-api-key"
  context_length: 128000

compression:
  enabled: true
  threshold: 0.50

auxiliary:
  compression:
    provider: openrouter
    model: "google/gemini-3-flash-preview"
  vision:
    provider: openrouter
    model: "openai/gpt-4o"

# .env 文件需要:
# OPENROUTER_API_KEY=sk-or-your-key
```

## 配置优先级

1. CLI 参数（最高优先级）：`hermes --model xxx --provider custom`
2. `~/.hermes/config.yaml` 配置文件
3. `~/.hermes/.env` 环境变量
4. 内置默认值（最低优先级）

**规则：**
- API 密钥等敏感信息建议放在 `.env` 文件，权限设为 600
- 非敏感配置放在 `config.yaml`
- 可以在 `config.yaml` 中用 `${VAR_NAME}` 引用 `.env` 中的变量

## 目录结构

```
~/.hermes/
├── config.yaml          # 主配置文件（非敏感设置）
├── .env                 # 环境变量（API密钥等敏感信息，权限 600）
├── auth.json            # OAuth 认证信息（Nous Portal, Codex 等）
├── SOUL.md              # AI 身份/人格定义
├── memories/            # 持久化记忆
│   ├── MEMORY.md        # 对话记忆
│   └── USER.md          # 用户档案
├── skills/              # 自定义技能
├── sessions/            # 会话数据
├── cron/                # 定时任务
└── logs/                # 日志文件
    └── errors.log       # 错误日志
```

## 常用命令

### 启动和对话

```bash
hermes                          # 启动交互式对话
hermes --model xxx              # 使用指定模型启动
hermes --provider custom        # 强制使用自定义端点
hermes --resume SESSION_ID      # 恢复之前的会话
```

### 配置管理

```bash
hermes config                   # 查看当前配置
hermes config edit              # 编辑配置文件
hermes config set model.provider custom   # 设置配置值
hermes config set model.base_url "http://localhost:11434/v1"
hermes doctor                   # 系统诊断
```

### 模型管理

```bash
hermes model                    # 交互式选择模型/提供商
/model                          # 在对话中切换模型
/model custom                   # 切换到自定义端点的模型
/model openrouter:claude-sonnet-4  # 切换到 OpenRouter 模型
```

### 会话管理

```bash
hermes sessions                 # 列出所有会话
hermes sessions export ID       # 导出会话
```

## 调试技巧

```bash
# 启用详细日志
export HERMES_LOG_LEVEL=DEBUG
python hermes

# 查看错误日志
tail -f ~/.hermes/logs/errors.log

# 测试端点连通性
curl https://your-endpoint/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY"

# 检查配置文件语法
python -c "import yaml; yaml.safe_load(open('~/.hermes/config.yaml'))"
```

## 参考资料

- **官方文档**: https://hermes-agent.nousresearch.com/docs/
- **提供商配置**: https://hermes-agent.nousresearch.com/docs/integrations/providers
- **配置详解**: https://hermes-agent.nousresearch.com/docs/user-guide/configuration
- **GitHub**: https://github.com/nousresearch/hermes-agent
- **示例配置**: ~/development/hermes-agent/cli-config.yaml.example

---
