<# 
Remove-EdgeIE.ps1

Aggressively removes/disables Microsoft Edge and Internet Explorer remnants
on Windows 10 and Windows 11.

Run as Administrator.

Dry run:
  .\Remove-EdgeIE.ps1

Apply changes:
  .\Remove-EdgeIE.ps1 -Apply -AggressiveFolderRemoval

More aggressive:
  .\Remove-EdgeIE.ps1 -Apply -AggressiveFolderRemoval -IncludeUserProfiles

Optional, risky:
  .\Remove-EdgeIE.ps1 -Apply -AggressiveFolderRemoval -IncludeWebView2
#>

[CmdletBinding()]
param(
    [switch]$Apply,
    [switch]$AggressiveFolderRemoval,
    [switch]$IncludeUserProfiles,
    [switch]$IncludeWebView2
)

Set-StrictMode -Version 2.0
$ErrorActionPreference = "Continue"

$IsAdmin = ([Security.Principal.WindowsPrincipal] `
    [Security.Principal.WindowsIdentity]::GetCurrent()
).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)

if (-not $IsAdmin) {
    throw "Run this script from an elevated PowerShell session: right-click PowerShell and choose 'Run as administrator'."
}

$LogRoot = Join-Path $env:ProgramData "EdgeIECleanup"
New-Item -ItemType Directory -Path $LogRoot -Force | Out-Null
$LogFile = Join-Path $LogRoot ("cleanup-{0}.log" -f (Get-Date -Format "yyyyMMdd-HHmmss"))

Start-Transcript -Path $LogFile -Force | Out-Null

$Os = Get-CimInstance Win32_OperatingSystem
$Build = [int]$Os.BuildNumber

Write-Host ""
Write-Host "Detected OS: $($Os.Caption) build $Build"
Write-Host "Log file: $LogFile"

if (-not $Apply) {
    Write-Warning "Dry run mode. No changes will be made. Re-run with -Apply to actually remove items."
}

if ($IncludeWebView2) {
    Write-Warning "IncludeWebView2 is enabled. This may break apps that use Microsoft Edge WebView2 Runtime."
}

function Invoke-IfApply {
    param(
        [Parameter(Mandatory = $true)][string]$Label,
        [Parameter(Mandatory = $true)][scriptblock]$ScriptBlock
    )

    Write-Host ""
    Write-Host "== $Label =="

    if (-not $Apply) {
        Write-Host "Dry run: skipped."
        return
    }

    try {
        & $ScriptBlock
    }
    catch {
        Write-Warning "$Label failed: $($_.Exception.Message)"
    }
}

function Invoke-ProcessLogged {
    param(
        [Parameter(Mandatory = $true)][string]$FilePath,
        [Parameter(Mandatory = $true)][string[]]$Arguments
    )

    if (-not (Test-Path -LiteralPath $FilePath)) {
        return
    }

    Write-Host "Run: `"$FilePath`" $($Arguments -join ' ')"

    if (-not $Apply) {
        return
    }

    try {
        $p = Start-Process -FilePath $FilePath `
            -ArgumentList $Arguments `
            -Wait `
            -PassThru `
            -WindowStyle Hidden

        Write-Host "Exit code: $($p.ExitCode)"
    }
    catch {
        Write-Warning "Failed to run $FilePath : $($_.Exception.Message)"
    }
}

function Remove-PathSafe {
    param(
        [Parameter(Mandatory = $true)][string]$Path
    )

    if ([string]::IsNullOrWhiteSpace($Path)) {
        return
    }

    if (-not (Test-Path -LiteralPath $Path)) {
        return
    }

    Write-Host "Remove path: $Path"

    if (-not $Apply) {
        return
    }

    try {
        Remove-Item -LiteralPath $Path -Recurse -Force -ErrorAction Stop
        Write-Host "Removed: $Path"
        return
    }
    catch {
        Write-Warning "Normal removal failed for $Path : $($_.Exception.Message)"
    }

    if ($AggressiveFolderRemoval) {
        Write-Warning "Taking ownership and retrying removal: $Path"

        try {
            & takeown.exe /F "$Path" /R /D Y | Out-Null
            & icacls.exe "$Path" /grant "*S-1-5-32-544:F" /T /C | Out-Null
            Remove-Item -LiteralPath $Path -Recurse -Force -ErrorAction Stop
            Write-Host "Removed after ownership change: $Path"
        }
        catch {
            Write-Warning "Aggressive removal failed for $Path : $($_.Exception.Message)"
        }
    }
    else {
        Write-Warning "Use -AggressiveFolderRemoval to take ownership and retry locked/protected folders."
    }
}

