# Belajar Struktur Odoo

Repositori ini menjalankan **Odoo 17** dan **PostgreSQL 15** menggunakan Docker Compose. Odoo adalah aplikasi bisnis berbasis modul: fitur seperti penjualan, inventori, dan akuntansi disediakan melalui modul atau *addons*.

## 1. Struktur repo saat ini

```text
odoo-belajar/
├── docker-compose.yml  # Definisi layanan, jaringan, dan penyimpanan
└── README.md           # Dokumentasi ini
```

Kode sumber Odoo berada di image Docker `odoo:17`, bukan di repo ini. Folder modul kustom dan konfigurasi Odoo terpisah belum disediakan.

## 2. Struktur layanan

```text
Browser → localhost:8069 → Odoo (odoo-app)
                              │
                              ├── PostgreSQL (odoo-db:5432)
                              └── File aplikasi (/var/lib/odoo)
```

| Layanan Compose | Image | Nama container | Fungsi |
| --- | --- | --- | --- |
| `odoo` | `odoo:17` | `odoo-app` | Menjalankan aplikasi web dan logika bisnis Odoo |
| `db` | `postgres:15-alpine` | `odoo-db` | Menyimpan data terstruktur Odoo |

Kedua layanan berkomunikasi melalui jaringan bawaan Compose. Variabel `HOST: db` membuat Odoo menghubungi layanan PostgreSQL dengan nama `db`. Variabel `USER` dan `PASSWORD` pada layanan Odoo disesuaikan dengan kredensial PostgreSQL pada Compose.

Port `8069:8069` meneruskan port 8069 komputer ke port 8069 container Odoo. `expose: 5432` pada layanan database tidak menerbitkan port database ke komputer; komunikasi antar layanan di jaringan Compose tetap dapat berlangsung tanpa `expose`.

> `depends_on: db` mengatur urutan startup, tetapi tidak menjamin PostgreSQL sudah siap menerima koneksi. Komentar terakhir pada layanan Odoo belum diikuti konfigurasi pemeriksaan kesiapan database.

## 3. Penyimpanan data

| Volume Docker | Lokasi di container | Isi |
| --- | --- | --- |
| `db-data` | `/var/lib/postgresql/data` | Data PostgreSQL, termasuk database Odoo yang dibuat melalui aplikasi |
| `odoo-data` | `/var/lib/odoo` | Data file Odoo, termasuk *filestore* untuk lampiran yang disimpan sebagai file |

Database menyimpan record bisnis, pengguna, konfigurasi, dan informasi modul. Filestore menyimpan berkas yang terkait dengan record database. Untuk pencadangan lengkap, database dan filestore terkait perlu dicadangkan bersama.

`POSTGRES_DB: postgres` membuat database awal PostgreSQL bernama `postgres`. Nilai ini bukan deklarasi bahwa database aplikasi Odoo harus bernama `postgres`; database aplikasi dapat dibuat melalui halaman pengelolaan database Odoo.

## 4. Struktur internal Odoo

Odoo menggunakan pola **Model–View–Controller (MVC)**. Dalam pengembangan sehari-hari, bagian yang paling sering digunakan adalah:

| Bagian | Bentuk umum | Tanggung jawab |
| --- | --- | --- |
| Model | Python di `models/` | Mendefinisikan data, relasi, aturan, dan logika bisnis melalui ORM |
| View | XML di `views/` | Mendefinisikan tampilan form, daftar, pencarian, menu, dan action |
| Controller | Python di `controllers/` | Menangani rute HTTP untuk kebutuhan web atau integrasi |
| Security | CSV/XML di `security/` | Mengatur hak akses model dan pembatasan record |
| Data | XML/CSV di `data/` | Menyediakan data awal atau konfigurasi modul |
| Assets | Berkas di `static/` | Menyediakan JavaScript, CSS/SCSS, gambar, dan template frontend |

Alur penggunaan sederhana:

1. Pengguna membuka menu atau form melalui browser.
2. Web client meminta data atau menjalankan aksi ke server Odoo.
3. Model memproses permintaan melalui ORM (*Object-Relational Mapping*), dengan pemeriksaan hak akses.
4. ORM membaca atau menulis data di PostgreSQL.
5. Hasil dikirim kembali untuk ditampilkan di browser.

