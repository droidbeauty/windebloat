# Installs Google Chrome via WinGet, then opens Windows Default Apps settings.

param(
    # Optional: import default app associations for deployment/new-user scenarios.
    # This is not a guaranteed silent override for the current signed-in user.
    [switch]$ConfigureNewUserDefaults,

    # Optional: do not open Windows Default Apps settings after install.
    [switch]$NoOpenSettings
)

$ErrorActionPreference = "Stop"

function Test-IsAdmin {
    $identity = [Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = New-Object Security.Principal.WindowsPrincipal($identity)
    return $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

function Get-WinGetPath {
    $cmd = Get-Command winget.exe -ErrorAction SilentlyContinue
    if ($cmd) {
        return $cmd.Source
    }

    $candidate = Join-Path $env:LOCALAPPDATA "Microsoft\WindowsApps\winget.exe"
    if (Test-Path $candidate) {
        return $candidate
    }

    $desktopInstaller = Get-ChildItem "C:\Program Files\WindowsApps" -Filter "winget.exe" -Recurse -ErrorAction SilentlyContinue |
        Where-Object { $_.FullName -like "*Microsoft.DesktopAppInstaller*" } |
        Select-Object -First 1

    if ($desktopInstaller) {
        return $desktopInstaller.FullName
    }

    throw "WinGet was not found. Install or update 'App Installer' from Microsoft Store, then run this script again."
}

if (-not (Test-IsAdmin)) {
    Write-Host "Restarting as Administrator..."
    Start-Process powershell.exe -Verb RunAs -ArgumentList @(
        "-NoProfile",
        "-ExecutionPolicy", "Bypass",
        "-File", "`"$PSCommandPath`"",
        $(if ($ConfigureNewUserDefaults) { "-ConfigureNewUserDefaults" }),
        $(if ($NoOpenSettings) { "-NoOpenSettings" })
    )
    exit
}

$winget = Get-WinGetPath

Write-Host "Installing Google Chrome with WinGet..."

& $winget install `
    --id Google.Chrome `
    --exact `
    --source winget `
    --silent `
    --accept-package-agreements `
    --accept-source-agreements

if ($LASTEXITCODE -ne 0) {
    throw "WinGet failed to install Google Chrome. Exit code: $LASTEXITCODE"
}

Write-Host "Google Chrome installation command completed."

# Optional deployment/new-user default associations.
# For existing signed-in users, Windows may ignore or reset this because default-app choices are protected.
if ($ConfigureNewUserDefaults) {
    $xmlPath = Join-Path $env:TEMP "ChromeDefaultAssociations.xml"

    @'
<?xml version="1.0" encoding="UTF-8"?>
<DefaultAssociations>
  <Association Identifier=".htm"  ProgId="ChromeHTML" ApplicationName="Google Chrome" />
  <Association Identifier=".html" ProgId="ChromeHTML" ApplicationName="Google Chrome" />
  <Association Identifier="http"  ProgId="ChromeHTML" ApplicationName="Google Chrome" />
  <Association Identifier="https" ProgId="ChromeHTML" ApplicationName="Google Chrome" />
</DefaultAssociations>
'@ | Set-Content -Path $xmlPath -Encoding UTF8

    Write-Host "Importing default app associations XML for deployment/new-user scenarios..."
    & dism.exe /Online /Import-DefaultAppAssociations:$xmlPath

    if ($LASTEXITCODE -ne 0) {
        Write-Warning "DISM default association import failed. Exit code: $LASTEXITCODE"
    } else {
        Write-Host "Default association XML imported."
    }
}

if (-not $NoOpenSettings) {
    Write-Host ""
    Write-Host "Windows requires user confirmation to change the current default browser."
    Write-Host "In the Settings window that opens, select Google Chrome and set it as default."
    Start-Process "ms-settings:defaultapps"
}

Write-Host "Done."