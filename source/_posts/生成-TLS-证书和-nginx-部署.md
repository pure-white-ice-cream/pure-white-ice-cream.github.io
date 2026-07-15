---
title: 生成 TLS 证书和 nginx 部署
date: 2026/7/15 12:00:00
---
# Windows 本地开发 TLS 证书：一键生成、安装与卸载

本地用 HTTPS（例如 nginx 反代、局域网用 IP 访问）时，自签证书常会触发浏览器「不安全」警告。比较稳妥的做法是：

1. 自己发一张 **本地 Root CA**
2. 用这张 CA 签发服务器证书（带 SAN：域名 + IP）
3. **只把 Root CA 导入系统信任库**（服务器私钥不要导入）
4. nginx 使用 `tls.crt` + `tls.key`

本文配套脚本 `New-LocalTlsCert.ps1`：**只负责生成文件，不写入系统证书存储**。信任由你手动导入，卸载也更可控。

---

## 你会得到什么

脚本跑完后，目录里只保留 3 个文件：

| 文件 | 用途 |
|------|------|
| `RootCA.crt` | 根证书（给系统/浏览器信任用） |
| `tls.crt` | 服务器证书（给 nginx `ssl_certificate`） |
| `tls.key` | 服务器私钥（给 nginx `ssl_certificate_key`） |

中间产物（CA 私钥、CSR、扩展配置、序列号等）写在临时目录 `.cert-tmp`，结束后会自动删除。  
**CA 私钥不会保留在磁盘上**——同一张 RootCA 无法再签发新服务器证；需要新证就重新跑脚本（并重新导入新的 RootCA）。

默认证书内容大致是：

- Root CA：`CN=Local Dev Root CA`，有效期约 10 年
- 服务器证：`CN=<本机计算机名>`
- SAN DNS：`localhost`、本机计算机名
- SAN IP：`127.0.0.1` + `192.168.1.0`～`192.168.1.255`

适合本机与同网段 `192.168.1.x` 用 IP 访问的开发场景。

---

## 环境准备

### 1. PowerShell

脚本要求 PowerShell **5.1+**（Windows 10/11 自带通常即可）。

### 2. 安装 OpenSSL

脚本会按下面顺序找 `openssl`：

1. PATH 中的 `openssl`
2. `C:\Program Files\OpenSSL-Win64\bin\openssl.exe`
3. `C:\Program Files\OpenSSL\bin\openssl.exe`

没有则安装（示例用 winget）：

```powershell
winget install --id ShiningLight.OpenSSL.Light
```

装完后**新开一个** PowerShell 窗口，确认：

```powershell
openssl version
```

---

## 脚本全文

将下列内容保存为 `New-LocalTlsCert.ps1`（与输出证书同一目录即可）：

