# WORKSTEPS APP — Database, Backend & Frontend
> **Scope:** Infrastruktur aplikasi untuk Dynamic Food Clearing House Simulator
> File ini terpisah dari `WORKSTEPS.md` (ML/data pipeline).
> Dikerjakan **setelah** Step 7–14 di WORKSTEPS.md sudah menghasilkan output stabil.

---

## Dependency Map

```
WORKSTEPS.md (ML Pipeline)
  └── Output: CSVs + trained models
        └── WORKSTEPS_APP.md (App Layer)
              ├── Step A1: Database Schema (Supabase)
              ├── Step A2: Data Ingestion (CSV → Supabase)
              ├── Step A3: Backend API (FastAPI)
              ├── Step A4: Frontend Dashboard
              └── Step A5: Developer/CMS Page
```

---

## Step A1 — Database Schema (Supabase)

**Goal:** Desain dan setup schema PostgreSQL di Supabase yang merepresentasikan seluruh output simulator — batch, events, dispatch, cluster — sehingga bisa diakses real-time dari frontend.

**Kenapa Supabase:**
- PostgreSQL cloud + REST API + Realtime subscription otomatis
- Built-in auth untuk Developer/CMS page
- Row Level Security (RLS) untuk isolasi data per user/role

### Tasks
- [ ] Buat project baru di [supabase.com](https://supabase.com)
- [ ] Simpan `SUPABASE_URL` dan `SUPABASE_ANON_KEY` di `.env`
- [ ] Desain & buat tabel `batches` — kolom: `batch_id`, `product_id`, `category`, `quantity`, `expiry_date`, `price_myr`, `store_id`, `risk_label`, `waste_risk_score`, `recommended_route`, `created_at`
- [ ] Desain & buat tabel `simulation_events` — kolom: `event_id`, `batch_id`, `event_type` (received/scored/dispatched/wasted), `timestamp`, `metadata` (JSONB)
- [ ] Desain & buat tabel `dispatch_log` — kolom: `dispatch_id`, `batch_id`, `receiver_id`, `distance_km`, `eta_hours`, `pickup_time`, `status`
- [ ] Desain & buat tabel `receivers` — kolom: `receiver_id`, `name`, `type` (food_bank/pantry/cafe), `lat`, `lng`, `capacity`, `acceptance_rate`, `accepted_categories` (array)
- [ ] Desain & buat tabel `cluster_results` — kolom: `entity_type` (store/product/receiver), `entity_id`, `cluster_label`, `waste_rate`, `velocity`, `run_date`
- [ ] Setup Row Level Security (RLS): read-only untuk `anon`, full access untuk `authenticated` (admin)
- [ ] Buat Supabase function/view: `v_dashboard_summary` — agregat KPI utama (total batch, total waste saved, avg risk score)
- [ ] Test koneksi dari Python: `supabase-py` client berhasil query semua tabel

---

## Step A2 — Data Ingestion (CSV → Supabase)

**Goal:** Setelah simulasi di WORKSTEPS.md selesai, upload semua output CSV ke Supabase secara otomatis agar frontend bisa langsung baca data terbaru.

**ML/Teknik:** Tidak ada — ETL script Python

### Tasks
- [ ] Install dependensi: `supabase-py`, `pandas`, `python-dotenv`
- [ ] Buat script `ingest/upload_batches.py` — baca `canonical_batch_table.csv` → upsert ke tabel `batches`
- [ ] Buat script `ingest/upload_events.py` — baca `simulation_events.csv` → upsert ke tabel `simulation_events`
- [ ] Buat script `ingest/upload_dispatch.py` — baca `dispatch_log.csv` → upsert ke tabel `dispatch_log`
- [ ] Buat script `ingest/upload_clusters.py` — baca output clustering Step 14 → upsert ke tabel `cluster_results`
- [ ] Buat script master `ingest/run_all.py` — jalankan semua upload sekaligus dengan error handling
- [ ] Tambahkan idempotency: gunakan `upsert` dengan conflict pada primary key agar aman di-run ulang
- [ ] Test: jalankan `run_all.py`, verifikasi data muncul di Supabase Table Editor

---

## Step A3 — Backend API (FastAPI)

**Goal:** Membangun REST API yang menjadi jembatan antara Supabase dan frontend. API ini juga menjadi titik untuk menjalankan ulang simulasi dan memicu re-scoring secara on-demand.

**Stack:** Python + FastAPI + Uvicorn + supabase-py

### Tasks
- [ ] Setup project struktur: `backend/main.py`, `backend/routers/`, `backend/services/`, `backend/schemas/`
- [ ] Install dependensi: `fastapi`, `uvicorn`, `supabase-py`, `python-dotenv`, `pydantic`
- [ ] Implementasi endpoint `GET /api/batches` — list semua batch dengan filter opsional (store_id, risk_label, route)
- [ ] Implementasi endpoint `GET /api/batches/{batch_id}` — detail satu batch + event history-nya
- [ ] Implementasi endpoint `GET /api/dashboard/summary` — KPI: total batch, total waste saved, avg risk score, dispatch success rate
- [ ] Implementasi endpoint `GET /api/dispatch` — list dispatch log dengan filter status
- [ ] Implementasi endpoint `GET /api/clusters` — hasil clustering per entity type
- [ ] Implementasi endpoint `POST /api/simulate/run` — trigger ulang simulasi Python (subprocess call ke WORKSTEPS pipeline)
- [ ] Implementasi endpoint `GET /api/receivers` — list semua penerima + kapasitas tersisa
- [ ] Setup CORS: izinkan request dari domain frontend
- [ ] Setup Supabase auth middleware: endpoint `/api/simulate/run` dan `/api/admin/*` hanya bisa diakses dengan Supabase JWT
- [ ] Tambahkan OpenAPI docs otomatis di `/docs` (default FastAPI)
- [ ] Test semua endpoint via Swagger UI (`/docs`)

---

## Step A4 — Frontend Dashboard (Next.js)

**Goal:** Dashboard yang bisa diakses publik (read-only) untuk memvisualisasikan hasil simulasi secara real-time — KPI cards, chart waste risk, peta dispatch, tabel batch.

**Stack:** Next.js (App Router) + Tailwind CSS + Recharts / Chart.js + Supabase JS client

### Tasks
- [ ] Init project: `npx create-next-app@latest frontend --typescript --tailwind --app`
- [ ] Install dependensi: `@supabase/supabase-js`, `recharts`, `leaflet` (untuk peta), `react-leaflet`
- [ ] Setup Supabase client di `lib/supabase.ts` dengan env vars
- [ ] Buat halaman `/dashboard` — layout utama dengan sidebar navigasi
- [ ] Implementasi KPI Cards component: Total Batch, Waste Saved (kg), Avg Risk Score, Dispatch Success Rate — data dari `/api/dashboard/summary`
- [ ] Implementasi chart: Waste Risk Distribution (Bar/Pie chart) — distribusi HIGH/MEDIUM/LOW per kategori
- [ ] Implementasi chart: Batch Route Funnel — berapa batch di tiap jalur (normal_sale → discount → donation → wasted)
- [ ] Implementasi chart: Demand Forecast vs Aktual (Line chart) — output Step 8
- [ ] Implementasi peta interaktif: plot lokasi donor & penerima, garis dispatch, warna berdasarkan status
- [ ] Implementasi tabel Batch List: filter by store, category, risk_label, route — dengan pagination
- [ ] Implementasi real-time update: subscribe Supabase Realtime pada tabel `simulation_events` — dashboard auto-refresh saat simulasi berjalan
- [ ] Responsive design: pastikan tampil baik di desktop & tablet
- [ ] Deploy ke Vercel: `vercel deploy` + set env vars di Vercel dashboard

---

## Step A5 — Developer / CMS Page (Admin Panel)

**Goal:** Halaman khusus yang dilindungi auth untuk: monitor simulasi, trigger ulang run, edit parameter, lihat raw log, dan manage daftar penerima — semua dari browser tanpa perlu buka terminal.

**Stack:** Next.js (route `/admin`) + Supabase Auth (magic link / email+password)

### Tasks

#### Auth
- [ ] Setup Supabase Auth: aktifkan Email provider di Supabase dashboard
- [ ] Buat halaman `/admin/login` — form email + password / magic link
- [ ] Implementasi middleware Next.js: redirect ke `/admin/login` jika tidak ada Supabase session
- [ ] Buat halaman `/admin` — layout admin dengan header "Developer Console"

#### Simulation Control
- [ ] Implementasi panel "Run Simulation": form parameter (horizon hari, jumlah toko, seed) → POST `/api/simulate/run`
- [ ] Tampilkan live log output simulasi (stream dari backend via SSE / WebSocket)
- [ ] Tampilkan status run terakhir: timestamp, duration, jumlah batch generated, errors

#### Data Management
- [ ] Implementasi tabel `Receivers Management`: tambah / edit / hapus penerima (food bank, pantry, kafe)
- [ ] Implementasi tabel `Batches Raw View`: lihat semua batch dengan semua kolom, bisa export CSV
- [ ] Implementasi tabel `Dispatch Log`: filter by status (success/failed/pending), bisa update status manual

#### Monitoring
- [ ] Implementasi halaman `/admin/clusters` — visualisasi hasil clustering dengan scatter plot interaktif
- [ ] Implementasi halaman `/admin/models` — info model ML: versi, tanggal training, metrik evaluasi (F1, RMSE, dll.)
- [ ] Implementasi halaman `/admin/logs` — raw event log `simulation_events` dengan search & filter

#### Deployment
- [ ] Pastikan route `/admin/*` tidak terindex oleh search engine (`robots.txt`)
- [ ] Test full auth flow: login → akses admin → logout
- [ ] Deploy bersama frontend ke Vercel (satu project, route terpisah)

---

## Struktur Folder Akhir

```
final-ait201/
├── data/                    # dataset mentah (gitignore besar)
├── notebooks/               # Step 7-14 Jupyter Notebooks
├── models/                  # trained model files (.pkl/.joblib)
├── output/                  # CSV output simulator
│   ├── canonical_batch_table.csv
│   ├── simulation_events.csv
│   └── dispatch_log.csv
├── ingest/                  # Step A2: upload scripts
│   ├── upload_batches.py
│   ├── upload_events.py
│   ├── upload_dispatch.py
│   ├── upload_clusters.py
│   └── run_all.py
├── backend/                 # Step A3: FastAPI
│   ├── main.py
│   ├── routers/
│   ├── services/
│   └── schemas/
├── frontend/                # Step A4 & A5: Next.js
│   ├── app/
│   │   ├── dashboard/
│   │   └── admin/
│   ├── components/
│   └── lib/
│       └── supabase.ts
├── .env                     # SUPABASE_URL, SUPABASE_ANON_KEY, dll.
├── WORKSTEPS.md             # ML/data pipeline steps
└── WORKSTEPS_APP.md         # File ini
```

---

## Progress Tracker

| Step | Status |
|---|---|
| Step A1 — Supabase Schema | ⬜ Belum dimulai |
| Step A2 — Data Ingestion | ⬜ Belum dimulai |
| Step A3 — Backend API (FastAPI) | ⬜ Belum dimulai |
| Step A4 — Frontend Dashboard | ⬜ Belum dimulai |
| Step A5 — Developer/CMS Page | ⬜ Belum dimulai |

> **Catatan urutan:** A1 → A2 → A3 → A4 → A5. Jangan mulai A2 sebelum minimal Step 10 di WORKSTEPS.md selesai (butuh output CSV).
