# DitLux — Arsitektur

**Versi bahasa mudah.** DitLux adalah aplikasi Windows yang membuat frame tambahan secara buatan,
sehingga game yang jalan di 30 fps terlihat seperti 60 fps. Cara kerjanya: ambil gambar yang sedang
ditampilkan game, lalu **menebak apa yang terjadi di antara dua gambar itu**, dan selipkan hasilnya
di antaranya.

Status: dokumen desain. **Belum ada satu baris kode pun.**
Ditulis ulang pada 2026-10-13 karena versi pertama memakai istilah tanpa menjelaskan artinya.

---

## 0. Cara membaca dokumen ini

Setiap istilah teknis di bawah ditulis satu kali di sini, dengan contoh dari Unity. Setelah itu,
di bagian berikutnya istilahnya langsung dipakai tanpa dijelaskan ulang.

| Istilah | Artinya sederhana | Padanan yang sudah kamu kenal |
|---|---|---|
| **frame** | potongan memori berisi angka warna semua titik layar | `RenderTexture` 1920×1200 |
| **pixel** | satu titik. Disimpan sebagai 4 angka: R, G, B, A | `Color32` |
| **RGBA8** | format frame: 8 bit per kanal = 4 byte per pixel | `TextureFormat.RGBA32` |
| **swapchain** | sepasang frame yang bergantian: satu digambar, satu dibaca layar | konsep balik-belik kertas |
| **Present** | panggilan "gambar saya sudah jadi, tukar sekarang" | akhir frame di Unity |
| **vsync** | aturan: tukar hanya boleh saat layar mulai baca baru. Mencegah sobekan | `QualitySettings.vSyncCount` |
| **DWM** | program Windows yang menggabungkan semua window jadi satu gambar desktop, lalu menampilkannya | kompositor |
| **tearing** | atas gambar baru, bawah gambar lama → ada garis sobekan | — |
| **windowed / borderless** | game menggambar ke window, DWM yang tayangkan ke layar | windowed & fullscreen tanpa border |
| **exclusive fullscreen** | game ambil alih layar sendiri, DWM disuruh minggir | mode "Fullscreen" lama |
| **capture** | mengambil salinan gambar game dari luar | OBS Display/Window Capture |
| **WGC** | API Windows: ambil gambar **satu window** | OBS Window Capture |
| **Duplication** | API Windows: ambil gambar **satu monitor** | OBS Display Capture |
| **QPC** | stopwatch Windows, resolusi mikrodetik, selalu maju | `Stopwatch` (bukan `DateTime`) |
| **ring buffer** | deretan slot tetap yang dipakai ulang, melingkar | array tetap + indeks geser |
| **nol alloc/frame** | loop tidak boleh membuat objek baru sama sekali | "jangan `new` di dalam `Update()`" |
| **compute shader** | program kecil yang jalan di GPU, dikerjakan serentak oleh jutaan thread | Unity Compute Shader + `Dispatch` |
| **flow map** | gambar khusus isinya **arah gerak**, bukan warna. Tiap titik bilang "aku geser sekian pixel" | — |
| **warp** | memindahkan pixel mengikuti arah gerak tadi | — |
| **interpolate** | hasilkan keadaan di waktu tengah antara dua keadaan yang diketahui | `Lerp(a, b, 0.5f)` |
| **ghosting** | bayangan ganda di tepi benda bergerak. Artefak utama frame generation | — |
| **occlusion / disocclusion** | benda tertutup di frame A, muncul di frame B → tidak ada data untuk menegakkan tengah-tengahnya | — |
| **linear vs sRGB** | dua cara menyimpan kecerahan. Campur dua frame di ruang yang salah = warnanya berubah | Project Settings → Color Space |
| **GPU-bound / CPU-bound** | mana yang jadi pengerat waktu: VGA atau prosesor | profiler Unity, waktu Rendering vs Scripts |
| **UMA** | iGPU memakai RAM utama sebagai memori grafis, dibagi dengan CPU | — |
| **LSFG** | Lossless Scaling Frame Generation. Produk yang jadi acuan DitLux | — |

