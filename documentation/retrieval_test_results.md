# Laporan Pengujian Diagnostik Relevansi Semantic Search & LLM Response

Dokumen ini mendokumentasikan hasil pengujian komprehensif 5 skenario kueri pada sistem *Semantic Search & LLM Response Generator* untuk layanan pelanggan e-commerce.

---

## 1. Ringkasan Metodologi Pengujian

* **Model Embedding:** `text-embedding-004` / Gemini Embedding 768-dimensi dense vector.
* **Vector Index:** FAISS `IndexFlatL2` (dimensi = 768, total vektor = 59).
* **LLM Model:** `gemini-3.5-flash-lite` dengan konfigurasi deterministik (`temperature = 0.0`, `max_output_tokens = 500`).
* **Kriteria Evaluasi:**
  1. **Akurasi Retrieval:** Apakah FAQ yang dicari berada pada Top-1 ranking?
  2. **Skor Jarak (L2 Distance):** Seberapa dekat vektor kueri dengan vektor FAQ di FAISS (semakin kecil, semakin mirip).
  3. **Faktualitas Respons LLM:** Apakah jawaban 100% konsisten dengan konteks FAQ tanpa menambah informasi spekulatif?
  4. **Kepatuhan Guardrail:** Apakah sistem mampu mengakui ketiadaan informasi dengan sopan saat kueri berada di luar basis pengetahuan?

---

## 2. Tabel Matriks Hasil Pengujian 5 Skenario

| ID | Skenario | Kueri Pelanggan | Top-1 FAQ Terambil (Kategori) | L2 Distance | Status Retrieval | Kepatuhan Guardrail / Faktualitas |
| :---: | :--- | :--- | :--- | :---: | :---: | :---: |
| **Q1** | **Skenario A** (Exact Match) | *"Berapa lama estimasi waktu pengiriman barang?"* | Berapa lama estimasi waktu pengiriman barang? (*Pengiriman*) | **0.0558** | **Sangat Tepat (Top-1)** | **100% Faktual** (Menjawab 2-5 hari dalam kota, 5-10 hari luar pulau) |
| **Q2** | **Skenario A** (Exact Match) | *"Metode pembayaran apa saja yang tersedia?"* | Metode pembayaran apa saja yang tersedia? (*Pembayaran*) | **0.0986** | **Sangat Tepat (Top-1)** | **100% Faktual** (Menyebutkan transfer bank, kartu kredit/debit, e-wallet, minimarket) |
| **Q3** | **Skenario B** (Paraphrased / Informal) | *"barang rusak pas nyampe bisa dibalikin gak?"* | Apa yang harus dilakukan jika paket diterima dalam kondisi rusak? (*Pengiriman*) | **0.1243** | **Tepat Semantik (Top-1)** | **100% Faktual & Solutif** (Instruksi foto/video & lapor komplain max 2x24 jam) |
| **Q4** | **Skenario B** (Paraphrased / Informal) | *"cara bayar pake cicilan gimana ya?"* | Apakah saya bisa membayar dengan cicilan? (*Pembayaran*) | **0.0815** | **Tepat Semantik (Top-1)** | **100% Faktual & Jelas** (Menjelaskan syarat kartu kredit & nominal minimal bank) |
| **Q5** | **Skenario C** (Out-of-Scope / Unanswered) | *"Apakah toko ini menjual tiket konser musik?"* | Apakah bisa menukar barang dengan ukuran/warna lain daripada refund? (*Pengembalian*) | **0.2803** | **Irrelevan (Jarak Jauh > 0.25)** | **Guardrail Aktif (Zero Hallucination)** (Menolak sopan & arahkan ke live chat) |

---

## 3. Detail Log Pengujian & Analisis Kritis AI Engineer

### Uji 1 — Skenario A (Exact Match)
* **Kueri:** `"Berapa lama estimasi waktu pengiriman barang?"`
* **Hasil Top-3 Retrieval:**
  1. `[Rank 1]` Distance: `0.0558` | Kategori: `Pengiriman` | Q: *"Berapa lama estimasi waktu pengiriman barang?"*
  2. `[Rank 2]` Distance: `0.1560` | Kategori: `Pengiriman` | Q: *"Bagaimana jika paket saya belum sampai melebihi estimasi?"*
  3. `[Rank 3]` Distance: `0.1714` | Kategori: `Pemesanan` | Q: *"Berapa lama waktu yang dibutuhkan sebelum pesanan diproses penjual?"*
