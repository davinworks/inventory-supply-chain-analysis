# BigQuery Exploration — thelook_ecommerce
## Project: Inventory & Supply Chain Analysis
**Tanggal eksplorasi:** Juni 2026  
**Dataset:** `bigquery-public-data.thelook_ecommerce`  
**Tools:** Google BigQuery (SQL)

---

## 1. Tabel yang Tersedia

| Tabel | Digunakan | Alasan |
|-------|-----------|--------|
| `inventory_items` | ✅ Ya | Data stok utama — kapan barang masuk gudang, kapan terjual |
| `orders` | ✅ Ya | Data transaksi order — status, waktu proses, pengiriman |
| `order_items` | ✅ Ya | Detail item per order — produk, harga, lead time |s
| `products` | ✅ Ya | Info produk — kategori, harga modal, harga jual |
| `distribution_centers` | ✅ Ya | Lokasi 10 gudang di seluruh AS |
| `users` | ❌ Tidak | Data demografi customer — tidak relevan untuk analisis supply chain |
| `events` | ❌ Tidak | Data event website — tidak relevan untuk analisis inventory |

---

## 2. Struktur & Skala Data

| Tabel | Jumlah Baris |
|-------|-------------|
| `inventory_items` | 490.356 |
| `orders` | 125.309 |
| `order_items` | 181.934 |
| `products` | 29.120 |
| `distribution_centers` | 10 |

**Rasio inventory vs order items:** 490.356 / 181.934 ≈ 2.7x  
artinya hanya 37% dari total inventory yang berhasil diorder. Ini sinyal awal potensi dead stock yang perlu diinvestigasi lebih dalam.

---

## 3. Detail Kolom Per Tabel

### inventory_items
| Kolom | Keterangan |
|-------|------------|
| `id` | ID unik tiap unit barang fisik di gudang |
| `product_id` | Referensi ke tabel products |
| `created_at` | Waktu barang masuk gudang |
| `sold_at` | Waktu barang terjual — **NULL berarti belum terjual** |
| `cost` | Harga modal barang |
| `product_category` | Kategori produk |
| `product_retail_price` | Harga jual ke customer |
| `product_distribution_center_id` | Barang ini berada di gudang mana |

**Catatan:** Kolom `sold_at` yang bernilai NULL adalah indikator utama untuk identifikasi dead stock.

### orders
| Kolom | Keterangan |
|-------|------------|
| `order_id` | ID unik order |
| `user_id` | ID customer |
| `status` | Status order: Complete, Shipped, Processing, Cancelled, Returned |
| `created_at` | Waktu order dibuat |
| `shipped_at` | Waktu order dikirim |
| `delivered_at` | Waktu order diterima customer |
| `returned_at` | Waktu order dikembalikan (jika ada) |
| `num_of_item` | Jumlah item dalam satu order |

### order_items
| Kolom | Keterangan |
|-------|------------|
| `id` | ID unik item dalam order |
| `order_id` | Referensi ke tabel orders |
| `product_id` | Referensi ke tabel products |
| `inventory_item_id` | Referensi ke tabel inventory_items |
| `status` | Status item |
| `created_at` | Waktu order dibuat |
| `shipped_at` | Waktu item dikirim |
| `delivered_at` | Waktu item diterima |
| `returned_at` | Waktu item dikembalikan |
| `sale_price` | Harga jual aktual |

### products
| Kolom | Keterangan |
|-------|------------|
| `id` | ID unik produk |
| `cost` | Harga modal |
| `category` | Kategori produk |
| `name` | Nama produk |
| `brand` | Brand produk |
| `retail_price` | Harga jual standar |
| `department` | Departemen: Men / Women |
| `sku` | Kode unik produk |
| `distribution_center_id` | Gudang utama produk ini |

### distribution_centers
| Kolom | Keterangan |
|-------|------------|
| `id` | ID gudang |
| `name` | Nama & lokasi gudang |
| `latitude` / `longitude` | Koordinat geografis |

**10 gudang aktif**, semua berlokasi di Amerika Serikat: Memphis TN, Chicago IL, Los Angeles CA, Houston TX, New Orleans LA, Port Authority NY/NJ, Philadelphia PA, Mobile AL, Charleston SC, Savannah GA.

---

## 4. Temuan Awal (Pre-EDA)

### Temuan 1: Distribusi Status Order
```
Complete    : 31.210 (24,91%)
Shipped     : 37.647 (30,04%)
Processing  : 25.188 (20,10%)
Cancelled   : 18.823 (15,02%)
Returned    : 12.441 ( 9,93%)
```
**Insight awal:** Cancelled (15%) dan Returned (9,93%) secara gabungan mencapai ~25% dari total order. Ini signifikan dan berdampak langsung pada inventory — barang yang cancelled/returned berpotensi menjadi dead stock.

### Temuan 2: Anomali Timestamp di order_items
Ditemukan kasus di mana `shipped_at` lebih awal dari `created_at` — barang tercatat dikirim sebelum order dibuat. Kemungkinan penyebab: timezone mismatch atau data entry error.  
**Tindakan:** Perlu dibersihkan di tahap Python cleaning. Record dengan kondisi `shipped_at < created_at` akan diflag sebagai anomali.

### Temuan 3: Sinyal Dead Stock di inventory_items
Dari sampling awal, ditemukan banyak baris dengan `sold_at = NULL` — artinya barang sudah masuk gudang namun belum terjual. Dengan rasio 2.7x antara total inventory dan total order items, investigasi dead stock menjadi prioritas EDA.

### Temuan 4: Catatan Dataset
Dataset `thelook_ecommerce` adalah data sintetis yang dirancang oleh Google untuk keperluan demonstrasi dan pembelajaran. Meskipun strukturnya realistis dan mencerminkan pola e-commerce nyata, temuan dari analisis ini tidak merepresentasikan performa bisnis perusahaan nyata.

---

## 5. Rencana EDA Selanjutnya

Berdasarkan eksplorasi awal, EDA akan difokuskan pada:

1. **Dead Stock Analysis** — Berapa persen inventory yang belum terjual? Sudah berapa lama menumpuk?
2. **Cancelled & Returned Order Impact** — Seberapa besar dampaknya terhadap pergerakan inventory?
3. **Lead Time Analysis** — Berapa rata-rata waktu dari order dibuat hingga diterima customer? Ada bottleneck di mana?
4. **Inventory Distribution** — Apakah stok tersebar merata di 10 gudang atau menumpuk di lokasi tertentu?
5. **Kategori & Margin** — Kategori mana yang paling banyak dead stock? Berapa kerugian potensialnya?

---

*Eksplorasi dilakukan menggunakan Google BigQuery SQL. Langkah selanjutnya: export data ke PostgreSQL dan cleaning menggunakan Python (Pandas).*
