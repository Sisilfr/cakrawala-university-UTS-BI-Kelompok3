# Batas Desain — K3T2 POS UMKM (Outlet A+B+C, 12 bulan, slice k8)

## Pertanyaan yang tetap tidak terjawab

> **"Berapa persen omzet Outlet A+B+C sepanjang 2025 yang berasal dari pelanggan berulang (repeat), dan berapa hari rata-rata jarak antar kunjungan mereka?"**

## Mengapa desain sekarang tidak bisa menjawabnya

1. **`fact_transaksi_item` tidak punya kunci pelanggan.** Kolom `customer_id` sebenarnya ada di `transactions.csv`, tetapi kolom tersebut tidak dibawa ke fact dan juga belum ada `dim_customer` karena keputusan scope cut di `docs/D7_scope_cut.md`.

2. **Sebagian besar transaksi masih anonim.** Dari 15.671 `transaction_id` unik, sebanyak 3.987 (±25%) tidak memiliki `customer_id`. Untuk transaksi seperti ini, status "repeat" tidak dapat ditentukan. Akibatnya, penyebut yang digunakan untuk menghitung persentase juga menjadi tidak jelas.

3. **Definisi "repeat" belum ditetapkan.** Belum ada kesepakatan mengenai berapa kali kunjungan dan dalam rentang waktu berapa hari yang bisa dikategorikan sebagai repeat. Selain itu, belum ditentukan apakah transaksi PENDING/VOID/REFUND akan dianggap sebagai kunjungan atau tidak.

Data profiling sendiri menunjukkan bahwa `customers.csv` berisi 800 pelanggan dan seluruh `customer_id` yang terdapat di transaksi juga ada di master tersebut (0 yatim). Jadi, masalah utamanya bukan pada integritas kunci, tetapi pada desain dan kelengkapan data yang digunakan.

## Yang dibutuhkan untuk menjawabnya

| Kebutuhan | Bentuk |
|---|---|
| Dimensi pelanggan | `dim_customer` (customers.csv, 800 baris + anggota `UNKNOWN` untuk transaksi anonim), SCD Type 1 |
| Kunci di fact | Kolom `customer_sk` di fact, diambil dari `transactions.csv` (di-join di level transaksi, bukan per item) |
| Definisi bisnis | Disepakati dengan pemilik UMKM, misalnya "repeat = pelanggan dengan ≥2 transaksi PAID dalam 90 hari" |
| Penanganan anonim | Dilaporkan terpisah ("tanpa identitas ±25% transaksi"), tidak dianggap pelanggan baru |
| Privasi | Kolom `telepon` tidak dimuat ke warehouse (atau di-hash) |
| Data tambahan (opsional) | Program loyalitas / keanggotaan agar pelanggan anonim bisa diidentifikasi |

## Catatan

Pertanyaan ini sengaja dipilih karena berhubungan langsung dengan artefak yang dipotong di D7. Kelompok memutuskan untuk tidak membangun `dim_customer` karena target build hanya 1 fact + 1 conformed dimension.

