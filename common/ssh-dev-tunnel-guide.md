# SSH 开发隧道说明（脱敏版）

## 0. 不用脚本，直接使用（启动 + 停止）

### 0.1 直接启动（写 PID 文件）

```bash
mkdir -p .cache
ssh -N \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -p {ssh_port} \
  -L {local_keycloak_port}:127.0.0.1:{remote_keycloak_port} \
  -L {local_openfga_port}:127.0.0.1:{remote_openfga_port} \
  -L {local_pg_port}:127.0.0.1:{remote_pg_port} \
  -L {local_ollama_port}:127.0.0.1:{remote_ollama_port} \
  -L {local_runtime_port}:127.0.0.1:{remote_runtime_port} \
  {ssh_user}@{ip} > /dev/null 2>&1 &
echo $! > .cache/dev_tunnel_ssh.pid
```

停止：

```bash
kill "$(cat .cache/dev_tunnel_ssh.pid)" && rm -f .cache/dev_tunnel_ssh.pid
```

### 0.2 直接启动（不写 PID 文件，推荐）

```bash
mkdir -p .cache
SOCK=.cache/dev_tunnel.sock
ssh -M -S "$SOCK" -fN \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -p {ssh_port} \
  -L {local_keycloak_port}:127.0.0.1:{remote_keycloak_port} \
  -L {local_openfga_port}:127.0.0.1:{remote_openfga_port} \
  -L {local_pg_port}:127.0.0.1:{remote_pg_port} \
  -L {local_ollama_port}:127.0.0.1:{remote_ollama_port} \
  -L {local_runtime_port}:127.0.0.1:{remote_runtime_port} \
  {ssh_user}@{ip}
```

停止：

```bash
SOCK=.cache/dev_tunnel.sock
ssh -S "$SOCK" -O exit {ssh_user}@{ip}
rm -f "$SOCK"
```

兜底（按端口结束监听进程）：

```bash
lsof -tiTCP:{local_keycloak_port} -sTCP:LISTEN | xargs -r kill
```

## 1. SSH 隧道是什么

SSH 本地端口转发（`ssh -L`）可以理解成：

- 你访问本机 `127.0.0.1:{local_port}`
- SSH 把流量安全转发到远端机器的 `127.0.0.1:{remote_port}`

本质是通过 SSH 会话临时建立专用通信通道，而不是把远端服务直接暴露到公网。

## 2. 端口映射关系（示例模板）

| 本地地址 | 远端地址 | 服务 |
| --- | --- | --- |
| `127.0.0.1:{local_keycloak_port}` | `127.0.0.1:{remote_keycloak_port}` | Keycloak |
| `127.0.0.1:{local_openfga_port}` | `127.0.0.1:{remote_openfga_port}` | OpenFGA |
| `127.0.0.1:{local_pg_port}` | `127.0.0.1:{remote_pg_port}` | PostgreSQL |
| `127.0.0.1:{local_ollama_port}` | `127.0.0.1:{remote_ollama_port}` | Ollama |
| `127.0.0.1:{local_runtime_port}` | `127.0.0.1:{remote_runtime_port}` | Runtime |

## 3. 前置条件

- 本机可用 `ssh` 命令。
- 本机可用 `lsof`（用于排查端口占用/查 PID）。
- 你能正常登录远端：

```bash
ssh -p {ssh_port} {ssh_user}@{ip}
```

- 远端对应服务已经启动并监听在预期端口。

## 4. 如何按需修改参数

### 4.1 修改 SSH 主机 / 端口 / 用户

```bash
ssh -N -p {ssh_port} -L {local_port}:127.0.0.1:{remote_port} {ssh_user}@{ip}
```

### 4.2 修改本地监听端口（避免冲突）

```bash
ssh -N -p {ssh_port} -L {new_local_port}:127.0.0.1:{remote_port} {ssh_user}@{ip}
```

### 4.3 同时转发多个端口

```bash
ssh -N -p {ssh_port} \
  -L {local_port_a}:127.0.0.1:{remote_port_a} \
  -L {local_port_b}:127.0.0.1:{remote_port_b} \
  {ssh_user}@{ip}
```

## 5. 如何确认隧道是否生效

- 查看本地端口监听：

```bash
lsof -iTCP:{local_keycloak_port} -sTCP:LISTEN
```

- 直接访问本地映射地址（按实际服务接口）：

```bash
curl -fsS http://127.0.0.1:{local_openfga_port}/healthz
curl -fsS http://127.0.0.1:{local_runtime_port}/info
```

如果本地端口在监听但访问失败，通常是远端服务未启动或远端端口不正确。

## 6. 常见问题排查

### 6.1 提示端口被占用

表现：`bind: Address already in use`

处理：

- 释放被占用端口。
- 或改用新的本地端口（并同步修改你的访问地址）。

### 6.2 SSH 命令退出或频繁断开

建议增加：

- `-o ServerAliveInterval=30`
- `-o ServerAliveCountMax=3`

### 6.3 隧道已建立但服务仍不可用

优先检查：

- 远端服务是否真的启动。
- 远端服务监听端口是否与你写的 `-L` 目标端口一致。
- 是否误把远端 `127.0.0.1` 写成了其他地址。

## 7. 安全建议

- 只开放 SSH 端口，不直接开放业务端口到公网。
- 隧道只在开发联调期间临时启用，用完及时关闭。
- 使用最小权限账号，避免长期使用高权限账号做日常联调。

## 8. 一句话总结

SSH 隧道不是“把远端服务暴露出来”，而是“把访问能力临时、安全地带到本机端口”。
