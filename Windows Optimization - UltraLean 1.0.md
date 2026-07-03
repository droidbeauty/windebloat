$IsAdmin = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()
).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)

if (-not $IsAdmin) {
    Write-Host "ERROR: Run as Administrator." -ForegroundColor Red
    exit 1
}

Write-Host "=== Windows 10 LTSC IoT – Ultra Lean Optimization (Emoji-Safe) ===" -ForegroundColor Cyan

# ------------------------------------------------
# SERVICE HELPERS
# ------------------------------------------------

function Set-ServiceSafe {
    param(
        [string]$Name,
        [string]$Startup
    )

    Get-Service -Name $Name -ErrorAction SilentlyContinue | ForEach-Object {
        try { Set-Service -Name $_.Name -StartupType $Startup -ErrorAction SilentlyContinue } catch {}
    }
}

function Stop-Disable {
    param(
        [string]$Name
    )

    Get-Service -Name $Name -ErrorAction SilentlyContinue | ForEach-Object {
        try { Stop-Service -Name $_.Name -Force -ErrorAction SilentlyContinue } catch {}
        try { Set-Service -Name $_.Name -StartupType Disabled -ErrorAction SilentlyContinue } catch {}
    }
}

function Remove-RegistryValueSafe {
    param(
        [string]$Path,
        [string]$Name
    )

    try {
        if (Test-Path $Path) {
            Remove-ItemProperty -Path $Path -Name $Name -ErrorAction SilentlyContinue
        }
    } catch {}
}

function New-RegistryDwordSafe {
    param(
        [string]$Path,
        [string]$Name,
        [int]$Value
    )

    try {
        if (!(Test-Path $Path)) {
            New-Item -Path $Path -Force | Out-Null
        }
        New-ItemProperty -Path $Path -Name $Name -Value $Value -PropertyType DWord -Force | Out-Null
    } catch {}
}

# ------------------------------------------------
# REPAIR FROM PREVIOUS RUNS
# ------------------------------------------------

Write-Host "Repairing settings that can break emoji/text input..." -ForegroundColor Yellow

# Required for emoji panel / expressive input / touch keyboard pipeline
Set-ServiceSafe TabletInputService Automatic
try { Start-Service -Name TabletInputService -ErrorAction SilentlyContinue } catch {}

# Restore shell/input-adjacent services that are risky to disable
Set-ServiceSafe CDPSvc Manual
Set-ServiceSafe DeviceAssociationService Manual

Get-Service -Name "CDPUserSvc*" -ErrorAction SilentlyContinue | ForEach-Object {
    try { Set-Service -Name $_.Name -StartupType Manual -ErrorAction SilentlyContinue } catch {}
    try { Start-Service -Name $_.Name -ErrorAction SilentlyContinue } catch {}
}

# Re-enable expressive input hotkey if present
$inputBase = "HKLM:\SOFTWARE\Microsoft\Input\Settings"
if (Test-Path $inputBase) {
    Get-ChildItem -Path $inputBase -Recurse -ErrorAction SilentlyContinue | ForEach-Object {
        try {
            $prop = Get-ItemProperty -Path $_.PSPath -Name "EnableExpressiveInputShellHotkey" -ErrorAction SilentlyContinue
            if ($null -ne $prop) {
                New-ItemProperty -Path $_.PSPath -Name "EnableExpressiveInputShellHotkey" -Value 1 -PropertyType DWord -Force | Out-Null
            }
        } catch {}
    }
}

# Ensure common path also contains the value
New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Microsoft\Input\Settings" `
    -Name "EnableExpressiveInputShellHotkey" `
    -Value 1

# Remove risky global background-app deny policy if it was set previously
Remove-RegistryValueSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppPrivacy" `
    -Name "LetAppsRunInBackground"

# Remove policies that can interfere with Microsoft Store / inbox app provisioning
Remove-RegistryValueSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
    -Name "RemoveWindowsStore"

Remove-RegistryValueSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
    -Name "DisableStoreApps"

# ------------------------------------------------
# CORE SERVICES (KEEP WORKING)
# ------------------------------------------------

Write-Host "Ensuring core services..." -ForegroundColor Yellow

# Networking
Set-ServiceSafe WlanSvc Automatic
Set-ServiceSafe Wcmsvc Automatic
Set-ServiceSafe Dhcp Automatic
Set-ServiceSafe Dnscache Automatic
Set-ServiceSafe NlaSvc Automatic
Set-ServiceSafe iphlpsvc Automatic
Set-ServiceSafe netprofm Manual

# Bluetooth
Set-ServiceSafe BthServ Manual
Set-ServiceSafe BTAGService Manual
Set-ServiceSafe BluetoothUserService Manual

# Updates / servicing
Set-ServiceSafe wuauserv Manual
Set-ServiceSafe BITS Manual
Set-ServiceSafe cryptsvc Automatic
Set-ServiceSafe UsoSvc Automatic
Set-ServiceSafe DoSvc Automatic
Set-ServiceSafe WaaSMedicSvc Manual
Set-ServiceSafe TrustedInstaller Manual