---

## 1. Batas mesin, dan kenapa bentuknya jadi begini

| Batas | Nilai | Apa yang dipaksakan ke desain |
|---|---|---|
| GPU | Radeon 680M (iGPU, 12 CU, ~3.7 TFLOPS). **Satu-satunya GPU di laptop ini** | Tidak ada GPU cadangan. Pekerjaan DitLux memakai jatah yang sama dengan game |
| NPU | **Tidak ada** | Semua "kecerdasan" harus jalan di GPU. Jaringan neural besar tidak realistis |
| Layar | 1920×1200 @ **60 Hz** | Atap keras. 30→60 masuk akal; 30→90 fisikanya tidak mungkin |
| RAM | 16 GB LPDDR5, UMA, ~51 GB/s | Bandwidth bukan masalah utama (lihat §8). Yang jadi soal: waktu GPU |
| Resolusi | sudah native | **Upscaling dihapus dari scope.** DitLux = pembuatan frame saja |

Konsekuensi yang diterima sejak awal, bukan kejutan nanti:

- Papan kualitas atas jalur ini kira-kira **LSFG generasi 1**. Yang membuat LSFG 3.x bagus adalah
  model neural, dan itu tidak bisa realtime di mesin ini.
- **Input tidak ikut membaik.** Yang naik cuma kelancaran gambar (§7).
- Teks dan HUD bergerak → akan berbayang. Ini sifat metode, bukan bug (§9).

---

## 2. Pengetahuan dasar: bagaimana satu gambar bisa sampai ke layarmu

Ini fondasi seluruh dokumen. Baca bagian ini dulu kalau §3 dan §4 terasa tiba-tiba.

```
  GAME               menulis angka warna ke  ->   frame buffer miliknya
                                                |
                        (windowed/borderless)   |   (exclusive fullscreen)
                                v               |          v
                        DWM menggabungkan        |   game memegang monitor
                        semua window jadi        |   sendiri. Tidak ada
                        satu gambar desktop      |   tahap "penggabungan"
                                |                |   untuk direkam orang luar
                                v                |
                        layar membaca 60x/detik <-+
```

Tiga hal yang harus menempel:

1. **Game tidak pernah menggambar langsung ke layar.** Layar sibuk dibaca 60 kali per detik; kalau
   game menulis sambil dibaca, hasilnya sobekan. Maka selalu ada dua tempat: yang digambar, yang
   dibaca, bergantian tiap refresh.
2. **Di mode windowed/borderless, pihak yang menayangkan ke layar adalah Windows (DWM), bukan
   game.** Game hanya menyerahkan gambar ke window-nya. Karena gambar itu tersimpan di tempat yang
   bisa dibaca, aplikasi lain bisa mengambilnya → itulah "capture".
3. **Di exclusive fullscreen, game yang memegang monitor.** Tidak ada tahap penggabungan, jadi dari
   luar tidak ada yang bisa dibaca. Sekaligus: tidak ada tempat kosong untuk DitLux menayangkan
   hasilnya sendiri, karena monitornya sedang dipakai game.

Dari (3) muncul syarat yang tadinya kutulis sebagai satu kalimat pendek. Syaratnya **dua**, dan
sekarang sudah kupisahkan karena penyebabnya berbeda:

| Syarat | Kenapa | Bisa dihindari? |
|---|---|---|
| Game harus windowed/borderless | game memegang monitor di exclusive → DitLux tidak bisa ikut menayangkan | hanya dengan menyuntik diri ke dalam game (§3, opsi C) |
| DitLux tidak boleh merekam monitornya sendiri | kalau DitLux menayangkan di monitor yang sama sambil merekam monitor itu, frame berikutnya berisi hasil buatannya sendiri → berputar dan meleleh | ya: rekam **window game**, bukan monitor |

---

## 3. Mengambil gambar game — tiga cara, dan mana yang kita pilih

