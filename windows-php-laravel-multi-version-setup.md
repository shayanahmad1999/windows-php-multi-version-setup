# Windows PHP Version Switcher + Laravel Setup Guide

This guide documents a clean setup for a new Windows PC so you can
switch between PHP 8.2, 8.3, 8.4, and 8.5 from PowerShell and use
different Laravel projects with the appropriate PHP version.

> **Important:** PHP and Laravel versions change over time. When setting
> up a future PC, download the PHP versions required by your projects
> and confirm each Laravel project's PHP requirements in its
> `composer.json`.

------------------------------------------------------------------------

## 1. Recommended Folder Structure

Use a dedicated folder for PHP versions:

``` text
C:\src\
├── php-8.3\
│   ├── php.exe
│   ├── php.ini
│   └── ext\
├── php-8.4\
│   ├── php.exe
│   ├── php.ini
│   └── ext\
├── php-8.5\
│   ├── php.exe
│   ├── php.ini
│   └── ext\
└── php-switch.ps1
```

In this setup, PHP 8.2 can optionally come from XAMPP:

``` text
C:\xampp\php
```

If you install standalone PHP 8.2 instead, use something like:

``` text
C:\src\php-8.2
```

and update the switch script accordingly.

------------------------------------------------------------------------

## 2. Install PHP Versions

Download the Windows PHP builds required by your projects and extract
each version into its own folder, for example:

``` text
C:\src\php-8.3
C:\src\php-8.4
C:\src\php-8.5
```

Verify that each folder contains:

``` text
php.exe
php.ini-development
php.ini-production
ext\
```

Create a `php.ini` for each PHP installation if one does not already
exist:

``` powershell
Copy-Item C:\src\php-8.5\php.ini-development C:\src\php-8.5\php.ini
```

Repeat for other versions as necessary.

------------------------------------------------------------------------

## 3. Configure php.ini for Each PHP Version

Open the configuration for the version you are configuring:

``` powershell
notepad C:\src\php-8.5\php.ini
```

Find `extension_dir` and set:

``` ini
extension_dir = "ext"
```

Using `ext` instead of a hard-coded path allows every PHP installation
to use its own extension directory.

Enable the extensions required by Laravel and your application. Common
examples include:

``` ini
extension=curl
extension=fileinfo
extension=gd
extension=intl
extension=mbstring
extension=openssl
extension=pdo_mysql
extension=pdo_pgsql
extension=pdo_sqlite
extension=pgsql
extension=sodium
extension=xsl
extension=zip
```

Only enable extensions that exist in that PHP distribution and that your
project needs.

For example, confirm mbstring exists:

``` powershell
Test-Path C:\src\php-8.5\ext\php_mbstring.dll
```

Then verify it is loaded:

``` powershell
php -m | findstr mbstring
```

or:

``` powershell
php -r "var_dump(extension_loaded('mbstring'));"
```

Expected:

``` text
bool(true)
```

------------------------------------------------------------------------

## 4. Avoid XAMPP php.ini Overriding Standalone PHP

After switching PHP, check:

``` powershell
php --ini
```

For PHP 8.5, the desired result is:

``` text
Loaded Configuration File: C:\src\php-8.5\php.ini
```

If it unexpectedly loads:

``` text
C:\xampp\php\php.ini
```

check the `PHPRC` environment variable:

``` powershell
$env:PHPRC

[Environment]::GetEnvironmentVariable("PHPRC", "User")
[Environment]::GetEnvironmentVariable("PHPRC", "Machine")
```

Also check:

``` powershell
$env:PHP_INI_SCAN_DIR

[Environment]::GetEnvironmentVariable("PHP_INI_SCAN_DIR", "User")
[Environment]::GetEnvironmentVariable("PHP_INI_SCAN_DIR", "Machine")
```

If `PHPRC` is forcing XAMPP, remove it from the current session:

``` powershell
Remove-Item Env:PHPRC -ErrorAction SilentlyContinue
```

Remove the User variable permanently:

``` powershell
[Environment]::SetEnvironmentVariable("PHPRC", $null, "User")
```

If it exists at Machine level, run PowerShell as Administrator:

``` powershell
[Environment]::SetEnvironmentVariable("PHPRC", $null, "Machine")
```

If `PHP_INI_SCAN_DIR` is incorrectly forcing another PHP installation,
remove it in the same way when appropriate:

