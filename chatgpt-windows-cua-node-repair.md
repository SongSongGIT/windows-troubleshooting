# ChatGPT Windows 桌面版打不开：从后台进程到 cua_node 运行时修复

点击 ChatGPT 后，任务管理器里能看到 ChatGPT.exe，但桌面一直没有窗口。这次故障发生在 Microsoft Store / MSIX 安装的 **OpenAI.Codex 26.1029.7803.0**。按社区 issue 的方法，将安装包自带的 cua_node 运行时完整复制到当前用户的缓存后，应用成功打开。

## 排查过程

| 检查项 | 本次观察 |
| --- | --- |
| 应用包 | OpenAI.Codex 26.1029.7803.0 |
| 运行时状态 | NotAvailable |
| 进程和窗口 | ChatGPT.exe 存在，MainWindowHandle 为 0，未发现带 --type=renderer 的 ChatGPT.exe |
| 运行时缓存 | %LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node 下积累了 20 个 .staging-* 目录 |
| 临时目录缺失项 | bin\node_repl.exe 和 manifest.json |
| 安装包源文件 | 对应文件存在于安装包的 app\resources\cua_node |
| 修复结果 | xcopy /E /I /H /Y /G 复制 2461 个文件到 cua_node\9c26d707817e90a7，重新启动后窗口打开 |

1. **先确认程序启动到了哪一步。** 主进程已经创建，但窗口句柄为 0，且缺少 renderer。单看句柄为 0 只能说明检查时没有可识别的主窗口。和 renderer 缺失、运行时 NotAvailable 一起看，才值得继续追查启动初始化。
2. **检查运行时临时目录。** 缓存里有 20 个 .staging-* 目录。检查发现临时目录缺少 node_repl.exe 和 manifest.json，说明运行时没有完整落盘。
3. **对照同版本的安装包。** 安装包里的对应文件都在，排查范围因此缩小到从安装目录搬运运行时到用户缓存的环节。
4. **完整复制并重启。** 从失败的 staging 目录名识别出运行时 ID 9c26d707817e90a7。使用带 /G 的 xcopy 将整套目录复制到该 ID 对应的正式缓存位置，终端报告复制了 2461 个文件。重新启动后窗口成功打开。

## 为什么这个方法能恢复启动

从目录结构和现象推断，应用准备 cua_node 时会先写入 .staging-<运行时 ID>-<随机后缀>，再让完整内容出现在以运行时 ID 命名的正式目录。复制安装包里的完整运行时后，应用具备了启动所需的缓存文件。运行时 ID 和 MSIX 版本号是两个不同的值。

