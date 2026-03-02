# UV + Python 3.13 统一虚拟环境复刻指南

本文档用于在一台新的 Linux 服务器上，复刻与当前机器一致的 Python 环境方案：
- 安装 `uv`
- 创建统一虚拟环境：`/opt/venvs/main/py13`
- 配置快捷别名：`v13in` / `v13out`
- 安装一组通用依赖

## 1. 安装 uv

优先使用官方安装脚本：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

把 `uv` 加入 PATH（当前会话 + 永久）：

```bash
export PATH="$HOME/.local/bin:$PATH"
grep -q 'export PATH="$HOME/.local/bin:$PATH"' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

验证：

```bash
which uv
uv --version
```

## 2. 创建 Python 3.13 虚拟环境

推荐使用你服务器里现有的 `python3.13`：

```bash
mkdir -p /opt/venvs/main
python3.13 -m venv /opt/venvs/main/py13
```

如果 `python3.13` 不存在，可先装解释器后再建环境（uv 方式）：

```bash
uv python install 3.13
uv venv /opt/venvs/main/py13 --python 3.13
```

验证：

```bash
/opt/venvs/main/py13/bin/python --version
/opt/venvs/main/py13/bin/pip --version
```

## 3. 配置进入/退出虚拟环境别名

写入 `~/.bashrc`：

```bash
grep -q "alias v13in='source /opt/venvs/main/py13/bin/activate'" ~/.bashrc || echo "alias v13in='source /opt/venvs/main/py13/bin/activate'" >> ~/.bashrc
grep -q "alias v13out='deactivate'" ~/.bashrc || echo "alias v13out='deactivate'" >> ~/.bashrc
source ~/.bashrc
```

使用：

```bash
v13in
v13out
```

## 4. 在该虚拟环境中安装依赖（等价替代 uv add）

说明：`uv add` 主要用于“项目依赖声明（pyproject.toml）”。  
你这里是“统一环境预装包”，应使用 `uv pip install`。

执行：

```bash
uv pip install --python /opt/venvs/main/py13/bin/python -U \
  langchain \
  langchain-mcp-adapters \
  langgraph \
  langchain-community \
  pypdf \
  "langchain[openai]" \
  langgraph-cli \
  "langgraph-cli[inmem]" \
  deepagents \
  langchain-deepseek \
  fastapi \
  "uvicorn[standard]" \
  requests \
  python-dotenv \
  langgraph-checkpoint-postgres \
  "psycopg[binary]" \
  langchain-openai
```

## 5. 常见问题

### Q1: 激活虚拟环境后 `uv: command not found`

原因：`uv` 在 `~/.local/bin/uv`，但 PATH 没带上 `~/.local/bin`。  
修复：

```bash
export PATH="$HOME/.local/bin:$PATH"
hash -r
uv --version
```

### Q2: 其他项目能否共用这些依赖？

可以。前提是项目运行时使用同一个解释器：

```bash
/opt/venvs/main/py13/bin/python your_script.py
```

或者先 `v13in` 再 `python your_script.py`。

## 6. 快速自检清单

```bash
which uv
uv --version
/opt/venvs/main/py13/bin/python --version
source /opt/venvs/main/py13/bin/activate
python -c "import langchain,fastapi,requests; print('ok')"
deactivate
```

如果以上全部成功，说明复刻完成。