* **Respons LLM:**
  > *"Halo! Estimasi waktu pengiriman barang umumnya adalah 2-5 hari kerja untuk area dalam kota dan 5-10 hari kerja untuk luar pulau, tergantung jasa kirim yang Anda pilih. Ada hal lain yang bisa saya bantu?"*
* **Analisis Kritis AI Engineer:**
  - **Efektivitas Retrieval:** Jarak L2 sangat rendah (`0.0558`), membuktikan representasi vektor kueri dan dokumen FAQ hampir berhimpitan secara geometris. Ranking 2 dan 3 menyajikan konteks pendukung yang masih koheren seputar durasi pengiriman.
  - **Kualitas Generasi:** LLM mengekstraksi informasi numerik secara akurat tanpa menambah estimasi hari baru. Gaya komunikasi ramah dan profesional.

---

### Uji 2 — Skenario A (Exact Match)
* **Kueri:** `"Metode pembayaran apa saja yang tersedia?"`
* **Hasil Top-3 Retrieval:**
  1. `[Rank 1]` Distance: `0.0986` | Kategori: `Pembayaran` | Q: *"Metode pembayaran apa saja yang tersedia?"*
  2. `[Rank 2]` Distance: `0.1708` | Kategori: `Pembayaran` | Q: *"Apakah bisa membayar menggunakan lebih dari satu metode sekaligus?"*
  3. `[Rank 3]` Distance: `0.2040` | Kategori: `Pembayaran` | Q: *"Apakah saya bisa membayar dengan cicilan?"*
* **Respons LLM:**
  > *"Halo! Kami mendukung beberapa metode pembayaran, yaitu transfer bank, kartu kredit/debit, e-wallet, dan pembayaran melalui minimarket rekanan. Jika ada hal lain yang ingin ditanyakan, silakan beri tahu ya!"*
* **Analisis Kritis AI Engineer:**
  - **Efektivitas Retrieval:** Top-1 tepat sasaran dengan L2 distance `0.0986`. Top-2 dan Top-3 merupakan entri dalam kategori yang sama (`Pembayaran`), menunjukkan pengelompokan semantik di ruang laten FAISS sangat terstruktur (*clustering behavior*).
  - **Kualitas Generasi:** LLM menyebutkan seluruh opsi pembayaran yang tercantum pada basis pengetahuan tanpa mengurangi atau mengarang metode baru (seperti COD atau kripto).

---

### Uji 3 — Skenario B (Paraphrased / Synonym Match)
* **Kueri:** `"barang rusak pas nyampe bisa dibalikin gak?"`
* **Hasil Top-3 Retrieval:**
  1. `[Rank 1]` Distance: `0.1243` | Kategori: `Pengiriman` | Q: *"Apa yang harus dilakukan jika paket diterima dalam kondisi rusak?"*
  2. `[Rank 2]` Distance: `0.1379` | Kategori: `Pengembalian` | Q: *"Bagaimana prosedur pengembalian barang yang tidak sesuai?"*
  3. `[Rank 3]` Distance: `0.1577` | Kategori: `Pengembalian` | Q: *"Saya menerima barang yang salah, apa yang harus saya lakukan?"*
* **Respons LLM:**
  > *"Halo! Tentu saja bisa. Jika paket diterima dalam kondisi rusak, Anda harus segera mengambil foto atau video kondisi paket saat diterima dan melaporkannya melalui menu komplain dalam waktu 2x24 jam sejak barang diterima."*
* **Analisis Kritis AI Engineer:**
  - **Efektivitas Retrieval:** Kueri pengguna menggunakan kosakata non-formal (*"pas nyampe"*, *"dibalikin gak"*). Sistem pencarian semantik berhasil memetakan konsep *"barang rusak pas nyampe"* ke entri *"paket diterima dalam kondisi rusak"* (skor `0.1243`) serta prosedur pengembalian barang (`0.1379`). Mesin pencari leksikal tradisional berbasis keyword pasti mengalami penurunan recall drastis pada kasus ini.
  - **Kualitas Generasi:** LLM mengonfirmasi kemungkinan pengembalian secara langsung dan menjelaskan syarat prosedural (foto/video dan batas waktu 2x24 jam) secara runtut.

