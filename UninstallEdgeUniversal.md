#Requires -RunAsAdministrator
<#
Remove-Edge-IE-All.ps1

Universal Windows 10/11 script to:
- Stop Edge / IE related processes
- Disable Internet Explorer optional feature
- Remove legacy Edge AppX packages if present
- Run Edge Chromium uninstallers if present
- Remove Edge services and scheduled tasks
- Remove system-level Edge / IE leftover folders
- Remove selected Edge / IE registry leftovers
- Recreate HKLM:\SOFTWARE\Microsoft\EdgeUpdate
- Set DWORD DoNotUpdateToEdgeWithChromium = 1

Usage:
  Dry run:
    .\Remove-Edge-IE-All.ps1

  Apply:
    .\Remove-Edge-IE-All.ps1 -Apply

  Apply with protected-folder ownership takeover:
    .\Remove-Edge-IE-All.ps1 -Apply -AggressiveFolderRemoval

  Also clean user profile data:
    .\Remove-Edge-IE-All.ps1 -Apply -AggressiveFolderRemoval -IncludeUserProfiles

  Also remove WebView2 Runtime:
    .\Remove-Edge-IE-All.ps1 -Apply -AggressiveFolderRemoval -IncludeWebView2
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

$LogRoot = Join-Path $env:ProgramData "EdgeIECleanup"
New-Item -ItemType Directory -Path $LogRoot -Force | Out-Null

$LogFile = Join-Path $LogRoot ("cleanup-{0}.log" -f (Get-Date -Format "yyyyMMdd-HHmmss"))
Start-Transcript -Path $LogFile -Force | Out-Null

$OS = Get-CimInstance Win32_OperatingSystem
Write-Host ""
Write-Host "Detected OS: $($OS.Caption) build $($OS.BuildNumber)"
Write-Host "Log file: $LogFile"

if (-not $Apply) {
    Write-Warning "Dry run mode. No changes will be made. Re-run with -Apply to actually remove items."
}

if ($IncludeWebView2) {
    Write-Warning "IncludeWebView2 enabled. Removing WebView2 may break apps that depend on it."
}

function Invoke-Action {
    param(
        [string]$Name,
        [scriptblock]$Action
    )

    Write-Host ""
    Write-Host "== $Name =="

    try {
        & $Action
    }
    catch {
        Write-Warning "$Name failed: $($_.Exception.Message)"
    }
}

function Stop-NamedProcesses {
    param([string[]]$Names)

    foreach ($name in $Names) {
        Get-Process -Name $name -ErrorAction SilentlyContinue | ForEach-Object {
            Write-Host "Stopping process: $($_.ProcessName) PID $($_.Id)"
            if ($Apply) {
                Stop-Process -Id $_.Id -Force -ErrorAction SilentlyContinue
            }
        }
    }
}