| Opsi | Cara kerja | Kelebihan | Kekurangan | Keputusan |
|---|---|---|---|---|
| **A. Rekam monitor** (Duplication / `IDXGIOutputDuplication`) | minta salinan hasil akhir satu monitor | universal; tidak menyentuh proses game | **melanggar syarat ke-2 di §2** saat DitLux ikut menayangkan di monitor itu. Juga ada batas: satu perekam per monitor | **fallback**, bukan utama |
| **B. Rekam satu window** (WGC / `Windows.Graphics.Capture`) | minta isi window tertentu lewat judul/`HWND`-nya | aman dari umpan-balik; hanya dapat gambar game; dapat tekstur GPU langsung, tidak lewat CPU; bisa minta kursor tidak diambil | butuh Windows 10 1803+; window harus sedang tampil (tidak di-minimize); gambar hasil gabungan DWM, jadi exclusive tetap tidak tertangkap | **PEILIH UTAMA** |
| **C. Menyuntik ke dalam game** (hook `Present` / `vkQueuePresentKHR`) | sisip DLL, dapatkan buffer game sebelum diserahkan ke OS | paling hemat (tanpa salin); satu-satunya yang bisa exclusive fullscreen; kamulah yang mengatur waktu tayang | harus menyuntik ke proses game → anti-cheat/DRM, dan wajib per-API (DX11/DX12/Vulkan) | **ditunda** — proyek lain, bukan pelajaran yang kamu mau sekarang |

**Koreksi atas dokumennya sendiri:** versi pertama dokumen ini menulis "Capture = DXGI Desktop
Duplication" sambil juga menuliskan "DitLux menayangkan di window fullscreen miliknya sendiri". Dua
hal itu saling membatalkan — see syarat ke-2 di §2. Sumber frame yang benar adalah **WGC per-window**.
Duplication tetap punya tempat, tapi untuk mode "rekam monitor kedua", bukan untuk setup utama.

---

## 4. Rangkaian utama (runtime)

Cara baca: kotak besar = satu tahap. Tanda `[M…]` = milestone tempat tahap itu mulai ada.
Garis putus-putus = jalur saat pengembangan, bukan jalur runtime.

```
   GAME (harus windowed / borderless -- alasannya di 2)
        |
        v
  +======================================================+
  | 1. AMBIL GAMBUK                          [M0]        |
  |    WGC: rekam window game, bukan monitor             |
  |    tiap frame diberi stempel waktu (QPC)             |
  |    salin ke RING BUFFER 4 slot, RGBA8                |
  |    semua slot dibuat sekali di awal: nol alloc       |
  +======================================================+
        |
        |  dua frame berurutan: A (lama) dan B (baru)
        v
  +======================================================+      +-------------------+
  | 2. CARA GERAKNYA                        [M2-M3]      |      | 3. POTONG ADEGAN  |
  |    perkecil dulu (pyramid) supaya murah              |      |    cut terdeteksi |
  |    gerak besar global: seluruh layar geser?          |----->|    -> jangan      |
  |    gerak lokal: blok-match -> FLOW MAP               |      |       campurkan   |
  |    rapikan flow (median, halus)                       |      +-------------------+
  +======================================================+               |
        | flow map + A + B                                               v
        v                                                          GOVERNOR
  +======================================================+
  | 4. BIKIN FRAME TENGAH                   [M4-M5]      |
  |    geser A maju, geser B mundur -> ketemu di tengah  |
  |    bagian yang bolong (bekas tertutup) -> diisi       |
  |    campur dua hasil itu                               |
  |    !! PENTING: campur di ruang warna LINEAR !!       |
  +======================================================+
        | frame hasil (G)
        v
  +======================================================+     JALUR PENGEMBANGAN
  | 5. YANG MENENTUKAN KAPAN                 [M1, M6]    |
  |    GOVERNOR: GENERATE / SKIP / ULANG / MATI          |     dua gambar yang
  |    lihat §6 sebagai diagram keputusan                |     sudah disimpan
  |    syarat: masih ada sisa waktu GPU?                 |     ke disk (PNG)
  +======================================================+     boleh masuk ke
        |                                                       tahap 2-4 lewat
        v                                                       jalur yang sama,
  +======================================================+     tapi TANPA urusan
  | 6. TAYANGKAN                             [M0, M1]     |     waktu Windows
  |    swapchain flip-model, window DitLux               |
  |    Present dipanggil tepat pada grid vsync 60 Hz     |
  +======================================================+
        |
        v
   LAYAR 1920x1200 @ 60 Hz

   Pelapis aplikasi:
     profil per game (rasio, tombol, mode) ----> masuk ke GOVERNOR
     telemetri (FPS + graf waktu frame) <------ datang dari QPC dan Present
```

