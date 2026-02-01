# Git 多用户配置 SOP（标准操作流程）

**文档版本**：1.0  
**更新日期**：2026-01-30  
**适用场景**：多人共用同一台公司电脑进行 Git 操作

---

## 📋 目录
1. [概述](#概述)
2. [前提条件](#前提条件)
3. [一次性配置（管理员）](#一次性配置管理员)
4. [日常使用流程（开发者）](#日常使用流程开发者)
5. [常见问题](#常见问题)
6. [安全注意事项](#安全注意事项)

---

## 概述

本文档描述如何在一台电脑上配置多个 Git 账户，支持两位或更多开发者轻松切换身份推送代码。

**核心组件**：
- **Git 用户配置文件** (`~/.git-profiles/`)：存储各账户的名称和邮箱
- **快速切换脚本** (`switch-git-user`)：一键切换 Git 全局身份
- **SSH host 别名** (`~/.ssh/config`)：为不同账户绑定不同的 SSH 密钥
- **1Password SSH Agent**：安全管理 SSH 私钥（无需本地明文存储）

---

## 前提条件

### 硬件 / 系统
- macOS（本文档基于 macOS 编写；Linux 需微调）
- 网络连接（用于 GitHub/Git 托管平台）

### 软件
- Git（通常预装；若未安装可 `brew install git`）
- 1Password 桌面应用（或其他 SSH Agent，本文以 1Password 为例）
- Homebrew（可选，但推荐用于管理包）

### 账户准备
- 两个（或多个）GitHub 账户，各自已添加公钥（详见下文）
- 两个（或多个）1Password 账户或同一账户内的多个 SSH Key 条目

---

## 一次性配置（管理员）

### 1️⃣ 创建 Git 用户配置文件

创建 `~/.git-profiles/` 目录并为每个开发者创建配置文件。

**示例：Penn_Lam 的配置**  
创建文件 `~/.git-profiles/Penn_Lam.conf`：
```
name=Penn_Lam
email=lemonmonlucy@163.com
```

**示例：charon-autopia 的配置**  
创建文件 `~/.git-profiles/charon-autopia.conf`：
```
name=charon-autopia
email=charon@autopia.chat
```

**自动化脚本**（复制粘贴到终端）：
```bash
mkdir -p ~/.git-profiles

# 创建 Penn_Lam 配置
cat > ~/.git-profiles/Penn_Lam.conf << 'EOF'
name=Penn_Lam
email=lemonmonlucy@163.com
EOF

# 创建 charon-autopia 配置
cat > ~/.git-profiles/charon-autopia.conf << 'EOF'
name=charon-autopia
email=charon@autopia.chat
EOF

echo "✓ Git 配置文件创建完成"
```

---

### 2️⃣ 安装快速切换脚本

创建 `~/.local/bin/switch-git-user` 脚本，允许快速切换用户。

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/switch-git-user << 'EOF'
#!/usr/bin/env bash
# switch-git-user <profile>|list - Git user switcher (SSH managed by 1Password)
PROFILES_DIR="$HOME/.git-profiles"
if [ "$1" = "list" ]; then
  ls -1 "$PROFILES_DIR" 2>/dev/null | sed 's/\.conf$//'
  exit 0
fi
PROFILE="$1"
if [ -z "$PROFILE" ]; then
  echo "Usage: switch-git-user <profile>|list"
  exit 1
fi
CFG="$PROFILES_DIR/${PROFILE}.conf"
if [ ! -f "$CFG" ]; then
  echo "Profile '$PROFILE' not found. Use 'switch-git-user list'"
  exit 2
fi
NAME="$(grep '^name=' "$CFG" | cut -d'=' -f2-)"
EMAIL="$(grep '^email=' "$CFG" | cut -d'=' -f2-)"
[ -n "$NAME" ] && git config --global user.name "$NAME"
[ -n "$EMAIL" ] && git config --global user.email "$EMAIL"
echo "✓ Git user switched to: $(git config --global user.name) <$(git config --global user.email)>"
EOF

chmod +x ~/.local/bin/switch-git-user
echo "✓ switch-git-user 脚本创建完成"
```

**确保 `~/.local/bin` 在 PATH 中**：
```bash
# 检查是否已在 PATH
echo $PATH | grep -q ".local/bin" && echo "✓ 已在 PATH" || echo "⚠️ 不在 PATH"

# 如果不在，添加到 ~/.zshrc（推荐）或 ~/.bashrc
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

---

### 3️⃣ 配置 SSH Host 别名

编辑或创建 `~/.ssh/config`，为每个账户的 GitHub 添加 host 别名。这样不同账户可使用不同的 SSH 密钥。

```bash
cat >> ~/.ssh/config << 'EOF'

# GitHub - Penn_Lam account
Host github-pennlam
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_Penn_Lam
  IdentitiesOnly yes

# GitHub - charon-autopia account
Host github-charon
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_charon-autopia
  IdentitiesOnly yes
EOF

chmod 600 ~/.ssh/config
echo "✓ SSH config 已更新"
```

**说明**：
- `Host github-pennlam` 和 `Host github-charon` 是自定义别名
- `IdentityFile` 指向本地私钥位置（注意：我们已删除本地私钥，改由 1Password 管理；此处仅作参考，实际认证由 1Password agent 完成）
- `IdentitiesOnly yes` 确保 SSH 仅使用指定的密钥

---

### 4️⃣ 在 1Password 中保存 SSH 密钥

**前提**：两个 GitHub 账户各自已有 SSH 公钥配置。若未配置，先生成：
```bash
# 生成 Penn_Lam 密钥（仅演示；本文档中已删除）
# ssh-keygen -t ed25519 -C "lemonmonlucy@163.com" -f ~/.ssh/id_ed25519_Penn_Lam -N ""

# 生成 charon-autopia 密钥（仅演示；本文档中已删除）
# ssh-keygen -t ed25519 -C "charon@autopia.chat" -f ~/.ssh/id_ed25519_charon-autopia -N ""
```

**在 1Password 中操作**：
1. 打开 1Password 桌面应用
2. 进入公司保险库（例：**世另我**）
3. 新建两条 SSH Key 项：
   - **Penn_Lam MacBook**：粘贴 Penn_Lam 的私钥
   - **charon-autopia MacBook**：粘贴 charon-autopia 的私钥
4. 在每条项的 Notes 中记录：
   - GitHub 账户
   - 邮箱
   - 用途（公司项目）

**配置 1Password SSH Agent**（可选，推荐）：
1. 打开 1Password 设置 → SSH Agent
2. 找到或编辑 SSH Agent 配置文件（通常位于 `~/.config/1Password/ssh/config.toml`）
3. 确保内容如下：
```toml
[[ssh-keys]]
vault = "世另我"
```
4. 保存，重启 1Password 应用

**验证密钥已加载**：
```bash
SSH_AUTH_SOCK=~/Library/Group\ Containers/2BUA8C4S2C.com.1password/t/agent.sock ssh-add -l
```
应显示两个 ED25519 公钥指纹。

---

### 5️⃣ 在 GitHub 上添加公钥

对于每个账户：
1. 登录对应的 GitHub 账户
2. 进入 **Settings** → **SSH and GPG keys**
3. 点击 **New SSH key**
4. **Title** 填写便于识别的名称（例：`Penn_Lam MacBook`）
5. **Key type** 选择 **Authentication Key**
6. **Key** 粘贴对应的公钥内容（从 1Password 获取或 `cat ~/.ssh/id_ed25519_*.pub`）
7. 点击 **Add SSH key**

---

## 日常使用流程（开发者）

### 快速开始

**1. 查看可用账户**
```bash
switch-git-user list
```
输出：
```
Penn_Lam
charon-autopia
```

**2. 切换到指定账户**
```bash
switch-git-user Penn_Lam
```
输出：
```
✓ Git user switched to: Penn_Lam <lemonmonlucy@163.com>
```

**3. 验证当前身份**
```bash
git config --global user.name
git config --global user.email
```

---

### 克隆仓库（新项目）

使用 SSH 别名克隆，确保后续推送使用正确的账户身份：

```bash
# Penn_Lam 克隆自己的仓库
switch-git-user Penn_Lam
git clone git@github-pennlam:Penn_Lam/my-repo.git

# charon-autopia 克隆自己的仓库
switch-git-user charon-autopia
git clone git@github-charon:charon-autopia/my-repo.git
```

**注意**：仓库的 remote URL 会自动使用别名，后续推送会使用对应的 SSH 密钥和 Git 身份。

---

### 修改现有仓库的 remote（如果需要切换账户）

```bash
# 当前在某个仓库
cd /path/to/repo

# 切换到新的 Git 账户
switch-git-user charon-autopia

# 修改 remote URL 为对应别名
git remote set-url origin git@github-charon:charon-autopia/repo-name.git

# 验证
git remote -v

# 推送
git commit -m "work by charon-autopia"
git push
```

---

### 一键推送（标准流程）

```bash
# 1. 切换用户
switch-git-user charon-autopia

# 2. 进入仓库（如果还未进入）
cd /path/to/repo

# 3. 确保 remote 使用对应别名（如果是新仓库）
git remote set-url origin git@github-charon:org/repo.git

# 4. 编码、提交
git add .
git commit -m "feature: description"

# 5. 推送
git push
```

---

## 常见问题

### Q1: 切换用户后为什么推送还是失败？

**可能原因**：
1. Remote URL 不是 SSH 别名（仍为 HTTPS）
2. 1Password SSH agent 未运行

**排查步骤**：
```bash
# 检查 remote URL
git remote -v
# 应该显示 git@github-pennlam:... 或 git@github-charon:... 形式

# 检查 1Password SSH agent 是否运行
SSH_AUTH_SOCK=~/Library/Group\ Containers/2BUA8C4S2C.com.1password/t/agent.sock ssh-add -l
# 应列出两个公钥指纹

# 如果没有，重启 1Password 或确认 SSH Agent 已启用
```

**修复**：
```bash
# 改 remote 为 SSH 别名
git remote set-url origin git@github-charon:org/repo.git

# 重新推送
git push
```

---

### Q2: 提交者信息显示错误？

**原因**：Git 身份未正确切换。

**解决**：
```bash
# 检查当前 Git 配置
git config --global user.name
git config --global user.email

# 重新切换用户
switch-git-user Penn_Lam

# 验证
git config --global user.name
```

如果已有错误的提交，可使用以下命令修改（仅修改本地，未推送的提交）：
```bash
git commit --amend --reset-author
```

---

### Q3: SSH 连接超时或拒绝？

**可能原因**：SSH 密钥不匹配或 GitHub 公钥未配置。

**排查**：
```bash
# 使用详细模式连接测试
SSH_AUTH_SOCK=~/Library/Group\ Containers/2BUA8C4S2C.com.1password/t/agent.sock ssh -vvv -T git@github-pennlam

# 查看输出中是否有 "Offering public key" 和 "Authenticated as" 行
```

**解决**：
1. 确认 1Password 中的 SSH 密钥是最新的
2. 确认 GitHub 上的公钥已添加且未过期
3. 重启 1Password SSH agent

---

### Q4: 同时在两个账户的仓库工作时，如何避免混乱？

**建议**：
1. 为不同账户的项目创建单独的目录（如 `~/Projects/penn-lam/` 和 `~/Projects/charon/`）
2. 进入各目录前先切换用户
3. 使用 shell 提示符显示当前 Git 身份（可选）：
```bash
# 在 ~/.zshrc 中添加
RPROMPT='$(git config --global user.name 2>/dev/null && echo " [$(git config --global user.name)]")'
```

---

## 安全注意事项

### ✅ 推荐做法
- ✅ **使用 1Password 存储 SSH 私钥**，磁盘上无明文存储
- ✅ **定期审计 GitHub 已授权的 SSH 密钥**（Settings → SSH keys）
- ✅ **不要在仓库或脚本中硬编码密钥**
- ✅ **使用强密码保护 1Password 账户**
- ✅ **定期更新 1Password 应用**

### ⚠️ 禁止事项
- ❌ **不要使用密码认证推送 GitHub**（GitHub 已禁用）
- ❌ **不要在终端历史中记录 SSH 密钥**
- ❌ **不要在非安全网络上使用该电脑处理敏感代码**
- ❌ **不要与他人共享 1Password 账户密钥**
- ❌ **不要把 SSH 私钥备份到公网**

---

## 版本历史

| 版本 | 日期 | 变更 |
|-----|------|------|
| 1.0 | 2026-01-30 | 初版发布 |

---

## 联系与支持

如遇问题或需要更新此文档，请：
1. 检查上述"常见问题"部分
2. 查阅 [1Password SSH Agent 官方文档](https://developer.1password.com/docs/ssh/agent/)
3. 查阅 [GitHub SSH 文档](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)

---

**文档维护人**：[待补充]  
**最后更新**：2026-01-30