function Run-Exe {
    param(
        [string]$Path,
        [string[]]$Args
    )

    if (-not (Test-Path -LiteralPath $Path)) {
        return
    }

    Write-Host "Running: `"$Path`" $($Args -join ' ')"

    if ($Apply) {
        try {
            $p = Start-Process -FilePath $Path -ArgumentList $Args -Wait -PassThru -WindowStyle Hidden
            Write-Host "Exit code: $($p.ExitCode)"
        }
        catch {
            Write-Warning "Failed to run $Path : $($_.Exception.Message)"
        }
    }
}

function Remove-PathSafe {
    param([string]$Path)

    if ([string]::IsNullOrWhiteSpace($Path)) {
        return
    }

    if (-not (Test-Path -LiteralPath $Path)) {
        return
    }

    Write-Host "Removing path: $Path"

    if (-not $Apply) {
        return
    }

    try {
        Remove-Item -LiteralPath $Path -Recurse -Force -ErrorAction Stop
        return
    }
    catch {
        Write-Warning "Normal removal failed: $Path"
    }

    if ($AggressiveFolderRemoval) {
        Write-Warning "Taking ownership and retrying: $Path"

        try {
            & takeown.exe /F "$Path" /R /D Y | Out-Null
            & icacls.exe "$Path" /grant "*S-1-5-32-544:F" /T /C | Out-Null
            Remove-Item -LiteralPath $Path -Recurse -Force -ErrorAction Stop
        }
        catch {
            Write-Warning "Aggressive removal failed: $Path : $($_.Exception.Message)"
        }
    }
}

function Remove-RegPathSafe {
    param([string]$Path)

    if (-not (Test-Path -LiteralPath $Path)) {
        return
    }

    Write-Host "Removing registry path: $Path"

    if ($Apply) {
        try {
            Remove-Item -LiteralPath $Path -Recurse -Force -ErrorAction Stop
        }
        catch {
            Write-Warning "Registry removal failed: $Path : $($_.Exception.Message)"
        }
    }
}

function Disable-InternetExplorer {
    $features = Get-WindowsOptionalFeature -Online -ErrorAction SilentlyContinue |
        Where-Object { $_.FeatureName -like "Internet-Explorer-Optional*" }

    if (-not $features) {
        Write-Host "Internet Explorer optional feature not found."
        return
    }

    foreach ($feature in $features) {
        Write-Host "Disabling/removing IE feature: $($feature.FeatureName)"

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
                Write-Warning "Retrying without -Remove for $($feature.FeatureName)"

                Disable-WindowsOptionalFeature `
                    -Online `
                    -FeatureName $feature.FeatureName `
                    -NoRestart `
                    -ErrorAction SilentlyContinue | Out-Host
            }
        }
    }
}

function Remove-LegacyEdgeAppx {
    Get-AppxPackage -AllUsers -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -like "Microsoft.MicrosoftEdge*" } |
        ForEach-Object {
            Write-Host "Removing AppX package: $($_.PackageFullName)"

            if ($Apply) {
                Remove-AppxPackage -Package $_.PackageFullName -AllUsers -ErrorAction SilentlyContinue
            }
        }

    Get-AppxProvisionedPackage -Online -ErrorAction SilentlyContinue |
        Where-Object { $_.DisplayName -like "Microsoft.MicrosoftEdge*" } |
        ForEach-Object {
            Write-Host "Removing provisioned AppX package: $($_.PackageName)"

            if ($Apply) {
                Remove-AppxProvisionedPackage -Online -PackageName $_.PackageName -ErrorAction SilentlyContinue | Out-Host
            }
        }
}

function Get-EdgeSetups {
    param([string[]]$Products)

    $bases = @(
        $env:ProgramFiles,
        ${env:ProgramFiles(x86)}
    ) | Where-Object { -not [string]::IsNullOrWhiteSpace($_) } | Select-Object -Unique

    $setups = @()

    foreach ($base in $bases) {
        foreach ($product in $Products) {
            $appRoot = Join-Path $base "Microsoft\$product\Application"
            $root = Join-Path $base "Microsoft\$product"

            foreach ($scanRoot in @($appRoot, $root)) {
                if (-not (Test-Path -LiteralPath $scanRoot)) {
                    continue
                }

                Get-ChildItem -LiteralPath $scanRoot -Directory -ErrorAction SilentlyContinue | ForEach-Object {
                    $setup = Join-Path $_.FullName "Installer\setup.exe"

                    if (Test-Path -LiteralPath $setup) {
                        $setups += Get-Item -LiteralPath $setup
                    }
                }
            }
        }
    }

    $setups | Sort-Object FullName -Unique
}

function Uninstall-EdgeChromium {
    $setups = Get-EdgeSetups -Products @("Edge", "EdgeCore")

    foreach ($setup in $setups) {
        Run-Exe -Path $setup.FullName -Args @(
            "--uninstall",
            "--system-level",
            "--verbose-logging",
            "--force-uninstall",
            "--msedge"
        )

        Run-Exe -Path $setup.FullName -Args @(
            "--uninstall",
            "--system-level",
            "--verbose-logging",
            "--force-uninstall"
        )
    }
}

function Uninstall-WebView2 {
    if (-not $IncludeWebView2) {
        return
    }

    $setups = Get-EdgeSetups -Products @("EdgeWebView")

    foreach ($setup in $setups) {
        Run-Exe -Path $setup.FullName -Args @(
            "--uninstall",
            "--system-level",
            "--verbose-logging",
            "--force-uninstall",
            "--msedgewebview"
        )
    }
}

function Remove-EdgeServicesAndTasks {
    $services = @(
        "edgeupdate",
        "edgeupdatem",
        "MicrosoftEdgeElevationService"
    )

    foreach ($service in $services) {
        $svc = Get-Service -Name $service -ErrorAction SilentlyContinue

        if ($null -ne $svc) {
            Write-Host "Disabling/deleting service: $service"

            if ($Apply) {
                Stop-Service -Name $service -Force -ErrorAction SilentlyContinue
                Set-Service -Name $service -StartupType Disabled -ErrorAction SilentlyContinue
                & sc.exe delete "$service" | Out-Null
            }
        }
    }

    Get-ScheduledTask -ErrorAction SilentlyContinue |
        Where-Object {
            $_.TaskName -like "MicrosoftEdgeUpdate*" -or
            $_.TaskPath -like "\Microsoft\EdgeUpdate*"
        } |
        ForEach-Object {
            Write-Host "Removing scheduled task: $($_.TaskPath)$($_.TaskName)"

            if ($Apply) {
                Unregister-ScheduledTask `
                    -TaskName $_.TaskName `
                    -TaskPath $_.TaskPath `
                    -Confirm:$false `
                    -ErrorAction SilentlyContinue
            }
        }
}

function Remove-SystemFolders {
    $pf = $env:ProgramFiles
    $pf86 = ${env:ProgramFiles(x86)}
    $pd = $env:ProgramData
    $publicDesktop = Join-Path $env:PUBLIC "Desktop"

    $paths = @(
        $(if ($pf) { Join-Path $pf "Microsoft\Edge" }),
        $(if ($pf) { Join-Path $pf "Microsoft\EdgeCore" }),
        $(if ($pf) { Join-Path $pf "Microsoft\EdgeUpdate" }),

        $(if ($pf86) { Join-Path $pf86 "Microsoft\Edge" }),
        $(if ($pf86) { Join-Path $pf86 "Microsoft\EdgeCore" }),
        $(if ($pf86) { Join-Path $pf86 "Microsoft\EdgeUpdate" }),

        $(if ($pd) { Join-Path $pd "Microsoft\EdgeUpdate" }),
        $(if ($pd) { Join-Path $pd "Microsoft\Windows\Start Menu\Programs\Microsoft Edge.lnk" }),

        $(if ($publicDesktop) { Join-Path $publicDesktop "Microsoft Edge.lnk" }),

        $(if ($pf) { Join-Path $pf "Internet Explorer" }),
        $(if ($pf86) { Join-Path $pf86 "Internet Explorer" }),
        $(if ($pd) { Join-Path $pd "Microsoft\Windows\Start Menu\Programs\Accessories\Internet Explorer.lnk" })
    )

    if ($IncludeWebView2) {
        $paths += @(
            $(if ($pf) { Join-Path $pf "Microsoft\EdgeWebView" }),
            $(if ($pf86) { Join-Path $pf86 "Microsoft\EdgeWebView" })
        )
    }

    $paths |
        Where-Object { -not [string]::IsNullOrWhiteSpace($_) } |
        Select-Object -Unique |
        ForEach-Object {
            Remove-PathSafe -Path $_
        }
}

function Remove-UserProfileFolders {
    if (-not $IncludeUserProfiles) {
        return
    }

    $usersRoot = Join-Path $env:SystemDrive "Users"

    if (-not (Test-Path -LiteralPath $usersRoot)) {
        return
    }

    Get-ChildItem -LiteralPath $usersRoot -Directory -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -notin @("Default", "Default User", "Public", "All Users") } |
        ForEach-Object {
            $profile = $_.FullName

            $paths = @(
                (Join-Path $profile "AppData\Local\Microsoft\Edge"),
                (Join-Path $profile "AppData\Local\Microsoft\Edge SxS"),
                (Join-Path $profile "AppData\Local\Microsoft\EdgeUpdate"),
                (Join-Path $profile "AppData\Local\Microsoft\Internet Explorer"),
                (Join-Path $profile "AppData\Roaming\Microsoft\Internet Explorer"),
                (Join-Path $profile "AppData\Local\Packages\Microsoft.MicrosoftEdge_8wekyb3d8bbwe")
            )

            if ($IncludeWebView2) {
                $paths += @(
                    (Join-Path $profile "AppData\Local\Microsoft\EdgeWebView")
                )
            }

            $paths | ForEach-Object {
                Remove-PathSafe -Path $_
            }
        }
}

function Remove-EdgeIERegistryLeftovers {
    $regPaths = @(
        "HKLM:\SOFTWARE\Microsoft\Edge",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Edge",
        "HKLM:\SOFTWARE\Microsoft\EdgeUpdate",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate",
        "HKLM:\SOFTWARE\Clients\StartMenuInternet\Microsoft Edge",
        "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\msedge.exe",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\App Paths\msedge.exe",
        "HKLM:\SOFTWARE\Microsoft\Internet Explorer",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Internet Explorer"
    )

    if ($IncludeWebView2) {
        $regPaths += @(
            "HKLM:\SOFTWARE\Microsoft\EdgeWebView",
            "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeWebView"
        )
    }

    foreach ($path in $regPaths) {
        Remove-RegPathSafe -Path $path
    }

    $uninstallRoots = @(
        "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall"
    )

    foreach ($root in $uninstallRoots) {
        if (-not (Test-Path -LiteralPath $root)) {
            continue
        }

        Get-ChildItem -LiteralPath $root -ErrorAction SilentlyContinue | ForEach-Object {
            $props = Get-ItemProperty -LiteralPath $_.PSPath -ErrorAction SilentlyContinue
            $displayName = $props.DisplayName

            if ([string]::IsNullOrWhiteSpace($displayName)) {
                return
            }

            $match = $displayName -eq "Microsoft Edge" -or
                     $displayName -eq "Microsoft Edge Update" -or
                     $displayName -like "Internet Explorer*" -or
                     ($IncludeWebView2 -and $displayName -like "Microsoft Edge WebView2*")

            if ($match) {
                Remove-RegPathSafe -Path $_.PSPath
            }
        }
    }
}

function Set-EdgeUpdateBlockRegistryKey {
    $path = "HKLM:\SOFTWARE\Microsoft\EdgeUpdate"
    $name = "DoNotUpdateToEdgeWithChromium"

    Write-Host "Creating registry key: $path"
    Write-Host "Setting DWORD: $name = 1"

    if (-not $Apply) {
        return
    }

    try {
        New-Item -Path $path -Force | Out-Null

        New-ItemProperty `
            -Path $path `
            -Name $name `
            -PropertyType DWord `
            -Value 1 `
            -Force | Out-Null

        Write-Host "Registry value set successfully."
    }
    catch {
        Write-Warning "Failed to set EdgeUpdate registry value: $($_.Exception.Message)"
    }
}

