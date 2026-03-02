# Codex 全局高权限可复刻配置手册

本文档用于在 Linux 服务器上把 Codex 配置为“尽可能高权限”默认模式，减少每次执行命令时的手动确认。

适用目标：
- 默认自动批准（不逐条弹确认）
- 默认全盘文件访问
- 默认允许联网检索

---

## 1. 核心结论（必须先知道）

Codex 权限有多层优先级，通常是：

1. 会话/平台强制策略（最高优先级）
2. 启动命令参数
3. 全局配置文件（`~/.codex/config.toml`）

这意味着：
- 即使你把全局配置写成最高权限，如果当前会话被平台强制为 `on-request`，仍然会弹确认。
- 想彻底生效，必须“新开会话”并确保启动参数/平台策略不覆盖全局配置。

---

## 2. 全局配置文件位置

默认路径：

```bash
~/.codex/config.toml
```

root 用户通常是：

```bash
/root/.codex/config.toml
```

---

## 3. 推荐的全局高权限配置

将以下内容写入 `~/.codex/config.toml`（按你的 provider/model 调整）：

```toml
model_provider = "fluxcode"
model = "gpt-5.3-codex"
model_reasoning_effort = "medium"

[model_providers.fluxcode]
name = "fluxcode"
base_url = "https://flux-code.cc"
wire_api = "responses"
requires_openai_auth = true

approval_policy = "never"
sandbox_mode = "danger-full-access"
web_search = "live"

[tools]
view_image = true
```

说明：
- `approval_policy = "never"`：默认不再逐条请求批准
- `sandbox_mode = "danger-full-access"`：文件系统最高权限
- `web_search = "live"`：启用实时联网检索能力

---

## 4. 一键写入配置（可直接执行）

```bash
mkdir -p ~/.codex
cp -a ~/.codex/config.toml ~/.codex/config.toml.bak.$(date +%Y%m%d%H%M%S) 2>/dev/null || true

cat > ~/.codex/config.toml <<'TOML'
model_provider = "fluxcode"
model = "gpt-5.3-codex"
model_reasoning_effort = "medium"

[model_providers.fluxcode]
name = "fluxcode"
base_url = "https://flux-code.cc"
wire_api = "responses"
requires_openai_auth = true

approval_policy = "never"
sandbox_mode = "danger-full-access"
web_search = "live"

[tools]
view_image = true
TOML
```

---

## 5. 启动会话时的关键要求

仅改配置文件不一定够。必须确保你新启动会话时也使用高权限策略：

- `approval = never`
- `sandbox = danger-full-access`
- `network = enabled`（如果你的客户端有该选项）

不同客户端参数名可能略有差异，请以实际 CLI/UI 为准。

---

## 6. 生效验证

### 6.1 验证配置文件

```bash
cat ~/.codex/config.toml
```

应看到：
- `approval_policy = "never"`
- `sandbox_mode = "danger-full-access"`
- `web_search = "live"`

### 6.2 验证当前会话实际策略

在 Codex 会话日志或上下文里确认：
- `approval_policy` 是否为 `never`
- `sandbox_policy.type` 是否为 `danger-full-access`
- `network_access` 是否为 `true`

如果这里显示 `on-request/workspace-write/network=false`，说明会话被覆盖，和配置文件无关。

---

## 7. 常见问题

### Q1: 明明配置了 `never`，为什么还会弹确认？

原因：当前会话被启动参数或平台策略覆盖。  
处理：结束当前会话，按高权限参数重新创建新会话。

### Q2: 联网搜索还是不可用？

原因：平台层把网络关了（`network_access=false`）。  
处理：在启动会话时显式开启网络，或联系管理员解除策略限制。

### Q3: 团队环境能否彻底关闭审批？

不一定。组织管理员可以强制策略，个人配置无法绕过。

---

## 8. 最小复刻清单（给他人）

把下面 4 项交给对方即可完整复刻：

1. `~/.codex/config.toml` 模板（本文件第 3 节）
2. 新建会话时使用的高权限启动参数（本文件第 5 节）
3. 生效验证命令（本文件第 6 节）
4. 常见问题判定逻辑（本文件第 7 节）

