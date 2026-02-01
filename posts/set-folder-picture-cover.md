---
title: "PowerShell+ImageMagick：一键自动化设置 Windows 文件夹封面"
published: 2025-12-29
# URL 优化：保持英文短横线命名，利于 SEO 和分享
alias: windows-folder-cover-automation
author: Rinhon
description: "编写 PowerShell 脚本调用 ImageMagick，实现自动裁剪图片并设为文件夹封面，解决本地漫画管理痛点。包含注册表右键菜单集成方案。"
# 建议提取文中那张效果对比图作为文章头图
# image: "./assets/folder-cover-preview.jpg" 
tags: 
  - PowerShell
  - ImageMagick
  - Windows
category: 脚本自动化
draft: false
pinned: false
lang: zh-CN
---

个人需求：没啥顺手的漫画阅读器。有个叫 `Rulia` 的我还蛮喜欢，但作者创业去了，久未更新，且一直不支持读取文件夹图片（Cover）的功能。试了个连 `Kavita` 的插件，压根连不上。

于是转念一想：**如果直接修改 Windows 文件夹图标，资源管理器不就是最好的书架吗？**

虽然市面上有改压缩包封面的工具，但这不太符合我“解压即看”的习惯。所以我决定自己写个脚本，利用 **PowerShell** 结合 **ImageMagick** 来自动化这个过程。ImageMagick 负责高性能的图片裁剪（处理成正方形封面），PowerShell 负责与 Windows 底层交互（写入配置、刷新缓存）。

### 前置准备

本方案依赖命令行图像处理工具，请先完成安装：

1.  **下载 ImageMagick**:
    * 前往 [ImageMagick 官网](https://imagemagick.org/script/download.php#windows) 下载 Windows 版本。
    * 推荐下载：`ImageMagick-x.x.x-Q16-HDRI-x64-dll.exe`。
2.  **关键安装步骤**:
    * 安装过程中，**务必勾选**以下两项，否则脚本无法运行：
        * `Install legacy utilities (e.g. convert)`
        * `Add application directory to your system path`

---

### 第一步：核心逻辑脚本 (`Set-FolderCover.ps1`)

在任意位置（建议固定目录，如 `C:\Scripts\FolderCover`）创建文件 `Set-FolderCover.ps1`。

**脚本功能升级点：**
* **智能裁剪**：调用 ImageMagick 进行“中心 1:1 裁剪”，保留图片主体。
* **强制刷新**：内置 C# 方法调用 Windows API (`SHChangeNotify`)，**无需重启资源管理器即可看到封面变化**。
* **错误日志**：自动记录失败原因到文档目录。

```powershell
param (
    [string]$TargetItem
)

# --- 配置部分 ---
$ErrorLogPath = [System.IO.Path]::Combine([Environment]::GetFolderPath("MyDocuments"), "FolderCover_ErrorLog.txt")
$ImageExtensions = @(".jpg", ".jpeg", ".png", ".bmp", ".webp", ".tif", ".tiff")

# --- 嵌入 C# 代码：用于强制刷新 Windows 图标缓存 ---
$code = @"
[System.Runtime.InteropServices.DllImport("Shell32.dll")]
public static extern int SHChangeNotify(int eventId, int flags, IntPtr item1, IntPtr item2);
"@
$Win32 = Add-Type -MemberDefinition $code -Name "Win32SHChangeNotify" -Namespace Win32Functions -PassThru

# --- 函数定义 ---

function Write-Log {
    param ([string]$Message)
    $Timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    Add-Content -Path $ErrorLogPath -Value "[$Timestamp] $Message" -Encoding UTF8
}

function Refresh-Explorer {
    # 0x08000000 = SHCNE_ASSOCCHANGED (通知系统文件关联/图标已变更)
    # 虽是核弹级刷新，但对文件夹图标生效最快
    $null = $Win32::SHChangeNotify(0x08000000, 0, [IntPtr]::Zero, [IntPtr]::Zero)
}

function Set-CoverImage {
    param (
        [string]$FolderPath,
        [string]$SourceImagePath
    )

    try {
        if (-not (Test-Path $SourceImagePath)) { return }

        $DestPath = Join-Path -Path $FolderPath -ChildPath "folder.jpg"

        # 1. 清理旧文件（需先移除只读/隐藏属性才能删除）
        if (Test-Path $DestPath) {
            $existing = Get-Item $DestPath
            $existing.Attributes = "Normal"
            Remove-Item -Path $DestPath -Force
        }

        # 2. 调用 ImageMagick 处理图片
        # -gravity center: 居中定位
        # -extent: 强制调整画布为 1:1 (宽=高，取原图中较小的一边)
        $magickCmd = "magick"
        $argsList = @(
            "`"$SourceImagePath`"", 
            "-gravity", "center", 
            "-extent", "`"%[fx:w<h?w:h]x%[fx:w<h?w:h]`"", 
            "`"$DestPath`""
        )
        
        $process = Start-Process -FilePath $magickCmd -ArgumentList $argsList -WindowStyle Hidden -Wait -PassThru
        
        if ($process.ExitCode -ne 0) {
            throw "ImageMagick CLI 执行失败，ExitCode: $($process.ExitCode)"
        }

        # 3. 设置 folder.jpg 为系统+隐藏
        $file = Get-Item $DestPath
        $file.Attributes = "Hidden, System"

        # 4. 配置 desktop.ini
        # 关键：文件夹必须设为 ReadOnly，Windows 才会去读 desktop.ini
        $folder = Get-Item $FolderPath
        $folder.Attributes = "ReadOnly"

        $iniPath = Join-Path -Path $FolderPath -ChildPath "desktop.ini"
        if (Test-Path $iniPath) {
            (Get-Item $iniPath).Attributes = "Normal"
        }
        
        # 写入配置 (使用 Unicode 编码以兼容特殊字符，虽然 folder.jpg 是纯英文)
        $iniContent = "[.ShellClassInfo]`r`nLogo=folder.jpg"
        Set-Content -Path $iniPath -Value $iniContent -Encoding Unicode -Force
        
        # desktop.ini 必须是 Hidden + System
        (Get-Item $iniPath).Attributes = "Hidden, System"
        
        # 5. 触发刷新
        Refresh-Explorer
    }
    catch {
        Write-Log "处理失败 [$FolderPath]: $_"
    }
}

