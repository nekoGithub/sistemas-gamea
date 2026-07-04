# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Laravel 12 admin application ("GAMEA") for IT asset/inventory management: tracks **sistemas** (software systems) and their **versiones** (releases), the **servidores** they run on, **bases de datos**, **credenciales**, SSL certificates, technologies, and organizational **unidades**/**responsables**. Includes audit logging, uptime/SSL monitoring with Telegram alerts, PDF/Excel reporting, and a chunked large-file upload pipeline for version artifacts (source code, manuals, DB dumps).

Auth/scaffolding is Laravel Jetstream (Livewire-based) + Fortify + Sanctum. Authorization is Spatie `laravel-permission` (roles: `admin`, `tecnico`, `visitante`). The admin UI is built on top of a Bootstrap 5 "Gamea"-style admin theme (jQuery + DataTables + a large library of vendor JS widgets), not a Vue/React SPA — despite Vue components existing under `resources/js/components` for Jetstream's legacy scaffolding.

## Commands

```bash
# Backend dev server + queue worker + logs + Vite, all at once
composer dev

# Just the PHP dev server
php artisan serve

# Frontend build
npm run dev      # Vite dev server
npm run build    # production build

# Tests (Pest, via artisan)
composer test
php artisan test
php artisan test --filter=TestName
php artisan test tests/Feature/SomeTest.php

# Queue worker (required for version uploads, SSL checks, notifications)
php artisan queue:listen --tries=1

# Reverb (WebSocket server for real-time server/system status broadcasts)
php artisan reverb:start
```

There is no PHP linter/formatter config wired into a script beyond `laravel/pint` being a dev dependency (`vendor/bin/pint`).

## Architecture

### Domain model core

`Sistema` (system) belongs to a `Unidad` (org unit) and optionally an `Ssl` cert, and has many `SistemaVersion` records. Each `SistemaVersion` is the real hub of complexity — it belongs-to-many `Tecnologia`, `Servidor`, `BaseDato`, `Credencial`, and `Documento` (via dedicated pivot tables named `sistema_version_*`), tracks which is `es_actual` (current/live release), and stores uploaded artifacts (`codigo_fuente`, `manual_tecnico`, `manual_usuario`, `archivo_bd`, `imagen`) as paths on the `public` disk.

Almost every admin-facing model uses `SoftDeletes`, and every index page follows the same pattern: fetch active records, fetch `onlyTrashed()` records for a separate "papelera" (trash) UI section, and provide `restore($id)` and hard `destroy()` endpoints — see `SistemaController` as the reference implementation.

### Auditing

The `Auditable` trait (`app/Traits/Auditable.php`) is mixed into models that need history tracking (`Sistema`, `SistemaVersion`, etc.). It hooks `created`/`updated`/`deleted`/`restored` model events and writes to the `Auditoria` model automatically, redacting sensitive fields (`password`, `api_token`, etc.) before persisting. Don't manually log CRUD actions on these models — it happens implicitly via Eloquent events. `AuditoriaController` exposes browsing/exporting of this log.

### Authorization

Every admin controller applies `$this->middleware('can:admin.<module>.<action>')` per-method in its constructor (see `SistemaController::__construct`). Permission strings follow the `admin.<module>.<index|store|edit|update|destroy|restore>` convention and are registered/assigned to roles in `database/seeders/RolesSeeder.php` — when adding a new module/action, add the permission there too, not just the route/middleware.

### Controller response convention

`index()` actions render a Blade view with the full dataset (active + trashed) passed via `compact()` — pagination/filtering happens client-side via DataTables (see `resources/js/datatables/datatables-*.js`, one file per module, wired up as individual Vite entry points). `store`/`update`/`destroy`/`restore` actions are AJAX endpoints that return JSON (`{success, ...}`), not redirects — validation errors surface via Laravel's default 422 JSON response, consumed by the corresponding Blade modal + JS.

### Chunked version uploads

Large version artifacts are uploaded in chunks via `SistemaVersionController` endpoints (`iniciar-upload`, `upload-chunk`, `completar-upload`, plus status/cancel), tracked by a `VersionUpload` record. Once all chunks land, `ProcessVersionUpload` (queued job) assembles each file type from `storage/app/chunks/{identifier}`, writes finals to `storage/app/public/versiones/{codigo|manuales|bases_datos}/`, creates/updates the `SistemaVersion` row, syncs its pivot relations, handles additional `Documento` attachments, cleans up temp chunks, and fires a Telegram notification (recorded as a `Notificacion` row) — all with granular progress percentages written back to `VersionUpload`. When touching this flow, preserve the progress-percentage checkpoints; the frontend polls upload status off of them.

### Monitoring & notifications

Console commands `MonitorearServidores` (ping-based server health, long-running loop) and `MonitorearSistemasWeb` (HTTP status of "external" systems) update model state and `broadcast()` Reverb events (`EstadoServidorActualizado`, `EstadoSistemaWebActualizado`) for live dashboard updates, and call `TelegramService` to alert on state transitions to down/inactive (not on every check — only on the transition). `CheckSslExpiration` handles SSL expiry alerts similarly. `TelegramService` includes its own rate-limiting/retry logic — reuse it rather than calling the Telegram API directly.

### PDF/Excel reporting

`app/Services/ReportePDF.php` and `AuditoriaPDF.php` extend TCPDF with shared branding/header helpers (watermark, section banners). `ReporteController` and `SistemaController::reportePdf` build reports by eager-loading nested relations up front (e.g. `unidad.responsables`, `versiones.tecnologias`, `versiones.servidores.sistemaOperativo`) then rendering via closures for repeated layout blocks (`$seccion`, `$fila`). Excel export uses `phpoffice/phpspreadsheet`.

### Frontend build

`vite.config.js` lists **every** JS/CSS entry point explicitly (no glob/manifest magic) — vendor widget CSS/JS, per-page scripts under `resources/js/pages/`, and per-module DataTables configs under `resources/js/datatables/`. **When adding a new admin module with a DataTable or a new demo/vendor page, you must add its entry to the `input` array in `vite.config.js` or it will not be bundled.** Styling is a SCSS build (`resources/scss/app.scss`) plus Tailwind (`resources/css/app.css`) coexisting — Tailwind is present for Jetstream's default components, the admin theme itself is Bootstrap 5.

### Email verification

Standard Laravel email verification is layered with a custom numeric-code step: `EnsureEmailVerifiedCode` middleware force-logs-out any authenticated-but-unverified user and redirects to `EmailCodeVerificationController`'s code entry form (encrypting the email in the redirect). This middleware is chained after `verified` on the `/dashboard` route group — apply the same chain to any other route that should require full verification.

## Environment specifics

- DB is PostgreSQL in dev (`DB_CONNECTION=pgsql`); tests run against in-memory SQLite (`phpunit.xml`) — don't rely on Postgres-only SQL in code paths covered by tests.
- Broadcasting uses Laravel Reverb (`BROADCAST_CONNECTION=reverb`); queue driver is `database` in dev, `sync` in tests.
- Telegram bot token/chat ID and Reverb app credentials live in `.env` (`services.telegram.*` config) — never hardcode them or read them into logs/output.
