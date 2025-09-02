# AI contributor guide for this repo

This is a Laravel 10 HRIS app (attendance, leave, payroll, recruitment, notifications) with Blade views, Vite, Yajra DataTables, DOMPDF, and Maatwebsite Excel. Use these conventions to be productive fast.

## Architecture and data model

-   Core models: `User` (PK `id_user`), `DataKaryawan` (PK `id_data_karyawan`, FK `user_id`), `Absensi`, `Cuti`, `Gaji`, `KomponenGaji`, `Rekrutmen`, `Notifikasi`.
    -   Models set custom primary keys and explicit relations. Example: `User::dataKaryawan()` is `hasOne(DataKaryawan, 'user_id')`; `DataKaryawan` has `belongsTo(User, 'user_id')` and `hasMany` to dependent tables.
-   Migrations define FK columns explicitly (e.g., `data_karyawan.user_id -> users.id_user`). Respect these names when joining and when adding relations/queries.
-   Dashboard aggregates per-user stats by resolving the logged-in user to `DataKaryawan` via `user_id` (see `AllController@dashboard`).

## Routing, auth, and roles

-   Routes are in `routes/web.php`, grouped by middleware:
    -   `auth` + `role:Administrator` for admin resources: `datakaryawan`, `rekrutmen`, `daftarabsensi`, `persetujuancuti`, `penggajian`.
    -   Employee routes: `pengajuancuti`, `riwayatgaji`, `riwayatabsensi`.
-   Role checking uses alias `role` -> `App\\Http\\Middleware\\CheckRole`. Use like: `Route::middleware(['role:Administrator'])`.
-   Login accepts username or email (see `Auth\\LoginController@authenticate`). For the public attendance terminal, use the master gate:
    -   `CheckMasterPassword` middleware alias `master` + `.env` key `MASTER_PASSWORD` to access `/absensi`.

## DataTables, exports, and PDFs

-   Server-side tables use Yajra DataTables. Pattern:
    -   Build an Eloquent/query with joins when needed, then `datatables()->of($query)->addIndexColumn()->addColumn('...')->toJson()`.
    -   Example join: `Absensi::with('dataKaryawan')->select('absensi.*','data_karyawan.nama as nama_karyawan')->join('data_karyawan','absensi.data_karyawan_id','=','data_karyawan.id_data_karyawan')`.
-   Exports: `Maatwebsite\\Excel` (e.g., `Excel::download(new DataKaryawanExport, 'Data Karyawan.xlsx')`).
-   PDFs: `barryvdh/laravel-dompdf` with Blade views (e.g., `PDF::loadView('admin.daftarabsensi.export_pdf', $data)->download(...)`).
-   Alerts: `realrashid/sweet-alert`. Calls like `Alert::success(...)` and `confirmDelete()` are used across controllers/views.

## Frontend build (Vite)

-   Vite entrypoints: see `vite.config.js` and `resources/views/layouts/app.blade.php` (`@vite('resources/js/jquery.js')`, `app.js`, `sidebar.js`, and CSS/SCSS). jQuery, Bootstrap 5, and DataTables BS5 are used.
-   Place assets under `resources/assets/**` and reference via `import.meta.glob(['../assets/**'])` if needed.

## Developer workflows

-   PHP deps: Composer; JS deps: npm.
-   Typical flow:
    -   Install deps and build: `composer install`, `npm ci`, `npm run dev` (or `npm run build`).
    -   Database: uses Laravel migrations; a `database/database.sqlite` exists. Configure `.env` accordingly, then `php artisan migrate --seed`.
    -   Run app: `php artisan serve`.
-   Seeds/factories:
    -   `UserSeeder` inserts two users with roles and links `data_karyawan` rows to them. Passwords: admin `admin@igi`, employee `employee@igi` (bcrypt in seeder).
    -   `DataKaryawanFactory` creates a `User` first, then `DataKaryawan`. Be mindful of potential duplication if running factories and the seeder together.

## Conventions and gotchas

-   Custom PKs mean you must not assume `id`—use the defined PKs/FKs in queries, relations, and route-model binding.
-   Timezone/locale: Carbon often set to `Asia/Jakarta` and Indonesian locale in attendance and exports; keep consistent when adding date logic.
-   Validation messages often Indonesian; follow existing message style and input naming patterns (e.g., `*Edit` fields for update forms).
-   When adding new DataTables endpoints, return proper empty JSON shape when data is 0 (see `AdminControllerThree@getAbsensiHariIni`).

Reference hotspots:

-   Routes: `routes/web.php`.
-   Middleware aliases: `app/Http/Kernel.php`; custom: `CheckRole`, `CheckMasterPassword`, `NoCache`.
-   Controllers: `app/Http/Controllers/*` (Admin/Employee/All/Auth).
-   Models: `app/Models/*`.
-   Configs: `config/datatables.php`, `config/dompdf.php`, `config/excel.php`.

If anything here feels off or incomplete (e.g., additional roles, missing build steps), tell me what to clarify and I’ll refine this doc.