function Remove-RegPathSafe {
    param(
        [Parameter(Mandatory = $true)][string]$Path
    )

    if (-not (Test-Path -LiteralPath $Path)) {
        return
    }

    Write-Host "Remove registry path: $Path"

    if (-not $Apply) {
        return
    }

    try {
        Remove-Item -LiteralPath $Path -Recurse -Force -ErrorAction Stop
    }
    catch {
        Write-Warning "Registry removal failed for $Path : $($_.Exception.Message)"
    }
}

function Stop-ProcessSafe {
    param([string[]]$Names)

    foreach ($Name in $Names) {
        $procs = Get-Process -Name $Name -ErrorAction SilentlyContinue
        foreach ($p in $procs) {
            Write-Host "Stop process: $($p.ProcessName) PID $($p.Id)"
            if ($Apply) {
                try {
                    Stop-Process -Id $p.Id -Force -ErrorAction Stop
                }
                catch {
                    Write-Warning "Could not stop process $Name : $($_.Exception.Message)"
                }
            }
        }
    }
}

function Get-InstallerSetups {
    param([string[]]$ProductFolders)

    $roots = New-Object System.Collections.Generic.List[string]

    foreach ($base in @($env:ProgramFiles, ${env:ProgramFiles(x86)})) {
        if ([string]::IsNullOrWhiteSpace($base)) {
            continue
        }

        foreach ($product in $ProductFolders) {
            $root = Join-Path $base ("Microsoft\{0}\Application" -f $product)
            if (Test-Path -LiteralPath $root) {
                $roots.Add($root)
            }

            $rootNoApplication = Join-Path $base ("Microsoft\{0}" -f $product)
            if (Test-Path -LiteralPath $rootNoApplication) {
                $roots.Add($rootNoApplication)
            }
        }
    }

    $setups = @()

    foreach ($root in ($roots | Select-Object -Unique)) {
        try {
            $items = Get-ChildItem -LiteralPath $root -Directory -ErrorAction SilentlyContinue
            foreach ($item in $items) {
                $setup = Join-Path $item.FullName "Installer\setup.exe"
                if (Test-Path -LiteralPath $setup) {
                    $setups += Get-Item -LiteralPath $setup
                }
            }
        }
        catch {
            Write-Warning "Could not scan $root : $($_.Exception.Message)"
        }
    }

    $setups | Sort-Object FullName -Unique
}

function Disable-EdgeServices {
    $serviceNames = @(
        "edgeupdate",
        "edgeupdatem",
        "MicrosoftEdgeElevationService"
    )

    foreach ($svcName in $serviceNames) {
        $svc = Get-Service -Name $svcName -ErrorAction SilentlyContinue
        if ($null -eq $svc) {
            continue
        }

        Write-Host "Disable/delete service: $svcName"

        if ($Apply) {
            try {
                Stop-Service -Name $svcName -Force -ErrorAction SilentlyContinue
                Set-Service -Name $svcName -StartupType Disabled -ErrorAction SilentlyContinue
                & sc.exe delete "$svcName" | Out-Null
            }
            catch {
                Write-Warning "Could not remove service $svcName : $($_.Exception.Message)"
            }
        }
    }
}