function Final-Check {
    $pf = $env:ProgramFiles
    $pf86 = ${env:ProgramFiles(x86)}

    $paths = @(
        $(if ($pf) { Join-Path $pf "Microsoft\Edge" }),
        $(if ($pf86) { Join-Path $pf86 "Microsoft\Edge" }),
        $(if ($pf) { Join-Path $pf "Internet Explorer" }),
        $(if ($pf86) { Join-Path $pf86 "Internet Explorer" })
    )

    if ($IncludeWebView2) {
        $paths += @(
            $(if ($pf) { Join-Path $pf "Microsoft\EdgeWebView" }),
            $(if ($pf86) { Join-Path $pf86 "Microsoft\EdgeWebView" })
        )
    }

    foreach ($path in ($paths | Where-Object { -not [string]::IsNullOrWhiteSpace($_) } | Select-Object -Unique)) {
        if (Test-Path -LiteralPath $path) {
            Write-Warning "Still exists: $path"
        }
        else {
            Write-Host "Gone: $path"
        }
    }

    $blockKey = "HKLM:\SOFTWARE\Microsoft\EdgeUpdate"
    $value = Get-ItemProperty -Path $blockKey -Name "DoNotUpdateToEdgeWithChromium" -ErrorAction SilentlyContinue

    if ($value.DoNotUpdateToEdgeWithChromium -eq 1) {
        Write-Host "Confirmed registry value: HKLM\SOFTWARE\Microsoft\EdgeUpdate\DoNotUpdateToEdgeWithChromium = 1"
    }
    else {
        Write-Warning "Registry value was not confirmed."
    }
}