function Find-FirstImage {
    param ([string]$Path)
    try {
        # 排除 folder.jpg 自身，防止递归错误
        $images = Get-ChildItem -Path $Path -File | Where-Object { 
            $ImageExtensions -contains $_.Extension.ToLower() -and $_.Name -ne "folder.jpg" 
        }
        if ($images) {
            return ($images | Select-Object -First 1).FullName
        }
    }
    catch {
        Write-Log "搜图失败 [$Path]: $_"
    }
    return $null
}

# --- 主入口 ---

try {
    # 清洗传入路径（去除注册表可能带来的多余引号）
    $TargetItem = $TargetItem.Trim('"')

    if (Test-Path $TargetItem -PathType Container) {
        # === 场景 A: 用户右键了文件夹 ===
        $img = Find-FirstImage -Path $TargetItem
        if ($img) {
            Set-CoverImage -FolderPath $TargetItem -SourceImagePath $img
        } else {
            Write-Log "跳过: 文件夹内无可用图片 - $TargetItem"
        }
    }
    elseif (Test-Path $TargetItem -PathType Leaf) {
        # === 场景 B: 用户右键了图片 ===
        $ext = [System.IO.Path]::GetExtension($TargetItem).ToLower()
        if ($ImageExtensions -contains $ext) {
            $parentDir = [System.IO.Path]::GetDirectoryName($TargetItem)
            Set-CoverImage -FolderPath $parentDir -SourceImagePath $TargetItem
        }
    }
}
catch {
    Write-Log "致命错误: $_"
}

