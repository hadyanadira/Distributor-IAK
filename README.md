PT IKAN TERBANG MAKMUR SEJAHTERA Tbk — Distributor & SCM Integration (Flask + Firebase)

Berikut penjelasan lengkap setiap bagian dalam bentuk paragraf, menggunakan bahasa baku yang akrab bagi sesama pengembang.

**Ringkasan Fitur**
Aplikasi ini adalah layanan Distributor berbasis Flask yang menyatukan antarmuka web publik, Admin Dashboard, serta serangkaian API untuk kebutuhan rantai pasok. Fitur intinya meliputi perhitungan biaya kirim berdasarkan tabel rute, pembuatan pengiriman single-item maupun multi-item, pelacakan status resi, serta penyajian daftar pengiriman aktif dan histori. Di sisi integrasi, sistem menyediakan mekanisme webhook bertanda tangan HMAC dengan kebijakan retry dan dead-letter queue agar notifikasi perubahan status tetap andal. Aplikasi juga mampu melakukan siaran langsung (direct broadcast) ke endpoint milik pihak retail menggunakan pemetaan id_retail. Seluruh data operasional disimpan pada Google Firestore, sedangkan login admin untuk keperluan demonstrasi memanfaatkan file Excel yang dibaca saat proses autentikasi sesi.

**Arsitektur**
Secara arsitektural, proyek ini terdiri atas satu proses Flask yang melayani web dan API pada port yang sama, modul autentikasi berbasis session cookie, lapisan akses data ke Firestore, serta komponen integrasi keluar berupa pengirim webhook dan broadcaster langsung. Alur bisnis utama dimulai dari permintaan klien (web atau API), validasi input, pengambilan atau pembaruan dokumen di koleksi Firestore, lalu penyusunan respons JSON atau render templat Jinja untuk web. Saat status pengiriman berubah di Admin Dashboard, sistem membangkitkan payload event, menandatangani dengan HMAC-SHA256, mencoba mengirim ke subscriber terdaftar dengan backoff bertahap, dan bila tetap gagal akan menyimpan catatan ke dead-letter queue untuk ditangani kemudian.

**Struktur Direktori**
Struktur direktori dirancang sederhana agar mudah dipahami: berkas utama app.py berisi inisialisasi aplikasi, definisi rute web dan API, serta helper integrasi; direktori templates/ memuat antarmuka index.html untuk pelacakan resi, admin.html untuk dashboard, dan login.html untuk autentikasi; direktori static/ (opsional) menampung aset; file kredensial service account Firestore (mis. DistributorD.json) ditempatkan di akar proyek; dan berkas auth.xlsx berfungsi sebagai sumber akun admin contoh untuk kebutuhan uji coba.

**Prasyarat & Instalasi**
Lingkungan standar yang diperlukan ialah Python 3.10+ beserta pustaka Flask, firebase-admin untuk Firestore, pandas dan openpyxl untuk membaca Excel, serta requests untuk HTTP keluar; pada Windows disarankan menambah waitress sebagai WSGI server yang lebih stabil untuk akses publik. Proses instalasi dilakukan dengan membuat virtual environment, mengaktifkannya, lalu memasang dependensi melalui pip. Setelah itu, letakkan berkas service account Google (JSON) pada lokasi yang dikonfigurasi, pastikan proyek Firestore aktif, dan siapkan file Excel kredensial admin jika belum tersedia.

**Konfigurasi Lingkungan**
Konfigurasi minimal meliputi kunci rahasia Flask untuk sesi (FLASK_SECRET_KEY), port eksekusi (PORT), jalur berkas service account Firestore, dan ID proyek GCP. Selain itu, aplikasi mengandalkan beberapa konstanta operasional seperti tabel rute lokal sebagai cadangan, serta peta id_retail → endpoint untuk keperluan broadcast langsung. Pada tahap produksi, disarankan menaruh seluruh nilai ini dalam variabel lingkungan agar aman dan mudah dikelola.

**Menjalankan Aplikasi**
Aplikasi dapat dijalankan langsung dengan python app.py untuk mode pengembangan atau melalui waitress-serve pada Windows untuk beban yang lebih stabil. Setelah server aktif, web publik dapat diakses pada root path dan endpoint kesehatan (/health) dapat digunakan untuk memastikan konfigurasi dasar sudah benar. Jika perlu diakses dari internet untuk demonstrasi, jalur akses lokal tersebut dapat ditunnel menggunakan ngrok, sehingga domain ngrok yang sama melayani halaman web dan seluruh endpoint API.

