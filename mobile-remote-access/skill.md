# Skill: Tailscale SSH 远程连接配置

> 帮助用户在 macOS 上配置 Tailscale，实现手机远程 SSH 连接

## 基本信息

| 属性 | 值 |
|------|-----|
| **Skill ID** | tailscale-ssh-setup |
| **名称** | Tailscale SSH 远程连接配置 |
| **适用平台** | macOS + iOS/Android |
| **难度** | 中等 |
| **预计时间** | 10-15 分钟 |

## 前置条件

- [ ] macOS 10.15+ (Intel 或 Apple Silicon)
- [ ] 具有管理员权限的用户账户
- [ ] 已安装 Homebrew（如未安装，先执行 `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`）
- [ ] 手机端可安装 Tailscale App

## 执行步骤

### Step 1: 环境检查

```bash
# 检查系统版本
sw_vers -productVersion

# 检查 Homebrew
which brew && brew --version

# 检查当前用户
whoami
echo "当前用户: $(whoami)"
```

**预期输出**: macOS 版本 10.15+，Homebrew 已安装

---

### Step 2: 安装 Tailscale

```bash
# 使用 Homebrew 安装
brew install tailscale

# 验证安装
which tailscale
tailscale --version
```

**预期输出**: 显示 tailscale 路径和版本号

---

### Step 3: 启动 Tailscale 服务

```bash
# 启动系统服务（需要密码）
sudo tailscaled install-system-daemon

# 登录（会弹出浏览器窗口）
tailscale up

# 等待用户完成浏览器登录后，检查状态
tailscale status
```

**预期输出**: 
- 显示当前设备信息
- 分配到的 IP 地址（100.x.x.x）
- 状态为 "Connected"

**获取关键信息**:
```bash
# 记录这个 IP，后续需要
LOCAL_IP=$(tailscale ip -4)
echo "Tailscale IP: $LOCAL_IP"
```

---

### Step 4: 开启 SSH 服务

```bash
# 方法 1：命令行开启
sudo systemsetup -setremotelogin on

# 或者方法 2：检查是否已开启
sudo systemsetup -getremotelogin
```

**预期输出**: "Remote Login: On"

**验证 SSH 是否工作**:
```bash
# 本地测试 SSH
ssh localhost -o ConnectTimeout=5
# 输入密码后能登录即成功
# 按 Ctrl+D 退出
```

---

### Step 5: 配置开机自启（PM2）

```bash
# 安装 PM2
brew install pm2

# 创建 Tailscale 守护脚本
TAILSCALE_KEEPER="$HOME/tailscale-keeper.sh"

cat > "$TAILSCALE_KEEPER" << 'EOF'
#!/bin/bash
LOG_FILE="$HOME/.pm2/logs/tailscale-keeper.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "Tailscale Keeper 启动..."
log "当前用户: $(whoami)"

while true; do
    if tailscale status &>/dev/null; then
        CURRENT_IP=$(tailscale ip -4 2>/dev/null)
        log "✓ Tailscale 连接正常 - IP: $CURRENT_IP"
    else
        log "✗ Tailscale 未连接，尝试重新连接..."
        if tailscale up --accept-routes 2>&1 | tee -a "$LOG_FILE"; then
            NEW_IP=$(tailscale ip -4 2>/dev/null)
            log "✓ 重新连接成功 - IP: $NEW_IP"
        else
            log "✗ 重新连接失败，30秒后重试..."
        fi
    fi
    sleep 60
done
EOF

chmod +x "$TAILSCALE_KEEPER"

# 使用 PM2 启动
pm2 start "$TAILSCALE_KEEPER" --name tailscale-keeper

# 保存 PM2 配置
pm2 save
```

**预期输出**: 
- PM2 显示 tailscale-keeper 状态为 "online"
- 日志显示 "Tailscale 连接正常"

---

### Step 6: 设置 PM2 开机自启

```bash
# 生成启动脚本
pm2 startup

# 根据提示执行生成的命令（通常是）：
# sudo env PATH=$PATH:/opt/homebrew/bin pm2 startup launchd -u $(whoami) --hp $HOME
```