---

## 5. Tahap demi tahap, dan apa yang kamu pelajari darinya

| # | Tahap | Tugasnya | Yang bikin kamu tambah pintar | Kalau salah, gejala yang kamu lihat |
|---|---|---|---|---|
| 1 | **Ambil gambar** (WGC) | dapat gambar game + jam-nya, simpan ke ring buffer | API Windows untuk window, batas umur gambar pinjaman, format warna | gambar basi; warna berubah; stutter periodik |
| 2 | **Ring buffer** | pegang beberapa frame sekaligus tanpa alloc baru | `new` di jalur cepat = GC = frame hilang | flicker acak tak reproducible (slot ditimpa saat dibaca) |
| 3 | **Gerak global** | satu geseran/besaran untuk seluruh layar | ruang koordinat & arah warp; kenapa **warp ≠ campur dua gambar** | kamera pan tetap berbayang |
| 4 | **Block-match flow** | arah gerak per petak kecil, ditulis ke flow map | compute shader: thread serentak, memori bersama, ukuran grup, format `R16G16_FLOAT` | objek melaju = bayangan; tekstur datar = flow acak |
| 5 | **Disocclusion** | temukan bagian yang tidak ada datanya, lalu isi | ini soal riset yang belum selesai; sumber artefak terbesar | lubang/molase di tepi benda yang baru muncul |
| 6 | **Potong adegan + rapikan** | hentikan interpolasi saat pindah shot; halirkan flow | konsistensi antar waktu; deteksi perubahan besar | dua shot meleleh jadi satu; tepi bergetar |
| 7 | **Campur** | gabungkan dua hasil jadi satu frame | **warna linear vs sRGB** (lihat §9) | warna di tepi gerak lebih gelap/geser |
| 8 | **Governor** | putuskan tiap slot: bikin, lewati, atau ulang | pengaturan waktu = bagian tersulit yang bukan algoritma | gambar bagus tapi patah-patah |
| 9 | **Tayangkan** | Present di grid vsync milik sendiri | vsync, flip model, kesegarisan refresh | tearing, atau 2 frame numpuk di slot yang sama |

Tahap 4 adalah tempat debug-view **flow map** (digambar jadi panah atau warna) pertama kali
memberimu pemahaman nyata. Itu alat utama yang akan kamu pakai sampai akhir proyek.

---

## 6. Keputusan tiap slot waktu (isi kotak 5)

```
setiap 16,7 ms (satu slot refresh)
        |
        v
  ukur kecepatan game sesungguhnya (rata-rata, naik-turun, tidak stabil)
        |
        v
  sisa waktu GPU masih ada?  --TIDAK-->  [LEWATI] tayangkan ulang frame terakhir
        | YA                          (game sedang GPU-bound: membuat frame
        v                             tambahan = mencuri FPS nyata)
  berapa fps sumbernya, dibandingkan atap 60 Hz?
        |
        +-- >= 60 fps -------------> [MATI] lempar lewat saja, tidak ada gunanya
        +-- ~45 fps (1,33x) ------> [SISIP 1 tiap 3 frame asli]
        +-- ~40 fps (1,5x)  ------> [SISIP 1 tiap 2 frame, pola berselang-seling]
        +-- ~30 fps (2x)    ------> [SISIP 1 tiap 1 frame -> paling bersih]
        |
        v
  pasangan A dan B sudah lengkap?  --BELUM-->  [LEWATI]
        |
        v
  adegan sedang berpindah / fade?   --YA-->  [LEWATI]
        |
        v
  [BUAT] ambil A+B -> tahap 2-4 -> tayangkan di slot berikutnya
```