**Model Data & Koleksi Firestore**
Skema data terpusat pada beberapa koleksi: tb_quote menyimpan hasil kuotasi biaya kirim beserta rincian rute, faktor tarif, dan estimasi kedatangan; tb_pengiriman menyimpan dokumen pengiriman aktif dengan nomor resi, status, rincian asal-tujuan, daftar item, total kuantitas, biaya, serta cap waktu; tb_histori menampung arsip pengiriman yang telah selesai sehingga tidak mengotori daftar aktif; routes menyimpan master rute dan parameter tarif; webhook_subscribers menyimpan pendaftaran subscriber beserta rahasia tanda tangan; dan webhook_deadletter merekam kegagalan pengiriman event untuk ditindaklanjuti.

**Aturan Tarif & ETA**
Perhitungan biaya dimulai dengan mencari rute pada koleksi routes; jika tidak ditemukan, sistem menggunakan tabel rute fallback di memori. Setiap rute memiliki price_base yang sudah mencakup included_kg, serta per_kg_factor yang diterapkan pada selisih kuantitas di atas batas inklusif; formulanya secara ringkas adalah total = price_base + max(qty - included_kg, 0) * per_kg_factor * price_base dengan pembulatan ke nilai rupiah wajar. Perkiraan waktu tempuh (`eta_days`) kemudian dikonversi menjadi teks yang mudah dibaca dan diturunkan tanggal kedatangannya (eta_delivery_date) berdasarkan tanggal saat kuotasi dibuat.

**API Reference**
Antarmuka pemrograman aplikasi menyediakan operasi inti untuk menghitung biaya (POST /api/biaya), membuat pengiriman multi-item (POST /api/pengiriman), membuat pengiriman single-item (varian legacy di POST /shipments), mengecek status resi (GET /status), dan memperoleh daftar pengiriman aktif beserta histori (GET /api/shipments). Seluruh endpoint menerapkan validasi input yang memadai, mengembalikan kode status HTTP yang sesuai (200/201 untuk berhasil, 4xx untuk kesalahan input atau entitas tidak ditemukan), serta merapikan bentuk data agar konsisten bagi konsumen API.

**Admin Dashboard**
Dashboard admin disediakan untuk memantau dan mengelola operasional pengiriman secara visual. Halaman ini menampilkan daftar pengiriman aktif dan histori, menyediakan form untuk memperbarui status yang secara otomatis membangkitkan event integrasi, dan menghadirkan modul pengelolaan rute yang memungkinkan mengubah price_base, eta_days, per_kg_factor, dan included_kg. Untuk demonstrasi, aksesnya dilindungi dengan login berbasis Excel dan status autentikasi disimpan pada sesi sehingga rute yang bersifat administratif tidak dapat diakses publik.

**Keamanan & Validasi**
Keamanan dasar mencakup penggunaan session cookie dengan kunci rahasia, pembersihan sesi saat logout, serta pembatasan akses rute administratif menggunakan dekorator pengecekan sesi. Di sisi integrasi, tiap payload webhook ditandatangani dengan HMAC-SHA256 menggunakan rahasia milik subscriber, sehingga penerima dapat memverifikasi keaslian pesan; pengiriman event juga dilengkapi strategi retry dengan jeda meningkat dan pencatatan ke dead-letter queue jika semua percobaan gagal. Validasi input dilakukan pada setiap endpoint untuk memastikan kelengkapan dan tipe data, sementara rekomendasi produksi meliputi penggunaan hash untuk kata sandi, CSRF untuk form, serta pembatasan laju untuk mencegah brute force.

**Alur Uji Cepat (cURL)**
Pengujian cepat dapat dilakukan secara berurutan: pertama hitung biaya dengan mengirim JSON berisi asal, tujuan, dan kuantitas ke endpoint kuotasi lalu pastikan respons memuat total biaya dan estimasi kedatangan; berikutnya buat pengiriman multi-item dengan menyuplai id_order, id_retail, titik asal dan tujuan, serta daftar barang agar sistem membangkitkan nomor resi; setelah pengiriman tercatat, verifikasikan statusnya melalui endpoint pelacakan resi dengan menyertakan no_resi; untuk observabilitas harian gunakan endpoint yang mengembalikan daftar pengiriman aktif dan histori; terakhir, jika menguji integrasi keluar, siapkan URL dummy dan lakukan percobaan siaran agar alur event dapat ditelusuri dari log dan koleksi dead-letter saat diperlukan.

