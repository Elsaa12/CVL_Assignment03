# Pelacakan Objek pada Sekuens Dudek: Fusi Optical Flow Lucas–Kanade, Kalman Filter, dan Detektor Haar

Penugasan Tutorial Computer Vision (MKA), pengajar Wahyono, Ph.D.

| | |
|---|---|
| Nama | Elsa Aiziyah |
| NIM | 25/572243/PPA/07187 |
| Notebook | [Buka di Google Colab](https://colab.research.google.com/github/Elsaa12/CVL_Assignment03/blob/main/Tracking_Dudek_LK_KalmanFilter_Haar.ipynb) |
| Video hasil | `hasil_tracking_dudek.mp4` (berkas di repositori ini) |
| Laporan | `Laporan_Tracking_Dudek.pdf` (dan `.docx` yang dapat disunting) |

## Deskripsi singkat

Proyek ini melacak wajah dan kepala seorang pria pada sekuens **Dudek** dari benchmark OTB-100 (1145 frame, 720×480, 25 FPS). Sekuens ini menantang karena kamera dipegang tangan, ukuran kotak berubah sampai 2,1 kali, kepala menoleh dan miring, serta tangan sempat menutupi wajah.

Tiga pelacak dibandingkan pada sekuens yang sama. Seluruhnya diinisialisasi dari kotak anotasi pada frame 1, sesuai protokol OTB.

| Kode | Pelacak | Keterangan |
|---|---|---|
| A | LK saja | Titik sudut Shi–Tomasi dilacak dengan Pyramidal Lucas–Kanade, kotak digeser dan diskalakan oleh transformasi kesamaan (RANSAC). |
| B | LK + Kalman Filter | Hasil aliran optik dimasukkan ke Kalman Filter model Constant Velocity. |
| C | LK + Kalman Filter + Haar | Metode usulan. Detektor wajah Haar ditambahkan sebagai sensor posisi absolut. |

## Metode

1. **Aliran optik.** Titik Shi–Tomasi dipilih pada 80% bagian tengah kotak, dilacak dengan Pyramidal LK (jendela 21×21, tiga tingkat piramida), lalu difilter dengan galat maju–mundur (ambang 1,5 piksel). Transformasi kesamaan diestimasi dengan RANSAC (ambang 2 piksel), dan faktor skala per frame dibatasi pada [0,95; 1,05].
2. **Kalman Filter.** State `[cx, cy, vx, vy]` dengan model Constant Velocity, Δt = 1/25 s. Matriks Q disusun eksplisit dari percepatan acak σ_a, dan pembaruan memakai bentuk Joseph.
3. **Fusi dua sensor.**
   - Aliran optik diperlakukan sebagai **sensor kecepatan** (relatif), karena hanya mengukur perubahan posisi.
   - Detektor Haar diperlakukan sebagai **sensor posisi** (absolut) yang bebas drift tetapi lebih berisik. Detektor dijalankan tiap 5 frame atau ketika aliran optik gagal.
4. **Gerbang validasi.** Pengukuran diterima bila jarak Mahalanobis inovasinya (NIS) di bawah kuantil χ² dua derajat bebas: 99% untuk aliran optik dan 99,9% untuk detektor. Bila deteksi ditolak tetapi jaraknya tidak lebih dari satu lebar kotak, posisi filter diinisialisasi ulang pada deteksi tersebut.
5. **Penyegaran titik fitur.** Hanya ketika titik valid kurang dari 20 atau setelah inisialisasi ulang.

Parameter bawaan: σ_a = 2500 px/s², σ_v = 30 px/s, σ_det = 10 px, peredaman kecepatan saat tanpa pengukuran 0,85.

## Hasil

Evaluasi memakai protokol OTB pada 1145 frame.

| Metode | RMSE (px) | CLE rata-rata (px) | Presisi@20 px | AUC sukses | Sukses IoU>0,5 |
|---|---|---|---|---|---|
| A. LK saja | 101,20 | 91,31 | 11,9% | 0,257 | 18,3% |
| B. LK + KF | 20,71 | 16,49 | 75,8% | 0,753 | 95,7% |
| **C. LK + KF + Haar** | **17,58** | **13,51** | **79,9%** | **0,762** | 94,8% |

Beberapa temuan lain, selengkapnya ada di notebook dan laporan:

- Selisih C terhadap B kecil pada parameter bawaan. Namun B rapuh terhadap pilihan σ_a dan σ_v, sedangkan C tetap pada 15,9–18,6 px di seluruh 12 kombinasi yang diuji.
- Aliran optik paling berguna ketika detektor dijalankan jarang. Pada interval deteksi 100 frame (13 panggilan detektor), RMSE C 19,4 px, sedangkan tanpa aliran optik 114,1 px.
- Penyegaran titik fitur di setiap deteksi (varian C′) menaikkan RMSE menjadi 54,6 px, karena titik baru sebagian jatuh pada tangan yang menutupi wajah.

## Menjalankan

**Google Colab (disarankan).** Buka tautan notebook di atas, lalu pilih *Runtime → Run all*. Seluruh data diunduh otomatis. Satu kali jalan memerlukan sekitar 3–8 menit.

**Lokal.** Python 3.10 atau lebih baru dengan paket berikut, lalu jalankan notebook secara berurutan.

```bash
pip install numpy pandas matplotlib opencv-python scipy
```

Video keluaran ditulis dengan `ffmpeg` (H.264), sehingga `ffmpeg` harus tersedia di PATH. Bila notebook dijalankan lokal, keluaran disimpan di `content_lokal/keluaran_tracking/`; di Colab, di `/content/keluaran_tracking/`.

## Reprodusibilitas

- Data diunduh dari repositori `opencv/opencv_extra` pada **commit tetap** `697e5c3de828f3154e82f222dbe902d622bbe19a` (direktori `testdata/cv/tracking/dudek`), lalu diverifikasi dengan SHA-256.
- Seed acak OpenCV diset, dan deteksi Haar disimpan dalam cache agar hasil identik antarpercobaan.
- Angka dapat berbeda sedikit pada versi OpenCV yang lain.

## Isi repositori

```
├── Tracking_Dudek_LK_KalmanFilter_Haar.ipynb   Notebook (sudah dijalankan, beserta keluarannya)
├── hasil_tracking_dudek.mp4                    Video hasil tracking
├── Laporan_Tracking_Dudek.pdf                  Laporan singkat
├── Laporan_Tracking_Dudek.docx                 Laporan (dapat disunting)
└── README.md
```

## Keterbatasan

- Evaluasi hanya pada satu sekuens.
- Detektor Haar yang dipakai adalah detektor wajah frontal, sehingga hanya efektif saat wajah menghadap kamera.
- Kesalahan yang tersisa terutama muncul saat kepala menoleh atau miring, karena konvensi kotak anotasi mencakup seluruh kepala.
- Kebijakan penyegaran titik fitur dipilih setelah kegagalan varian C′ teramati pada sekuens ini.

## Sumber

Wu, Y., Lim, J., & Yang, M.-H. (2013). Online object tracking: A benchmark. *IEEE CVPR*, 2411–2418.