# Search / notifications
Set-ServiceSafe WSearch Automatic
Set-ServiceSafe WpnService Automatic
Set-ServiceSafe WpnUserService Automatic

# Text input / emoji / touch keyboard
Set-ServiceSafe TabletInputService Automatic
try { Start-Service -Name TabletInputService -ErrorAction SilentlyContinue } catch {}

# Shell/input-adjacent services that should not be disabled
Set-ServiceSafe CDPSvc Manual
Set-ServiceSafe DeviceAssociationService Manual
Get-Service -Name "CDPUserSvc*" -ErrorAction SilentlyContinue | ForEach-Object {
    try { Set-Service -Name $_.Name -StartupType Manual -ErrorAction SilentlyContinue } catch {}
}

# Store / AppX / inbox app deployment compatibility
Set-ServiceSafe AppXSvc Manual
Set-ServiceSafe ClipSVC Manual
Set-ServiceSafe InstallService Manual
Set-ServiceSafe LicenseManager Manual
Set-ServiceSafe TokenBroker Manual
Set-ServiceSafe StateRepository Manual
Set-ServiceSafe AppReadiness Manual
Set-ServiceSafe wlidsvc Manual

# ------------------------------------------------
# DISABLE SERVICES
# ------------------------------------------------

Write-Host "Disabling non-essential services..." -ForegroundColor Yellow

$services = @(
    "DiagTrack",
    "dmwappushservice",

    # Removed:
    # "CDPSvc",
    # "CDPUserSvc*",

    "XblAuthManager",
    "XblGameSave",
    "XboxGipSvc",
    "XboxNetApiSvc",
    "BcastDVRUserService*",

    "edgeupdate",
    "edgeupdatem",

    "WMPNetworkSvc",
    "PhoneSvc",
    "WalletService",

    "SysMain",
    "TrkWks",

    "RemoteRegistry",
    "CscService",
    "Fax",
    "RetailDemo",
    "MapsBroker",

    "lfsvc",
    "SensorService",
    "SensorDataService",
    "SensrSvc",

    # DO NOT disable TabletInputService
    # It is needed for emoji / text input pipeline

    "WbioSrvc",
    "WerSvc",

    "HomeGroupListener",
    "HomeGroupProvider",

    "PimIndexMaintenanceSvc*",

    "DPS",
    "WdiServiceHost",
    "WdiSystemHost",
    "wisvc",

    "MessagingService"

    # Removed:
    # "DeviceAssociationService"
)

foreach ($s in $services) {
    Stop-Disable $s
}

# ------------------------------------------------
# TELEMETRY / PRIVACY
# ------------------------------------------------

Write-Host "Applying telemetry/privacy restrictions..." -ForegroundColor Yellow