Controller kustom tidak wajib untuk setiap modul. Banyak fitur internal cukup dibuat dengan model, view, action, menu, dan aturan akses.

## 5. Struktur modul kustom

Berikut **contoh struktur untuk pengembangan berikutnya**, belum ada di repo ini:

```text
addons/
└── belajar_odoo/
    ├── __init__.py
    ├── __manifest__.py
    ├── models/
    │   ├── __init__.py
    │   └── buku.py
    ├── views/
    │   └── buku_views.xml
    ├── security/
    │   └── ir.model.access.csv
    ├── data/
    │   └── buku_data.xml
    ├── controllers/
    │   ├── __init__.py
    │   └── main.py
    └── static/
        └── description/
            └── icon.png
```

- `__manifest__.py`: metadata modul, dependensi (`depends`), dan daftar berkas data yang dimuat saat instalasi atau pembaruan.
- `__init__.py`: mengimpor paket Python modul. Contohnya, file utama mengimpor `models`, lalu `models/__init__.py` mengimpor `buku`.
- `models/buku.py`: definisi model, misalnya `belajar.buku`, beserta field dan method-nya.
- `views/buku_views.xml`: tampilan, action pembuka tampilan, dan menu untuk mengakses fitur.
- `security/ir.model.access.csv`: izin membaca, membuat, mengubah, dan menghapus data pada model. Pembatasan record lebih lanjut dapat dibuat melalui *record rules*.
- `data/buku_data.xml`: data awal yang dimuat jika dicantumkan dalam manifest.
- `controllers/` dan `static/`: ditambahkan sesuai kebutuhan fitur.

Berkas XML/CSV tidak otomatis dimuat hanya karena ada dalam folder. Cantumkan berkas yang diperlukan di manifest dan perhatikan urutannya jika suatu berkas merujuk record dari berkas lain.

### Menghubungkan folder addons ke container

Jika folder `addons/` sudah dibuat, tambahkan bind mount berikut pada `services.odoo.volumes`, sambil mempertahankan volume data yang ada:

```yaml
services:
  odoo:
    volumes:
      - odoo-data:/var/lib/odoo
      - ./addons:/mnt/extra-addons
```

Image resmi Odoo menyediakan `/mnt/extra-addons` sebagai lokasi addons tambahan dalam konfigurasi bawaannya. Setelah mengubah Compose, buat ulang layanan dengan `docker compose up -d`, aktifkan mode pengembang di Odoo, lalu perbarui daftar aplikasi. Modul baru dapat diinstal dari menu Apps; hapus filter Apps bila modul tidak ditandai sebagai aplikasi.

Perubahan Python biasanya memerlukan restart layanan. Perubahan definisi field, view, security, atau data modul juga dapat memerlukan upgrade modul agar diterapkan ke database.

## 6. Menjalankan proyek

Prasyarat: Docker Engine atau Docker Desktop dengan Docker Compose tersedia dan berjalan.

Jalankan dari folder repo:

```bash
docker compose up -d
```

Buka <http://localhost:8069>. Pada instalasi baru, ikuti halaman pembuatan database Odoo.

Perintah yang sering digunakan:

```bash
# Melihat status layanan
docker compose ps

# Mengikuti log Odoo
docker compose logs -f odoo

# Mengikuti log PostgreSQL
docker compose logs -f db

# Restart aplikasi setelah perubahan kode Python
docker compose restart odoo

# Menghentikan dan menghapus container, dengan volume data tetap tersimpan
docker compose down
```

Hindari menambahkan opsi `-v` pada `docker compose down` jika data masih diperlukan: opsi tersebut menghapus named volume proyek, termasuk database dan filestore.

## 7. Catatan lingkungan belajar

Compose saat ini menggunakan kredensial database yang ditulis langsung di file dan belum menyediakan healthcheck atau konfigurasi deployment produksi. Gunakan konfigurasi ini untuk belajar lokal; sebelum deployment produksi, siapkan pengelolaan rahasia, pembatasan akses, HTTPS, dan pencadangan.