Dua aturan yang tidak boleh dilanggar, dan ini pelajaran yang biasanya baru dibeli dengan
kesulitan:

1. **Jangan pernah menayangkan frame basi** sebagai pengganti frame yang belum selesai. Lebih baik
   ulangi frame terakhir — itu masih gambar yang benar.
2. **Rasio yang bukan bilangan bulat (1,5× dan 1,33×) adalah sumber patah yang paling sering.**
   Itu harus jadi kasus yang kamu uji dengan sengaja, bukan kejutan di tahap akhir.

---

## 7. Kenapa FPS terlihat naik tapi mainnya tetap terasa 30 fps

```
  game 30 fps.  F = frame asli.  G = frame buatan.  U = diulang.

  waktu (ms)   dari game      yang tampil di layar
  ---------    -----------    ----------------------------
      0,0      F 0     --->   F 0
     16,7                      U 0   (belum bisa apa-apa: F 1 belum datang)
     33,3      F 1     --->   F 1
     50,0                      G 0   (ini frame waktu 16,7, tapi baru bisa
                                      dihitung setelah F 1 datang di 33,3)
     66,7      F 2     --->   F 2
     83,3                      G 1
```

Sebabnya satu kalimat: **frame tengah butuh dua frame asli**, jadi ia tidak pernah bisa ada sebelum
frame kedua datang. Akibatnya jalur tampilan selalu mundur satu frame sumber.

| Yang | Naik? |
|---|---|
| Kelancaran yang kamu **lihat** | ya — 30 → 60 |
| Kecepatan simulasi game | tetap 30 |
| Seberapa cepat tombolmu diproses | tetap seperti 30 fps (+ sedikit tambahan) |

Untuk game sektarian/manajemen seperti ToI, tukar itu menguntungkan. Untuk tembak-menembak
kompetitif, tidak — dan ini wajib disebut di README, supaya tidak ada yang salah paham.

---

## 8. Biaya perhitungan di 680M (angka ini aritmetika, belum pengukuran)

| Yang dihitung | Cara hitung | Hasil |
|---|---|---|
| 1 frame RGBA8 | 1920 × 1200 × 4 byte | **9,2 MB** |
| sekali menyalin ke slot sendiri | baca + tulis | ~18 MB |
| satu frame buatan | baca A, baca B, tulis hasil | ~28 MB |
| 30 frame buatan per detik | 28 MB × 30 | ~0,83 GB/s → **~2% dari ~51 GB/s** |
| block-match, grid diperkecil 4×, petak 8×8, cari ±15 px | ±2.250 petak × 961 posisi × 64 pixel | ~0,4 miliar operasi → **< 0,5 ms** |
| warp + campur ukuran penuh | dibatasi bandwidth | **~1 ms** |
| **satu frame buatan** | | **~1,5–2,5 ms** |
| beban di 30 frame buatan/detik | | **~5–8% waktu GPU** |

Dua kesimpulan yang berdiri di atas angka ini:

- **Yang murah adalah bandwidth; yang mahal adalah waktu GPU, jam kerja, dan listrik.** Karena
  iGPU dan CPU berada di kemasan yang sama dan dibagi satu anggaran daya, pekerjaan DitLux bisa
  membuat game kebagian clock lebih rendah. Ini tidak terlihat di aritmetika di atas.
- **Neural flow tidak muat.** Skala ~100 miliar operasi per frame di 12 CU tidak akan realtime.
  Ini batas desain, bukan masalah optimasi.

