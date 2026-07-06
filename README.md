# WRF Simulation Setup with Docker

Repositori ini berisi *write-up* dan konfigurasi untuk menjalankan simulasi Weather Research and Forecasting (WRF) menggunakan environment Docker. Dokumentasi ini mencakup langkah-langkah dari penentuan domain (WPS) hingga eksekusi `wrf.exe`.

## 📌 Prerequisites

Sebelum memulai, pastikan sistem sudah memenuhi persyaratan berikut:
* **OS/Environment:** [Misal: Ubuntu 22.04 / WSL2 Windows]
* **Docker & Docker Compose:** Terinstall dan berjalan dengan baik.
* **Hardware:** Minimal RAM [X] GB dan CPU [X] Core.
* **WRF Docker Image:** `[nama-image-docker-yang-dipakai, misal: jbeezley/wrf]`

## 📁 Struktur Direktori

Pastikan struktur direktori diatur seperti ini agar *mounting* ke Docker lebih mudah:

```text
.
├── docker-compose.yml
├── data/
│   ├── grib/               # Tempat menaruh data input meteorologi (GFS/FNL)
│   └── geog/               # Data geografis statis
├── config/                 # (atau folder 'powy' jika kamu menggunakan nama khusus)
│   ├── namelist.wps        # Konfigurasi WPS
│   └── namelist.input      # Konfigurasi WRF
└── output/                 # Tempat hasil wrfout disimpan (di-ignore oleh git)
