---
title: 解决powershell无法运行npm和ps1文件
date: 2025/10/22 11:00:00
---

# 报错现象

`powershell` 执行 `npm` 或 `xxx.ps1`:
``` ps1
PS C:\Users\username> npm -v
npm : 由于此系统上的策略禁止运行脚本，无法加载文件 C:\nvm4w\nodejs\npm.ps1。有关详细信息，请参阅 "about_Execution_Policies" (https://go.microsoft.com/fwlink/?LinkID=135170)。
所在位置 行:1 字符:1
+ npm -v
+ ~~~
    + CategoryInfo          : 安全错误: (:) []，PSSecurityException
    + FullyQualifiedErrorId : UnauthorizedAccess
```

# 报错原因
- 默认拒绝执行脚本，为了防止恶意脚本利用 PowerShell 自动化系统。

# 解决方法
`powershell` 执行以下命令:
``` ps1
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

# 检测是否生效
`powershell` 执行以下命令:
``` ps1
Get-ExecutionPolicy -List
```

预期得到结果:
``` ps1
PS C:\Users\username> Get-ExecutionPolicy -List

        Scope ExecutionPolicy
        ----- ---------------
MachinePolicy       Undefined
   UserPolicy       Undefined
      Process       Undefined
  CurrentUser    RemoteSigned
 LocalMachine       Undefined
```

`powershell` 执行 `npm` 或 `xxx.ps1` 预期得到结果:
``` ps1
PS C:\Users\username> npm -v
10.9.3
```

# 命令详解
- 作用：允许执行本地脚本（如 npm.ps1），而仍然防止运行从互联网下载但未签名的脚本。
- 设置范围：-Scope CurrentUser 只影响当前用户，不会修改系统级策略。
- 该命令等价于在 注册表 `\HKEY_CURRENT_USER\Software\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell` 添加 `ExecutionPolicy`: `RemoteSigned` (类型为`REG_SZ`)

# 扩展内容
| 策略值                       | 含义             | 风险        |
| ------------------------- | -------------- | --------- |
| **Restricted**            | 禁止一切脚本（默认）     | 最安全       |
| **AllSigned**             | 只允许运行签名的脚本     | 安全但繁琐     |
| **RemoteSigned**          | 本地脚本可运行，下载的需签名 | 折中（推荐开发者） |
| **Unrestricted / Bypass** | 允许所有脚本         | 高风险       |

| Scope 值                        | 说明                          | 优先级 | 适合场景     |
| ------------------------------ | --------------------------- | --- | -------- |
| **Process**                    | 仅当前 PowerShell 会话有效，关闭窗口后失效 | 最高  | 临时修改     |
| **CurrentUser**                | 仅当前登录用户有效                   | 中等  | 推荐开发者使用  |
| **LocalMachine**               | 整个系统有效（所有用户）                | 较低  | 需要管理员权限  |
| **MachinePolicy / UserPolicy** | 由组策略设置（企业环境）                | 最高  | 企业 IT 管控 |


# 回滚操作
`powershell` 执行以下命令即可撤销之前的命令:
``` ps1
Set-ExecutionPolicy Undefined -Scope CurrentUser
```