Validasi = §11. Jangan pakai angka ini sebagai fakta sampai kamu mengukur sendiri.

---

## 9. Jebakan yang paling sering menjatuhkan: ruang warna

Frame yang kamu ambil dari Windows disimpan dengan **sRGB** — kecerahan sudah "ditekuk" supaya
terlihat alami di monitor. Menjumlahkan angka yang sudah ditekuk bukan rata-rata kecerahan yang
benar.

```
  SALAH   hasil = (A + B) / 2            <- dilakukan di angka sRGB
  BENAR   hasil = (toLinear(A) + toLinear(B)) / 2, lalu ke sRGB lagi
```

Gejala kalau salah: **tepi benda bergerak jadi lebih gelap atau warnanya bergeser**, sementara
bagian diam tampak normal. Bugnya kelihatan seperti "algoritma flow-nya jelek", padahal tidak ada
hubungannya dengan flow. Ini alasan aturan itu muncul dua kali di dokumen ini (§4 kotak 4, §5 baris
7). Di Unity kamu sudah kenal ini sebagai Color Space **Gamma vs Linear**.

---

## 10. Cara kerja yang kupakai supaya bisa belajar tanpa berkelahi dengan Windows

Seluruh tahap gerak dan pembuatan frame (§4 kotak 2 dan 4) dikerjakan terhadap **dua gambar yang
sudah tersimpan sebagai PNG di disk**, bukan ke aliran langsung game. Empat alasannya:

- **Uji A/B bisa diulang persis sama** — pasangan gambar yang sama, kernel berbeda, bandingkan.
- **Artefak bisa disentuh per pixel** — di-zoom, dibandingkan, bukan dikejar sambil main game.
- **Kamu berhenti menebak.** Kalau jelek, itu algoritmanya — bukan waktu Windows, bukan GC, bukan
  mode layar.
- Ada **regression set** tetap (misalnya adegan ToI yang paling sering bikin patah) yang dipakai terus,
  jadi setiap perubahan kernel ketahuan maju atau mundurnya.

Setelah tahap itu bagus, sumber gambarnya ditukar ke WGC. **Otaknya tidak berubah** — itu gunanya
memisahkan otak dari pipa.

---

## 11. Tangga pengerjaan, dan apa yang dibuka tiap anak tangga

```
M0  ambil gambar window + tayangkan di 60 Hz
 |  belajar: WGC, swapchain, flip model, format warna
 v
M1  buat frame tambahan dengan cara paling bodoh: ULANGI frame
 |  belajar: waku & tata-cara tayang. Belum ada matematika sama sekali
 |  <-- anak tangga yang paling sering dilewati orang, dan itu sumber frustrasinya
 v
M2  satu geseran global + warp, campur di ruang LINEAR
 v
M3  FLOW MAP dengan block-match di compute shader
 v
M4  dua arah (A maju, B mundur) + bagian bolong
 v
M5  rapikan flow + potong adegan + bobot memudar
 v
M6  governor + rasio 1,5x/1,33x + waktu yang berayun-ayun
 v
M7  coba model kecil. Perkiraan: gagal realtime. Pelajarannya justru di situ
```

Satu syarat lulus sebelum boleh naik dari M1: **bukukan waktu tayang harus datar di 16,7 ms**
pada ribuan frame dengan algoritma yang masih bodoh. Kalau grafnya bergigi, itu alokasi/GC,
dan semua artefak di M3–M5 akan kamu tuding ke penyebab yang salah.

---

## 12. Yang masih belum terbukti (jangan dihitung sebagai bagian desain)