**验证开机自启**:
```bash
# 检查 LaunchAgents
ls ~/Library/LaunchAgents/ | grep pm2

# 应该存在类似 com.pm2.user.plist 的文件
```

---

### Step 7: 生成连接信息

```bash
# 收集连接信息
echo ""
echo "═══════════════════════════════════════════════════"
echo "📱 手机 SSH 连接信息"
echo "═══════════════════════════════════════════════════"
echo "主机: $(tailscale ip -4)"
echo "端口: 22"
echo "用户名: $(whoami)"
echo "密码: [你的 macOS 登录密码]"
echo "═══════════════════════════════════════════════════"
echo ""
echo "手机端配置步骤:"
echo "1. 安装 Tailscale App"
echo "2. 使用相同账号登录"
echo "3. 使用上述信息配置 SSH 客户端"
```

---

## 验证检查点

| 检查项 | 命令 | 预期结果 |
|--------|------|---------|
| Tailscale 安装 | `which tailscale` | 显示路径 |
| 服务运行 | `tailscale status` | 显示 IP，无错误 |
| SSH 开启 | `sudo lsof -i :22` | sshd 进程在监听 |
| PM2 运行 | `pm2 status` | tailscale-keeper online |
| 网络连通 | `tailscale status | grep -E "direct|active"` | 显示 active |

---

## 可选配置：Exit Node

如果用户需要让手机流量经过 Mac（翻墙场景）：

```bash
# 开启 Exit Node
sudo tailscale up --advertise-exit-node --accept-routes

# 提示用户：
echo "请在 https://login.tailscale.com/admin/machines 开启 'Use as exit node'"
echo "然后在手机 Tailscale App 中选择 'Use as exit node'"
```

---

## 故障处理

### 问题 1: tailscale 命令找不到

**解决**:
```bash
# 重新安装
brew reinstall tailscale

# 检查 PATH
export PATH="/opt/homebrew/bin:$PATH"
```

### 问题 2: SSH 连接失败

**排查步骤**:
```bash
# 1. 检查 SSH 服务
sudo lsof -i :22

# 2. 检查防火墙
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
# 如开启，尝试关闭或添加例外

# 3. 检查 Tailscale 连接
tailscale status
```

### 问题 3: PM2 启动失败

**解决**:
```bash
# 检查 PM2 日志
pm2 logs tailscale-keeper --lines 50

# 重启
pm2 restart tailscale-keeper
```

### 问题 4: 手机无法发现电脑

**解决**:
```bash
# 检查电脑 Tailscale 状态
tailscale status

# 如显示离线，重新登录
tailscale up

# 确保手机和电脑使用同一账号登录 Tailscale
```

---

## 交互变量

执行过程中需要向用户获取的信息：

| 变量名 | 说明 | 获取方式 |
|--------|------|---------|
| `USER_PASSWORD` | macOS 登录密码 | 询问用户（sudo 需要） |
| `TAILSCALE_AUTH` | Tailscale 登录授权 | 浏览器弹窗，用户手动完成 |
| `PHONE_SSH_APP` | 手机 SSH 客户端选择 | 询问用户：Termius/Prompt/其他 |

---

## 成功后输出模板

```
✅ Tailscale SSH 配置完成！

📱 手机连接信息：
   主机: 100.x.x.x
   端口: 22
   用户名: [用户名]
   密码: [系统密码]

🔧 管理命令：
   tailscale status      # 查看状态
   pm2 status            # 查看进程
   pm2 logs tailscale-keeper  # 查看日志

💡 使用场景：
   - 手机 SSH 远程执行命令
   - 使用 Kimi CLI 等 AI 工具
   - 远程重启 OpenClaw 等服务
   - 随时随地管理你的 Mac
```

---

## 参考

- 完整教程：https://github.com/jqlong17/tutorials/blob/main/mobile-remote-access/tailscale-ssh-mac.md
- Tailscale 文档：https://tailscale.com/kb
- macOS 远程登录：https://support.apple.com/guide/mac-help/allow-a-remote-computer-to-access-your-mac-mchlp1066/mac

---

**版本**: 1.0  
**更新日期**: 2026-02-20  
**作者**: @ruska
