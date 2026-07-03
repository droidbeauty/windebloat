#Requires -RunAsAdministrator

[CmdletBinding()]
param(
    [switch]$SkipScheduledTasks,
    [switch]$SkipServices
)

$ErrorActionPreference = "Continue"

$LogDir = "$env:ProgramData\TelemetryDisable"
$LogFile = Join-Path $LogDir "Disable-Telemetry-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"

New-Item -ItemType Directory -Force -Path $LogDir | Out-Null
Start-Transcript -Path $LogFile -Force | Out-Null

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
        Write-Warning "Failed to set $Path\$Name : $($_.Exception.Message)"
    }
}

function Set-String {
    param(
        [Parameter(Mandatory)] [string]$Path,
        [Parameter(Mandatory)] [string]$Name,
        [Parameter(Mandatory)] [string]$Value
    )

    try {
        if (-not (Test-Path $Path)) {
            New-Item -Path $Path -Force | Out-Null
        }

        New-ItemProperty -Path $Path -Name $Name -Value $Value -PropertyType String -Force | Out-Null
        Write-Host "Set STRING: $Path\$Name = $Value"
    }
    catch {
        Write-Warning "Failed to set $Path\$Name : $($_.Exception.Message)"
    }
}

function Disable-ServiceSafe {
    param([Parameter(Mandatory)] [string]$Name)

    try {
        $svc = Get-Service -Name $Name -ErrorAction SilentlyContinue
        if ($null -eq $svc) {
            Write-Host "Service not present: $Name"
            return
        }

        if ($svc.Status -ne "Stopped") {
            Stop-Service -Name $Name -Force -ErrorAction SilentlyContinue
        }

        Set-Service -Name $Name -StartupType Disabled -ErrorAction Stop
        Write-Host "Disabled service: $Name"
    }
    catch {
        Write-Warning "Failed to disable service $Name : $($_.Exception.Message)"
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

Write-Host "`n=== Applying machine-wide telemetry policies ==="

$DataCollection = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection"

# Core diagnostic-data policy.
Set-Dword $DataCollection "AllowTelemetry" 0
Set-Dword $DataCollection "MaxTelemetryAllowed" 0

# Windows 11 diagnostic data restrictions.
Set-Dword $DataCollection "LimitDiagnosticLogCollection" 1
Set-Dword $DataCollection "LimitDumpCollection" 1

# Prevent users from re-enabling diagnostic settings in Settings UI.
Set-Dword $DataCollection "ConfigureTelemetryOptInSettingsUx" 1
Set-Dword $DataCollection "ConfigureTelemetryOptInChangeNotification" 1

# Disable viewer and feedback prompts.
Set-Dword $DataCollection "DisableDiagnosticDataViewer" 1
Set-Dword $DataCollection "DoNotShowFeedbackNotifications" 1

# Reduce diagnostic metadata and cloud config behavior.
Set-Dword $DataCollection "AllowDeviceNameInDiagnosticData" 0
Set-Dword $DataCollection "DisableEnterpriseAuthProxy" 1
Set-Dword $DataCollection "DisableOneSettingsDownloads" 1

# Mirror core setting in non-policy location used by Settings/status surfaces.
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" "AllowTelemetry" 0
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" "MaxTelemetryAllowed" 0

Write-Host "`n=== Disabling Windows Error Reporting ==="

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Windows Error Reporting" "Disabled" 1
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting" "Disabled" 1
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting" "DontSendAdditionalData" 1
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\Consent" "DefaultConsent" 1
Set-Dword "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\Consent" "DefaultOverrideBehavior" 1

try {
    Disable-WindowsErrorReporting -ErrorAction SilentlyContinue | Out-Null
    Write-Host "Disable-WindowsErrorReporting invoked."
}
catch {
    Write-Warning "Disable-WindowsErrorReporting failed or is unavailable: $($_.Exception.Message)"
}

Write-Host "`n=== Disabling CEIP / SQM / compatibility telemetry policies ==="

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\SQMClient\Windows" "CEIPEnable" 0
Set-Dword "HKLM:\SOFTWARE\Microsoft\SQMClient\Windows" "CEIPEnable" 0

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppCompat" "AITEnable" 0
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppCompat" "DisableInventory" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppCompat" "DisablePCA" 1

Write-Host "`n=== Disabling tailored experiences, advertising ID, and activity upload ==="

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AdvertisingInfo" "DisabledByGroupPolicy" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent" "DisableTailoredExperiencesWithDiagnosticData" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent" "DisableWindowsConsumerFeatures" 1
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent" "DisableSoftLanding" 1

Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" "EnableActivityFeed" 0
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" "PublishUserActivities" 0
Set-Dword "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" "UploadUserActivities" 0

Write-Host "`n=== Disabling telemetry WMI autologgers ==="

Set-Dword "HKLM:\SYSTEM\CurrentControlSet\Control\WMI\AutoLogger\AutoLogger-Diagtrack-Listener" "Start" 0
Set-Dword "HKLM:\SYSTEM\CurrentControlSet\Control\WMI\AutoLogger\SQMLogger" "Start" 0

Write-Host "`n=== Applying current-user privacy settings ==="

Set-Dword "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\AdvertisingInfo" "Enabled" 0
Set-Dword "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Privacy" "TailoredExperiencesWithDiagnosticDataEnabled" 0
Set-Dword "HKCU:\SOFTWARE\Microsoft\Siuf\Rules" "NumberOfSIUFInPeriod" 0
Set-Dword "HKCU:\SOFTWARE\Microsoft\Siuf\Rules" "PeriodInNanoSeconds" 0

Write-Host "`n=== Applying loaded-user hive privacy settings ==="

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

if (-not $SkipServices) {
    Write-Host "`n=== Disabling telemetry-related services ==="

    $Services = @(
        "DiagTrack",                              # Connected User Experiences and Telemetry
        "dmwappushservice",                      # WAP Push Message Routing Service
        "diagnosticshub.standardcollector.service",
        "WerSvc"                                 # Windows Error Reporting Service
    )

    foreach ($Service in $Services) {
        Disable-ServiceSafe -Name $Service
    }
}

if (-not $SkipScheduledTasks) {
    Write-Host "`n=== Disabling telemetry-related scheduled tasks ==="

    $Tasks = @(
        @{ Path = "\Microsoft\Windows\Application Experience\"; Name = "Microsoft Compatibility Appraiser" },
        @{ Path = "\Microsoft\Windows\Application Experience\"; Name = "ProgramDataUpdater" },
        @{ Path = "\Microsoft\Windows\Application Experience\"; Name = "StartupAppTask" },

        @{ Path = "\Microsoft\Windows\Autochk\"; Name = "Proxy" },

        @{ Path = "\Microsoft\Windows\Customer Experience Improvement Program\"; Name = "Consolidator" },
        @{ Path = "\Microsoft\Windows\Customer Experience Improvement Program\"; Name = "KernelCeipTask" },
        @{ Path = "\Microsoft\Windows\Customer Experience Improvement Program\"; Name = "UsbCeip" },

        @{ Path = "\Microsoft\Windows\DiskDiagnostic\"; Name = "Microsoft-Windows-DiskDiagnosticDataCollector" },

        @{ Path = "\Microsoft\Windows\Feedback\Siuf\"; Name = "DmClient" },
        @{ Path = "\Microsoft\Windows\Feedback\Siuf\"; Name = "DmClientOnScenarioDownload" },

        @{ Path = "\Microsoft\Windows\PI\"; Name = "Sqm-Tasks" },

        @{ Path = "\Microsoft\Windows\Power Efficiency Diagnostics\"; Name = "AnalyzeSystem" },

        @{ Path = "\Microsoft\Windows\Windows Error Reporting\"; Name = "QueueReporting" }
    )

    foreach ($Task in $Tasks) {
        Disable-TaskSafe -TaskPath $Task.Path -TaskName $Task.Name
    }
}

Write-Host "`n=== Refreshing local policy ==="

try {
    gpupdate.exe /target:computer /force | Out-Host
}
catch {
    Write-Warning "gpupdate failed: $($_.Exception.Message)"
}

Write-Host "`n=== Result summary ==="

Write-Host "`nDiagnostic data policy:"
Get-ItemProperty -Path $DataCollection -ErrorAction SilentlyContinue |
    Select-Object AllowTelemetry, MaxTelemetryAllowed, LimitDiagnosticLogCollection, LimitDumpCollection, ConfigureTelemetryOptInSettingsUx, DisableDiagnosticDataViewer |
    Format-List

Write-Host "`nTelemetry services:"
"DiagTrack", "dmwappushservice", "diagnosticshub.standardcollector.service", "WerSvc" |
    ForEach-Object {
        Get-Service -Name $_ -ErrorAction SilentlyContinue
    } |
    Select-Object Name, Status, StartType |
    Format-Table -AutoSize

Write-Host "`nDone. Reboot the device to fully apply service, autologger, and policy changes."
Write-Host "Log written to: $LogFile"

Stop-Transcript | Out-Null