**Keterbatasan Dikenal**
Login admin berbasis Excel dimaksudkan murni untuk kepentingan akademik sehingga tidak memenuhi standar produksi karena menyimpan kredensial dalam bentuk terbaca; di sistem nyata, mekanisme ini perlu diganti dengan penyimpanan terproteksi dan verifikasi berbasis hash. Selain itu, pemetaan endpoint retail masih dikelola sebagai konfigurasi aplikasi sehingga perubahan target memerlukan pembaruan konfigurasi. Tabel rute lokal hanya bersifat cadangan; apabila rute tidak tersedia di Firestore maupun fallback, permintaan terkait tarif atau pembuatan pengiriman akan ditolak.

**Deployment Singkat (Opsional)**
Untuk demonstrasi, aplikasi cukup dijalankan pada port lokal lalu diekspos melalui ngrok sehingga web dan API tersedia pada satu domain publik sementara; langkah ini mempermudah validasi antarmuka dan uji integrasi webhook tanpa menyiapkan infrastruktur penuh. Pada lingkungan yang lebih permanen, jalankan aplikasi menggunakan WSGI server seperti Waitress atau Gunicorn, kelola variabel lingkungan secara aman, dan pastikan kredensial Firestore tersedia di host yang menjalankan layanan.

**FAQ**
Pertanyaan yang sering muncul antara lain bagaimana menambah rute baru dan bagaimana memvalidasi webhook. Penambahan rute dilakukan dari Admin Dashboard yang akan menulis atau memperbarui dokumen pada koleksi routes, kemudian semua perhitungan biaya otomatis mengikuti nilai terbaru. Untuk validasi webhook, penerima cukup menghitung HMAC-SHA256 atas payload yang diterima menggunakan rahasia yang sama dan membandingkannya dengan nilai tanda tangan pada header; bila verifikasi gagal, permintaan dapat ditolak dan ID event dicatat untuk audit.

**Lisensi**
Proyek ini disediakan untuk kebutuhan pembelajaran dan demonstrasi integrasi Supply Chain Management. Jenis lisensi mengikuti pilihan pemilik repositori; apabila hendak dipakai di luar konteks akademik, sertakan berkas lisensi resmi (misalnya MIT atau Apache-2.0) di akar proyek agar hak dan kewajiban penggunaan terdokumentasi dengan jelas.

**Ringkasan Fitur API Tarif & Pengiriman**
Layanan tarif dan pengiriman membentuk inti API dengan kontrak yang konsisten, validasi ketat, dan format respons yang mudah dikonsumsi aplikasi pihak ketiga. Seluruh operasi dirancang idempotent sejauh memungkinkan, memanfaatkan nomor resi dan ID dokumen sebagai kunci referensi, dan menyimpan hasilnya ke Firestore agar setiap perubahan dapat dilacak.

**Hitung biaya ongkir (POST /api/biaya)**
Endpoint ini menerima asal, tujuan, dan kuantitas (dalam kilogram atau satuan yang ditetapkan), kemudian mencari rute yang sesuai, menghitung biaya menurut aturan tarif, menghasilkan ID kuotasi, menyimpan hasilnya ke koleksi tb_quote, dan mengembalikan nilai total biaya beserta estimasi hari serta tanggal kedatangan. Jika rute tidak tersedia atau input tidak valid, layanan mengembalikan pesan kesalahan yang informatif dengan kode status HTTP 4xx.

**Buat pengiriman multi-item (POST /api/pengiriman)**
Endpoint ini dipakai untuk membuat satu pengiriman yang berisi lebih dari satu item. Klien harus menyediakan id_order, id_retail, informasi asal dan tujuan, serta daftar barang_dipesan berisi ID, nama, dan kuantitas setiap item. Layanan menghitung total kuantitas, memperoleh tarif, membangkitkan nomor resi unik, menyimpan dokumen ke tb_pengiriman, dan menyiapkan status awal sehingga pengiriman siap dilacak dan diperbarui.

**Cek status resi (GET /status)**
Endpoint ini menerima parameter no_resi dan mengembalikan status terkini beserta informasi rute dan estimasi kedatangan. Pencarian dilakukan terlebih dahulu di koleksi pengiriman aktif dan, bila tidak ditemukan, dilanjutkan ke arsip tb_histori. Bila nomor resi tidak valid atau tidak ada, layanan memberi respons 404 dengan pesan yang jelas.

