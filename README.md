Windows PHP & Laravel Multi-Version Setup

A practical Windows development setup for managing multiple PHP versions
and working with Laravel projects that require different PHP versions.

This repository documents how to configure PHP 8.2, 8.3, 8.4, and
8.5, switch between them from PowerShell, configure PHP extensions
correctly, and use Composer and Laravel without PATH or php.ini
conflicts.

Features

Switch between PHP 8.2, 8.3, 8.4, and 8.5

Persist the selected PHP version after restarting PowerShell

Keep a separate php.ini for each PHP version

Avoid XAMPP and standalone PHP configuration conflicts

Configure required Laravel PHP extensions

Use Composer with the currently selected PHP version

Install and use the Laravel installer globally

Work with old and new Laravel projects on the same PC

Troubleshoot PATH, php.ini, and missing-extension issues

Quick Usage

Switch PHP:

php-switch 8.2
php-switch 8.3
php-switch 8.4
php-switch 8.5

Verify the active version:

php -v

Check which PHP executable is being used:

where.exe php

Check the active PHP configuration:

php --ini

Check the Laravel framework version inside a project:

php artisan --version

Check the global Laravel installer:

laravel --version

Documentation

See the complete setup guide:

windows-php-laravel-multi-version-setup.md

It contains the full process for configuring a new Windows PC, including
PHP installation, PowerShell profile setup, PATH management, Composer,
Laravel, extensions, and troubleshooting.

Example Workflow

For a newer Laravel project:

php-switch 8.5
cd C:\path\to\project
php -v
php artisan --version
composer install
php artisan serve

For an older project requiring PHP 8.2:

php-switch 8.2
cd C:\path\to\older-project
php -v
php artisan --version
composer install
php artisan serve

Important

Always switch to the PHP version required by the project before
running Composer or Artisan commands.

Check the project's composer.json to confirm its PHP requirements when
you are unsure which PHP version to use.

Purpose

The goal of this repository is to provide a reusable reference for
setting up the same PHP/Laravel development environment on a new Windows
PC without repeating the configuration and troubleshooting process from
scratch.