New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" `
    -Name "AllowTelemetry" `
    -Value 0

New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" `
    -Name "EnableActivityFeed" `
    -Value 0

New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" `
    -Name "PublishUserActivities" `
    -Value 0

New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" `
    -Name "UploadUserActivities" `
    -Value 0

New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\LocationAndSensors" `
    -Name "DisableLocation" `
    -Value 1

$ws = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Windows Search"
if (!(Test-Path $ws)) { New-Item $ws -Force | Out-Null }
try {
    Set-ItemProperty -Path $ws -Name "DisableWebSearch" -Type DWord -Value 1
    Set-ItemProperty -Path $ws -Name "ConnectedSearchUseWeb" -Type DWord -Value 0
} catch {}

# ------------------------------------------------
# BACKGROUND APPS (KEEP SHELL/STORE COMPATIBILITY)
# ------------------------------------------------

Write-Host "Keeping background app policy shell-safe..." -ForegroundColor Yellow

# Per-user background app container exists
$bg = "HKCU:\Software\Microsoft\Windows\CurrentVersion\BackgroundAccessApplications"
if (!(Test-Path $bg)) { New-Item -Path $bg -Force | Out-Null }

# Do not globally disable background apps
try {
    Set-ItemProperty -Path $bg -Name "GlobalUserDisabled" -Type DWord -Value 0
} catch {}

# Important:
# Do NOT set:
# HKLM:\SOFTWARE\Policies\Microsoft\Windows\AppPrivacy\LetAppsRunInBackground = 0
# because that can break or degrade Windows app/shell features.

# ------------------------------------------------
# SCHEDULED TASKS
# ------------------------------------------------

Write-Host "Disabling selected scheduled tasks..." -ForegroundColor Yellow

$tasks = @(
    "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser",
    "\Microsoft\Windows\Application Experience\ProgramDataUpdater",
    "\Microsoft\Windows\Application Experience\StartupAppTask",

    "\Microsoft\Windows\Customer Experience Improvement Program\Consolidator",
    "\Microsoft\Windows\Customer Experience Improvement Program\KernelCeipTask",
    "\Microsoft\Windows\Customer Experience Improvement Program\UsbCeip",

    "\Microsoft\Windows\DiskDiagnostic\Microsoft-Windows-DiskDiagnosticDataCollector",

    "\Microsoft\Windows\Feedback\Siuf\DmClient",
    "\Microsoft\Windows\Feedback\Siuf\DmClientOnScenarioDownload",

    "\Microsoft\Windows\Maps\MapsUpdateTask",
    "\Microsoft\Windows\Maps\MapsToastTask",

    "\Microsoft\Windows\AppID\SmartScreenSpecific",
    "\Microsoft\Windows\Application Experience\AitAgent",
    "\Microsoft\Windows\Autochk\Proxy",
    "\Microsoft\Windows\DiskFootprint\Diagnostics",
    "\Microsoft\Windows\Maintenance\WinSAT",
    "\Microsoft\Windows\Shell\IndexerAutomaticMaintenance",
    "\Microsoft\Windows\UpdateOrchestrator\Schedule Scan",
    "\Microsoft\Windows\UpdateOrchestrator\USO_UxBroker"
)

foreach ($t in $tasks) {
    try {
        $taskPath = $t.Substring(0, $t.LastIndexOf("\") + 1)
        $taskName = $t.Split("\")[-1]
        $taskObj = Get-ScheduledTask -TaskPath $taskPath -TaskName $taskName -ErrorAction SilentlyContinue
        if ($taskObj) {
            Disable-ScheduledTask -InputObject $taskObj -ErrorAction SilentlyContinue | Out-Null
        }
    } catch {}
}

# ------------------------------------------------
# CAPABILITY SERVICES
# ------------------------------------------------

Write-Host "Disabling selected capability services..." -ForegroundColor Yellow

$capServices = @(
)

foreach ($svc in $capServices) {
    Get-Service -Name $svc -ErrorAction SilentlyContinue | ForEach-Object {
        try { Set-Service -Name $_.Name -StartupType Disabled -ErrorAction SilentlyContinue } catch {}
        try { Stop-Service -Name $_.Name -Force -ErrorAction SilentlyContinue } catch {}
    }
}

# ------------------------------------------------
# EXTRA PRIVACY REGISTRY SETTINGS
# ------------------------------------------------

Write-Host "Applying privacy-related registry settings..." -ForegroundColor Yellow

# 1. Telemetry
New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" `
    -Name "AllowTelemetry" `
    -Value 0

# 2. Advertising ID
New-RegistryDwordSafe `
    -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\AdvertisingInfo" `
    -Name "Enabled" `
    -Value 0

# 3. Activity Feed
New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" `
    -Name "EnableActivityFeed" `
    -Value 0

# 4. Consumer features
New-RegistryDwordSafe `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CloudContent" `
    -Name "DisableWindowsConsumerFeatures" `
    -Value 1

Write-Host "Registry values applied." -ForegroundColor Green

# ------------------------------------------------
# OPTIONAL HEALTH CHECK
# ------------------------------------------------

Write-Host ""
Write-Host "Post-check:" -ForegroundColor Cyan

try {
    $svc = Get-Service -Name TabletInputService -ErrorAction Stop
    Write-Host ("TabletInputService status: " + $svc.Status + " / " + $svc.StartType) -ForegroundColor Gray
} catch {
    Write-Host "TabletInputService not found." -ForegroundColor DarkYellow
}

try {
    $svc = Get-Service -Name AppXSvc -ErrorAction Stop
    Write-Host ("AppXSvc status: " + $svc.Status + " / " + $svc.StartType) -ForegroundColor Gray
} catch {
    Write-Host "AppXSvc not found." -ForegroundColor DarkYellow
}

try {
    $svc = Get-Service -Name ClipSVC -ErrorAction Stop
    Write-Host ("ClipSVC status: " + $svc.Status + " / " + $svc.StartType) -ForegroundColor Gray
} catch {
    Write-Host "ClipSVC not found." -ForegroundColor DarkYellow
}

try {
    $svc = Get-Service -Name InstallService -ErrorAction Stop
    Write-Host ("InstallService status: " + $svc.Status + " / " + $svc.StartType) -ForegroundColor Gray
} catch {
    Write-Host "InstallService not found." -ForegroundColor DarkYellow
}

try {
    $svc = Get-Service -Name CDPSvc -ErrorAction Stop
    Write-Host ("CDPSvc status: " + $svc.Status + " / " + $svc.StartType) -ForegroundColor Gray
} catch {
    Write-Host "CDPSvc not found." -ForegroundColor DarkYellow
}

try {
    $svc = Get-Service -Name DeviceAssociationService -ErrorAction Stop
    Write-Host ("DeviceAssociationService status: " + $svc.Status + " / " + $svc.StartType) -ForegroundColor Gray
} catch {
    Write-Host "DeviceAssociationService not found." -ForegroundColor DarkYellow
}

# Restart text input monitor quietly
try {
    Get-Process ctfmon -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
    Start-Process "$env:WINDIR\System32\ctfmon.exe" -ErrorAction SilentlyContinue
    Write-Host "ctfmon.exe restarted." -ForegroundColor Gray
} catch {
    Write-Host "ctfmon.exe restart skipped." -ForegroundColor DarkYellow
}

# ------------------------------------------------
# FINAL
# ------------------------------------------------

Write-Host ""
Write-Host "=== DONE ===" -ForegroundColor Green
Write-Host "REBOOT REQUIRED." -ForegroundColor Yellow