function Remove-EdgeScheduledTasks {
    try {
        $tasks = Get-ScheduledTask -ErrorAction SilentlyContinue | Where-Object {
            $_.TaskName -like "MicrosoftEdgeUpdate*" -or
            $_.TaskPath -like "\Microsoft\EdgeUpdate*"
        }

        foreach ($task in $tasks) {
            Write-Host "Remove scheduled task: $($task.TaskPath)$($task.TaskName)"
            if ($Apply) {
                try {
                    Unregister-ScheduledTask `
                        -TaskName $task.TaskName `
                        -TaskPath $task.TaskPath `
                        -Confirm:$false `
                        -ErrorAction Stop
                }
                catch {
                    Write-Warning "Could not remove task $($task.TaskName): $($_.Exception.Message)"
                }
            }
        }
    }
    catch {
        Write-Warning "Scheduled task scan failed: $($_.Exception.Message)"
    }
}

function Disable-InternetExplorerFeature {
    try {
        $features = Get-WindowsOptionalFeature -Online -ErrorAction SilentlyContinue |
            Where-Object { $_.FeatureName -like "Internet-Explorer-Optional*" }

        if (-not $features) {
            Write-Host "No Internet Explorer optional feature found on this OS."
            return
        }

        foreach ($feature in $features) {
            Write-Host "Disable/remove Windows optional feature: $($feature.FeatureName) [$($feature.State)]"

            if ($Apply) {
                try {
                    Disable-WindowsOptionalFeature `
                        -Online `
                        -FeatureName $feature.FeatureName `
                        -NoRestart `
                        -Remove `
                        -ErrorAction Stop | Out-Host
                }
                catch {
                    Write-Warning "Disable with -Remove failed for $($feature.FeatureName). Retrying without -Remove."
                    try {
                        Disable-WindowsOptionalFeature `
                            -Online `
                            -FeatureName $feature.FeatureName `
                            -NoRestart `
                            -ErrorAction Stop | Out-Host
                    }
                    catch {
                        Write-Warning "Could not disable $($feature.FeatureName): $($_.Exception.Message)"
                    }
                }
            }
        }
    }
    catch {
        Write-Warning "Internet Explorer feature removal failed: $($_.Exception.Message)"
    }
}

function Remove-LegacyEdgeAppx {
    Write-Host "Scan/remove legacy Microsoft Edge AppX packages."

    try {
        $packages = Get-AppxPackage -AllUsers -ErrorAction SilentlyContinue |
            Where-Object {
                $_.Name -like "Microsoft.MicrosoftEdge*"
            }

        foreach ($pkg in $packages) {
            Write-Host "Remove AppX package: $($pkg.Name) $($pkg.PackageFullName)"
            if ($Apply) {
                try {
                    Remove-AppxPackage -Package $pkg.PackageFullName -AllUsers -ErrorAction Stop
                }
                catch {
                    Write-Warning "Could not remove AppX package $($pkg.PackageFullName): $($_.Exception.Message)"
                }
            }
        }
    }
    catch {
        Write-Warning "AppX package scan failed: $($_.Exception.Message)"
    }

    try {
        $provisioned = Get-AppxProvisionedPackage -Online -ErrorAction SilentlyContinue |
            Where-Object {
                $_.DisplayName -like "Microsoft.MicrosoftEdge*"
            }

        foreach ($pkg in $provisioned) {
            Write-Host "Remove provisioned AppX package: $($pkg.DisplayName) $($pkg.PackageName)"
            if ($Apply) {
                try {
                    Remove-AppxProvisionedPackage -Online -PackageName $pkg.PackageName -ErrorAction Stop | Out-Host
                }
                catch {
                    Write-Warning "Could not remove provisioned package $($pkg.PackageName): $($_.Exception.Message)"
                }
            }
        }
    }
    catch {
        Write-Warning "Provisioned AppX package scan failed: $($_.Exception.Message)"
    }
}

function Remove-EdgeUninstallRegistryEntries {
    $roots = @(
        "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall"
    )

    foreach ($root in $roots) {
        if (-not (Test-Path -LiteralPath $root)) {
            continue
        }

        try {
            $children = Get-ChildItem -LiteralPath $root -ErrorAction SilentlyContinue
            foreach ($child in $children) {
                $props = Get-ItemProperty -LiteralPath $child.PSPath -ErrorAction SilentlyContinue
                $displayName = $props.DisplayName

                if ([string]::IsNullOrWhiteSpace($displayName)) {
                    continue
                }

                $isEdge = $displayName -eq "Microsoft Edge" -or
                    $displayName -eq "Microsoft Edge Update" -or
                    ($IncludeWebView2 -and $displayName -like "Microsoft Edge WebView2*")

                if ($isEdge) {
                    Remove-RegPathSafe -Path $child.PSPath
                }
            }
        }
        catch {
            Write-Warning "Could not scan uninstall registry root $root : $($_.Exception.Message)"
        }
    }
}

Invoke-IfApply "Create a restore point" {
    try {
        Checkpoint-Computer `
            -Description "Before Microsoft Edge and Internet Explorer cleanup" `
            -RestorePointType "MODIFY_SETTINGS" `
            -ErrorAction Stop
    }
    catch {
        Write-Warning "Restore point creation failed or is disabled: $($_.Exception.Message)"
    }
}

Write-Host ""
Write-Host "== Stop browser/update processes =="
Stop-ProcessSafe -Names @(
    "msedge",
    "MicrosoftEdgeUpdate",
    "MicrosoftEdgeCP",
    "MicrosoftEdgeSH",
    "browser_broker",
    "iexplore"
)

if ($IncludeWebView2) {
    Stop-ProcessSafe -Names @("msedgewebview2")
}

Write-Host ""
Write-Host "== Disable/remove Internet Explorer optional feature =="
Disable-InternetExplorerFeature

Write-Host ""
Write-Host "== Remove legacy Edge AppX packages on Windows 10 if present =="
Remove-LegacyEdgeAppx

Write-Host ""
Write-Host "== Run Microsoft Edge Chromium uninstallers if present =="
$edgeSetups = Get-InstallerSetups -ProductFolders @("Edge", "EdgeCore")

foreach ($setup in $edgeSetups) {
    Invoke-ProcessLogged -FilePath $setup.FullName -Arguments @(
        "--uninstall",
        "--system-level",
        "--verbose-logging",
        "--force-uninstall",
        "--msedge"
    )

    Invoke-ProcessLogged -FilePath $setup.FullName -Arguments @(
        "--uninstall",
        "--system-level",
        "--verbose-logging",
        "--force-uninstall"
    )
}

if ($IncludeWebView2) {
    Write-Host ""
    Write-Host "== Run Microsoft Edge WebView2 uninstallers if present =="
    $wvSetups = Get-InstallerSetups -ProductFolders @("EdgeWebView")

    foreach ($setup in $wvSetups) {
        Invoke-ProcessLogged -FilePath $setup.FullName -Arguments @(
            "--uninstall",
            "--system-level",
            "--verbose-logging",
            "--force-uninstall",
            "--msedgewebview"
        )
    }
}

Write-Host ""
Write-Host "== Remove Edge services and scheduled update tasks =="
Disable-EdgeServices
Remove-EdgeScheduledTasks

Write-Host ""
Write-Host "== Remove system-level folders and shortcuts =="

$pf = $env:ProgramFiles
$pf86 = ${env:ProgramFiles(x86)}
$programData = $env:ProgramData
$publicDesktop = Join-Path $env:PUBLIC "Desktop"

$systemPaths = @(
    $(if ($pf) { Join-Path $pf "Microsoft\Edge" }),
    $(if ($pf) { Join-Path $pf "Microsoft\EdgeCore" }),
    $(if ($pf) { Join-Path $pf "Microsoft\EdgeUpdate" }),
    $(if ($pf86) { Join-Path $pf86 "Microsoft\Edge" }),
    $(if ($pf86) { Join-Path $pf86 "Microsoft\EdgeCore" }),
    $(if ($pf86) { Join-Path $pf86 "Microsoft\EdgeUpdate" }),
    $(if ($programData) { Join-Path $programData "Microsoft\EdgeUpdate" }),
    $(if ($programData) { Join-Path $programData "Microsoft\Windows\Start Menu\Programs\Microsoft Edge.lnk" }),
    $(if ($publicDesktop) { Join-Path $publicDesktop "Microsoft Edge.lnk" }),

    $(if ($pf) { Join-Path $pf "Internet Explorer" }),
    $(if ($pf86) { Join-Path $pf86 "Internet Explorer" }),
    $(if ($programData) { Join-Path $programData "Microsoft\Windows\Start Menu\Programs\Accessories\Internet Explorer.lnk" })
)

if ($IncludeWebView2) {
    $systemPaths += @(
        $(if ($pf) { Join-Path $pf "Microsoft\EdgeWebView" }),
        $(if ($pf86) { Join-Path $pf86 "Microsoft\EdgeWebView" })
    )
}

foreach ($path in ($systemPaths | Where-Object { -not [string]::IsNullOrWhiteSpace($_) } | Select-Object -Unique)) {
    Remove-PathSafe -Path $path
}

Write-Host ""
Write-Host "== Remove selected Edge registry remnants =="

$edgeRegPaths = @(
    "HKLM:\SOFTWARE\Microsoft\Edge",
    "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Edge",
    "HKLM:\SOFTWARE\Microsoft\EdgeUpdate",
    "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate",
    "HKLM:\SOFTWARE\Clients\StartMenuInternet\Microsoft Edge",
    "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\App Paths\msedge.exe",
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\msedge.exe"
)

foreach ($regPath in $edgeRegPaths) {
    Remove-RegPathSafe -Path $regPath
}

Remove-EdgeUninstallRegistryEntries

if ($IncludeWebView2) {
    $wvRegPaths = @(
        "HKLM:\SOFTWARE\Microsoft\EdgeWebView",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeWebView"
    )

    foreach ($regPath in $wvRegPaths) {
        Remove-RegPathSafe -Path $regPath
    }
}

if ($IncludeUserProfiles) {
    Write-Host ""
    Write-Host "== Remove Edge/IE data from user profiles =="

    $profilesRoot = Join-Path $env:SystemDrive "Users"

    if (Test-Path -LiteralPath $profilesRoot) {
        $profiles = Get-ChildItem -LiteralPath $profilesRoot -Directory -ErrorAction SilentlyContinue |
            Where-Object {
                $_.Name -notin @(
                    "Default",
                    "Default User",
                    "Public",
                    "All Users"
                )
            }

        foreach ($profile in $profiles) {
            $userPaths = @(
                (Join-Path $profile.FullName "AppData\Local\Microsoft\Edge"),
                (Join-Path $profile.FullName "AppData\Local\Microsoft\Edge SxS"),
                (Join-Path $profile.FullName "AppData\Local\Microsoft\EdgeUpdate"),
                (Join-Path $profile.FullName "AppData\Local\Microsoft\Internet Explorer"),
                (Join-Path $profile.FullName "AppData\Roaming\Microsoft\Internet Explorer"),
                (Join-Path $profile.FullName "AppData\Local\Packages\Microsoft.MicrosoftEdge_8wekyb3d8bbwe")
            )

            foreach ($path in $userPaths) {
                Remove-PathSafe -Path $path
            }
        }
    }
}

Write-Host ""
Write-Host "== Final check =="

$checkPaths = @(
    $(if ($pf) { Join-Path $pf "Microsoft\Edge" }),
    $(if ($pf86) { Join-Path $pf86 "Microsoft\Edge" }),
    $(if ($pf) { Join-Path $pf "Internet Explorer" }),
    $(if ($pf86) { Join-Path $pf86 "Internet Explorer" })
)

if ($IncludeWebView2) {
    $checkPaths += @(
        $(if ($pf) { Join-Path $pf "Microsoft\EdgeWebView" }),
        $(if ($pf86) { Join-Path $pf86 "Microsoft\EdgeWebView" })
    )
}

foreach ($path in ($checkPaths | Where-Object { -not [string]::IsNullOrWhiteSpace($_) } | Select-Object -Unique)) {
    if (Test-Path -LiteralPath $path) {
        Write-Warning "Still exists: $path"
    }
    else {
        Write-Host "Gone: $path"
    }
}

Write-Host ""
Write-Host "Cleanup finished."
Write-Host "A reboot is strongly recommended."

Stop-Transcript | Out-Null