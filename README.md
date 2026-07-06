# 🌪️ Simulasi Numerik WRF-ARW: Dinamika Siklon Tropis Cempaka (26–29 November 2017)

![WRF](https://img.shields.io/badge/WRF--ARW-v4.3-blue) ![WPS](https://img.shields.io/badge/WPS-v4.3-orange) ![Docker](https://img.shields.io/badge/Container-dtcenter%2Fwps__wrf-2496ED?logo=docker&logoColor=white) ![Data](https://img.shields.io/badge/Input-ERA5%20Reanalysis-green) ![License](https://img.shields.io/badge/License-MIT-lightgrey)

Repositori ini berisi konfigurasi, dokumentasi teknis, dan alur kerja (*workflow*) lengkap untuk menjalankan simulasi numerik **WRF-ARW (Weather Research and Forecasting – Advanced Research WRF) v4.3** guna membedah struktur dinamika dan penjalaran sabuk hujan (*rainband*) **Siklon Tropis Cempaka**.

---

## 📋 Daftar Isi

1. [Pendahuluan & Prerequisites](#1-📖-pendahuluan--prerequisites)
2. [Konfigurasi Domain (Nesting)](#2-🗺️-konfigurasi-domain-wrf-domain-wizard)
3. [Persiapan Data Input](#3-💾-persiapan-data-input-di-luar-container)
4. [Fase WPS-4.3 (Preprocessing)](#4-⚙️-fase-wps-43-wrf-preprocessing-system)
5. [Fase WRF-4.3 (Core Model Running)](#5-🌀-fase-wrf-43-core-model-running)
6. [Troubleshooting & Tips Efisiensi](#6-🛠️-troubleshooting--tips-efisiensi)
7. [Struktur Direktori Proyek](#7-📂-struktur-direktori-proyek)
8. [Referensi](#8-📚-referensi)

---

## 1. 📖 Pendahuluan & Prerequisites

### 1.1 Latar Belakang Kasus

**Siklon Tropis Cempaka** terbentuk di selatan Pulau Jawa pada akhir November 2017 dan menjadi salah satu siklon tropis pertama yang tercatat tumbuh sangat dekat dengan garis pantai selatan Jawa dalam catatan modern BMKG. Interaksi antara sirkulasi siklonik Cempaka dengan topografi pegunungan selatan Jawa memicu konvergensi masa udara basah yang intens, menghasilkan curah hujan ekstrem secara terus-menerus di sepanjang **26–29 November 2017**. Dampaknya sangat signifikan di:

- 🏞️ **Kabupaten Cilacap**
- 🏞️ **Kabupaten Kebumen**
- 🏞️ **Kabupaten Purworejo**
- 🏞️ **D.I. Yogyakarta**

berupa **banjir bandang dan longsor** yang meluas. Proyek ini bertujuan merekonstruksi ulang kejadian tersebut secara numerik untuk memahami mekanisme fisis penjalaran sabuk hujan (*rainband propagation*) siklon terhadap topografi pesisir selatan Jawa.

### 1.2 Prerequisites

| Komponen | Keterangan |
|---|---|
| **Environment** | Docker Desktop, menjalankan container `dtcenter/wps_wrf` (tanpa volume mounting) |
| **Model** | WRF-ARW v4.3 & WPS v4.3 |
| **Data Meteorologi** | ERA5 Reanalysis (format GRIB) |
| **Data Geografis Statis** | WPS_GEOG (Complete Dataset) |
| **RAM Disarankan** | ≥ 16 GB (untuk domain 3 nest) |
| **Storage Disarankan** | ≥ 100 GB ruang kosong |

### 1.3 Instalasi Docker Image Resmi (NCAR/DTCenter)

Tarik *image* resmi yang sudah berisi *toolchain* WPS + WRF terkompilasi (NETCDF, MPI, JASPER, dsb.):

```bash
docker pull dtcenter/wps_wrf
```

### 1.4 Menjalankan Container via Docker Desktop

Pendekatan proyek ini **tidak menggunakan volume mounting**. Container dijalankan langsung lewat **Docker Desktop** agar operasi *Linux-based* lebih mudah dikelola secara visual.

**Langkah singkat:**

1. Buka **Docker Desktop** → tab *Images* → jalankan (**Run**) image `dtcenter/wps_wrf`.
2. Setelah container berjalan (tab *Containers*), klik container tersebut → buka tab **Exec** untuk langsung masuk ke terminal Linux di dalam container.
3. Container image `dtcenter/wps_wrf` sudah menyediakan toolchain terkompilasi secara *default*, tanpa perlu instalasi tambahan. Begitu masuk (`pwd`), Anda akan berada di direktori kerja:
   ```bash
   comsoftware/wrf/
   ```
   yang di dalamnya sudah tersedia folder `WPS`, `WPS-4.3`, dan `WRF-4.3` siap pakai.

**Alternatif via terminal (tanpa GUI Docker Desktop):**

```bash
# 1) Lihat daftar container yang sedang berjalan, catat CONTAINER ID-nya
docker ps

# 2) Masuk ke shell container tersebut
docker exec -it <id-container> /bin/bash
```

> 💡 Karena tidak ada *mounting*, memasukkan/mengeluarkan file (data ERA5, `WPS_GEOG`, hasil `wrfout`) dari/ke container dilakukan dengan **`docker cp`** — lihat Bagian 3.3.

> ⚠️ **Catatan penting:** Tanpa *mounting*, data yang ada di dalam container **akan hilang jika container dihapus** (`docker rm`). Selama container hanya dihentikan (`docker stop`) lalu dijalankan lagi (`docker start`), data di dalamnya tetap aman.

---

## 2. 🗺️ Konfigurasi Domain (WRF Domain Wizard)

Penentuan wilayah kajian (*bounding box*) dilakukan secara konseptual menggunakan **WRF Domain Wizard**, dengan menempatkan pusat domain (*center lat/lon*) kira-kira di sekitar perbatasan Jawa Tengah–DIY bagian selatan agar lintasan Siklon Cempaka dan wilayah terdampak banjir bandang tercakup penuh pada domain resolusi tertinggi.

### 2.1 Skema Nesting 3 Domain

Digunakan skema **triple nesting (1-way/2-way)** dengan rasio pembesaran grid standar **1:3**, mengikuti kaidah CFL (Courant–Friedrichs–Lewy) agar interaksi antar-domain tetap stabil secara numerik:

| Domain | Peran | Resolusi Grid | Cakupan Wilayah | Rasio Terhadap Induk |
|---|---|---|---|---|
| **D01** | Skala Makro/Regional | **27 km** | Regional Jawa – Samudra Hindia selatan (menangkap sirkulasi siklon skala penuh) | — (*mother domain*) |
| **D02** | Skala Provinsi | **9 km** | Pulau Jawa (fokus Jawa Tengah & DIY) | 1:3 dari D01 |
| **D03** | Skala Mikro (resolusi tinggi) | **3 km** | Pesisir selatan Jateng–DIY: Cilacap, Kebumen, Purworejo, DIY | 1:3 dari D02 |

> ⚙️ **Catatan teknis:** Rasio 27 km → 9 km → 3 km dipilih agar `time_step` model (umumnya `6× resolusi_D01_dalam_km` sebagai estimasi awal, atau ±150–162 detik untuk D01 27 km) tetap stabil, sekaligus D03 3 km sudah masuk kategori *convection-permitting scale* sehingga skema kumulus pada domain terdalam dapat dinonaktifkan (`cu_physics = 0` untuk D03) demi merepresentasikan konveksi secara eksplisit — krusial untuk menangkap sabuk hujan siklon secara realistis.
<img width="616" height="322" alt="Screenshot 2026-07-06 172031" src="https://github.com/user-attachments/assets/3a4e22cd-c827-4660-b886-d7c70e2072b4" />


---

## 3. 💾 Persiapan Data Input (di Luar Container)

Tahap ini idealnya dilakukan **di host/direktori lokal** (di luar container) agar proses unduh besar tidak terikat siklus hidup container.

### 3.1 Data Statis Bumi — WPS_GEOG

```bash
# Unduh dataset WPS_GEOG (Complete) dari server resmi NCAR
wget -c https://www2.mmm.ucar.edu/wrf/src/wps_files/geog_complete.tar.gz

# Ekstrak ke direktori khusus data statis
mkdir -p /path/lokal/anda/WPS_GEOG
tar -xzvf geog_complete.tar.gz -C /path/lokal/anda/WPS_GEOG --strip-components=1
```

> 📌 Ekstraksi dilakukan di **host** (di luar container). Path hasil ekstraksi ini akan dipindahkan ke dalam container lewat `docker cp` (lihat 3.3), lalu dirujuk pada variabel `geog_data_path` di `&geogrid` (lihat bagian 4.1).

### 3.2 Data Sinoptik/Gangguan Cuaca — ERA5 Reanalysis

Data ERA5 diunduh dari **Copernicus Climate Data Store (CDS)** mencakup dua jenis data untuk rentang **26–29 November 2017**:

| Jenis Data | Level | Fungsi |
|---|---|---|
| **Pressure Levels** | Multi-level (1000–1 hPa) | Variabel 3D: geopotential, temperature, u/v wind, relative humidity |
| **Single/Surface Levels** | Permukaan | Variabel 2D: SST, soil temperature/moisture, mslp, 2m temp, 10m wind, snow cover |

WEBSITE CLIMATE DATA STORE : https://cds.climate.copernicus.eu/datasets

<img width="386" height="122" alt="Screenshot 2026-07-06 172443" src="https://github.com/user-attachments/assets/0b57c46a-5752-4d8a-8705-4c98d3b45951" />


### 3.3 Memindahkan Data ke Dalam Container (`docker cp`)

Karena container dijalankan tanpa *mounting*, semua data hasil unduhan di host dipindahkan ke dalam container menggunakan `docker cp` (dijalankan dari terminal **host**, bukan dari dalam container):

```bash
# Catat dulu CONTAINER ID / nama container yang berjalan
docker ps

# Salin WPS_GEOG ke dalam container
docker cp /path/lokal/anda/WPS_GEOG/ <id-container>:comsoftware/wrf/WPS_GEOG

# Salin data ERA5 (GRIB) ke dalam container
docker cp /path/lokal/anda/data_era5/ <id-container>:comsoftware/wrf/WPS-4.3/data_era5
```

> 🔁 Untuk mengeluarkan hasil akhir (`wrfout_d0*`) dari container ke host setelah simulasi selesai, gunakan arah sebaliknya:
> ```bash
> docker cp <id-container>:comsoftware/wrf/WRF-4.3/test/em_real/wrfout_d01_2017-11-26_00:00:00 /path/lokal/anda/output/
> ```

---

## 4. ⚙️ Fase WPS-4.3 (WRF Preprocessing System)

Semua langkah berikut dijalankan **di dalam container** (via tab *Exec* Docker Desktop atau `docker exec -it`), pada direktori `WPS-4.3`.

```bash
cd comsoftware/wrf/WPS-4.3
```

### 4.1 Konfigurasi `namelist.wps`

```fortran
&share
 wrf_core = 'ARW',
 max_dom = 3,
 start_date = '2017-11-26_00:00:00','2017-11-26_00:00:00','2017-11-26_00:00:00',
 end_date   = '2017-11-29_23:00:00','2017-11-29_23:00:00','2017-11-29_23:00:00',
 interval_seconds = 3600,
/

&geogrid
 parent_id         = 1,   1,   2,
 parent_grid_ratio = 1,   3,   3,
 i_parent_start    = 1,   30,  35,
 j_parent_start    = 1,   25,  30,
 e_we              = 150, 220, 232,
 e_sn              = 130, 214, 214,
 geog_data_res     = 'default','default','default',
 dx = 27000, dy = 27000,
 map_proj = 'mercator',
 ref_lat  = -8.5,
 ref_lon  = 109.5,
 truelat1 = -8.5,
 stand_lon = 109.5,
 geog_data_path = 'comsoftware/wrf/WPS_GEOG'
/

&ungrib
 out_format = 'WPS',
 prefix = 'ERA5'
/

&metgrid
 fg_name = 'ERA5',
 io_form_metgrid = 2,
/
```

> ⚠️ Sesuaikan `e_we`, `e_sn`, `i_parent_start`, `j_parent_start` dengan hasil final dari **WRF Domain Wizard** pada Bab 2 — nilai di atas adalah contoh ilustratif.

### 4.2 Menjalankan Geogrid

Membentuk grid geografis statis (`geo_em.d0*.nc`) untuk tiap domain:

```bash
./geogrid.exe >& log.geogrid
tail -n 5 log.geogrid   # pastikan berakhir dengan "Successful completion"
ls geo_em.d0*.nc
```

### 4.3 Menjalankan Ungrib

**a) Linking Vtable ERA5:**

```bash
ln -sf ungrib/Variable_Tables/Vtable.ERA-interim.pl Vtable
# Alternatif (jika tersedia Vtable khusus ERA5 pada instalasi Anda):
# ln -sf ungrib/Variable_Tables/Vtable.ERA5 Vtable
```

**b) Linking file GRIB ERA5 menggunakan `link_grib.csh`:**

```bash
./link_grib.csh data_era5/ERA5_PL_cempaka_20171126_29.grib
./link_grib.csh data_era5/ERA5_SFC_cempaka_20171126_29.grib
```

**c) Eksekusi Ungrib:**

```bash
./ungrib.exe >& log.ungrib
tail -n 5 log.ungrib
ls FILE:2017-11-*   # file intermediate hasil ungrib
```

### 4.4 Menjalankan Metgrid

Mengawinkan data statis geografis (`geo_em.d0*`) dengan data reanalysis (`FILE:*`):

```bash
./metgrid.exe >& log.metgrid
tail -n 5 log.metgrid
ls met_em.d0*.nc
```

---

## 5. 🌀 Fase WRF-4.3 (Core Model Running)

Pindah ke direktori kerja utama model:

```bash
cd comsoftware/wrf/WRF-4.3/test/em_real
```

### 5.1 Soft-Linking Hasil Metgrid

```bash
ln -sf ../../../WPS-4.3/met_em.d0* .
ls met_em.d0*
```

### 5.2 Konfigurasi `namelist.input`

Parameter waktu krusial:

```fortran
&time_control
 run_days                = 3,
 run_hours               = 23,
 run_minutes             = 0,
 run_seconds             = 0,
 start_date              = '2017-11-26_00:00:00','2017-11-26_00:00:00','2017-11-26_00:00:00',
 end_date                = '2017-11-29_23:00:00','2017-11-29_23:00:00','2017-11-29_23:00:00',
 interval_seconds        = 3600,
 input_from_file         = .true.,.true.,.true.,
 history_interval        = 60,  60,  60,
 auxinput4_inname        = 'wrflowinp_d<domain>',
 auxinput4_interval      = 360, 360, 360,
 io_form_auxinput4       = 2,
/
```

### 5.3 ⭐ Konfigurasi Standard SST (Sea Surface Temperature)

Karena Cempaka tumbuh dan menguat di atas Samudra Hindia, **update SST harian sangat krusial** agar fluks energi laut-atmosfer terepresentasi akurat dan intensitas siklon tidak *underestimate*. Aktifkan di bagian `&physics`:

```fortran
&physics
 sst_update = 1,
/
```

Serta pastikan konfigurasi `auxinput4` (lihat bagian 5.2) sudah menunjuk pada file `wrflowinp_d<domain>` yang memuat variasi SST harian tersebut.

> 🌊 **Kenapa penting?** Tanpa `sst_update = 1`, WRF akan menggunakan SST tunggal dari kondisi awal (`wrfinput`) sepanjang simulasi, padahal siklon berlangsung 4 hari penuh — SST statis dapat menyebabkan simulasi **melesetkan intensitas dan lintasan siklon**.

### 5.4 Eksekusi Real Model

`real.exe` memeriksa konsistensi vertikal & horizontal data, lalu menghasilkan kondisi awal dan batas domain:

```bash
./real.exe >& log.real
tail -n 5 log.real
ls wrfbdy_d01 wrfinput_d0* wrflowinp_d0*   # wrflowinp muncul jika sst_update=1
```

### 5.5 Eksekusi WRF Core

**Single-core:**

```bash
./wrf.exe >& log.wrf
```

**Multi-core (disarankan untuk domain 3 nest):**

```bash
mpirun -np 8 ./wrf.exe >& log.wrf
```

Output akhir yang dihasilkan:

```bash
ls wrfout_d01* wrfout_d02* wrfout_d03*
```

---

## 6. 🛠️ Troubleshooting & Tips Efisiensi

### 6.1 Pengecekan Log Saat Crash

| File Log | Kapan Diperiksa | Perintah Cepat |
|---|---|---|
| `rsl.error.0000` | Error utama (fatal error, segfault, CFL violation) | `tail -n 50 rsl.error.0000` |
| `rsl.out.0000` | Progres normal tiap timestep, indikasi macet di mana | `tail -n 50 rsl.out.0000` |
| Semua rank MPI | Crash spesifik satu proses saja | `grep -i "error\|fatal" rsl.error.*` |

> 🔎 Kata kunci umum yang menandakan masalah: `CFL`, `NaN`, `SIGSEGV`, `d01/d02/d03 domain not fully covered`. Jika muncul CFL violation berulang, pertimbangkan memperkecil `time_step` pada `&domains`.

### 6.2 Manajemen Storage di Docker Desktop

Output `wrfout` dan `wrfrst` berukuran sangat besar dan menumpuk di dalam *disk image* Docker Desktop (`Docker Desktop Data.vhdx` / *disk image* WSL2 backend Docker Desktop). Langkah pembersihan aman:

**Langkah 1 — Setelah `wrfout` selesai diambil keluar via `docker cp` (lihat 3.3) dan divisualisasikan, hapus file besar di dalam container:**

```bash
rm -f wrfout_d0*_2017-11-2[6-9]* wrfrst_d0*
```

**Langkah 2 — Bersihkan sisa layer/volume Docker yang tidak terpakai dari sisi host (jalankan di terminal host, bukan di dalam container):**

```bash
docker system prune -a --volumes
```

> 💽 Perintah ini membersihkan *image*, *container* berhenti, dan *volume* menganggur yang menumpuk selama proses trial-and-error simulasi, sehingga ruang disk yang dialokasikan Docker Desktop tidak terus membengkak.
>
> ⚠️ Pastikan container `wrf_cempaka_2017` masih Anda perlukan sebelum menjalankan perintah ini — `prune` akan menghapus container yang sedang **tidak berjalan** beserta datanya (yang belum di-`docker cp` keluar).

---

## 7. 📂 Struktur Direktori Proyek

**Di host** (sebelum dipindahkan dengan `docker cp`, lihat 3.3):

```
WRF_CEMPAKA_2017/                   # Folder proyek di laptop/PC Anda
├── WPS_GEOG/                       # Hasil ekstraksi data statis geografis
├── data_era5/
│   ├── ERA5_PL_cempaka_20171126_29.grib
│   └── ERA5_SFC_cempaka_20171126_29.grib
├── output/                         # Tujuan `docker cp` untuk hasil wrfout
├── postprocessing/                 # Script visualisasi (Python/NCL/GrADS)
└── README.md
```

**Di dalam container** (`dtcenter/wps_wrf`, setelah `docker cp`):

```
comsoftware/wrf/
├── WPS_GEOG/                       # <- disalin dari host
├── WPS-4.3/
│   ├── data_era5/                  # <- disalin dari host
│   ├── namelist.wps
│   ├── geo_em.d0*.nc
│   ├── FILE:2017-11-*
│   └── met_em.d0*.nc
└── WRF-4.3/test/em_real/
    ├── namelist.input
    ├── wrfbdy_d01
    ├── wrfinput_d0*
    ├── wrflowinp_d0*
    └── wrfout_d0*                  # <- diambil keluar via docker cp
```

---

## 8. 📚 Referensi

- Skamarock, W. C., et al. (2021). *A Description of the Advanced Research WRF Model Version 4.3*. NCAR Technical Note.
- ECMWF – Copernicus Climate Change Service. *ERA5 Reanalysis Documentation*.
- BMKG. *Laporan Kejadian Siklon Tropis Cempaka, November 2017*.
- DTCenter. *WPS/WRF Docker Container Documentation* — [github.com/NCAR/container-wrf-wps](https://github.com/NCAR/container-wrf-wps)

---

**📌 Catatan Akhir:** Repositori ini disusun untuk keperluan riset/akademik terkait rekonstruksi numerik Siklon Tropis Cempaka. Silakan sesuaikan parameter domain, `time_step`, dan skema fisis (mikrofisika, kumulus, PBL) sesuai kebutuhan validasi terhadap observasi curah hujan aktual (misalnya data GSMaP/TRMM atau pos hujan BMKG setempat).