---

### Uji 4 — Skenario B (Paraphrased / Synonym Match)
* **Kueri:** `"cara bayar pake cicilan gimana ya?"`
* **Hasil Top-3 Retrieval:**
  1. `[Rank 1]` Distance: `0.0815` | Kategori: `Pembayaran` | Q: *"Apakah saya bisa membayar dengan cicilan?"*
  2. `[Rank 2]` Distance: `0.2113` | Kategori: `Pembayaran` | Q: *"Apakah bisa membayar menggunakan lebih dari satu metode sekaligus?"*
  3. `[Rank 3]` Distance: `0.2115` | Kategori: `Pemesanan` | Q: *"Bagaimana cara melakukan pemesanan di platform ini?"*
* **Respons LLM:**
  > *"Halo! Fitur cicilan tersedia untuk transaksi dengan kartu kredit tertentu pada nominal minimal yang ditentukan pihak bank. Jika ada hal lain yang ingin ditanyakan, silakan beri tahu saya ya!"*
* **Analisis Kritis AI Engineer:**
  - **Efektivitas Retrieval:** Jarak L2 sangat dekat (`0.0815`), memperlihatkan pemahaman semantik yang kuat antara frasa *"cara bayar pake cicilan"* dengan *"membayar dengan cicilan"*.
  - **Kualitas Generasi:** Model memberikan jawaban langsung mengenai batasan cicilan (hanya kartu kredit tertentu dan syarat nominal) sesuai data FAQ.

---

### Uji 5 — Skenario C (Out-of-Scope / Unanswered)
* **Kueri:** `"Apakah toko ini menjual tiket konser musik?"`
* **Hasil Top-3 Retrieval:**
  1. `[Rank 1]` Distance: `0.2803` | Kategori: `Pengembalian` | Q: *"Apakah bisa menukar barang dengan ukuran/warna lain daripada refund?"*
  2. `[Rank 2]` Distance: `0.2807` | Kategori: `Produk` | Q: *"Bagaimana cara mengetahui apakah suatu produk asli atau palsu?"*
  3. `[Rank 3]` Distance: `0.2807` | Kategori: `Pemesanan` | Q: *"Bagaimana cara melakukan pemesanan di platform ini?"*
* **Respons LLM:**
  > *"Mohon maaf, informasi mengenai hal tersebut belum tersedia di basis pengetahuan kami. Untuk bantuan lebih lanjut, silakan hubungi Customer Support kami melalui live chat."*
* **Analisis Kritis AI Engineer:**
  - **Efektivitas Retrieval:** Skor L2 distance melonjak ke rentang `0.2803` s.d. `0.2807` (jauh di atas rata-rata dokumen relevan yang berada di bawah `< 0.16`). Hal ini menunjukkan bahwa FAISS secara akurat memposisikan kueri ini berjauhan dari seluruh klaster FAQ e-commerce.
  - **Kepatuhan Guardrail:** Meskipun dokumen Top-3 tetap disuntikkan ke prompt sebagai kandidat terdekat, instruksi *strict guardrail* berhasil mencegah LLM berhalusinasi atau berspekulasi mengenai tiket konser. LLM menjalankan instruksi *fallback* standar secara sempurna.

---

## 4. Kesimpulan & Rekomendasi Teknis

1. **Pemisahan Klaster Berdasarkan Skor Jarak (Distance Thresholding):**
   - Skor L2 `< 0.16` menunjukkan kueri yang sangat relevan dan memiliki padanan langsung/parafrase di knowledge base.
   - Skor L2 `> 0.25` dapat dijadikan acuan batas (*similarity threshold*) untuk memotong (*short-circuit*) proses pemanggilan LLM, sehingga menghemat biaya inferensi API.
2. **Robustness terhadap Bahasa Slang/Informal:**
   Model embedding Gemini (`text-embedding-004` / Gemini Embedding 768-dim) menunjukkan pemahaman morfologis dan sintaksis bahasa Indonesia informal yang sangat baik.
3. **Zero-Hallucination Rate:**
   Kombinasi context injection terstruktur dengan temperature deterministik (`0.0`) berhasil mengeliminasi halusinasi model sebesar 100% pada seluruh skenario pengujian.
