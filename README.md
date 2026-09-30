# Data Warehouse POS UMKM Multi-Outlet — Kelompok 3

> **Business Intelligence Systems (SDA2161)** — Universitas Cakrawala  
> **Topik Project**: T2 — POS UMKM Multi-Outlet  
> **Slice Scope**: `k8` (Outlet A + B + C, 12 Bulan / Periode 2025)  

---

## 👥 Informasi Tim & Deliverable UTS

* **Kelompok**: Kelompok 3
* **Topik Target**: T2 POS UMKM (`data/raw/t2_umkm/`)
* **Slice Datasets**: `k8` (Outlet A, B, C — 12 Bulan Kalender)

---

## 📐 Arsitektur Data Warehouse & Star Schema

Sistem Data Warehouse dirancang menggunakan arsitektur **Star Schema** yang difokuskan pada granularitas transaksi item terkecil (*item level grain*) untuk fleksibilitas analisis bauran produk dan omzet outlet.

```
       +--------------------+          +-------------------------+
       |     dim_outlet     |          |       dim_date          |
       +--------------------+          +-------------------------+
       | outlet_sk (PK)     |          | date_sk (PK)            |
       | outlet_id (NK)     |          | full_date               |
       | nama_outlet, kota  |          | tahun, triwulan, bulan  |
       +---------+----------+          +------------+------------+
                 |                                  |
                 |      +---------------------+     |
                 +----->| fact_transaksi_item |<----+
                        +---------------------+
                 +----->| date_sk (FK)        |<----+
                 |      | product_sk (FK)     |     |
                 |      | outlet_sk (FK)      |     |
                 |      | status_sk (FK)      |     |
                 |      | transaction_id      |     |
                 |      | qty (ADDITIVE)      |     |
                 |      | subtotal_rupiah     |     |
                 |      | diskon (ADDITIVE)   |     |
                 |      | total_bayar_trx     |     |
                 |      +---------------------+     |
                 |                                  |
       +---------+----------+          +------------+------------+
       |    dim_product     |          |   dim_status_transaksi  |
       +--------------------+          +-------------------------+
       | product_sk (PK)    |          | status_sk (PK)          |
       | product_id (NK)    |          | status_code (NK)        |
       | valid_from, valid_to|          | label_indonesia         |
       | is_current (SCD 2) |          | kategori_final          |
       +--------------------+          +-------------------------+
```

