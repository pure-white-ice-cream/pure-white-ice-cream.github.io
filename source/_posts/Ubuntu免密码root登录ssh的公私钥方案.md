---
title: Ubuntu免密码root登录ssh的公私钥方案
date: 2025/10/28 14:00:00
---

# 简介

- 使用 `MobaXterm` 连接远程服务器的时候经常需要 `sudo -i` 不方便而且 `sftp` 权限受阻
- 通过本教程可以无需密码连接远程服务器, 并且 `sftp` 是 `root` 权限, 方便同时在一定程度上比密码登录更安全

# 生成密钥对（推荐：ed25519）

## 在 PowerShell / CMD（Windows 10/11 已内置 OpenSSH 客户端）或 Git Bash 都可以用 ssh-keygen：

- PowerShell / CMD / Git Bash：
``` shell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_设置一个名字"
```

- 如果需要 RSA（兼容性更好，但要用更长的密钥）:
``` shell
ssh-keygen -t rsa -b 4096 -f "$env:USERPROFILE\.ssh\id_设置一个名字"
```

## 执行后会提示：

- 存储路径（默认 C:\Users\你的用户名\.ssh\id_设置一个名字）

- 输入 passphrase（可留空或输入密码以增加安全, 本教程为免密码, 所以推荐留空）

# 查看 / 复制公钥到剪贴板

- 在资源管理器中查看公钥文件 `C:\Users\你的用户名\.ssh\id_设置一个名字.pub` 并复制内容

- 或者以下命令中的一个

- PowerShell:
``` shell
Get-Content $env:USERPROFILE\.ssh\id_设置一个名字.pub | Set-Clipboard
```

- CMD:
``` shell
type %USERPROFILE%\.ssh\id_设置一个名字.pub | clip
```

- Git Bash:
``` shell
cat ~/.ssh/id_设置一个名字.pub | clip.exe
# 或者直接显示出来然后手动复制：
cat ~/.ssh/id_设置一个名字.pub
```

# 把公钥添加到远程主机

- 在服务器执行
``` shell
sudo -i # 获取管理员权限
vi /root/.ssh/authorized_keys
```

- 输入 `i` 插入复制好的公钥
- 按 `ESC` 输入 `:wq` 保存并推出
- 如果有多个公钥, 用回车空行来区分

# 开启 `root` 登录

- 在服务器执行
``` shell
sudo -i # 获取管理员权限
vi /etc/ssh/sshd_config.d/50-cloud-init.conf # 如果没有就创建
```

- 将文件编辑为
``` conf
PermitRootLogin prohibit-password
```

# 应用 `ssh` 配置

- 仅重载配置而不中断现有连接（不重启）:
``` shell
sudo systemctl reload ssh
```

# 无密码连接

- 在 `MobaXterm` 跟服务器连接配置里修改用户名为 `root`, 验证改为私钥, 并导入`C:\Users\你的用户名\.ssh\id_设置一个名字` 之后可以直接连接, 并且之后不需要再导入

# 安全提示
- `id_设置一个名字.pub` 的公钥是可以公开的, 即使泄露了也没有安全风险
- 保存好你的私钥文件,  `C:\Users\你的用户名\.ssh\id_设置一个名字`, 一旦泄露, 及时更换