```

---

### 第二步：注册右键菜单 (`Register-RightClick.ps1`)

将此脚本放在与 `Set-FolderCover.ps1` **相同的目录**下。

**脚本功能：**

* 自动识别当前路径，写入注册表。
* 同时注册到 **“文件夹右键”** 和 **“图片文件右键”**。
* **自动提权**：双击运行即可，无需手动右键以管理员运行。

```powershell
# --- 自动提权模块 ---
$currentPrincipal = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
if (-not $currentPrincipal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Start-Process powershell.exe "-NoProfile -ExecutionPolicy Bypass -File `"$PSCommandPath`"" -Verb RunAs
    Exit
}

# --- 环境检查 ---
$ScriptDir = Split-Path -Parent $MyInvocation.MyCommand.Definition
$TargetScript = Join-Path -Path $ScriptDir -ChildPath "Set-FolderCover.ps1"

if (-not (Test-Path $TargetScript)) {
    Write-Host "错误：未找到主脚本 Set-FolderCover.ps1" -ForegroundColor Red
    Write-Host "请确保两个文件在同一目录。"
    Pause; Exit
}

if (-not (Get-Command magick -ErrorAction SilentlyContinue)) {
    Write-Host "警告：未检测到 'magick' 命令！" -ForegroundColor Yellow
    Write-Host "请确保已安装 ImageMagick 且添加到了 PATH 环境变量。"
    Write-Host "按任意键继续（如果确认已安装）..."
    Pause
}

# --- 注册表操作 ---
$MenuText = "设置纯图封面 (1:1)"
# 使用系统自带图标，也可替换为 .ico 文件路径
$IconPath = "imageres.dll,-68" 

function Register-Menu {
    param (
        [string]$RegPath,
        [string]$MenuName
    )
    
    $KeyPath = "Registry::HKEY_CLASSES_ROOT\$RegPath\shell\SetFolderCover"
    $CommandPath = "$KeyPath\command"

    # 1. 创建菜单项
    if (-not (Test-Path $KeyPath)) { New-Item -Path $KeyPath -Force | Out-Null }
    Set-ItemProperty -Path $KeyPath -Name "(default)" -Value $MenuName
    Set-ItemProperty -Path $KeyPath -Name "Icon" -Value $IconPath

    # 2. 注入命令 (优先使用 pwsh, 降级使用 powershell)
    if (-not (Test-Path $CommandPath)) { New-Item -Path $CommandPath -Force | Out-Null }
    
    $Executor = if (Get-Command pwsh -ErrorAction SilentlyContinue) { "pwsh.exe" } else { "powershell.exe" }
    $Command = "$Executor -WindowStyle Hidden -ExecutionPolicy Bypass -File `"$TargetScript``" `"%1`""

    Set-ItemProperty -Path $CommandPath -Name "(default)" -Value $Command
}

try {
    Write-Host "正在注入注册表..." -ForegroundColor Cyan
    
    # 注册位置 1: 文件夹背景/图标右键
    Register-Menu -RegPath "Directory" -MenuName $MenuText
    
    # 注册位置 2: 所有图片格式右键
    Register-Menu -RegPath "SystemFileAssociations\image" -MenuName "设为当前文件夹封面 (1:1)"

    Write-Host "✅ 注册成功！" -ForegroundColor Green
    Write-Host "现在你可以右键文件夹或图片使用了。"
}
catch {
    Write-Host "❌ 注册失败：$_" -ForegroundColor Red
}

Pause

```

### 使用指南

1. **部署**：
* 将上述两个 `.ps1` 文件放在同一永久目录（如 `D:\Tools\FolderCover\`）。
* 双击运行 `Register-RightClick.ps1`（会自动请求管理员权限）。


2. **操作**：
* **方法 A**：在资源管理器中，右键点击某个漫画**文件夹** -> 选择 `设置纯图封面 (1:1)`。
* **方法 B**：进入文件夹，右键点击你喜欢的某张**图片** -> 选择 `设为当前文件夹封面 (1:1)`。


3. **批量处理**：
* 选中几百个文件夹，右键一次性执行，脚本会自动为每个文件夹找第一张图做封面。



### 效果原理

脚本的核心在于这一行 ImageMagick 命令：

```powershell
magick "input.jpg" -gravity center -extent "%[fx:w<h?w:h]x%[fx:w<h?w:h]" "folder.jpg"

```

它利用 `fx` 表达式动态计算宽高，取较小值作为边长，配合 `center` 重心，实现**智能正方形裁切**。相比直接缩放，这样能最大程度保留封面的人物主体，在书架视图下也更整齐美观。