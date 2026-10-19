# WORKSTEPS — Dynamic Food Clearing House Simulator
> **Scope:** Step 7 (FEFO/Stok) → Step 14 (Monitoring)
> Step 1–6 (produksi) hanya konteks; tidak dikerjakan di proyek ini.

---

## Step 7 — FEFO & Manajemen Stok Gudang

**Goal:** Membangun modul yang memprioritaskan batch berdasarkan expiry date (FEFO), menggantikan logika FIFO reaktif yang ada saat ini. Output-nya adalah urutan prioritas pengiriman per batch.

**ML/Teknik:** Supervised — spoilage-risk classification

**Dataset:**
- 📦 [Grocery Inventory (Kaggle)](https://www.kaggle.com/datasets/willianoliveiragibin/grocery-inventory)
- 📦 [Perishable Goods Management (Kaggle)](https://www.kaggle.com/datasets/likithagedipudi/perishable-goods-management)

### Tasks
- [ ] Download & load dataset Grocery Inventory
- [ ] Download & load dataset Perishable Goods Management
- [ ] EDA: distribusi expiry date, stock quantity, sales volume, inventory turnover rate
- [ ] Feature engineering: `days_to_expiry`, `turnover_ratio`, `batch_age`
- [ ] Buat label: klasifikasi spoilage-risk (HIGH / MEDIUM / LOW) berdasarkan `days_to_expiry` + `Reorder_Level`
- [ ] Training model klasifikasi spoilage-risk (Decision Tree / Random Forest)
- [ ] Evaluasi model (accuracy, precision, recall, F1)
- [ ] Implementasi FEFO logic: sort batch berdasarkan `Expiration_Date` ascending
- [ ] Output: tabel prioritas batch → kolom `batch_id`, `category`, `days_to_expiry`, `risk_label`, `dispatch_priority`

---

## Step 8 — Perencanaan Pengiriman (Demand Forecasting)

**Goal:** Memprediksi permintaan per toko per produk sehingga alokasi stok bisa dilakukan secara proaktif, bukan reaktif terhadap pesanan yang masuk.

**ML/Teknik:** Supervised — time series demand forecasting

**Dataset:**
- 📊 [M5 / Walmart Store Sales (Kaggle)](https://www.kaggle.com/competitions/store-sales-time-series-forecasting/data)
- 📊 [Retail Sales Forecast Dataset 2026 (Kaggle)](https://www.kaggle.com/datasets/mmumairkhattak/retail-sales-forecast-dataset-2026)

### Tasks
- [ ] Download & load dataset M5/Walmart (date, store_nbr, item_id, sales, onpromotion, holiday, oil_price)
- [ ] Download & load dataset Retail Sales Forecast 2026 sebagai suplemen
- [ ] EDA: pola demand harian, efek promo & hari libur, seasonality
- [ ] Feature engineering: `lag_7`, `lag_14`, `rolling_mean_7`, `is_promo`, `is_holiday`
- [ ] Split data: train/test per time window
- [ ] Training model forecasting (SARIMA / LightGBM / Prophet)
- [ ] Evaluasi model (MAE, RMSE, MAPE)
- [ ] Mapping pola M5 ke konteks Malaysia (kalibrasi skala demand)
- [ ] Output: `demand_forecast` per `store_id` x `item_id` x `date` untuk 7-14 hari ke depan

---

## Step 9 — Mapping Tables & Canonical Schema

**Goal:** Menyatukan semua dataset dari Step 7 & 8 ke dalam satu skema kanonikal sebelum masuk ke engine simulator. Tanpa ini, data dari sumber berbeda tidak bisa diproses bersama.

**ML/Teknik:** Tidak ada — data engineering murni

### Tasks
- [ ] Definisikan Canonical Schema: field wajib (`batch_id`, `product_id`, `category`, `quantity`, `expiry_date`, `price_myr`, `store_id`, `demand_forecast`, `risk_label`)
- [ ] Buat mapping table: kategori Grocery Inventory ↔ kategori M5 ↔ kategori Perishable Goods
- [ ] Buat mapping unit (kg/unit/pack → unit standar)
- [ ] Normalisasi harga ke MYR menggunakan dataset PriceCatcher DOSM
- [ ] Normalisasi timestamp ke timezone Malaysia (UTC+8)
- [ ] Buat fungsi `merge_to_canonical(df_inventory, df_forecast, df_price)` → DataFrame kanonikal
- [ ] Validasi: tidak ada null pada field wajib, tidak ada duplikat `batch_id`
- [ ] Output: `canonical_batch_table.csv` / DataFrame siap masuk simulator engine

---

## Step 10 — Simulator Engine (Batch Generator)

**Goal:** Mensimulasikan aliran batch pangan secara sintetis namun terkalibrasi dari data riil — menggantikan kebutuhan akan mitra nyata untuk MVP.

**ML/Teknik:** Tidak ada — rule-based simulation

### Tasks
- [ ] Definisikan parameter simulasi: jumlah toko, jumlah produk, horizon waktu (30/60/90 hari)
- [ ] Implementasi generator batch sintetis (pakai distribusi dari Canonical Schema)
- [ ] Kalibrasi waste rate per kategori menggunakan FAO FLW benchmark
- [ ] Kalibrasi demand per toko menggunakan output Step 8
- [ ] Kalibrasi harga menggunakan PriceCatcher MYR
- [ ] Jalankan simulasi end-to-end: hasilkan event stream batch (received, scored, dispatched, wasted)
- [ ] Output: `simulation_events.csv` — log semua event batch selama horizon simulasi

---

## Step 11 — Deteksi Surplus & Waste-Risk Scoring

**Goal:** Setiap batch di simulator mendapatkan skor risiko waste otomatis (persentase kemungkinan terbuang dalam Y jam). Ini adalah inti sistem untuk intervensi proaktif.

**ML/Teknik:** Supervised — waste-risk scoring (regression + classification)

**Dataset:**
- 🗑️ [Food Waste Prediction & Demand Forecasting (Kaggle)](https://www.kaggle.com/datasets/kp00011/food-waste-prediction-and-demand-forecasting-dataset)
- 🗑️ [AI-Powered Food Waste Management (Kaggle)](https://www.kaggle.com/datasets/vinay24baghira/ai-powered-food-waste-management-dataset-v1/data)
- 💰 [PriceCatcher DOSM Malaysia](https://open.dosm.gov.my/data-catalogue/pricecatcher) — untuk fitur `price_index`

### Tasks
- [ ] Download & load dataset Food Waste Prediction (inventory, sales, expiry, temperature, kategori, label risiko)
- [ ] Download & load dataset AI-Powered Food Waste Management (skenario restoran/supermarket/kafe)
- [ ] EDA: distribusi label risiko HIGH/MEDIUM/LOW, korelasi fitur dengan waste
- [ ] Feature engineering: `days_to_expiry`, `demand_gap` (forecast vs stok), `price_index_myr`, `category_waste_rate_fao`, `temperature_ok`
- [ ] Training model waste-risk (Gradient Boosting / XGBoost untuk regression skor 0-1 + classification HIGH/MED/LOW)
- [ ] Evaluasi model (R² untuk regression, F1 untuk classification)
- [ ] Integrasi ke simulator: setiap batch di Step 10 mendapat `waste_risk_score` & `waste_risk_label`
- [ ] Output: field tambahan `waste_risk_score`, `waste_risk_label`, `time_to_waste_hours` pada `simulation_events.csv`

---

## Step 12 — Rekomendasi Jalur Surplus (Rule Engine)

**Goal:** Berdasarkan skor risiko dari Step 11, sistem merekomendasikan jalur terbaik untuk setiap batch: jual normal → diskon dinamis → penawaran komersial → donasi → recovery. Urutan ini anti-moral-hazard by design.

**ML/Teknik:** Rule engine + dynamic pricing (bukan ML)

**Referensi:**
- 📋 [ReFED Measurement Framework](https://refed.org/downloads/2026-refed-measurement-report.pdf) — taksonomi 10 destinasi surplus
- 💰 PriceCatcher DOSM — threshold harga diskon berbasis harga pasar riil

### Tasks
- [ ] Baca & pahami taksonomi destinasi surplus ReFED (10 jalur)
- [ ] Definisikan rule hierarchy: `normal_sale` → `discount_sale` → `commercial_offer` → `donation` → `animal_feed` → `composting`
- [ ] Implementasi threshold per jalur berdasarkan `days_to_expiry` + `waste_risk_score`
- [ ] Implementasi dynamic pricing: hitung `discount_percent` berdasarkan `days_to_expiry` & `price_index_myr`
- [ ] Implementasi kuota donasi per donor (anti-moral-hazard: donasi tidak boleh jadi default pertama)
- [ ] Implementasi reputasi donor: track rasio `donated / total_surplus` agar tidak overproduce sengaja
- [ ] Jalankan rule engine pada semua batch di `simulation_events.csv`
- [ ] Output: field `recommended_route`, `discount_percent`, `expected_value_myr` pada setiap batch

---

## Step 13 — Matching & Dispatch ke Penerima

**Goal:** Untuk batch yang masuk jalur donasi, sistem mencocokkan dengan food bank / pantry / kafe terdekat secara otomatis, dan menjadwalkan pengambilan sebelum batch kadaluarsa.

**ML/Teknik:** Greedy optimization / nearest-neighbor matching (bukan ML)

**Tools:**
- 🗺️ [Nominatim / OpenStreetMap](https://nominatim.org/release-docs/develop/api/Search/) — geocoding alamat donor & penerima
- 🚗 [OSRM (Open Source Routing Machine)](https://github.com/Project-OSRM) — hitung jarak & ETA

### Tasks
- [ ] Buat daftar simulasi penerima (food bank, pantry, kafe) dengan alamat fiktif tapi realistis di Malaysia
- [ ] Geocoding: konversi alamat → koordinat via Nominatim API
- [ ] Implementasi OSRM API call: hitung jarak & ETA antara donor ↔ penerima
- [ ] Implementasi greedy matching: pilih penerima dengan skor tertinggi = `min(distance) + max(capacity_remaining) + max(acceptance_rate)`
- [ ] Implementasi constraint: penerima hanya menerima kategori tertentu (mis. food bank tidak menerima alkohol)
- [ ] Implementasi dispatch schedule: `pickup_time` = `current_time + eta_hours`, harus < `expiry_time`
- [ ] Simulasikan acceptance rate penerima (tidak selalu 100% menerima)
- [ ] Output: tabel `dispatch_log.csv` — kolom `batch_id`, `receiver_id`, `distance_km`, `eta_hours`, `pickup_time`, `status`

---

## Step 14 — Monitoring & Clustering

**Goal:** Menganalisis output agregat simulator untuk menemukan pola tersembunyi — toko mana yang paling boros, produk apa yang paling sering surplus, penerima mana yang paling efektif — menggunakan unsupervised learning.

**ML/Teknik:** Unsupervised — K-Means / DBSCAN clustering

**Data:** Output agregat dari Step 7–13 (tidak butuh dataset eksternal)

### Tasks
- [ ] Agregasi output simulator: `waste_rate_per_store`, `velocity_per_product`, `acceptance_rate_per_receiver`, `avg_risk_score_per_category`
- [ ] EDA: visualisasi distribusi metrik per entitas (store, product, receiver)
- [ ] Feature scaling: normalisasi fitur sebelum clustering
- [ ] Tentukan jumlah cluster optimal: Elbow Method + Silhouette Score
- [ ] Jalankan K-Means clustering pada profil toko & produk
- [ ] Jalankan DBSCAN untuk deteksi outlier (toko dengan waste rate anomali tinggi)
- [ ] Interpretasi cluster: beri label human-readable (mis. "high-waste fast-moving", "low-waste stable")
- [ ] Buat visualisasi dashboard hasil clustering (scatter plot, heatmap)
- [ ] Output: `cluster_report.md` + visualisasi PNG/HTML

---

## Deliverables Akhir

| Deliverable | Berasal dari Step |
|---|---|
| `canonical_batch_table.csv` | Step 9 |
| `simulation_events.csv` (dengan risk score & route) | Step 10-12 |
| `dispatch_log.csv` | Step 13 |
| `cluster_report.md` + visualisasi | Step 14 |
| Model file: spoilage classifier | Step 7 |
| Model file: demand forecaster | Step 8 |
| Model file: waste-risk scorer | Step 11 |

---

## Progress Tracker

| Step | Status |
|---|---|
| Step 7 — FEFO & Stok | ⬜ Belum dimulai |
| Step 8 — Demand Forecasting | ⬜ Belum dimulai |
| Step 9 — Canonical Schema | ⬜ Belum dimulai |
| Step 10 — Simulator Engine | ⬜ Belum dimulai |
| Step 11 — Waste-Risk Scoring | ⬜ Belum dimulai |
| Step 12 — Rule Engine Jalur | ⬜ Belum dimulai |
| Step 13 — Matching & Dispatch | ⬜ Belum dimulai |
| Step 14 — Monitoring & Clustering | ⬜ Belum dimulai |
