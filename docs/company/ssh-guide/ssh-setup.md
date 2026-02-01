# SSH 安全配置指南

## 设备信息

| 设备 | IP 地址 | 用户名 | 用途 |
|------|---------|--------|------|
| Mac Mini | `192.168.x.x` | apple | 服务器 |
| MacBook | - | apple | 客户端 |

---

## 1. 客户端（MacBook）配置

### 1.1 检查现有 SSH 密钥

```bash
ls ~/.ssh/*.pub
```

### 1.2 复制公钥到服务器

```bash
# 默认密钥
ssh-copy-id apple@mini的ip

# 自定义密钥
ssh-copy-id -i ~/.ssh/你的密钥.pub apple@mini的ip
```

---

## 2. 服务器（Mac Mini）配置

### 2.1 开启 SSH 服务

**方法一：系统设置**
> 系统设置 → 通用 → 共享 → 远程登录 → 开启

**方法二：命令行**

```bash
sudo systemsetup -setremotelogin on
```

### 2.2 编辑 SSH 配置

```bash
export TERM=xterm-256color && sudo nano /etc/ssh/sshd_config
```

添加或修改以下内容：

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

### 2.3 重启 SSH 服务

```bash
sudo launchctl unload /System/Library/LaunchDaemons/ssh.plist
sudo launchctl load /System/Library/LaunchDaemons/ssh.plist
```

---

## 3. 连接方式

### 3.1 从 MacBook 连接到 Mini

```bash
ssh apple@192.168.x.x
```

### 3.2 退出连接

```bash
# 方法1
exit

# 方法2
Ctrl+D
```

---

## 4. 查看 Mini 的 IP 地址

```bash
ipconfig getifaddr en0
```

---

## 5. 常用命令

| 操作 | 命令 |
|------|------|
| 查看 SSH 状态 | `sudo systemsetup -getremotelogin` |
| 关闭 SSH | `sudo systemsetup -setremotelogin off` |
| 查看防火墙状态 | `/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate` |

---

## 6. 故障排除

### 连接被拒绝

确认 Mini 上 SSH 已开启，且两台设备在同一局域网。

### 密钥认证失败

检查 MacBook 上的密钥权限：

```bash
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

### 忘记 Mini IP 地址

在 Mini 上运行：

```bash
ifconfig | grep "inet " | grep -v 127.0.0.1
```