Invoke-Action "Create restore point" {
    if ($Apply) {
        Checkpoint-Computer `
            -Description "Before Edge and IE removal" `
            -RestorePointType "MODIFY_SETTINGS" `
            -ErrorAction SilentlyContinue
    }
}

Invoke-Action "Stop Edge and IE processes" {
    $processes = @(
        "msedge",
        "MicrosoftEdgeUpdate",
        "MicrosoftEdgeCP",
        "MicrosoftEdgeSH",
        "browser_broker",
        "iexplore"
    )

    if ($IncludeWebView2) {
        $processes += "msedgewebview2"
    }

    Stop-NamedProcesses -Names $processes
}

Invoke-Action "Disable Internet Explorer optional feature" {
    Disable-InternetExplorer
}

Invoke-Action "Remove legacy Edge AppX packages" {
    Remove-LegacyEdgeAppx
}

Invoke-Action "Run Edge Chromium uninstallers" {
    Uninstall-EdgeChromium
}

Invoke-Action "Run WebView2 uninstallers if requested" {
    Uninstall-WebView2
}

Invoke-Action "Remove Edge services and scheduled tasks" {
    Remove-EdgeServicesAndTasks
}

Invoke-Action "Remove system-level folders and shortcuts" {
    Remove-SystemFolders
}

Invoke-Action "Remove user profile folders if requested" {
    Remove-UserProfileFolders
}

Invoke-Action "Remove Edge and IE registry leftovers" {
    Remove-EdgeIERegistryLeftovers
}

Invoke-Action "Create EdgeUpdate registry key and block Edge Chromium update" {
    Set-EdgeUpdateBlockRegistryKey
}

Invoke-Action "Final check" {
    Final-Check
}

Write-Host ""
Write-Host "Cleanup finished."
Write-Host "A reboot is strongly recommended."
Write-Host "Log file: $LogFile"

Stop-Transcript | Out-Null
