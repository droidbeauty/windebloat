Set-ExecutionPolicy Bypass -Scope Process -Force

<#
.SYNOPSIS
  Aggressively disables Windows telemetry on Windows 10 IoT Enterprise LTSC.

.DESCRIPTION
  Optimized for Windows 10 IoT Enterprise LTSC appliance / fixed-function images.

  This script targets:
    - Windows diagnostic data policy
    - Connected User Experiences and Telemetry
    - CEIP / SQM
    - Application Compatibility telemetry / inventory
    - Windows Error Reporting
    - Feedback prompts
    - Tailored experiences / advertising ID / activity upload
    - Telemetry-related scheduled tasks
    - Telemetry WMI autologgers

  It intentionally does NOT disable:
    - Windows Search / WSearch
    - Windows Update
    - BITS
    - Defender / Security Center
    - Event Log
    - Task Scheduler
    - Store/AppX services
    - Networking services

.NOTES
  Save as:
    Disable-Telemetry-Windows10-IoT-LTSC.ps1

  Run:
    powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Disable-Telemetry-Windows10-IoT-LTSC.ps1

  Reboot after running.
#>

$ErrorActionPreference = "Continue"

# ------------------------------------------------------------
# Self-elevate
# ------------------------------------------------------------

function Test-IsAdmin {
    $identity = [Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = New-Object Security.Principal.WindowsPrincipal($identity)
    return $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

if (-not (Test-IsAdmin)) {
    Write-Host "Restarting PowerShell as Administrator..."

    if (-not $PSCommandPath) {
        Write-Error "This script must be saved as a .ps1 file before it can self-elevate."
        exit 1
    }

    Start-Process powershell.exe -Verb RunAs -ArgumentList @(
        "-NoProfile",
        "-ExecutionPolicy", "Bypass",
        "-File", "`"$PSCommandPath`""
    )

    exit
}

# ------------------------------------------------------------
# Logging
# ------------------------------------------------------------

$LogDir = "$env:ProgramData\TelemetryDisable-Windows10IoTLTSC"
$LogFile = Join-Path $LogDir "Disable-Telemetry-Windows10IoTLTSC-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"

New-Item -ItemType Directory -Force -Path $LogDir | Out-Null
Start-Transcript -Path $LogFile -Force | Out-Null

Write-Host "=== Windows 10 IoT LTSC Telemetry Disable ===" -ForegroundColor Cyan
Write-Host "Log: $LogFile" -ForegroundColor Gray

# ------------------------------------------------------------
# Helpers
# ------------------------------------------------------------

function Set-Dword {
    param(
        [Parameter(Mandatory)] [string]$Path,
        [Parameter(Mandatory)] [string]$Name,
        [Parameter(Mandatory)] [int]$Value
    )

    try {
        if (-not (Test-Path $Path)) {
            New-Item -Path $Path -Force | Out-Null
        }

        New-ItemProperty -Path $Path -Name $Name -Value $Value -PropertyType DWord -Force | Out-Null
        Write-Host "Set DWORD: $Path\$Name = $Value"
    }
    catch {
        Write-Warning "Failed to set DWORD $Path\$Name : $($_.Exception.Message)"
    }
}

function Remove-RegValueSafe {
    param(
        [Parameter(Mandatory)] [string]$Path,
        [Parameter(Mandatory)] [string]$Name
    )

    try {
        if (Test-Path $Path) {
            Remove-ItemProperty -Path $Path -Name $Name -ErrorAction SilentlyContinue
            Write-Host "Removed registry value if present: $Path\$Name"
        }
    }
    catch {
        Write-Warning "Failed to remove registry value $Path\$Name : $($_.Exception.Message)"
    }
}

function Disable-ServiceSafe {
    param([Parameter(Mandatory)] [string]$Name)

    try {
        $services = Get-Service -Name $Name -ErrorAction SilentlyContinue

        if (-not $services) {
            Write-Host "Service not present: $Name"
            return
        }

        foreach ($svc in $services) {
            try {
                if ($svc.Status -ne "Stopped") {
                    Stop-Service -Name $svc.Name -Force -ErrorAction SilentlyContinue
                }

                Set-Service -Name $svc.Name -StartupType Disabled -ErrorAction Stop
                Write-Host "Disabled service: $($svc.Name)"
            }
            catch {
                Write-Warning "Could not disable service $($svc.Name) : $($_.Exception.Message)"
            }
        }
    }
    catch {
        Write-Warning "Could not query service $Name : $($_.Exception.Message)"
    }
}

function Set-ServiceStartupSafe {
    param(
        [Parameter(Mandatory)] [string]$Name,
        [Parameter(Mandatory)] [ValidateSet("Automatic", "Manual", "Disabled")] [string]$StartupType
    )

    try {
        $services = Get-Service -Name $Name -ErrorAction SilentlyContinue

        foreach ($svc in $services) {
            try {
                Set-Service -Name $svc.Name -StartupType $StartupType -ErrorAction Stop
                Write-Host "Set service startup: $($svc.Name) = $StartupType"
            }
            catch {
                Write-Warning "Could not set service startup for $($svc.Name) : $($_.Exception.Message)"
            }
        }
    }
    catch {
        Write-Warning "Could not query service $Name : $($_.Exception.Message)"
    }
}

function Disable-TaskSafe {
    param(
        [Parameter(Mandatory)] [string]$TaskPath,
        [Parameter(Mandatory)] [string]$TaskName
    )

    try {
        $task = Get-ScheduledTask -TaskPath $TaskPath -TaskName $TaskName -ErrorAction SilentlyContinue

        if ($null -eq $task) {
            Write-Host "Scheduled task not present: $TaskPath$TaskName"
            return
        }

        Disable-ScheduledTask -TaskPath $TaskPath -TaskName $TaskName -ErrorAction Stop | Out-Null
        Write-Host "Disabled scheduled task: $TaskPath$TaskName"
    }
    catch {
        Write-Warning "Failed to disable scheduled task $TaskPath$TaskName : $($_.Exception.Message)"
    }
}

function Disable-TaskByFullPathSafe {
    param([Parameter(Mandatory)] [string]$FullTaskPath)

    try {
        $taskPath = $FullTaskPath.Substring(0, $FullTaskPath.LastIndexOf("\") + 1)
        $taskName = $FullTaskPath.Split("\")[-1]
        Disable-TaskSafe -TaskPath $taskPath -TaskName $taskName
    }
    catch {
        Write-Warning "Invalid task path $FullTaskPath : $($_.Exception.Message)"
    }
}

# ------------------------------------------------------------
# Preserve local search and core infrastructure
# ------------------------------------------------------------

Write-Host "`n=== Preserving local search and core services ===" -ForegroundColor Yellow

# Critical: keep local Start/File Explorer search working.
Set-ServiceStartupSafe -Name "WSearch" -StartupType Automatic
try {
    Start-Service -Name "WSearch" -ErrorAction SilentlyContinue
    Write-Host "WSearch started or already running."
}
catch {
    Write-Warning "Could not start WSearch: $($_.Exception.Message)"
}

# Keep these untouched/healthy for general OS stability.
Set-ServiceStartupSafe -Name "EventLog" -StartupType Automatic
Set-ServiceStartupSafe -Name "Schedule" -StartupType Automatic
Set-ServiceStartupSafe -Name "BITS" -StartupType Manual
Set-ServiceStartupSafe -Name "wuauserv" -StartupType Manual

# ------------------------------------------------------------
# Windows diagnostic data policy
# ------------------------------------------------------------

Write-Host "`n=== Applying Windows 10 LTSC diagnostic data policies ===" -ForegroundColor Yellow

$DataCollectionPolicy = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection"
$DataCollectionCurrent = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection"

# Enterprise/LTSC-supported minimum diagnostic data level.
Set-Dword $DataCollectionPolicy "AllowTelemetry" 0
Set-Dword $DataCollectionPolicy "MaxTelemetryAllowed" 0

# Mirror into current-version policy location used by some status/reporting surfaces.
Set-Dword $DataCollectionCurrent "AllowTelemetry" 0
Set-Dword $DataCollectionCurrent "MaxTelemetryAllowed" 0

# Reduce diagnostic UX, prompts, and enterprise diagnostic behavior.
Set-Dword $DataCollectionPolicy "DoNotShowFeedbackNotifications" 1
Set-Dword $DataCollectionPolicy "ConfigureTelemetryOptInSettingsUx" 1
Set-Dword $DataCollectionPolicy "ConfigureTelemetryOptInChangeNotification" 1
Set-Dword $DataCollectionPolicy "DisableDiagnosticDataViewer" 1
Set-Dword $DataCollectionPolicy "AllowDeviceNameInDiagnosticData" 0
Set-Dword $DataCollectionPolicy "DisableEnterpriseAuthProxy" 1
Set-Dword $DataCollectionPolicy "DisableOneSettingsDownloads" 1
Set-Dword $DataCollectionPolicy "LimitEnhancedDiagnosticDataWindowsAnalytics" 1

# These are ignored on older builds that do not support them, but harmless on LTSC images that do.
Set-Dword $DataCollectionPolicy "LimitDiagnosticLogCollection" 1
Set-Dword $DataCollectionPolicy "LimitDumpCollection" 1

# ------------------------------------------------------------
# Windows Error Reporting
# ------------------------------------------------------------

Write-Host "`n=== Disabling Windows Error Reporting ===" -ForegroundColor Yellow

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Windows Error Reporting" "Disabled" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Windows Error Reporting" "DontSendAdditionalData" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Windows Error Reporting" "LoggingDisabled" 1

Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting" "Disabled" 1
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting" "DontSendAdditionalData" 1
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting" "DontSendData" 1
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting" "LoggingDisabled" 1

Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\Consent" "DefaultConsent" 1
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\Consent" "DefaultOverrideBehavior" 1

try {
    Disable-WindowsErrorReporting -ErrorAction SilentlyContinue | Out-Null
    Write-Host "Disable-WindowsErrorReporting invoked."
}
catch {
    Write-Warning "Disable-WindowsErrorReporting failed or is unavailable: $($_.Exception.Message)"
}

# ------------------------------------------------------------
# CEIP / SQM / Application Compatibility inventory
# ------------------------------------------------------------

Write-Host "`n=== Disabling CEIP, SQM, and AppCompat telemetry ===" -ForegroundColor Yellow

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\SQMClient\Windows" "CEIPEnable" 0
Set-Dword "HKLM:\SOFTWARE\Microsoft\SQMClient\Windows" "CEIPEnable" 0

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\SQMClient" "CorporateSQMURL" 0

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppCompat" "AITEnable" 0
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppCompat" "DisableInventory" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppCompat" "DisablePCA" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppCompat" "DisableUAR" 1

Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Appraiser" "HaveUploadedForTarget" 1

# ------------------------------------------------------------
# Privacy-adjacent telemetry surfaces
# ------------------------------------------------------------

Write-Host "`n=== Disabling tailored experiences, advertising ID, feedback, and activity upload ===" -ForegroundColor Yellow

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AdvertisingInfo" "DisabledByGroupPolicy" 1

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent" "DisableTailoredExperiencesWithDiagnosticData" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent" "DisableWindowsConsumerFeatures" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent" "DisableSoftLanding" 1

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" "EnableActivityFeed" 0
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" "PublishUserActivities" 0
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" "UploadUserActivities" 0

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\LocationAndSensors" "DisableLocation" 1

# Keep local search, disable web search from Start/search.
$WindowsSearchPolicy = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Windows Search"
Set-Dword $WindowsSearchPolicy "DisableWebSearch" 1
Set-Dword $WindowsSearchPolicy "ConnectedSearchUseWeb" 0
Set-Dword $WindowsSearchPolicy "AllowCortana" 0
Set-Dword $WindowsSearchPolicy "AllowCloudSearch" 0
Set-Dword $WindowsSearchPolicy "AllowSearchToUseLocation" 0

# Current user privacy values.
Set-Dword "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\AdvertisingInfo" "Enabled" 0
Set-Dword "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Privacy" "TailoredExperiencesWithDiagnosticDataEnabled" 0
Set-Dword "HKCU:\SOFTWARE\Microsoft\Siuf\Rules" "NumberOfSIUFInPeriod" 0
Set-Dword "HKCU:\SOFTWARE\Microsoft\Siuf\Rules" "PeriodInNanoSeconds" 0

# Apply user privacy values to already loaded user hives.
Write-Host "`n=== Applying loaded-user hive telemetry/privacy values ===" -ForegroundColor Yellow

$LoadedUserHives = Get-ChildItem "Registry::HKEY_USERS" -ErrorAction SilentlyContinue |
    Where-Object {
        $_.PSChildName -match "^S-1-5-21-" -and
        $_.PSChildName -notmatch "_Classes$"
    }

foreach ($Hive in $LoadedUserHives) {
    $Root = "Registry::HKEY_USERS\$($Hive.PSChildName)"

    Set-Dword "$Root\SOFTWARE\Microsoft\Windows\CurrentVersion\AdvertisingInfo" "Enabled" 0
    Set-Dword "$Root\SOFTWARE\Microsoft\Windows\CurrentVersion\Privacy" "TailoredExperiencesWithDiagnosticDataEnabled" 0
    Set-Dword "$Root\SOFTWARE\Microsoft\Siuf\Rules" "NumberOfSIUFInPeriod" 0
    Set-Dword "$Root\SOFTWARE\Microsoft\Siuf\Rules" "PeriodInNanoSeconds" 0
}

# ------------------------------------------------------------
# Disable telemetry WMI autologgers
# ------------------------------------------------------------

Write-Host "`n=== Disabling telemetry WMI autologgers ===" -ForegroundColor Yellow

$AutoLoggerKeys = @(
    "HKLM:\SYSTEM\CurrentControlSet\Control\WMI\AutoLogger\AutoLogger-Diagtrack-Listener",
    "HKLM:\SYSTEM\CurrentControlSet\Control\WMI\AutoLogger\SQMLogger"
)

foreach ($key in $AutoLoggerKeys) {
    Set-Dword $key "Start" 0
}

# ------------------------------------------------------------
# Disable telemetry services
# ------------------------------------------------------------

Write-Host "`n=== Disabling telemetry-related services ===" -ForegroundColor Yellow

$TelemetryServicesToDisable = @(
    "DiagTrack",                                # Connected User Experiences and Telemetry
    "dmwappushservice",                        # WAP Push Message Routing Service
    "diagnosticshub.standardcollector.service",# Diagnostics Hub Standard Collector
    "WerSvc",                                  # Windows Error Reporting
    "PcaSvc"                                   # Program Compatibility Assistant
)

foreach ($svc in $TelemetryServicesToDisable) {
    Disable-ServiceSafe -Name $svc
}

# Optional diagnostic infrastructure reduction.
# These are not required for local search or normal app execution, but disabling them removes built-in diagnostics/troubleshooting helpers.
$DiagnosticServicesToDisable = @(
    "DPS",             # Diagnostic Policy Service
    "WdiServiceHost",  # Diagnostic Service Host
    "WdiSystemHost"    # Diagnostic System Host
)

foreach ($svc in $DiagnosticServicesToDisable) {
    Disable-ServiceSafe -Name $svc
}

# ------------------------------------------------------------
# Disable telemetry scheduled tasks
# ------------------------------------------------------------

Write-Host "`n=== Disabling telemetry-related scheduled tasks ===" -ForegroundColor Yellow

$TasksToDisable = @(
    "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser",
    "\Microsoft\Windows\Application Experience\ProgramDataUpdater",
    "\Microsoft\Windows\Application Experience\StartupAppTask",
    "\Microsoft\Windows\Application Experience\AitAgent",

    "\Microsoft\Windows\Autochk\Proxy",

    "\Microsoft\Windows\Customer Experience Improvement Program\Consolidator",
    "\Microsoft\Windows\Customer Experience Improvement Program\KernelCeipTask",
    "\Microsoft\Windows\Customer Experience Improvement Program\UsbCeip",

    "\Microsoft\Windows\DiskDiagnostic\Microsoft-Windows-DiskDiagnosticDataCollector",

    "\Microsoft\Windows\Feedback\Siuf\DmClient",
    "\Microsoft\Windows\Feedback\Siuf\DmClientOnScenarioDownload",

    "\Microsoft\Windows\PI\Sqm-Tasks",

    "\Microsoft\Windows\Power Efficiency Diagnostics\AnalyzeSystem",

    "\Microsoft\Windows\Windows Error Reporting\QueueReporting",

    "\Microsoft\Windows\Maps\MapsToastTask",
    "\Microsoft\Windows\Maps\MapsUpdateTask"
)

foreach ($task in $TasksToDisable | Sort-Object -Unique) {
    Disable-TaskByFullPathSafe -FullTaskPath $task
}

# ------------------------------------------------------------
# Apply local policy
# ------------------------------------------------------------

Write-Host "`n=== Refreshing local policy ===" -ForegroundColor Yellow

try {
    gpupdate.exe /target:computer /force | Out-Host
}
catch {
    Write-Warning "gpupdate failed: $($_.Exception.Message)"
}

# ------------------------------------------------------------
# Verification
# ------------------------------------------------------------

Write-Host "`n=== Verification summary ===" -ForegroundColor Cyan

Write-Host "`nDiagnostic data policy:"
Get-ItemProperty -Path $DataCollectionPolicy -ErrorAction SilentlyContinue |
    Select-Object AllowTelemetry, MaxTelemetryAllowed, DoNotShowFeedbackNotifications, ConfigureTelemetryOptInSettingsUx, DisableDiagnosticDataViewer, AllowDeviceNameInDiagnosticData |
    Format-List

Write-Host "`nTelemetry services:"
$TelemetryServicesToDisable + $DiagnosticServicesToDisable |
    Sort-Object -Unique |
    ForEach-Object { Get-Service -Name $_ -ErrorAction SilentlyContinue } |
    Select-Object Name, Status, StartType |
    Format-Table -AutoSize

Write-Host "`nLocal search service:"
Get-Service -Name "WSearch" -ErrorAction SilentlyContinue |
    Select-Object Name, Status, StartType |
    Format-Table -AutoSize

Write-Host "`nKey scheduled tasks:"
$TasksToDisable |
    ForEach-Object {
        try {
            $taskPath = $_.Substring(0, $_.LastIndexOf("\") + 1)
            $taskName = $_.Split("\")[-1]
            Get-ScheduledTask -TaskPath $taskPath -TaskName $taskName -ErrorAction SilentlyContinue
        }
        catch {}
    } |
    Select-Object TaskPath, TaskName, State |
    Format-Table -AutoSize

Write-Host "`nDone. Reboot the device to fully apply service, autologger, and policy changes." -ForegroundColor Green
Write-Host "Log written to: $LogFile" -ForegroundColor Gray

Stop-Transcript | Out-Null