**Daftar pengiriman aktif & histori (GET /api/shipments)**
Endpoint ini menyajikan ringkasan dua himpunan data sekaligus: pengiriman yang masih aktif dan pengiriman yang telah dipindahkan ke histori. Respons telah dinormalisasi sehingga bidang penting seperti rute, biaya, status, eta, dan cap waktu tersedia dalam bentuk yang konsisten untuk kebutuhan dashboard atau analitik.

**Reliabilitas Integrasi. Webhook HMAC-SHA256 + exponential backoff (3x) + Dead Letter Queue**
Setiap perubahan status memicu pembuatan event yang ditandatangani menggunakan HMAC-SHA256 dengan rahasia spesifik subscriber dan dikirim ke URL yang terdaftar; bila respons tidak berhasil, pengirim akan mencoba kembali hingga tiga kali dengan jeda meningkat untuk menghindari banjir permintaan. Jika seluruh percobaan gagal, event dicatat ke koleksi dead-letter agar dapat dianalisis dan dikirim ulang secara manual, memastikan tidak ada informasi penting yang hilang tanpa jejak.

**Direct broadcast ke Retail Endpoint berdasarkan id_retail**
Selain mekanisme webhook berbasis pendaftaran, sistem melakukan siaran langsung ke endpoint yang dipetakan dari id_retail. Strategi ini memudahkan integrasi cepat dengan sistem retail yang belum mengadopsi skema berlangganan webhook; payload ringkas berisi nomor resi, identitas dokumen, status lama dan baru, serta informasi rute dikirim segera setelah pembaruan status berhasil.

**Admin Dashboard. Login berbasis Excel (auth.xlsx) untuk demonstrasi**
Akses ke dashboard dilindungi login sederhana yang membaca kredensial dari auth.xlsx, kemudian menyimpan penanda is_admin pada sesi saat autentikasi berhasil. Skema ini memudahkan uji fungsional tanpa menyiapkan direktori pengguna atau basis data, namun tidak ditujukan untuk produksi karena tidak memanfaatkan hash dan kontrol akses granular.

**Ubah status pesanan (memicu pengiriman event)**
Perubahan status dilakukan melalui form di dashboard. Setelah valid, sistem memperbarui dokumen pengiriman, mencatat waktu perubahan, dan segera membangkitkan event untuk dikirim ke subscriber webhook serta endpoint retail yang relevan. Jika status akhir (misalnya “Pesanan Selesai”) dipilih, dokumen otomatis dipindahkan ke koleksi histori agar daftar aktif tetap bersih.

**CRUD rute (basis tarif, ETA, faktor kg, inklusif kg) Analytics: grafik, status distribution, top routes.**
Dashboard menyediakan antarmuka untuk menambah, memperbarui, atau menghapus rute pada koleksi routes, termasuk pengaturan price_base, eta_days, per_kg_factor, dan included_kg. Selain fungsi editorial, data pada koleksi-koleksi tersebut dapat digunakan sebagai dasar visualisasi sederhana seperti persebaran status dan rute teratas, sehingga operator memperoleh gambaran ringkas performa pengiriman.

**Frontend Publik Landing + form pelacakan resi (/)**
Halaman publik pada root path menyajikan formulir pelacakan yang memungkinkan pengguna akhir atau pihak retail memasukkan nomor resi dan mendapatkan status terkini beserta informasi rute dan estimasi kedatangan. Implementasi ini memanfaatkan templat Jinja agar tampil konsisten dan ringan tanpa ketergantungan framework front-end tersendiri.

**Konektivitas Data Firestore collections: tb_pengiriman, tb_histori, tb_quote, routes, webhook_subscribers, webhook_deadletter.**
Seluruh data dipusatkan di Firestore dengan pembagian tanggung jawab yang tegas: tb_quote untuk kuotasi biaya, tb_pengiriman untuk entitas aktif, tb_histori untuk arsip, routes sebagai master tarif dan ETA, webhook_subscribers sebagai pendaftaran integrasi keluar, serta webhook_deadletter sebagai catatan kegagalan pengiriman event. Pola ini memudahkan audit dan pemisahan beban kerja.

**Catatan akademik: Mekanisme login admin via Excel adalah demonstrasi, tidak untuk produksi.**
Skema autentikasi menggunakan Excel sengaja dipertahankan untuk memudahkan penilaian dan replikasi tugas perkuliahan tanpa infrastruktur tambahan; pada sistem nyata, mekanisme ini harus digantikan oleh penyimpanan kredensial aman, verifikasi berbasis hash, kontrol peran, dan proteksi CSRF pada form, serta audit trail yang memadai.
