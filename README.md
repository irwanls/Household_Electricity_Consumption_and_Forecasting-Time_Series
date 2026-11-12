# Household Electricity Consumption & Forecasting

**Data Analytics | Time Series Forecasting | Python | Power BI**

Proyek ini menganalisis pola konsumsi listrik rumah tangga dan membangun model *time series forecasting* untuk memperkirakan kebutuhan daya pada periode berikutnya. Analisis mencakup pemeriksaan kualitas data, *exploratory data analysis* (EDA), identifikasi lonjakan arus, evaluasi pola penggunaan berdasarkan waktu, serta perbandingan tiga pendekatan prediksi: **Seasonal Naive, SARIMA, dan XGBoost**.

Hasil analisis divisualisasikan dalam dashboard Power BI untuk membantu pengguna memahami pola pemakaian listrik, memperhatikan potensi lonjakan beban, dan menggunakan hasil proyeksi sebagai bahan pertimbangan pengelolaan energi.

> **Hasil utama:** XGBoost memperoleh **MAE 0,3830 kW** pada data testing, lebih rendah dibandingkan SARIMA (0,4359 kW) dan Seasonal Naive (0,6064 kW). Model terpilih menghasilkan proyeksi rata-rata daya **1,23 kW** untuk Juli 2007.


---

## 1. Project Overview

Konsumsi listrik rumah tangga dipengaruhi oleh waktu dan penggunaan sejumlah peralatan secara bersamaan. Ketika penggunaan daya meningkat, pemilik rumah perlu memahami kapan beban mencapai puncak, kelompok peralatan apa yang berkaitan dengan lonjakan, dan bagaimana pola pemakaian diperkirakan berlanjut.

Tujuan proyek ini adalah:

1. Mengidentifikasi pola konsumsi listrik berdasarkan **jam** dan **hari**.
2. Menemukan periode dengan lonjakan arus dan meninjau kaitannya dengan sub-metering.
3. Menilai kualitas data pengukuran, terutama **missing values** dan outlier.
4. Membandingkan performa **Seasonal Naive, SARIMA, dan XGBoost** untuk memprediksi `Global_active_power`.
5. Menyusun rekomendasi pengelolaan beban dan efisiensi energi berdasarkan hasil analisis.

### Business / Analytical Questions

- Kapan rata-rata penggunaan daya mencapai nilai tertinggi dan terendah?
- Apakah pola konsumsi listrik berbeda antara hari kerja dan akhir pekan?
- Seberapa besar lonjakan arus tertinggi dan kapan lonjakan tersebut terjadi?
- Bagaimana hubungan antara `Global_active_power`, `Global_intensity`, dan ketiga variabel `Sub_metering`?
- Model apa yang menghasilkan kesalahan prediksi paling rendah pada data testing?
- Bagaimana proyeksi rata-rata daya untuk Juli 2007 dan apa implikasinya bagi pengelolaan listrik rumah tangga?

---

## 2. Data Source