| Status | Yang belum terbukti | Cara membuktikannya | Kenapa penting |
|---|---|---|---|
| belum diukur | beban GPU 680M saat ToI jalan normal vs saat lag | HWiNFO (GPU Load %) + waktu frame | kalau sudah ~100%, tidak ada sisa waktu GPU, dan §8 + keputusan rasio berubah total |
| belum diukur | ToI itu lambat karena cap, CPU, atau GPU | bedakan "terkunci 60" vs "turun ke 30" | kalau sudah stabil 60 fps, membuat frame tambahan tidak menghasilkan apa pun di layar 60 Hz |
| belum diuji | WGC bisa menangkap game targetmu dengan benar | coba di ToI dan 2-3 game lain; cek minimisasi, ukuran, kursor | seluruh arsitektur berdiri di atas ini |
| hipotesis | VCN / scaler di display bisa dipakai sebagai penyesuaian ukuran tanpa GPU-core | uji terpisah | kalau nyata, biaya turun jauh |
| hipotesis | `DX12 Video Motion Estimation` tersedia di driver iGPU | cek kemampuan yang dilaporkan driver | kalau nyata, menggantikan tahap 2 di §4 dengan jalan yang lebih murah |
| belum dicek | nama persis slider/preset LSFG di build 3.x | buka dokumentasi/Steam resmi | perlu kalau kamu mau menyamakan fitur |
| perkiraan | **semua angka di §8** | profiling sungguhan | itu aritmetika dari resolusi, bukan hasil ukur |

---

## 13. Risiko utama proyek ini

1. **Musuhmu adalah pengaturan waktu, bukan algoritma.** Kegagalan yang paling mungkin terjadi:
   hasilnya cantik tapi patah-patah. Lawan dengan M1 lebih dulu + telemetri sejak hari pertama.
2. **Ekspektasi kualitas.** block-match ≠ LSFG 3.x. Berbayang di tepi benda cepat dan di teks adalah
   sifat metode ini.
3. **Exclusive fullscreen tidak akan tertangkap.** Butuh pesan yang jelas kepada pengguna, bukan
   diam-diam gagal — lihat §2 syarat 1 dan §3 opsi C.
4. **Alokasi per frame** (GC di C#, atau sumber daya sementara di DirectX) menghasilkan patah yang
   akan kamu salah tuding ke flow estimation. §5 baris 2 dan §11 adalah penawarnya.
5. **Scope bocor.** Godaan menambah upscaling, HDR, atau pengaturan per-game sebelum pembuatan frame
   berjalan. Layarmu sudah native 1920×1200: upscaling tidak kamu butuhkan (§1).
6. **Satu keputusan besar masih terbuka:** menayangkan lewat **window milik DitLux sendiri**
   (butuh game borderless) atau **menyuntik ke dalam game** (bisa exclusive, lebih hemat, lebih
   rumit). Dokumen ini berasumsi yang pertama, dan itu pilihan yang bisa berubah setelah M0 jalan.

---

## 14. Yang berubah dari versi pertama dokumen ini

Dicatat supaya riwayat keputusannya tidak hilang:

| # | Versi lama | Versi ini | Alasan |
|---|---|---|---|
| 1 | Sumber frame = **DXGI Desktop Duplication** | Sumber frame = **WGC per-window** | merekam monitor tempat kamu sendiri menayangkan = merekam hasil sendiri → umpan-balik. Ditangkap dari pertanyaanmu "emang harus borderless?" |
| 2 | Alasan "borderless wajib" ditulis satu kalimat | dipecah jadi **dua syarat** dengan penyebab berbeda (memegang monitor vs umpan-balik) | satu di antaranya bisa dihindari, yang lain tidak |
| 3 | Tidak ada aturan ruang warna | **campur wajib di linear** (§4, §5, §9) | ini bug kualitas paling umum di frame generation buatan sendiri |
| 4 | Penuh istilah tanpa definisi (SRV/UAV, `memory_order_acquire`, p99) | §0 menjadi glosarium dengan padanan Unity | versi pertama tidak bisa kamu baca, dan itu salahku, bukan kekuranganmu |
| 5 | "Present hook" disebut sekilas sebagai mode kedua | jadi **opsi C di §3** + keputusan terbuka di §13 | perbedaannya perlu kelihatan, karena konsekuensinya besar (bisa exclusive) |
