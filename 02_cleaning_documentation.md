# Data Cleaning Documentation
## Project: Inventory & Supply Chain Analysis
**Tanggal:** Juni 2026  
**Tools:** Python (Pandas, SQLAlchemy), PostgreSQL  
**File:** `02_cleaning.ipynb`

---

## 1. Sumber Data

Data diload dari PostgreSQL (`inventory_supply_chain`) yang sebelumnya diimport dari Google BigQuery (`bigquery-public-data.thelook_ecommerce`).

| Tabel | Baris | Kolom |
|-------|-------|-------|
| `inventory_items` | 490.356 | 12 |
| `orders` | 125.309 | 9 |
| `order_items` | 63.921 | 11 |
| `products` | 29.120 | 9 |
| `distribution_centers` | 10 | 4 |

---

## 2. Temuan Missing Values

### inventory_items
| Kolom | Missing | Keterangan |
|-------|---------|------------|
| `sold_at` | 308.422 (62.9%) | **Bukan error** — NULL berarti barang belum terjual |
| `product_brand` | 406 | Minor — diisi 'Unknown' |
| `product_name` | 44 | Minor — diisi 'Unknown' |

### orders
| Kolom | Missing | Keterangan |
|-------|---------|------------|
| `returned_at` | 112.868 | **Bukan error** — hanya order yang return yang terisi |
| `delivered_at` | 81.658 | **Bukan error** — order belum/tidak terdelivery |
| `shipped_at` | 44.011 | **Bukan error** — order cancelled/processing tidak dikirim |

### order_items
| Kolom | Missing | Keterangan |
|-------|---------|------------|
| `returned_at` | 57.626 (90.2%) | **Bukan error** — hanya item yang return yang terisi |
| `delivered_at` | 41.583 | **Bukan error** — item belum/tidak terdelivery |
| `shipped_at` | 22.460 | **Bukan error** — item cancelled/processing |

### products
| Kolom | Missing | Keterangan |
|-------|---------|------------|
| `brand` | 24 | Minor — diisi 'Unknown' |
| `name` | 2 | Minor — diisi 'Unknown' |

---

## 3. Tipe Data

Semua kolom tanggal (`created_at`, `sold_at`, `shipped_at`, `delivered_at`, `returned_at`) tersimpan sebagai tipe **object** (string) di PostgreSQL. Perlu dikonversi ke **datetime** untuk analisis temporal.

**Tindakan:** Konversi menggunakan `pd.to_datetime(col, format='mixed', utc=True)` untuk menghandle format tanggal yang tidak konsisten (ada suffix "UTC" di beberapa nilai).

---

## 4. Anomali Timestamp

### Temuan
Ditemukan kasus di mana `shipped_at < created_at` — barang tercatat dikirim **sebelum** order dibuat. Ini tidak mungkin terjadi secara logika bisnis.

| Tabel | Jumlah Anomali | Persentase |
|-------|---------------|------------|
| `orders` | 0 | 0% |
| `order_items` | 12.545 | 19.6% |

### Analisis Penyebab
Anomali tidak ditemukan di `orders` tapi signifikan di `order_items` (19.6%) mengindikasikan **masalah sistemik**, bukan human error individual. Kemungkinan penyebab: perbedaan timezone antara sistem warehouse dan sistem order management, atau bug di pipeline integrasi data antar sistem.

### Tindakan
Baris anomali **tidak dihapus** karena data order-nya sendiri valid — hanya timestamp-nya yang bermasalah. Ditambahkan kolom flag `is_timestamp_anomaly` (True/False) di tabel `order_items`.

**Implikasi analisis:** Kolom ini digunakan sebagai filter saat menghitung lead time pengiriman — hanya baris dengan `is_timestamp_anomaly = False` yang diikutkan dalam perhitungan, untuk menghindari bias pada metrik waktu.

```
Baris normal (is_timestamp_anomaly = False) : 51.376 (80.4%)
Baris anomali (is_timestamp_anomaly = True) : 12.545 (19.6%)
```

---

## 5. Ringkasan Keputusan Cleaning

| # | Masalah | Tindakan | Alasan |
|---|---------|----------|--------|
| 1 | Kolom tanggal bertipe object | Dikonversi ke datetime (UTC) | Diperlukan untuk perhitungan selisih waktu dan analisis temporal |
| 2 | `sold_at` NULL 308.422 baris | Dibiarkan | NULL = belum terjual, valid secara bisnis dan menjadi indikator dead stock |
| 3 | `shipped_at`/`delivered_at`/`returned_at` NULL | Dibiarkan | NULL mencerminkan status order yang belum melewati tahap tersebut |
| 4 | `product_brand` NULL 406 baris | Diisi 'Unknown' | Supaya tidak hilang dari analisis per brand |
| 5 | `product_name` NULL 44 baris | Diisi 'Unknown' | Supaya tidak hilang dari analisis per produk |
| 6 | Anomali timestamp order_items 12.545 baris | Diflag `is_timestamp_anomaly` | Data order tetap valid untuk analisis non-temporal; exclude hanya saat hitung lead time |
| 7 | Data original | Tetap tersimpan di PostgreSQL | Prinsip: raw data tidak dimodifikasi; data bersih disimpan di tabel terpisah (`_clean`) |

---

## 6. Output

Data bersih disimpan ke PostgreSQL dalam tabel terpisah:

| Tabel Baru | Keterangan |
|------------|------------|
| `inventory_items_clean` | Data inventory dengan datetime terkonversi dan missing values handled |
| `orders_clean` | Data orders dengan datetime terkonversi |
| `order_items_clean` | Data order items dengan datetime terkonversi + kolom `is_timestamp_anomaly` |
| `products_clean` | Data produk dengan missing brand/name diisi 'Unknown' |

Raw data original (`inventory_items`, `orders`, `order_items`, `products`) tetap tersimpan dan tidak dimodifikasi.

---

*Langkah selanjutnya: Exploratory Data Analysis (EDA) — Dead Stock Analysis, Lead Time Analysis, Cancelled & Returned Order Impact, Inventory Distribution, dan Category & Margin Analysis.*