```powershell
#Requires -Version 5.1
<#
.SYNOPSIS
  一键生成本地 CA + TLS 证书，最终仅保留 RootCA.crt / tls.crt / tls.key（不导入系统）
.NOTES
  - nginx: ssl_certificate tls.crt; ssl_certificate_key tls.key;
  - 信任需自行导入 RootCA.crt（本机: Cert:\LocalMachine\Root；当前用户: Cert:\CurrentUser\Root）
#>
$ErrorActionPreference = 'Stop'

try {
    $OutDir = $PSScriptRoot
    if (-not $OutDir) { $OutDir = Get-Location }
    Set-Location $OutDir

    # --- 定位 OpenSSL ---
    $openssl = Get-Command openssl -ErrorAction SilentlyContinue
    if (-not $openssl) {
        $candidates = @(
            'C:\Program Files\OpenSSL-Win64\bin\openssl.exe',
            'C:\Program Files\OpenSSL\bin\openssl.exe'
        )
        foreach ($p in $candidates) {
            if (Test-Path $p) { $openssl = @{ Source = $p }; break }
        }
    }
    if (-not $openssl) {
        throw "未找到 openssl。请先安装: winget install --id ShiningLight.OpenSSL.Light"
    }
    $OpenSSL = $openssl.Source

    # --- 默认参数（可按需改） ---
    $CaDays   = 3650
    $TlsDays  = 3650
    $CaSubj   = '/CN=Local Dev Root CA'
    $hostName = $env:COMPUTERNAME
    $TlsSubj  = "/CN=$hostName"
    $SanDns   = @('localhost', $hostName)
    # 与 sec_portalsite_sp.ext 一致：127.0.0.1 + 192.168.1.0～255
    $SanIp    = @('127.0.0.1') + (0..255 | ForEach-Object { "192.168.1.$_" })

    # --- 临时文件 ---
    $tmp = Join-Path $OutDir '.cert-tmp'
    New-Item -ItemType Directory -Force -Path $tmp | Out-Null
    $caKey  = Join-Path $tmp 'rootCA.key'
    $caCrt  = Join-Path $OutDir 'RootCA.crt'
    $srvKey = Join-Path $OutDir 'tls.key'
    $srvCsr = Join-Path $tmp 'tls.csr'
    $srvCrt = Join-Path $OutDir 'tls.crt'
    $extFile = Join-Path $tmp 'tls.ext'
    $serial  = Join-Path $tmp 'rootCA.srl'

    # 直接调用外部程序；-out/-in 等会原样传给 openssl，不会被 PowerShell 解析
    function Invoke-OpenSSL([string[]]$OpenSslArgs) {
        & $OpenSSL $OpenSslArgs
        if ($LASTEXITCODE -ne 0) { throw "openssl 失败: $($OpenSslArgs -join ' ')" }
    }

    # --- ② 生成 Root CA ---
    Write-Host '生成 Root CA ...'
    Invoke-OpenSSL -OpenSslArgs @('genrsa', '-out', $caKey, '4096')
    Invoke-OpenSSL -OpenSslArgs @('req', '-x509', '-new', '-nodes', '-key', $caKey, '-sha256', '-days', "$CaDays", '-subj', $CaSubj, '-out', $caCrt)

    # --- ③ 生成服务器证书 ---
    Write-Host '生成 TLS 证书 ...'
    $sanLines = @('subjectAltName = @alt_names', '', '[alt_names]')
    $i = 1
    foreach ($d in $SanDns) { $sanLines += "DNS.$i = $d"; $i++ }
    $i = 1
    foreach ($ip in $SanIp) { $sanLines += "IP.$i = $ip"; $i++ }
    $sanLines | Set-Content -Path $extFile -Encoding ascii

    Invoke-OpenSSL -OpenSslArgs @('genrsa', '-out', $srvKey, '2048')
    Invoke-OpenSSL -OpenSslArgs @('req', '-new', '-key', $srvKey, '-subj', $TlsSubj, '-out', $srvCsr)
    Invoke-OpenSSL -OpenSslArgs @('x509', '-req', '-in', $srvCsr, '-CA', $caCrt, '-CAkey', $caKey, '-CAcreateserial', '-CAserial', $serial, '-out', $srvCrt, '-days', "$TlsDays", '-sha256', '-extfile', $extFile)

    # --- 清理临时文件（含 CA 私钥） ---
    Remove-Item -Recurse -Force $tmp

    Write-Host ''
    Write-Host '完成（仅生成，未导入系统）。输出文件:'
    Write-Host "  $caCrt"
    Write-Host "  $srvCrt"
    Write-Host "  $srvKey"
    Write-Host ''
    & $OpenSSL x509 -in $srvCrt -noout -issuer -subject
}
catch {
    Write-Host ''
    Write-Host '========== 发生错误 ==========' -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
    if ($_.ScriptStackTrace) {
        Write-Host ''
        Write-Host $_.ScriptStackTrace
    }
    Write-Host '==============================' -ForegroundColor Red
    Write-Host ''
    Read-Host '按 Enter 键退出'
    exit 1
}
```

---

## 生成证书

在脚本所在目录打开 PowerShell：

```powershell
cd C:\DevSource\powershell-tls
Set-ExecutionPolicy -Scope Process Bypass
.\New-LocalTlsCert.ps1
```

成功时会看到类似输出：

```text
生成 Root CA ...
生成 TLS 证书 ...
完成（仅生成，未导入系统）。输出文件:
  ...\RootCA.crt
  ...\tls.crt
  ...\tls.key

issuer=CN=Local Dev Root CA
subject=CN=你的计算机名
```

### 可按需修改的参数

打开脚本中的「默认参数」一段：

| 变量 | 含义 |
|------|------|
| `$CaDays` / `$TlsDays` | CA / 服务器证有效天数 |
| `$CaSubj` | Root CA 主题（卸载时按此名称查找） |
| `$SanDns` | SAN 域名列表 |
| `$SanIp` | SAN IP 列表（默认含整段 `192.168.1.0/24`） |

改完重新执行脚本即可覆盖生成新文件。

---

## 配置 nginx（使用服务器证书）

把 `tls.crt`、`tls.key` 拷到 nginx 证书目录，例如：

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     conf/certs/tls.crt;
    ssl_certificate_key conf/certs/tls.key;

    # ... 其余配置
}
```

改完后重载：

```powershell
nginx -s reload
```

此时若还没导入 RootCA，浏览器仍会报警——这是预期行为。

---

## 安装信任（导入 RootCA）

只导入 **`RootCA.crt`**。不要把 `tls.key` 导入系统。

### 方案 A：本机所有用户（推荐开发机）

需要**管理员** PowerShell：

```powershell
Import-Certificate `
  -FilePath 'C:\DevSource\powershell-tls\RootCA.crt' `
  -CertStoreLocation 'Cert:\LocalMachine\Root'
