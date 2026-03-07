# Debian: 替换 Docker Registry Mirror（华为云）

## 适用场景
- Debian 系统已安装并启用 Docker。
- 需要将 `registry-mirrors` 统一替换为华为云镜像源。

## 目标镜像源
- `https://dfe2ea7390e64005a3e5cb8e1e590e91.mirror.swr.myhuaweicloud.com`

## 操作步骤
1. 备份当前 Docker 配置文件：
   ```bash
   sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak.$(date +%Y%m%d%H%M%S)
   ```
2. 写入新的镜像源配置：
   ```bash
   sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
   {
     "registry-mirrors": ["https://dfe2ea7390e64005a3e5cb8e1e590e91.mirror.swr.myhuaweicloud.com"]
   }
   EOF
   ```
3. 重载 systemd 配置并重启 Docker：
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```
4. 验证配置是否生效：
   ```bash
   docker info | grep -i "registry"
   docker info | sed -n '/Registry Mirrors/,+3p'
   ```

## 预期结果
- `docker info` 中出现：
  - `Registry Mirrors:`
  - `https://dfe2ea7390e64005a3e5cb8e1e590e91.mirror.swr.myhuaweicloud.com/`

## 回滚（可选）
如需回滚，使用最新备份恢复后重启 Docker：
```bash
sudo cp /etc/docker/daemon.json.bak.<timestamp> /etc/docker/daemon.json
sudo systemctl daemon-reload
sudo systemctl restart docker
```
