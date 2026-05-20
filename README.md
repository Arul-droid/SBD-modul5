# 🎭 Praktikum Basis Data — Modul 5
### Sistem EO Pesta Gala | *Frieren: Beyond Journey's End*

> *"Masih ada manuskrip yang belum kamu tunjukkan?"*
> Secretary Maid menyodorkan satu gulungan terakhir. Frieren mengambilnya, membuka halaman pertama, lalu mengangguk. Ia mulai bekerja lagi.

---

## 📋 Daftar Isi

- [Gambaran Umum](#gambaran-umum)
- [Struktur Database](#struktur-database)
- [Soal 1 — INSERT Data](#soal-1--insert-data)
- [Soal 2 — Stored Procedure: Daftarkan Vendor](#soal-2--stored-procedure-daftarkan-vendor)
- [Soal 3 — Function: Kategori Kontrak](#soal-3--function-kategori-kontrak)
- [Soal 4 — Tabel Log & Trigger](#soal-4--tabel-log--trigger)
- [Soal 5 — Uji Stored Procedure](#soal-5--uji-stored-procedure)
- [Cara Menjalankan](#cara-menjalankan)

---

## Gambaran Umum

Modul ini mengimplementasikan sistem manajemen Event Organizer (EO) untuk **Pesta Gala** menggunakan fitur-fitur lanjutan MySQL:

| Fitur | Digunakan Pada |
|-------|---------------|
| `INSERT` / DML | Soal 1 |
| `STORED PROCEDURE` + parameter `OUT` | Soal 2 & 5 |
| `FUNCTION` | Soal 3 |
| `TRIGGER` + tabel log | Soal 4 |

**Database:** `gala_frieren`

---

## Struktur Database

```
gala_frieren
├── Acara          (acara_id, nama_acara, tanggal, lokasi)
├── Vendor         (vendor_id, nama_vendor, kategori, kontak)
├── Tamu           (tamu_id, nama_tamu, acara_id, rsvp_status)
├── Kontrak        (kontrak_id, vendor_id, acara_id, nilai_kontrak, status_kontrak)
└── LogPerubahanKontrak  (id_log, kontrak_id, status_lama, status_baru, waktu_perubahan)
```

**Relasi:**
- `Tamu.acara_id` → `Acara.acara_id`
- `Kontrak.vendor_id` → `Vendor.vendor_id`
- `Kontrak.acara_id` → `Acara.acara_id`

---

## Soal 1 — INSERT Data

**Tujuan:** Memasukkan seluruh data awal ke semua tabel sesuai dataset yang disediakan.

Data yang dimasukkan mencakup:
- **5 Acara** — dari *Gala Malam Pertama* hingga *Upacara Penghargaan*
- **8 Vendor** — berbagai kategori: Katering, Dekorasi, Hiburan, Teknis, Dokumentasi
- **15 Tamu** — tersebar di 5 acara dengan status RSVP masing-masing
- **10 Kontrak** — menghubungkan vendor dengan acara beserta nilai dan statusnya

```sql
-- Jalankan script dataset yang disediakan untuk mengisi semua tabel.
-- Script sudah mencakup INSERT untuk Acara, Vendor, Tamu, dan Kontrak.
USE gala_frieren;

-- Contoh sebagian data yang dimasukkan:
INSERT INTO Acara (nama_acara, tanggal, lokasi) VALUES
('Gala Malam Pertama',    '2024-03-10', 'Aula Kerajaan Sein'),
('Pesta Dansa Musim Semi', '2024-04-20', 'Taman Bunga Äußerst'),
-- ... (lihat dataset lengkap)
;
```

> ✅ Jalankan file `dataset.sql` yang disediakan untuk setup lengkap.

---

## Soal 2 — Stored Procedure: Daftarkan Vendor

**Tujuan:** Membuat Stored Procedure untuk mendaftarkan vendor ke sebuah acara dengan validasi kontrak aktif.

**Logika:**
1. Cek apakah vendor sudah memiliki kontrak `'Aktif'` di acara yang sama.
2. Jika **sudah ada** → keluarkan pesan **GAGAL** beserta nama vendor via parameter `OUT`.
3. Jika **belum ada** → tambahkan data kontrak baru dan keluarkan pesan **SUKSES**.

```sql
DELIMITER //

CREATE PROCEDURE DaftarkanVendor(
    IN  p_vendor_id      INT,
    IN  p_acara_id       INT,
    IN  p_nilai_kontrak  DECIMAL(15,2),
    OUT p_pesan          VARCHAR(255)
)
BEGIN
    DECLARE v_nama_vendor   VARCHAR(100);
    DECLARE v_kontrak_aktif INT DEFAULT 0;

    -- Ambil nama vendor
    SELECT nama_vendor INTO v_nama_vendor
    FROM Vendor
    WHERE vendor_id = p_vendor_id;

    -- Cek kontrak aktif di acara yang sama
    SELECT COUNT(*) INTO v_kontrak_aktif
    FROM Kontrak
    WHERE vendor_id      = p_vendor_id
      AND acara_id       = p_acara_id
      AND status_kontrak = 'Aktif';

    IF v_kontrak_aktif > 0 THEN
        SET p_pesan = CONCAT('GAGAL: Vendor "', v_nama_vendor,
                             '" sudah memiliki kontrak aktif di acara ini.');
    ELSE
        INSERT INTO Kontrak (vendor_id, acara_id, nilai_kontrak, status_kontrak)
        VALUES (p_vendor_id, p_acara_id, p_nilai_kontrak, 'Aktif');

        SET p_pesan = CONCAT('SUKSES: Vendor "', v_nama_vendor,
                             '" berhasil didaftarkan ke acara.');
    END IF;
END //

DELIMITER ;
```

**Parameter:**

| Parameter | Tipe | Arah | Keterangan |
|-----------|------|------|-----------|
| `p_vendor_id` | `INT` | `IN` | ID vendor yang akan didaftarkan |
| `p_acara_id` | `INT` | `IN` | ID acara tujuan |
| `p_nilai_kontrak` | `DECIMAL(15,2)` | `IN` | Nilai kontrak dalam rupiah |
| `p_pesan` | `VARCHAR(255)` | `OUT` | Pesan hasil eksekusi |

---

## Soal 3 — Function: Kategori Kontrak

**Tujuan:** Membuat Function yang menerima nilai kontrak dan mengembalikan kategorinya.

**Aturan kategorisasi:**

| Kondisi | Kategori |
|---------|----------|
| Nilai > Rp 10.000.000 | `'Kontrak Besar'` |
| Rp 5.000.000 ≤ Nilai ≤ Rp 10.000.000 | `'Kontrak Menengah'` |
| Nilai < Rp 5.000.000 | `'Kontrak Kecil'` |

```sql
DELIMITER //

CREATE FUNCTION KategoriKontrak(p_nilai DECIMAL(15,2))
RETURNS VARCHAR(20)
DETERMINISTIC
BEGIN
    DECLARE v_kategori VARCHAR(20);

    IF p_nilai > 10000000 THEN
        SET v_kategori = 'Kontrak Besar';
    ELSEIF p_nilai >= 5000000 THEN
        SET v_kategori = 'Kontrak Menengah';
    ELSE
        SET v_kategori = 'Kontrak Kecil';
    END IF;

    RETURN v_kategori;
END //

DELIMITER ;
```

**Contoh penggunaan:**

```sql
SELECT
    kontrak_id,
    vendor_id,
    nilai_kontrak,
    KategoriKontrak(nilai_kontrak) AS kategori
FROM Kontrak;
```

**Contoh hasil:**

| kontrak_id | vendor_id | nilai_kontrak | kategori |
|-----------|-----------|--------------|----------|
| 1 | 1 | 15,000,000 | Kontrak Besar |
| 2 | 2 | 8,500,000 | Kontrak Menengah |
| 4 | 4 | 4,500,000 | Kontrak Kecil |

---

## Soal 4 — Tabel Log & Trigger

**Tujuan:** Membuat sistem pencatatan otomatis setiap kali `status_kontrak` berubah.

### 4a. Buat Tabel LogPerubahanKontrak

```sql
CREATE TABLE LogPerubahanKontrak (
    id_log          INT PRIMARY KEY AUTO_INCREMENT,
    kontrak_id      INT         NOT NULL,
    status_lama     VARCHAR(20) NOT NULL,
    status_baru     VARCHAR(20) NOT NULL,
    waktu_perubahan DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**Kolom:**

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `id_log` | `INT AUTO_INCREMENT` | Primary key log |
| `kontrak_id` | `INT` | ID kontrak yang berubah |
| `status_lama` | `VARCHAR(20)` | Status sebelum perubahan |
| `status_baru` | `VARCHAR(20)` | Status setelah perubahan |
| `waktu_perubahan` | `DATETIME` | Timestamp perubahan terjadi |

### 4b. Buat Trigger AFTER UPDATE

```sql
DELIMITER //

CREATE TRIGGER trg_log_perubahan_kontrak
AFTER UPDATE ON Kontrak
FOR EACH ROW
BEGIN
    -- Hanya catat jika status_kontrak benar-benar berubah
    IF OLD.status_kontrak <> NEW.status_kontrak THEN
        INSERT INTO LogPerubahanKontrak (kontrak_id, status_lama, status_baru, waktu_perubahan)
        VALUES (OLD.kontrak_id, OLD.status_kontrak, NEW.status_kontrak, NOW());
    END IF;
END //

DELIMITER ;
```

**Cara kerja Trigger:**
- Dijalankan **AFTER UPDATE** pada tabel `Kontrak`
- Hanya mencatat log **jika status benar-benar berubah** (`OLD <> NEW`)
- Merekam kontrak mana yang berubah, dari status apa ke status apa, dan kapan waktunya

**Verifikasi Trigger:**

```sql
-- Ubah status kontrak untuk memicu trigger
UPDATE Kontrak SET status_kontrak = 'Selesai' WHERE kontrak_id = 1;

-- Cek hasil log
SELECT * FROM LogPerubahanKontrak;
```

---

## Soal 5 — Uji Stored Procedure

**Tujuan:** Menguji Stored Procedure `DaftarkanVendor` dengan dua skenario berlawanan.

### Skenario 1 — Pendaftaran Baru (SUKSES)

Mendaftarkan **Vendor 4 (Florist Fern)** ke **Acara 5 (Upacara Penghargaan)** — vendor ini belum punya kontrak aktif di acara tersebut.

```sql
CALL DaftarkanVendor(4, 5, 7500000, @pesan);
SELECT @pesan AS hasil;
```

**Expected Output:**
```
SUKSES: Vendor "Florist Fern" berhasil didaftarkan ke acara.
```

---

### Skenario 2 — Pendaftaran Duplikat (GAGAL)

Mendaftarkan **Vendor 4 (Florist Fern)** ke **Acara 5** lagi — sekarang sudah punya kontrak aktif dari Skenario 1.

```sql
CALL DaftarkanVendor(4, 5, 7500000, @pesan);
SELECT @pesan AS hasil;
```

**Expected Output:**
```
GAGAL: Vendor "Florist Fern" sudah memiliki kontrak aktif di acara ini.
```

---

## Cara Menjalankan

```bash
# 1. Login ke MySQL
mysql -u root -p

# 2. Jalankan script dataset (DDL + data awal)
source /path/to/dataset.sql

# 3. Jalankan script jawaban soal 2-4
source /path/to/jawaban.sql

# 4. Jalankan pengujian soal 5
CALL DaftarkanVendor(4, 5, 7500000, @pesan);
SELECT @pesan AS hasil;

CALL DaftarkanVendor(4, 5, 7500000, @pesan);
SELECT @pesan AS hasil;
```

---

*Praktikum Basis Data — Modul 5 | Departemen Teknologi Informasi, ITS*