```

图形界面：

1. 右键 `RootCA.crt` → **安装证书**
2. 存储位置选 **本地计算机**
3. 选 **将所有的证书都放入下列存储** → **受信任的根证书颁发机构**
4. 完成

查看：`Win + R` → **`certlm.msc`** → 受信任的根证书颁发机构 → 证书 → **Local Dev Root CA**

### 方案 B：仅当前用户（无需管理员）

普通 PowerShell：

```powershell
Import-Certificate `
  -FilePath 'C:\DevSource\powershell-tls\RootCA.crt' `
  -CertStoreLocation 'Cert:\CurrentUser\Root'
```

图形界面：安装时选 **当前用户**，同样放到「受信任的根证书颁发机构」。

查看：`Win + R` → **`certmgr.msc`**（不是 `certlm.msc`）→ 受信任的根证书颁发机构 → **Local Dev Root CA**

> **容易踩坑：**  
> - `certlm.msc` = 本地计算机  
> - `certmgr.msc` = 当前用户  
> 导入到「当前用户」却打开「本机」管理器，就会以为「没装上」，但浏览器其实已经信任了。

导入后**重启浏览器**（或新开无痕窗口），再访问 `https://localhost` 或 `https://192.168.1.x`。

---

## 卸载信任

### 从「当前用户」卸载

图形：`certmgr.msc` → 受信任的根证书颁发机构 → 删除 **Local Dev Root CA**

PowerShell：

```powershell
Get-ChildItem Cert:\CurrentUser\Root |
  Where-Object { $_.Subject -like '*Local Dev Root CA*' } |
  Remove-Item
```

### 从「本机」卸载（需管理员）

图形：`certlm.msc` → 受信任的根证书颁发机构 → 删除 **Local Dev Root CA**

PowerShell（管理员）：

```powershell
Get-ChildItem Cert:\LocalMachine\Root |
  Where-Object { $_.Subject -like '*Local Dev Root CA*' } |
  Remove-Item
```

### 不确定装在哪？先查再删

```powershell
$thumb = (New-Object System.Security.Cryptography.X509Certificates.X509Certificate2(
  'C:\DevSource\powershell-tls\RootCA.crt'
)).Thumbprint

foreach ($store in @(
  'Cert:\CurrentUser\Root',
  'Cert:\LocalMachine\Root'
)) {
  $hit = Get-ChildItem $store -ErrorAction SilentlyContinue |
    Where-Object { $_.Thumbprint -eq $thumb }
  if ($hit) {
    Write-Host "找到: $store  Thumbprint=$thumb"
  } else {
    Write-Host "未在: $store"
  }
}
```

注意：每重新生成一次 `RootCA.crt`，指纹都会变。旧 RootCA 若已导入，需要按**旧指纹或旧主题名**删，不能只靠新文件指纹。

---

## 端到端流程小结

```text
安装 OpenSSL
    ↓
运行 New-LocalTlsCert.ps1（仅生成）
    ↓
得到 RootCA.crt / tls.crt / tls.key
    ↓
nginx 配置 tls.crt + tls.key 并 reload
    ↓
手动导入 RootCA.crt（LocalMachine 或 CurrentUser）
    ↓
重启浏览器验证 HTTPS
```

卸载时：从对应存储删除 RootCA；磁盘上的三个文件可另行删除。

---

## 常见问题

### 1. 浏览器仍提示不安全

- RootCA 是否导入到了你正在用的那个存储（本机 / 当前用户）？
- 是否重启了浏览器？
- 访问的 IP/域名是否在 SAN 里？（例如 `192.168.0.x` 不在默认 `192.168.1.0/24` 内）
- 是否重新生成过证书，但系统里还是旧 RootCA、nginx 已换成新 `tls.crt`？（链对不上）

### 2. `certlm.msc` 里找不到 Local Dev Root CA

多半导入的是 **当前用户**。改用 `certmgr.msc`，或用上一节的查询命令。

### 3. 脚本报找不到 openssl

安装 OpenSSL 后新开终端；或把 `openssl.exe` 所在目录加入 PATH。

### 4. 执行策略限制无法运行 `.ps1`

仅对当前进程放宽即可：

```powershell
Set-ExecutionPolicy -Scope Process Bypass
```

### 5. 局域网其他电脑访问仍报警

其他电脑也需要安装**同一份** `RootCA.crt`（导入它们自己的信任库）。只在你本机导入，别的机器不会自动信任。

### 6. 安全提醒

- 这是**本地开发用** CA，不要用于生产公网站点。
- `tls.key` 等同于服务器私钥，勿提交到公开仓库、勿随手拷贝。
- 脚本故意不保留 CA 私钥，降低「被拿去乱签证书」的风险；代价是换证必须整套重发并重新导入 RootCA。

---

## 参考：两个证书管理器对照

| 工具 | 打开方式 | 对应存储 |
|------|----------|----------|
| 本机证书 | `certlm.msc` | `Cert:\LocalMachine\...` |
| 用户证书 | `certmgr.msc` | `Cert:\CurrentUser\...` |

开发机多人共用、希望「装一次全机生效」→ 用本机 Root。  
个人电脑、不想提权 → 用当前用户 Root 即可。