``` powershell
Remove-Item Env:PHP_INI_SCAN_DIR -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable("PHP_INI_SCAN_DIR", $null, "User")
```

Close and reopen PowerShell after changing permanent environment
variables.

------------------------------------------------------------------------

## 5. Remove Fixed PHP Versions from the Machine PATH

The PHP switcher should control which PHP version appears first.

Check both PATH values:

``` powershell
Write-Host "USER PATH:"
[Environment]::GetEnvironmentVariable("Path", "User")

Write-Host "`nMACHINE PATH:"
[Environment]::GetEnvironmentVariable("Path", "Machine")
```

Avoid permanently keeping multiple managed PHP paths in the Machine
PATH, such as:

``` text
C:\xampp\php
C:\src\php-8.3
C:\src\php-8.4
C:\src\php-8.5
```

If necessary, open PowerShell **as Administrator** and remove managed
PHP paths from the Machine PATH:

``` powershell
$phpPaths = @(
    "C:\xampp\php",
    "C:\src\php-8.3",
    "C:\src\php-8.4",
    "C:\src\php-8.5"
)

$machinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")

$cleaned = $machinePath -split ';' |
    Where-Object {
        $entry = $_.Trim().TrimEnd('\')
        $entry -and -not ($phpPaths | Where-Object {
            $_.TrimEnd('\') -eq $entry
        })
    }

[Environment]::SetEnvironmentVariable(
    "Path",
    ($cleaned -join ';'),
    "Machine"
)

Write-Host "Managed PHP paths removed from Machine PATH." -ForegroundColor Green
```

Close all terminal windows afterward.

------------------------------------------------------------------------

## 6. Create the PHP Switching Script

Create:

``` text
C:\src\php-switch.ps1
```

Put this inside:

``` powershell
param(
    [Parameter(Mandatory = $true)]
    [ValidateSet("8.2", "8.3", "8.4", "8.5")]
    [string]$Version
)

$phpVersions = @{
    "8.2" = "C:\xampp\php"
    "8.3" = "C:\src\php-8.3"
    "8.4" = "C:\src\php-8.4"
    "8.5" = "C:\src\php-8.5"
}

$phpPath = $phpVersions[$Version]
$phpExe  = Join-Path $phpPath "php.exe"

if (-not (Test-Path $phpExe)) {
    Write-Host "PHP $Version not found at $phpPath" -ForegroundColor Red
    exit 1
}

$managedPaths = $phpVersions.Values | ForEach-Object {
    $_.TrimEnd('\')
}

# Update User PATH permanently.
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")

$userEntries = $userPath -split ';' |
    Where-Object {
        if (-not $_) {
            return $false
        }

        $entry = $_.Trim().TrimEnd('\')
        $managedPaths -notcontains $entry
    }

$newUserPath = @($phpPath) + $userEntries

[Environment]::SetEnvironmentVariable(
    "Path",
    ($newUserPath -join ';'),
    "User"
)

# Rebuild current session PATH with selected PHP first.
$machinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")

$machineEntries = $machinePath -split ';' |
    Where-Object {
        if (-not $_) {
            return $false
        }

        $entry = $_.Trim().TrimEnd('\')
        $managedPaths -notcontains $entry
    }

$env:Path = (@($phpPath) + $userEntries + $machineEntries) -join ';'

Write-Host ""
Write-Host "PHP switched successfully!" -ForegroundColor Green
Write-Host "PHP Version: $Version" -ForegroundColor Cyan
Write-Host "PHP Path:    $phpPath" -ForegroundColor Cyan
Write-Host ""

php -v
```

If PHP 8.2 is standalone rather than XAMPP, change:

``` powershell
"8.2" = "C:\xampp\php"
```

to:

``` powershell
"8.2" = "C:\src\php-8.2"
```

------------------------------------------------------------------------

## 7. Create the PowerShell Profile

Check your profile path:

``` powershell
$PROFILE
```

If the directory/profile does not exist:

``` powershell
New-Item -ItemType Directory -Path (Split-Path $PROFILE) -Force
New-Item -ItemType File -Path $PROFILE -Force
```

Open it:

``` powershell
notepad $PROFILE
```

Add:

``` powershell
function php-switch {
    param(
        [Parameter(Mandatory = $true)]
        [ValidateSet("8.2", "8.3", "8.4", "8.5")]
        [string]$Version
    )

    & "C:\src\php-switch.ps1" $Version

    # Refresh PATH from permanent User + Machine environment.
    $env:Path = [Environment]::GetEnvironmentVariable("Path", "User") + ";" +
                [Environment]::GetEnvironmentVariable("Path", "Machine")
}
```

Save it and reload:

``` powershell
. $PROFILE
```

If PowerShell execution policy prevents your local script/profile from
running, inspect your policy first:

``` powershell
Get-ExecutionPolicy -List
```

Only change execution policy if required and according to the security
policy of the PC/organization.

------------------------------------------------------------------------

## 8. Switch PHP Versions

Switch to PHP 8.5:

``` powershell
php-switch 8.5
```

Switch to PHP 8.4:

``` powershell
php-switch 8.4
```

Switch to PHP 8.3:

``` powershell
php-switch 8.3
```

Switch to PHP 8.2:

``` powershell
php-switch 8.2
```

Verify:

``` powershell
php -v
```

Check which executable Windows resolves:

``` powershell
where.exe php
```

The selected PHP should be the first result.

For PHP 8.5:

``` text
C:\src\php-8.5\php.exe
```

Also verify the configuration:

``` powershell
php --ini
```

For PHP 8.5:

``` text
Loaded Configuration File: C:\src\php-8.5\php.ini
```

------------------------------------------------------------------------

## 9. Verify Switching Persists After Restarting PowerShell

Run:

``` powershell
php-switch 8.5
php -v
```

Close PowerShell completely.

Open a new PowerShell window and run:

``` powershell
php -v
where.exe php
php --ini
```

The selected PHP version should still be active.

If it goes back to another version, inspect:

``` powershell
[Environment]::GetEnvironmentVariable("Path", "User")
[Environment]::GetEnvironmentVariable("Path", "Machine")
```

A different PHP installation is probably still taking precedence.

------------------------------------------------------------------------

## 10. Install Composer

Install Composer for Windows.

After installation verify:

``` powershell
composer --version
```

Also verify the PHP version currently used by your shell:

``` powershell
php -v
```

When working with multiple PHP versions, **switch PHP before running
Composer commands for a project**.

Example:

``` powershell
php-switch 8.5
php -v
composer --version
```

------------------------------------------------------------------------

## 11. Install the Laravel Installer Globally

Switch to the PHP version you want to use for installing/running the
current Laravel installer:

``` powershell
php-switch 8.5
```

Then:

``` powershell
composer global require laravel/installer
```

Make sure Composer's global `vendor\bin` directory is available in PATH
if the `laravel` command is not found.

Verify:

``` powershell
laravel --version
```

Remember:

``` text
laravel --version
```

shows the **Laravel Installer** version, not the framework version of a
project.

------------------------------------------------------------------------

## 12. Create a New Laravel Project

For example:

``` powershell
php-switch 8.5
laravel new saraf
```

Then:

``` powershell
cd saraf
php artisan --version
```

This displays the actual Laravel Framework version installed in the
project.

Useful checks:

``` powershell
php -v
php artisan --version
laravel --version
composer --version
```

Their meanings are:

``` text
php -v                 -> active PHP CLI version
php artisan --version  -> Laravel Framework version for current project
laravel --version      -> globally installed Laravel Installer version
composer --version     -> Composer version
```

------------------------------------------------------------------------

## 13. Working with an Existing Laravel 11 Project

A global Laravel installer does **not** determine the framework version
of an existing Laravel project.

For an existing project, switch to a PHP version supported by that
project's dependencies:

``` powershell
php-switch 8.2
cd C:\path\to\laravel-11-project
php -v
php artisan --version
```

Then, when needed:

``` powershell
composer install
```

The project's Laravel/dependency versions are governed primarily by:

``` text
composer.json
composer.lock
vendor\
```

Always inspect the project's `composer.json` PHP requirement before
choosing a PHP version:

``` powershell
Get-Content composer.json
```

------------------------------------------------------------------------

## 14. Common Error: Laravel Says mbstring Is Missing

Example:

``` text
The following PHP extensions are required but are not installed: mbstring
```

First verify the active PHP:

``` powershell
php -v
php --ini
```

Open its `php.ini`, for example:

``` powershell
notepad C:\src\php-8.5\php.ini
```

Make sure:

``` ini
extension_dir = "ext"
```

and enable:

``` ini
extension=mbstring
```

Verify:

``` powershell
php -m | findstr mbstring
```

Then retry:

``` powershell
laravel new saraf
```

------------------------------------------------------------------------

## 15. Common Error: Unable to Load Dynamic Library

Example:

``` text
Unable to load dynamic library 'curl'
tried: C:\php\ext\php_curl.dll
```

If your active PHP is:

``` text
C:\src\php-8.5\php.exe
```

but extensions are being loaded from:

``` text
C:\php\ext
```

then `extension_dir` is wrong.

Check:

``` powershell
php --ini
```

Open the loaded `php.ini` and change:

``` ini
extension_dir = "C:\php\ext"
```

to:

``` ini
extension_dir = "ext"
```

Verify:

``` powershell
php -m
```

------------------------------------------------------------------------

## 16. Troubleshooting Commands

When PHP switching behaves unexpectedly, run:

``` powershell
php -v
where.exe php
php --ini
$env:PHPRC
$env:PHP_INI_SCAN_DIR
```

Inspect permanent environment values:

``` powershell
[Environment]::GetEnvironmentVariable("PHPRC", "User")
[Environment]::GetEnvironmentVariable("PHPRC", "Machine")

[Environment]::GetEnvironmentVariable("PHP_INI_SCAN_DIR", "User")
[Environment]::GetEnvironmentVariable("PHP_INI_SCAN_DIR", "Machine")

[Environment]::GetEnvironmentVariable("Path", "User")
[Environment]::GetEnvironmentVariable("Path", "Machine")
```

Check extensions:

``` powershell
php -m
```

Check a particular extension:

``` powershell
php -r "var_dump(extension_loaded('mbstring'));"
```

Check PHP extension directory:

``` powershell
php -i | Select-String "extension_dir"
```

------------------------------------------------------------------------

## 17. Recommended Daily Workflow

### New/current Laravel project

``` powershell
php-switch 8.5
cd C:\path\to\project
php -v
php artisan --version
composer install
php artisan serve
```

### Older project requiring PHP 8.2

``` powershell
php-switch 8.2
cd C:\path\to\older-project
php -v
php artisan --version
composer install
php artisan serve
```

Switch PHP **before** Composer or Artisan commands.

------------------------------------------------------------------------

## 18. New-PC Checklist

When moving to a new Windows PC:

-   Install Git.
-   Install Composer.
-   Install Node.js/npm if required by your Laravel frontend.
-   Install XAMPP only if you actually need it.
-   Download/extract required PHP versions.
-   Keep each PHP version in its own folder.
-   Create a separate `php.ini` for every PHP version.
-   Set `extension_dir = "ext"`.
-   Enable required PHP extensions.
-   Create `C:\src\php-switch.ps1`.
-   Create the `php-switch` function in `$PROFILE`.
-   Remove conflicting PHP paths from Machine PATH.
-   Remove unwanted `PHPRC`/`PHP_INI_SCAN_DIR` overrides.
-   Switch to the required PHP version.
-   Verify `php -v`.
-   Verify `where.exe php`.
-   Verify `php --ini`.
-   Install the Laravel installer globally if needed.
-   Verify `laravel --version`.
-   Enter each project and verify `php artisan --version`.
-   Run `composer install` inside existing projects.

------------------------------------------------------------------------

## 19. Final Verification

For PHP 8.5:

``` powershell
php-switch 8.5

php -v
where.exe php
php --ini
php -m
composer --version
laravel --version
```

Inside a Laravel project:

``` powershell
php artisan --version
```

Then restart PowerShell and verify again:

``` powershell
php -v
where.exe php
php --ini
```

If these still point to the selected version, the PHP switcher is
configured correctly.

------------------------------------------------------------------------

## Quick Reference

``` powershell
# Switch PHP
php-switch 8.2
php-switch 8.3
php-switch 8.4
php-switch 8.5

# Active PHP
php -v

# PHP executable resolution
where.exe php

# Loaded php.ini
php --ini

# PHP extensions
php -m

# Composer
composer --version

# Global Laravel Installer
laravel --version

# Laravel Framework version (run inside project)
php artisan --version
```