Dataset yang digunakan adalah **`household_power_consumption.csv`** dari [Kaggle - Household Electricity Consumption Records](https://www.kaggle.com/datasets/thedevastator/240000-household-electricity-consumption-records).

| Karakteristik | Nilai |
|---|---:|
| Jumlah baris | 260.640 |
| Jumlah kolom yang tercatat dalam proyek | 10 |
| Jenis data | Pengukuran konsumsi listrik rumah tangga |
| Granularitas | Pengamatan per menit |
| Target forecasting | `Global_active_power` (kW) |
| Horizon yang ditampilkan | Juli 2007 |

### Variabel utama

| Kolom | Deskripsi |
|---|---|
| `Date` | Tanggal pengukuran |
| `Time` | Waktu pengukuran |
| `Global_active_power` | Daya aktif total rumah tangga (kW) |
| `Global_reactive_power` | Daya reaktif total |
| `Voltage` | Tegangan listrik (V) |
| `Global_intensity` | Arus listrik (A) |
| `Sub_metering_1` | Pengukuran kelompok beban dapur |
| `Sub_metering_2` | Pengukuran kelompok beban laundry |
| `Sub_metering_3` | Pengukuran kelompok beban pemanas air dan AC |

**Catatan:** Dataset mencakup pengukuran daya dan energi pada beberapa kelompok peralatan. **Daya (kW) tidak sama dengan energi (kWh)**, sehingga hasil prediksi `Global_active_power` tidak dapat langsung dianggap sebagai tagihan listrik tanpa konversi dan informasi tarif.

---

## 3. Data Quality & Preparation

Pemeriksaan kualitas data meliputi tipe data, konsistensi nilai, missing values, duplikasi, outlier, dan pembuatan fitur waktu.

### 3.1 Data Types & Consistency

Sejumlah kolom pengukuran awalnya bertipe `string`, antara lain `Global_active_power`, `Global_reactive_power`, `Voltage`, `Global_intensity`, `Sub_metering_1`, dan `Sub_metering_2`.

Ditemukan nilai tidak valid berupa tanda `?` pada kolom-kolom tersebut. Nilai ini diperlakukan sebagai missing value saat konversi ke tipe numerik, bukan sebagai hasil pengukuran listrik.

### 3.2 Missing Values

| Pemeriksaan | Temuan |
|---|---:|
| Missing pada masing-masing kolom pengukuran setelah konversi | 3.771 baris |
| Proporsi missing | Sekitar 1,45% |
| Gap terpanjang yang teridentifikasi | 3.723 pengamatan berurutan |
| Periode gap terpanjang | 28 April 2007 00:21 – 30 April 2007 14:23 |
| Durasi gap | Sekitar 62 jam 3 menit |

Mayoritas missing value terkonsentrasi dalam satu periode yang panjang, bukan tersebar merata. Penyebab pasti gap tersebut tidak dapat dipastikan dari data. Dalam tahap pemeriksaan awal, nilai hilang **dipertahankan sebagai `NaN`**; penanganan untuk kebutuhan pemodelan perlu disesuaikan dengan pipeline forecasting yang digunakan.

### 3.3 Duplicates

Tidak ditemukan baris duplikat pada dataset (**0% duplicate rows**).

### 3.4 Outliers

Beberapa variabel memiliki nilai ekstrem yang cukup banyak:

| Variabel | Jumlah outlier | Persentase |
|---|---:|---:|
| `Sub_metering_1` | 23.199 | 8,90% |
| `Global_intensity` | 14.666 | 5,63% |
| `Global_active_power` | 13.664 | 5,24% |
| `Sub_metering_2` | 13.424 | 5,15% |
| `Global_reactive_power` | 2.210 | 0,85% |
| `Voltage` | 570 | 0,22% |

Outlier **tidak langsung dihapus**, karena dapat mencerminkan perubahan beban listrik yang benar-benar terjadi, bukan semata-mata kesalahan sensor. Analisis lanjutan diperlukan untuk membedakan lonjakan penggunaan yang valid dari anomali pengukuran.

### 3.5 Feature Engineering

Kolom `Date` dan `Time` digabungkan menjadi fitur **`datetime`** untuk mendukung pengurutan waktu, analisis pola temporal, dan pemodelan time series.

---

## 4. Exploratory Data Analysis (EDA)

### 4.1 Karakteristik tegangan dan arus

- **Tegangan minimum–maksimum:** 223,49–250,89 V.
- **Tegangan rata-rata:** 239,21 V.
- **Arus rata-rata:** 4,97 A.
- **Arus maksimum:** 46,4 A.

Sebagian besar variabel pengukuran memiliki distribusi *right-skewed*, sedangkan `Voltage` relatif terkonsentrasi di sekitar 239–241 V. Korelasi `Global_intensity` dengan `Global_active_power` mendekati 1,00, yang konsisten dengan hubungan antara arus dan daya pada tegangan yang relatif stabil.

Dalam analisis ini, `Global_active_power` dipilih sebagai target forecasting. Variabel tersebut juga memiliki korelasi positif dengan ketiga sub-metering, masing-masing sekitar **0,48; 0,47; dan 0,61**.

### 4.2 Lonjakan arus pada 4 Maret 2007

Pengamatan tertinggi mencapai **46,4 A pada 4 Maret 2007 pukul 19:33**. Visualisasi sub-metering memperlihatkan peningkatan pada beberapa kelompok beban dalam periode lonjakan tersebut.

Temuan ini **mengindikasikan** penggunaan beberapa kelompok peralatan secara bersamaan, tetapi data agregat sub-metering saja belum membuktikan secara pasti perangkat individual yang menyebabkannya.

**Implikasi:** kapasitas sambungan, kemampuan penghantar/kabel, serta rating MCB perlu disesuaikan dengan beban maksimum yang mungkin terjadi.

### 4.3 Pola konsumsi berdasarkan hari

| Hari | Rata-rata daya |
|---|---:|
| Minggu | 1,62 kW |
| Sabtu | 1,32 kW |
| Selasa | 0,96 kW |
| Jumat | 0,94 kW |

Rata-rata daya tertinggi terjadi pada **Minggu**, sedangkan yang terendah pada **Jumat**. Perbedaan ini dapat berkaitan dengan kebiasaan penghuni pada akhir pekan, tetapi jenis aktivitas penghuni tidak dapat dipastikan dari data listrik saja.

### 4.4 Pola konsumsi berdasarkan jam

- **03:00–05:00:** rata-rata terendah, sekitar **0,5 kW**.
- **07:00–08:00:** meningkat hingga sekitar **1,5 kW**.
- **21:00:** puncak rata-rata sekitar **2,2 kW**.

Pola ini mengindikasikan siklus konsumsi berulang dalam **24 jam**. Analisis ACF menunjukkan pola pada lag 24 jam dan kelipatannya, sedangkan PACF menunjukkan hubungan kuat pada lag 1.

---


## 5. Forecasting Methodology

### 5.1 Time-Based Data Split

Data dibagi secara **kronologis** agar urutan waktu tetap terjaga dan informasi masa depan tidak digunakan secara tidak sengaja untuk membangun prediksi masa lalu.

| Subset | Proporsi |
|---|---:|
| Training | 64% |
| Validation | 16% |
| Testing | 20% |

### 5.2 Stationarity Test (ADF)

Uji *Augmented Dickey-Fuller* dilakukan pada data training:

| Metrik | Hasil |
|---|---:|
| ADF Statistic | -6,6113 |
| p-value | 6,37 × 10⁻⁹ |
| Significance level | 0,05 |

Karena p-value lebih kecil dari 0,05, hipotesis nol adanya *unit root* ditolak pada seri training yang diuji. Hasil ADF ini merupakan satu indikator, bukan jaminan bahwa seluruh sifat statistik data tidak berubah sepanjang periode.

### 5.3 Model Comparison

Tiga pendekatan yang dibandingkan adalah:

1. **Seasonal Naive:** model pembanding berbasis pola musiman sebelumnya.
2. **SARIMA:** model statistik time series dengan komponen musiman.
3. **XGBoost:** model machine learning yang menggunakan informasi lag konsumsi listrik serta fitur jam dan hari.

Beberapa konfigurasi SARIMA dan XGBoost diuji sebelum evaluasi model terpilih pada data testing.

### 5.4 Test-Set Performance

Metrik evaluasi utama adalah **Mean Absolute Error (MAE)**. Nilai lebih rendah menunjukkan rata-rata kesalahan absolut prediksi yang lebih kecil.

| Model | MAE (kW) | Urutan performa |
|---|---:|---:|
| **XGBoost** | **0,3830** | **1 – terbaik** |
| SARIMA | 0,4359 | 2 |
| Seasonal Naive | 0,6064 | 3 |

**XGBoost dipilih** karena memberikan MAE terendah pada data testing dalam perbandingan yang dilaporkan.

### 5.5 Error Analysis by Hour

Performa XGBoost berbeda berdasarkan jam:

- **03:00:** MAE terendah, sekitar **0,07 kW**.
- **21:00:** MAE tertinggi, sekitar **0,83 kW**.

Model cenderung lebih sulit memprediksi konsumsi pada jam malam yang juga memiliki rata-rata penggunaan daya tinggi. Hal ini perlu diperhatikan saat hasil prediksi digunakan sebagai bahan perencanaan beban.

### 5.6 July 2007 Forecast

| Indikator | Hasil |
|---|---:|
| Rata-rata daya historis yang dilaporkan | 1,16 kW |
| Rata-rata daya prediksi Juli 2007 | **1,23 kW** |

Proyeksi menunjukkan pola daya yang berulang dengan rata-rata sedikit lebih tinggi dibandingkan rata-rata historis yang digunakan dalam presentasi.

---

## 6. Key Findings & Conclusion

1. Konsumsi listrik memperlihatkan **pola harian yang jelas**; penggunaan tertinggi secara rata-rata muncul sekitar pukul 21:00.
2. Arus maksimum **46,4 A** terjadi bersamaan dengan peningkatan pengukuran pada beberapa kelompok beban.
3. Data memiliki **3.771 missing values** pada kolom pengukuran, didominasi satu gap panjang selama sekitar 62 jam.
4. **XGBoost** memberikan performa testing terbaik dengan **MAE 0,3830 kW** di antara tiga model yang dibandingkan.
5. Proyeksi untuk **Juli 2007** menunjukkan rata-rata daya **1,23 kW**, dibandingkan nilai historis 1,16 kW.

---

## 7. Recommendations

### 1. Periksa kesiapan instalasi untuk beban puncak

Pastikan kapasitas sambungan dan rating **MCB** sesuai dengan beban maksimum dan kemampuan kabel yang dilindungi. Pemeriksaan teknis instalasi harus dilakukan oleh tenaga yang kompeten, bukan hanya berdasarkan satu nilai arus puncak.

### 2. Kelola penggunaan beban pada jam sibuk

Pertimbangkan penjadwalan penggunaan peralatan dengan daya besar agar tidak selalu bekerja bersamaan, khususnya pada periode penggunaan tinggi di malam hari.

### 3. Gunakan forecasting sebagai sistem pendukung keputusan

Hasil prediksi dapat digunakan untuk mengantisipasi kebutuhan daya dan mengidentifikasi periode yang layak diprioritaskan dalam program efisiensi energi.

### 4. Tingkatkan kualitas data pengukuran

Pantau gangguan pencatatan, identifikasi gap secara dini, dan dokumentasikan metode penanganan missing values sebelum model digunakan secara operasional.

---

## 8. Limitations

- **Penyebab lonjakan arus belum terbukti secara kausal.** Sub-metering menunjukkan pola kelompok beban, bukan identifikasi perangkat individual secara pasti.
- **Missing values terkonsentrasi dalam gap panjang.** Penanganannya dapat memengaruhi hasil model dan perlu dijelaskan dalam kode pemodelan.
- **Hasil forecasting menunjukkan perkiraan daya, bukan penghematan biaya yang telah terealisasi.** Perhitungan kWh dan biaya membutuhkan agregasi durasi serta informasi tarif listrik.
- **Evaluasi yang dilaporkan terbatas pada data dan skenario pengujian dalam proyek ini.** Akurasi untuk rumah atau periode lain memerlukan validasi tambahan.

---

## 9. Tools

- **Python / Jupyter Notebook** – data preparation, EDA, analisis time series, dan eksperimen forecasting.
- **Power BI** – dashboard dan data storytelling.
- **Seasonal Naive, SARIMA, XGBoost** – pendekatan yang dibandingkan dalam eksperimen prediksi.
- **Canva-style presentation export** – komunikasi temuan dalam PDF presentasi.
