# UTS Graf Sains Data: Neo4j, VSM, dan Sistem Rekomendasi KNN

Proyek Ujian Tengah Semester mata kuliah **Graf Sains Data**.

| | |
| :--- | :--- |
| **Nama** | Sandi Ardi Prayitno |
| **NIM** | 24031554037 |
| **Notebook** | `UTS_Graf_Sandi_Ardi_Prayitno_ca0e6b.ipynb` |
| **Dataset** | [Retailrocket e-commerce dataset](https://www.kaggle.com/datasets/retailrocket/ecommerce-dataset) (`events.csv`) |

## Ringkasan

Notebook ini membangun **graf interaksi pengunjung dan produk** di Neo4j dari data klik e-commerce Retailrocket, lalu memakainya untuk analisis dan sistem rekomendasi:

1. Menghubungkan Python ke Neo4j lewat driver resmi dan menjawab pertanyaan checkpoint.
2. Memfilter data berdasarkan `timestamp` menjadi dua periode (**Opsi 3, rentang 15 hari**).
3. Mengimpor data latih (P1) ke Neo4j secara batch, lalu memverifikasi hasilnya terhadap CSV.
4. Membangun *Vector Space Model* (VSM) User dan Item sebagai matriks sparse, lalu memproyeksikannya dengan NetworkX.
5. Membandingkan proyeksi User-User dan Item-Item.
6. Menghitung similarity antar-item (Jaccard lewat Cypher, Jaccard dan Cosine lewat Python) dan menuliskannya kembali ke graf sebagai relasi `SIMILAR_TO`.
7. Membuat rekomendasi **KNN (k = 5)** untuk satu user acak dan mengevaluasinya dengan **Hit@5** dan **NDCG@5**.

### Pembagian data

| Fold | Peran | Periode | Jumlah baris |
| :--- | :--- | :--- | ---: |
| **P1** | Training | 1 s.d. 8 September 2015 | 141.038 |
| **P2** | Testing (*ground truth*) | 9 s.d. 15 September 2015 | 139.521 |
| | **Total `events_filtered.csv`** | | **280.559** |

Graf Neo4j hanya dibangun dari **P1** (`events_p1.csv`). P2 dipakai semata-mata sebagai *ground truth* evaluasi sehingga tidak ada kebocoran data (*data leakage*).

## Model Graf

```
(:Visitor {visitorId}) -[:VIEWED         {timestamp}]------------------> (:Item {itemId})
(:Visitor {visitorId}) -[:ADDED_TO_CART  {timestamp}]------------------> (:Item {itemId})
(:Visitor {visitorId}) -[:BOUGHT         {timestamp, transactionId}]--> (:Item {itemId})
(:Item)                -[:SIMILAR_TO     {score, cosine}]-------------> (:Item)      (hasil similarity)
```

Kolom `event` pada CSV dipetakan ke tipe relasi: `view` -> `VIEWED`, `addtocart` -> `ADDED_TO_CART`, `transaction` -> `BOUGHT`. Pada `Visitor` dan `Item` terpasang constraint `UNIQUE` sehingga `MERGE` berjalan cepat dan simpul tidak terduplikasi.

## Persyaratan (Requirements)

### 1. Perangkat lunak

| Komponen | Keterangan |
| :--- | :--- |
| **Python** | 3.10 (notebook dikembangkan dengan kernel Python 3.10.10) |
| **Neo4j Server** | Versi 5.x (Community Edition cukup), aktif di `neo4j://127.0.0.1:7687`, database `neo4j` |
| **Jupyter** | Jupyter Notebook / JupyterLab / VS Code dengan ekstensi Jupyter |

### 2. Pustaka Python

Versi pada notebook tidak dikunci; satu-satunya batas minimal yang perlu diperhatikan adalah driver `neo4j >= 5.7`, karena notebook memakai parameter `notifications_min_severity`.

| Paket | Dipakai untuk |
| :--- | :--- |
| `neo4j >= 5.7` | koneksi, `execute_query`, `execute_write`, `RoutingControl` |
| `pandas` | membaca dan memfilter CSV, tabel hasil |
| `numpy` | operasi larik, pengacakan dengan *seed* |
| `scipy` | `csr_matrix` untuk matriks User-Item dan Item-User (sparse) |
| `scikit-learn` | `NearestNeighbors` (cosine, *brute force*) dan `normalize` |
| `networkx` | graf bipartit dan proyeksi (`nx.bipartite.projected_graph`), visualisasi |
| `matplotlib` | plot graf |
| `tqdm` | *progress bar* impor batch |
| `ipykernel` / `jupyter` | menjalankan notebook |

Instalasi:

```bash
python -m venv myenv
# Windows
myenv\Scripts\activate
# Linux / macOS
source myenv/bin/activate

pip install -r requirements.txt
```

Isi `requirements.txt`:

```text
neo4j>=5.7
pandas
numpy
scipy
scikit-learn
networkx
matplotlib
tqdm
jupyter
ipykernel
```

### 3. Data

* Unduh `events.csv` dari halaman dataset Retailrocket di atas (kolom: `timestamp`, `visitorid`, `event`, `itemid`, `transactionid`).
* Letakkan di folder `data/` pada proyek.

## Struktur Folder

```text
.
├── UTS_Graf_Sandi_Ardi_Prayitno_ca0e6b.ipynb
├── README.md
├── requirements.txt
├── data/
│   └── events.csv              # dataset mentah (unduh sendiri)
├── events_filtered.csv         # keluaran: P1 + P2
├── events_p1.csv               # keluaran: data latih (diimpor ke Neo4j)
└── events_p2.csv               # keluaran: data uji (ground truth)
```

Tiga file `events_*.csv` dibuat otomatis oleh notebook (bagian 3).

## Cara Menjalankan

1. **Aktifkan Neo4j** dan pastikan layanan berjalan di `127.0.0.1:7687`.
2. **Sesuaikan konfigurasi** di notebook. Nilai berikut tertulis langsung di beberapa sel (sel koneksi, pembersihan, impor, VSM, similarity, dan sebelum/sesudah):
   * `URI` / `AUTH` / `USER` / `PASSWORD`: sesuaikan dengan akun Neo4j Anda.
   * Path `events.csv` pada sel filter (bagian 3) yang masih berupa path Windows pribadi: ganti ke `data/events.csv` atau lokasi file Anda.
3. Buka notebook, lalu pilih **Kernel -> Restart & Run All**, atau jalankan sel **berurutan dari atas** dalam **satu sesi kernel**. Sel-sel berikutnya memakai variabel dari sel sebelumnya (misalnya `df_p1`, `df_interactions`, `vsm_user_matrix`, `df_sim`, `ref_item_id`).
4. Jalankan **impor (bagian 4) hanya sekali**. Setelahnya sel **Verifikasi** akan memeriksa bahwa jumlah relasi di Neo4j sama dengan CSV P1. Jika `assert` gagal, jalankan sel **Pembersihan Database** (bagian 2), lalu ulangi impor sampai selesai tanpa terputus.

> **Peringatan:** sel pembersihan database menghapus **seluruh simpul dan relasi** pada database `neo4j`. Gunakan database khusus untuk tugas ini.

Perkiraan waktu dari eksekusi pada notebook: impor 142 batch (1.000 baris per batch) sekitar 20 detik.

## Alur Notebook

| Bagian | Isi | Soal |
| :-: | :--- | :--- |
| 1 | Koneksi ke Neo4j dan tes `Person` / `KNOWS` | Tugas pendahuluan |
| Soal | Jawaban 3 pertanyaan checkpoint (nama database, akses database lain, jenis graf) | Tugas pendahuluan |
| 2 | Pembersihan database secara bertahap (5.000 simpul per transaksi) | - |
| 3 | Filter `timestamp` menjadi P1 dan P2, simpan CSV | 1, 2 |
| 4 | Impor batch P1 ke Neo4j (`UNWIND`, `MERGE`, `CREATE`) | - |
| Verifikasi | Cek jumlah relasi CSV vs Neo4j dengan `assert` | - |
| 5 | Query komposisi subgraph (simpul per label, relasi per tipe) | 3 |
| 6 | VSM User (matriks User x Item) dan proyeksi User-User | 4 |
| 7 | VSM Item (transpos) dan proyeksi Item-Item (30 item terpopuler) | 5 |
| 8 | Perbandingan proyeksi User-User vs Item-Item | 6 |
| 9 | Similarity: Jaccard (Cypher + *write-back*), Jaccard dan Cosine (Python) | 7 |
| 10 | Cypher dan visualisasi graf sebelum vs sesudah similarity | 8 |
| 11 | Analytics: rekomendasi KNN (k = 5), 3 item iklan, Hit@5 dan NDCG@5 | 2.2 |

## Ringkasan Hasil (dari eksekusi notebook)

**Komposisi graf (P1):**

| Elemen | Jumlah |
| :--- | ---: |
| Simpul `Visitor` | 82.958 |
| Simpul `Item` | 51.788 |
| Relasi `VIEWED` | 136.420 |
| Relasi `ADDED_TO_CART` | 3.532 |
| Relasi `BOUGHT` | 1.086 |

Jumlah relasi (141.038) sama dengan jumlah baris P1, sehingga verifikasi impor lulus.

**VSM:** matriks User-Item berukuran 82.958 x 51.788 dengan 114.068 elemen non-zero (*sparsity* 99,9973%). Matriks Item-User adalah transposnya sehingga nilainya sama.

**Perbandingan proyeksi:**

| Kriteria | User-User | Item-Item |
| :--- | ---: | ---: |
| Jumlah simpul | 82.958 | 51.788 |
| Sisi (batas atas) | 439.309 | 548.743 |
| Simpul terisolasi | 22.670 (27,33%) | 27.839 (53,76%) |

Proyeksi **Item-Item** dipilih untuk similarity karena jumlah simpulnya lebih sedikit dan similarity antar-item bersifat tahan lama serta dapat disimpan di graf.

**Similarity (item acuan `187946`, Jaccard):** lima item termirip adalah `133549` (0,0061), `181852`, `36956`, `57245` (masing-masing 0,0031), dan `372855` (0,0030). Hasil Python (Jaccard) konsisten dengan hasil Cypher; nilai Cosine ditambahkan sebagai properti `cosine` pada relasi `SIMILAR_TO`.

**Rekomendasi KNN:**

| Item | Nilai |
| :--- | :--- |
| User terpilih (seed 42) | Visitor `287281` |
| Top-5 rekomendasi | `229940`, `334401`, `34532`, `392898`, `346673` |
| 3 item iklan | `229940`, `334401`, `34532` |
| Ground truth P2 (item baru) | 11 item |
| **Hit@5** | **0,0000** |
| **NDCG@5** | **0,0000** |

Tidak ada item rekomendasi yang muncul pada data uji user tersebut. Evaluasi dilakukan pada **satu user** sehingga nilai ini belum mewakili kualitas model secara umum.

## Metode Singkat

* **Pembobotan VSM:** nilai sel = jumlah interaksi user terhadap item (`count(r)`).
* **Jaccard:** pada himpunan visitor; **Cosine** dihitung pada vektor item yang dinormalisasi L2.
* **KNN (k = 5):** *user-based*. Tetangga adalah 5 user dengan cosine tertinggi terhadap user target yang membawa minimal satu item baru. Skor item = jumlah (similarity tetangga x bobot interaksi), item yang sudah ada di riwayat P1 dikeluarkan, lalu diambil Top-5.
* **Pemilihan user:** acak terkontrol (`np.random.seed(42)`) dari user yang aktif di P1 dan P2, punya minimal 3 item di P1, memiliki *ground truth* item baru di P2, serta bisa diberi minimal 3 rekomendasi.
* **Hit@5:** bernilai 1 jika ada irisan Top-5 dengan ground truth.
* **NDCG@5:** relevansi biner

## Catatan dan Keterbatasan
* **Konsistensi metode:** kesimpulan bagian 8 memilih proyeksi Item-Item, sedangkan rekomendasi pada bagian 11 memakai KNN berbasis **user**. Keduanya sah, tetapi sebaiknya dijelaskan di laporan alasan perbedaannya.
* **Filter waktu:** batas atas `<= 23:59:59` melewatkan event pada milidetik terakhir hari terakhir tiap periode (dampaknya kecil).
* **Evaluasi:** hanya satu user, sehingga Hit@5 dan NDCG@5 sangat bergantung pada sampel. Untuk kesimpulan yang lebih kuat, rata-ratakan pada banyak user dan bandingkan dengan *baseline* popularitas.

## Pemecahan Masalah

| Gejala | Penyebab / solusi |
| :--- | :--- |
| `ServiceUnavailable` / gagal terhubung | Neo4j belum aktif atau `URI` salah. Cek layanan dan port 7687. |
| `AuthError` | Password atau user pada `AUTH` / `PASSWORD` tidak sesuai. |
| `FileNotFoundError` pada `events.csv` | Path pada sel filter belum disesuaikan; pastikan file ada di `data/`. |
| `AssertionError: Jumlah relasi tidak cocok` | Impor pernah terputus atau dijalankan dua kali. Jalankan pembersihan database, lalu impor ulang sekali sampai selesai. |
| `NameError` (mis. `df_p1`, `vsm_user_matrix`) | Sel dijalankan tidak berurutan atau kernel di-*restart*. Jalankan ulang dari atas. |
| `TypeError ... notifications_min_severity` | Driver `neo4j` terlalu lama. Jalankan `pip install -U "neo4j>=5.7"`. |
