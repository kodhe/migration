# Kodhe Migration

Package versioning skema database hasil refaktor library *Migrations* (CI Bonfire). Namespace `Kodhe\Framework\Migration`. Menyediakan dua kelas:

- `Migration` — runner modern berbasis tabel `schema_version`, mendukung migrasi **core** dan **per-module** (`app_<module>_`), bisa dijalankan otomatis saat page-load (`autoLatest()`) atau manual per versi.
- `Legacy` — runner klasik bergaya CI3 (`application/db/migrations/` + konfig `$config['migration_*']`) untuk proyek yang sudah punya file migrasi lama.

## Instalasi

```bash
composer require kodhe/migration
```

Persyaratan: PHP >= 8.1, koneksi database aktif (pakai query builder `kodhe/database` di dalam kelas abstrak `Migration` bawaan CI), serta konstanta CI (`BASEPATH`, `APPPATH`).

## Quick Start

```php
<?php

declare(strict_types=1);

use Kodhe\Framework\Migration\Migration;

// Dari config CI: $config['migrations_path'] = APPPATH.'Migration';
$migration = new Migration(['migrations_path' => APPPATH . 'Migration']);

// Jalankan semua migrasi core hingga versi terbaru
$migration->install();

// Cek hasil
echo $migration->getVersion();          // versi aktif sekarang
echo $migration->getVersion('', true);  // versi terbaru yang tersedia
```

## Struktur Direktori

```
src/
├── Migration.php   # Runner utama (tabel schema_version, modul-aware)
└── Legacy.php      # Runner kompatibel CI3 (db/migrations + migration_enabled)
```

## Penggunaan

### 1. Membuat file migrasi

File di `application/Migration/` dengan penomoran berurutan:

```
001_install_initial_schema.php
002_add_user_avatar.php
```

Isinya kelas yang meng-extend `Migration` abstrak CI dan wajib punya `up()`/`down()`:

```php
<?php // 002_add_user_avatar.php

class Migration_add_user_avatar extends CI_Migration
{
    public function up()
    {
        $this->dbforge->add_column('users', [
            'avatar' => ['type' => 'varchar', 'constraint' => 255, 'null' => true],
        ]);
    }

    public function down()
    {
        $this->dbforge->drop_column('users', 'avatar');
    }
}
```

### 2. Migrasi per modul (app migrations)

Nama file diawali `app_<module>_<nomor>_<deskripsi>` → runner memisahkan versi tiap modul:

```php
$migration->install('blog');            // hanya modul blog
$migration->version(3, 'blog');         // ke versi 3 milik modul blog
$versions = $migration->getModuleVersions(); // ['' => 5, 'blog' => 3, ...]
```

### 3. Auto-migrate saat aplikasi load

```php
// application/config/*.php
$config['migrate']['auto_core'] = TRUE;
$config['migrate']['auto_app']  = TRUE;

// Di hook/bootstrap:
$migration->autoLatest();
```

### 4. SQL murni tanpa kelas migrasi

```php
$migration->doSqlMigration("ALTER TABLE users ADD COLUMN token VARCHAR(64) NULL");
```

### 5. Menangani error

```php
if ($migration->version(4) === false) {
    echo $migration->getErrorMessage();  // gabungan pesan
    print_r($migration->getErrors());    // array mentah
}
```

### 6. Mode Legacy (kompatibel CI3 penuh)

```php
use Kodhe\Framework\Migration\Legacy;

// Butuh config CI3 standar:
// $config['migration_enabled'], ['migration_type'], ['migration_path'],
// ['migration_table'], ['migration_auto_latest'], ['migration_version']
$m = new Legacy();
$m->latest();                 // naik ke versi terbaru
$m->current();                // versi aktif
$m->version(2);               // pindah ke versi 2 (naik/turun via up/down)
echo $m->error_string();
```

## Referensi API (`Migration`)

| Method | Keterangan |
|---|---|
| `__construct(array $params)` | Opsional: `migrations_path`; buat tabel `schema_version` bila belum ada |
| `autoLatest()` | Jalankan auto-migrate core/app sesuai config `migrate` |
| `install($type = '')` | Naikkan ke versi terbaru untuk tipe/modul tertentu (`''` = core) |
| `version($version, $type = '')` | Pindah ke versi spesifik (maju/mundur) |
| `doSqlMigration($sql)` | Eksekusi statement SQL langsung & catat versinya |
| `getVersion($type, $getLatest = false)` | Versi aktif / versi terbaru tersedia |
| `getAvailableVersions($type)` | Daftar semua nomor versi di folder |
| `getModuleVersions()` | Peta versi per modul |
| `getErrors()` / `getErrorMessage()` | Ambil error |
| `setVerbose($state)` | Cetak progres ke output |

## Kompatibilitas CodeIgniter 3

Perilaku mengikuti CI_Bonfire Migrations; `Legacy` meniru `CI_Migration` asli. Tabel pelacak bernama `schema_version` (bukan `ci_migrations`) — saat bermigrasi dari CI3, sesuaikan nama tabel lewat properti `$migrationsTable`.

## Catatan

- Bergantung pada lingkungan CI (`get_instance()`, `show_error()`, dbforge) — bukan package standalone penuh.
- Nomor versi harus integer berurutan; file tanpa prefiks angka diabaikan.

## Pengujian

```bash
vendor/bin/phpunit --filter Migration
```