社区报告也记录了安装包内文件存在、staging 不完整，以及使用 /G 后恢复启动的现象。按 Microsoft 的说明，/G 允许在目标不支持加密时创建解密的目标文件。这与受保护的安装包资源搬运失败的解释相符。本次记录能确认文件缺失和修复结果，底层 Windows 错误码仍未确认。 [openai/codex #42412](https://github.com/openai/codex/issues/42412) · [Microsoft Learn 的 xcopy 文档](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/xcopy)

## 复现检查

启动应用后，在发生故障的 Windows 账号下打开 PowerShell，等待启动过程稳定，再执行以下只读检查。命令只展示必要字段。

~~~powershell
Get-AppxPackage -Name 'OpenAI.Codex' | Select-Object Name, Version
Get-Process -Name ChatGPT -ErrorAction SilentlyContinue |
    Select-Object Id, MainWindowHandle
Get-CimInstance Win32_Process -Filter "Name='ChatGPT.exe'" |
    Where-Object { $_.CommandLine -match '(?:^|\s)--type=renderer(?:\s|$)' } |
    Select-Object ProcessId
~~~

## 安全执行修复

1. 保存工作并退出 ChatGPT/Codex。若无窗口且进程残留，可在任务管理器结束属于本次应用的 ChatGPT.exe。使用同一个 Windows 账号，暂停手动更新，并在复制完成前保持应用关闭。
2. **先备份。** 将 %LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node 中需要保留的缓存复制到该目录之外的本地位置。若同 ID 的正式目录已存在，先备份并确认内容。只有确认该目录不完整、且没有进程使用它时，才手动将它移出原位置，给新副本留出位置。保留 staging 作为排查证据。
3. 先触发一次能复现故障的启动，再退出应用并尽快运行下面整段脚本。脚本只采纳最近 15 分钟创建的最新 staging，要求其缺失特征和本案例一致，并在写入前让你核对包版本和运行时 ID。若在这次故障启动后更新过应用，先用更新后的版本重新复现。
4. 脚本只从当前账号已安装的 MSIX 读取运行时文件。请使用本机的官方安装包作为来源。

~~~powershell
& {
    $ErrorActionPreference = 'Stop'
    $packages = @(Get-AppxPackage -Name 'OpenAI.Codex' |
        Where-Object { -not $_.IsFramework -and -not $_.IsResourcePackage })
    if ($packages.Count -ne 1 -or -not $packages[0].InstallLocation) {
        throw '无法唯一确认 OpenAI.Codex 安装包。'
    }
    $package = $packages[0]
    $source = Join-Path $package.InstallLocation 'app\resources\cua_node'
    $cache = Join-Path $env:LOCALAPPDATA 'OpenAI\Codex\runtimes\cua_node'

    # 遇到被重定向的缓存目录时停止。
    $directory = Get-Item -LiteralPath $cache -Force
    while ($directory) {
        if ($directory.Attributes -band [IO.FileAttributes]::ReparsePoint) {
            throw '缓存路径含有链接或联接点，请先核对实际位置。'
        }
        $directory = $directory.Parent
    }

    $staging = Get-ChildItem -LiteralPath $cache -Directory -Force -Filter '.staging-*' |
        Sort-Object CreationTime -Descending |
        Select-Object -First 1
    if (-not $staging -or $staging.CreationTime -lt (Get-Date).AddMinutes(-15) -or
        ($staging.Attributes -band [IO.FileAttributes]::ReparsePoint) -or
        $staging.Name -notmatch '^\.staging-([0-9a-f]{16})-[a-z0-9_-]+$') {
        throw '没有可用的新 staging，请重新复现故障并核对目录。'
    }
    $runtimeId = $Matches[1]
    $destination = Join-Path $cache $runtimeId
    $required = @('bin\node.exe', 'bin\node_repl.exe', 'manifest.json')
    foreach ($relative in $required) {
        if (-not (Test-Path -LiteralPath (Join-Path $source $relative) -PathType Leaf)) {
            throw "安装包源文件缺失或不可读：$relative"
        }
    }
    foreach ($relative in @('bin\node_repl.exe', 'manifest.json')) {
        if (Test-Path -LiteralPath (Join-Path $staging.FullName $relative)) {
            throw '最新 staging 的缺失特征与本案例不同，请先检查。'
        }
    }
    if (Test-Path -LiteralPath $destination) {
        throw '正式运行时目录已存在。请检查并备份，脚本会保留现有目录。'
    }

    Write-Host "应用包版本 $($package.Version)"
    Write-Host "最新 staging $($staging.Name)"
    Write-Host "目标运行时 ID $runtimeId"
    if ((Read-Host '确认已备份且应用已退出，输入运行时 ID 继续') -ne $runtimeId) {
        return
    }
    if (Get-Process -Name ChatGPT -ErrorAction SilentlyContinue) {
        throw '检测到 ChatGPT 进程，请先退出应用。'
    }

    # 先复制到独立目录。只有复制和校验成功后才启用该副本。
    $workName = '.repair-' + [guid]::NewGuid().ToString('N').Substring(0, 12)
    $work = Join-Path $cache $workName
    New-Item -Path $work -ItemType Directory | Out-Null
    $xcopy = Join-Path $env:SystemRoot 'System32\xcopy.exe'
    & $xcopy (Join-Path $source '*') $work /E /I /H /Y /G
    $copyExitCode = $LASTEXITCODE
    if ($copyExitCode -ne 0) {
        throw "xcopy 退出码 $copyExitCode。部分副本保留在 $workName。"
    }
    foreach ($relative in $required) {
        $sourceHash = (Get-FileHash -LiteralPath (Join-Path $source $relative)).Hash
        $copyHash = (Get-FileHash -LiteralPath (Join-Path $work $relative)).Hash
        if ($sourceHash -ne $copyHash) {
            throw "关键文件校验失败：$relative"
        }
    }
    $current = @(Get-AppxPackage -Name 'OpenAI.Codex' |
        Where-Object { $_.PackageFullName -eq $package.PackageFullName })
    if ($current.Count -ne 1 -or (Get-Process -Name ChatGPT -ErrorAction SilentlyContinue)) {
        throw "安装包或进程状态发生变化。副本保留在 $workName。"
    }
    Rename-Item -LiteralPath $work -NewName $runtimeId
    Write-Host "运行时 $runtimeId 已就绪。现在可从开始菜单启动 ChatGPT。"
}
~~~

/E 包含子目录和空目录，/I 将目标按目录处理，/H 包含隐藏及系统文件，/Y 省去覆盖提示，/G 处理上文所述的解密目标场景。这里的 /Y 用于新建的临时副本。重命名时如果正式目录已出现，PowerShell 会报错并保留临时副本。 [Microsoft Learn 的 xcopy 文档](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/xcopy)

## 验证与回退

- **文件。** 正式 ID 目录应包含 bin\node.exe、bin\node_repl.exe 和 manifest.json。脚本在启用副本前核对这三个文件的 SHA-256。复制数量随安装包变化，2461 是本次实测值。
- **界面。** 从开始菜单启动，确认窗口可见、能交互，再退出并启动一次。需要进一步确认时，重跑只读检查，查看是否出现 renderer 以及是否有 ChatGPT 进程取得非零主窗口句柄。对比本次启动前后的 staging，确认同一 ID 是否还在反复产生失败目录。
- **回退。** 退出应用后，将本次创建的正式 ID 目录移到缓存之外。如先前移出了同名目录，再把备份恢复到原位置。复制中断留下的 .repair-* 可保留用于检查。恢复时只处理本次对应的目录。

## 常见报错

| 现象 | 处理 |
| --- | --- |
| 找不到安装包，或有多个匹配项 | 在发生故障的账号下执行 Get-AppxPackage -Name OpenAI.Codex，确认实际包名、版本和安装类型。 |
| 最新 staging 太旧、目录格式不同，或两个文件的缺失特征不匹配 | 重新复现后查看新目录。如果仍不匹配，按当前启动日志定位故障。脚本只识别本案例使用的 16 位十六进制 ID。 |
| Access denied，或提示源文件不可读 | 核对源路径是否来自当前包的 InstallLocation，并确认以故障账号运行。如果官方包资源仍无法读取，检查 Microsoft Store 中应用的安装状态。保留 WindowsApps 的所有权和权限设置。 |
| xcopy 返回 1、2、4 或 5 | 分别检查是否找不到匹配文件、是否被中断、初始化或路径是否有问题、磁盘是否可写。返回 4 也可能涉及剩余空间，路径过长时可能出现内存不足提示。以终端的具体错误为准。 |
| 正式目录存在，或复制后仍没有窗口 | 先检查该目录三个关键文件、当前包版本以及最新启动时的 staging。若目录已完整，结合 renderer 状态和启动日志继续定位。 |

## 更新后复发怎么办

先重新查询 MSIX 版本，再观察这次失败启动产生的最新 staging。如果仍同时出现无窗口、renderer 缺失和 staging 文件缺失，就从当前安装包复制一份新的运行时，按新目录名识别 ID。每次都使用同版本包中的完整文件。社区 issue 记录了更新后复发的案例，因此将此流程作为启动故障的临时恢复手段。分享日志时仅保留包版本、运行时 ID、文件存在状态、复制退出码和窗口结果，去掉账号名及完整用户目录。 [openai/codex #42412](https://github.com/openai/codex/issues/42412)