### 1. Fact Table Utama
* **`fact_transaksi_item`** ([sql/30_fact_transaksi_item.sql](file:///D:/UTS_BI_Kel3/sql/30_fact_transaksi_item.sql))
  * **Grain**: 1 baris per item produk per transaksi per outlet per tanggal.
  * **Natural Key**: `(transaction_id, item_id)`
  * **Measures**:
    * `qty`: **ADDITIVE** (Total unit barang terjual/direset).
    * `subtotal_rupiah`: **ADDITIVE** (Nilai penjualan item setelah diskon).
    * `diskon`: **ADDITIVE** (Nilai potongan harga per item).
    * `total_bayar_trx`: **SEMI-ADDITIVE** (Header total faktur dari POS).

### 2. Dimension Tables
1. **`dim_date`** (Conformed Dimension - SCD Type 0)
   * Kunci `date_sk` format `YYYYMMDD` (2024-01-01 s.d. 2027-12-31 + Unknown `-1`).
2. **`dim_product`** (SCD Type 2 - Histori Perubahan Harga)
   * Menyimpan histori harga produk dengan atribut `valid_from`, `valid_to`, `is_current`. Menampung anggota unknown (`product_sk = -1`) untuk mengani FK orphan dari profiling.
3. **`dim_outlet`** (SCD Type 1 - Master Outlet)
   * Master outlet aktif Slice `k8` (Outlet A, B, C). Outlet D dikecualikan karena permanen tutup.
4. **`dim_status_transaksi`** (SCD Type 0 - Kamus Status POS)
   * Mengkategorikan status POS (`PAID`, `PENDING`, `CANCELLED`, `REFUNDED`, `PARTIAL`) dengan flag `kategori_final` dan `berdampak_pendapatan`.

---

## Batas Lingkup Capstone (Scope Cut - D7)

Berdasarkan formulir persetujuan batas lingkup ([docs/D7_scope_cut.md](file:///D:/UTS_BI_Kel3/docs/D7_scope_cut.md)):

### AKAN DIBANGUN
1. **Fact Table**: `fact_transaksi_item` (dedup 219 transaksi duplikat).
2. **4 Tabel Dimensi**: `dim_date`, `dim_product` (SCD 2), `dim_outlet` (SCD 1), `dim_status_transaksi` (SCD 0).
3. **3 Query Analitik**: Ukuran keranjang (`q01`), Pertumbuhan MoM (`q02`), Rasio Retur (`q03`).
4. **Kamus Metrik & Query Metrik**: Omzet Bersih Bulanan, Pertumbuhan MoM, Rasio Retur Triwulanan.
5. **Data Quality Tests**: 6 test kualitas data dengan severity & expected result.
6. **Pipeline Load Idempoten**: `sql/load.sql` dengan strategi `CREATE OR REPLACE TABLE`.

### TIDAK LAGI DIBANGUN (Scope Cut)
1. **`dim_customer`**: 25% transaksi anonim di POS & grain fact adalah level item.
2. **Analisis Pelanggan Berulang (RFM / Retention)**: Kunci pelanggan tidak dibawa ke fact item.
3. **Tabel `dim_kategori` Terpisah**: Kategori didenormalisasi langsung pada `dim_product`.
4. **Data Historis Outlet D**: Permanen tutup & di luar cakupan slice `k8`.

---

## Kualitas Data & Guard Tests (D4)

Pengujian kualitas data diatur dalam [tests/test_definitions.yml](file:///D:/UTS_BI_Kel3/tests/test_definitions.yml):

| Test Name | Severity | Expected | Masalah Data yang Ditangkap |
|---|---|---|---|
| `transaction_id_duplikat` | **blocking** | `fail` | 219 transaksi duplikat di POS yang dapat menggandakan omzet. |
| `item_fk_orphan` | **blocking** | `fail` | 1 baris item transaksi tanpa `product_id` di master produk. |
| `timestamp_utc_tanpa_zona` | **warning** | `fail` | 2.389 timestamp UTC tanpa tanda zona waktu. |
| `transaksi_tanpa_item` | **warning** | `fail` | 158 header transaksi tanpa rincian baris item. |
| `qty_negatif_bukan_refund` | **warning** | `fail` | 621 baris `qty < 0` yang tercatat pada status selain REFUND. |
| `harga_satuan_tidak_positif` | **blocking** | `pass` | 0 baris (memastikan tidak ada harga nol/negatif). |

---

## Struktur Direktori Repository

```
├── data/raw/t2_umkm/       # Seed CSV dataset T2 POS UMKM (customers, outlets, products, transactions, items)
├── docs/                   # Dokumen perancangan (kamus_metrik.md, D7_scope_cut.md, Batasan_desain.md, profile_t2_k8.md)
├── pipeline/               # Python script pipeline (load.py, profile.py, config.py, DESIGN_load.md)
├── sql/
│   ├── 10_dim_date.sql            # Conformed date dimension generator
│   ├── 20_dim_product.sql         # DDL Dimensi Produk (SCD Type 2)
│   ├── 20_dim_outlet.sql          # DDL Dimensi Outlet (SCD Type 1)
│   ├── 20_dim_status_transaksi.sql # DDL Dimensi Status Transaksi (SCD Type 0)
│   ├── 30_fact_transaksi_item.sql # DDL Fact Table Utama (Item Grain)
│   ├── 40_analytics/              # Query analitik (q01_contoh.sql, q02.sql, q03.sql)
│   ├── 50_metrics/                # Metrik bisnis (omzet_bersih_bulanan, m02_pertumbuhan_mom, m03_rasio_retur)
│   └── load.sql                   # Skrip eksekusi idempoten utama untuk DuckDB
├── tests/                         # Pengujian Kualitas Data (test_definitions.yml, run_tests.py)
├── checkpoint.py                  # Skrip verifikasi DoD checkpoint otomatis
└── warehouse/                     # Output DuckDB Warehouse (t2_k8.duckdb + fallback/)
```

---

## Cara Menjalankan & Verifikasi Project

### 1. Eksekusi Load Warehouse (Idempoten)
Jalankan proses load warehouse 2 kali (`--twice`) untuk membuktikan idempotensi skrip:
```bash
python -m pipeline.load --topic t2 --slice k8 --twice
```

### 2. Jalankan Data Quality Tests
Jalankan pengujian data kualitas berbasis definisi YAML:
```bash
python tests/run_tests.py --topic t2
```

### 3. Verifikasi DoD Checkpoint Sesi 8 / UTS
Jalankan checkpoint verifikasi DoD untuk memastikan kelengkapan perancangan:
```bash
python checkpoint.py verify --sesi 8 --topic t2
```
