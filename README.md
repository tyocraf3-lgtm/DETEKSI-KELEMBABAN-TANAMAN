# Alat Penyiram Tanaman Otomatis Berbasis IoT (ESP32 + Sensor NPK 7-in-1)

Repositori Tugas Besar UTS mata kuliah Internet of Things: Protokol, Platform, dan AI (AAK4WBB3).
Kelompok [ISI NOMOR KELOMPOK], dosen pengampu Zein Hanni Pradana.

Proyek ini terdiri atas dua bagian:

- **Bagian A:** alat penyiram tanaman otomatis. ESP32 membaca sensor tanah NPK 7-in-1 (kelembapan, suhu, EC, pH, N, P, K) melalui RS485 Modbus RTU, mengendalikan pompa DC 5 V lewat relay berdasarkan kelembapan dan jam siram, lalu mengirim data ke ThingSpeak setiap 30 detik.
- **Bagian B:** eksperimen protokol di Wokwi. Enam skenario (S1-S6) membandingkan HTTP POST dan MQTT (QoS 0 dan QoS 1) pada payload 20 byte dan 500 byte.

Analisis data dan eksperimen dikerjakan dengan skrip Python pada folder `analisis/`.

## 1. Struktur Berkas

```
tubes-iot-penyiraman/
├── README.md
├── .gitignore
├── bagian_a_alat/
│   ├── penyiraman.ino          # kode ESP32 alat penyiraman
│   └── secrets.example.h       # contoh kredensial (salin menjadi secrets.h)
├── bagian_b_eksperimen/
│   ├── S1_http_20B.ino
│   ├── S2_http_500B.ino
│   ├── S3_mqtt_qos0_20B.ino
│   ├── S4_mqtt_qos0_500B.ino
│   ├── S5_mqtt_qos1_20B.ino
│   └── S6_mqtt_qos1_500B.ino
├── analisis/
│   └── analisis_tubes_iot.py   # skrip analisis eksperimen dan dataset
└── data/
    ├── feeds.csv               # ekspor dataset ThingSpeak
    └── log_eksperimen/         # S1.csv ... S6.csv (salinan Serial Monitor Wokwi)
```

## 2. Perangkat yang Digunakan

Hardware (Bagian A):

- ESP32 Dev Module dan papan ekspansi
- Sensor tanah NPK 7-in-1 (RS485 Modbus RTU)
- Modul TTL-RS485 (pembaca sensor NPK)
- Relay 1 channel (active-low)
- Pompa DC 5 V
- Kabel jumper dan kabel USB
- Sumber daya (adaptor atau power bank)

Software:

- Arduino IDE (Bagian A) dan Wokwi (Bagian B)
- Library `WiFi` dan `time` (bawaan paket board ESP32)
- Library `ThingSpeak` oleh MathWorks (Library Manager)
- Python 3.10 atau lebih baru, dengan `pandas`, `numpy`, dan `matplotlib` (Bagian analisis)
- Platform: ThingSpeak (https://thingspeak.mathworks.com/)

Versi yang digunakan: ESP32 core [ISI VERSI], library ThingSpeak [ISI VERSI]. Analisis diuji dengan Python 3.13, pandas 3.0, numpy 2.5, dan matplotlib 3.11.

## 3. Rangkaian Alat (Bagian A)

Hubungkan komponen ke ESP32 dengan konfigurasi berikut:

| Komponen | Pin modul | Pin ESP32 |
|---|---|---|
| Modul RS485 | RO | GPIO16 (RX) |
| Modul RS485 | DI | GPIO17 (TX) |
| Modul RS485 | DE dan RE | GPIO4 |
| Relay | IN | GPIO27 |

Sensor NPK terhubung ke terminal A dan B modul RS485 sesuai penandaan pada modul. Pompa dihubungkan melalui kontak relay. Catu daya sensor dan pompa mengikuti datasheet masing-masing [ISI]. Pastikan seluruh kabel terpasang benar sebelum perangkat dinyalakan.

Pengaturan komunikasi sensor: alamat Modbus 0x01, baud 4800, 8N1, function code 0x03, register 0x0000 sampai 0x0006.

## 4. Persiapan Arduino IDE

### Instalasi board ESP32

1. Buka Arduino IDE, pilih File, lalu Preferences.
2. Pada Additional Boards Manager URLs, masukkan:
   `https://espressif.github.io/arduino-esp32/package_esp32_index.json`
3. Buka Tools, Board, Boards Manager, cari **esp32**, lalu instal paket dari Espressif Systems.

### Instalasi library

Buka Sketch, Include Library, Manage Libraries, cari **ThingSpeak** (MathWorks), lalu instal. Library `WiFi` sudah termasuk dalam paket board ESP32.

## 5. Konfigurasi Program (Bagian A)

### Kredensial (jangan diunggah ke GitHub)

Kode membaca Wi-Fi dan ThingSpeak dari berkas `secrets.h`, yang tidak ikut diunggah.

1. Salin `bagian_a_alat/secrets.example.h` menjadi `bagian_a_alat/secrets.h`.
2. Isi nama Wi-Fi, sandi Wi-Fi, Channel ID, dan Write API Key:

```cpp
const char* WIFI_SSID     = "NAMA_WIFI";
const char* WIFI_PASSWORD = "PASSWORD_WIFI";
unsigned long CHANNEL_ID    = 0;                 // Channel ID ThingSpeak
const char*   WRITE_API_KEY = "WRITE_API_KEY";   // Write API Key ThingSpeak
```

Berkas `.gitignore` sudah memuat `secrets.h` sehingga tidak ikut terunggah.

### Konfigurasi ThingSpeak

Buat channel baru, aktifkan delapan field, lalu salin Write API Key dari bagian API Keys.

| Field | Isi | Satuan |
|---|---|---|
| field1 | Kelembapan tanah | % |
| field2 | Suhu tanah | °C |
| field3 | EC | µS/cm |
| field4 | pH | - |
| field5 | Nitrogen (N) | mg/kg |
| field6 | Fosfor (P) | mg/kg |
| field7 | Kalium (K) | mg/kg |
| field8 | Status pompa | 0 = mati, 1 = menyala |

### Pengaturan penyiraman

Pengaturan berada di bagian atas `penyiraman.ino`:

| Parameter | Nilai | Keterangan |
|---|---|---|
| `MOISTURE_LOW` | 35 % | Di bawah nilai ini pompa mulai menyala |
| `MOISTURE_HIGH` | 60 % | Pada atau di atas nilai ini pompa berhenti |
| `JAM_MULAI` - `JAM_SELESAI` | 18:00 - 06:00 WIB | Pompa hanya boleh menyala pada jam siram |
| `PUMP_MAX_MS` | 10 menit | Batas aman menyala terus-menerus |
| `SOAK_MS` | 60 detik | Jeda meresap setelah batas aman tercapai |
| `SENSOR_LOST_MS` | 30 detik | Pompa berhenti jika sensor tidak terbaca selama waktu ini |
| `SENSOR_READ_MS` | 2 detik | Interval baca sensor |
| `SEND_INTERVAL_MS` | 30 detik | Interval kirim ke ThingSpeak |

Jam diambil dari NTP. Selama jam belum tersinkron, pompa tidak akan menyala.

## 6. Cara Menjalankan Program

### Bagian A (alat)

1. Hubungkan ESP32 ke komputer dengan kabel USB.
2. Buka `bagian_a_alat/penyiraman.ino` di Arduino IDE (pastikan `secrets.h` sudah ada di folder yang sama).
3. Pilih Tools, Board, ESP32 Arduino, **ESP32 Dev Module**, lalu pilih port COM.
4. Klik Verify, lalu Upload.
5. Buka Serial Monitor dengan baud rate **115200**. Perangkat akan terhubung ke Wi-Fi, menyinkronkan jam, membaca sensor tiap 2 detik, mengatur pompa, dan mengirim data ke ThingSpeak tiap 30 detik.

### Bagian B (eksperimen di Wokwi)

1. Buka proyek Wokwi [ISI TAUTAN PROYEK WOKWI] atau buat proyek ESP32 baru, lalu tempel salah satu kode `S1` sampai `S6` dari folder `bagian_b_eksperimen/`.
2. Jalankan simulasi. Jaringan yang dipakai adalah `Wokwi-GUEST`.
3. Setiap skenario mengirim 100 pesan dengan jeda 2 detik. Pesan yang tidak kembali dalam 5 detik dihitung hilang. Waktu tempuh (RTT) diukur dengan `millis()`.
4. Salin keluaran Serial Monitor ke `data/log_eksperimen/S1.csv` dan seterusnya.

| Skenario | Protokol | Payload | Tujuan |
|---|---|---|---|
| S1 | HTTP POST | 20 byte | `httpbin.org/post` (port 80) |
| S2 | HTTP POST | 500 byte | `httpbin.org/post` (port 80) |
| S3 | MQTT QoS 0 | 20 byte | `broker.hivemq.com` (port 1883) |
| S4 | MQTT QoS 0 | 500 byte | `broker.hivemq.com` (port 1883) |
| S5 | MQTT QoS 1 | 20 byte | `broker.hivemq.com` (port 1883) |
| S6 | MQTT QoS 1 | 500 byte | `broker.hivemq.com` (port 1883) |

Pada MQTT, RTT diukur dengan pesan yang dikirim lalu diterima kembali oleh perangkat yang sama (berlangganan pada topik sendiri).

## 7. Mengulang Analisis

Pasang dependensi, lalu jalankan skrip dari folder utama repositori:

```bash
pip install pandas numpy matplotlib
python analisis/analisis_tubes_iot.py --raw-dir data/log_eksperimen --feeds data/feeds.csv --out hasil
```

Skrip membaca `S1.csv` sampai `S6.csv` dan `feeds.csv`, lalu menghasilkan di folder `hasil/`:

- `exp_S1.csv` sampai `exp_S6.csv` (log yang sudah dibersihkan)
- `hasil_metrik.json` (PDR, rata-rata, median, simpangan baku, P95, overhead, dan statistik dataset)
- `rtt_per_skenario.png` (grafik RTT)

Hasil verifikasi (menjalankan perintah di atas pada data repositori ini): dataset ThingSpeak berisi 3.296 baris; S1 memiliki RTT rata-rata 1.038,32 ms dan PDR 100%; S3 memiliki PDR 74,0%. Angka ini sama dengan yang tercantum di laporan.

## 8. Troubleshooting

- **ESP32 tidak terdeteksi:** periksa kabel USB, driver USB-to-Serial, dan port COM.
- **Upload gagal:** pastikan board ESP32 Dev Module dan port sudah dipilih. Beberapa papan perlu menahan tombol BOOT saat proses upload dimulai.
- **Error `secrets.h: No such file`:** salin `secrets.example.h` menjadi `secrets.h` di folder yang sama dengan `penyiraman.ino`.
- **Wi-Fi tidak terhubung:** periksa nama dan sandi Wi-Fi. Program mencoba menyambung ulang setiap 30 detik.
- **Serial Monitor menampilkan "Sensor tidak merespon":** periksa kabel A/B pada modul RS485, pin DE/RE (GPIO4), baud 4800, dan alamat sensor 0x01.
- **Data tidak masuk ke ThingSpeak:** periksa Channel ID, Write API Key, koneksi internet, dan kode respons pada Serial Monitor (kode 200 berarti berhasil). Interval pengiriman minimal 15 detik pada akun gratis.
- **Pompa tidak menyala:** periksa jam siram (18:00-06:00 WIB), sinkronisasi jam NTP, dan apakah kelembapan di bawah 35%. Jika pompa baru berhenti karena batas aman, tunggu jeda 60 detik.
- **Nilai EC, N, P, K terbaca nol:** pada dataset ini kondisi tersebut terutama terjadi saat kelembapan tanah di bawah 35%. Pastikan probe tertancap pada media yang lembap.

## 9. Catatan Penggunaan

- Pastikan perangkat mendapat suplai daya yang cukup dan stabil. Pada pengambilan data, baterai power bank yang habis menyebabkan perangkat tidak menyala selama beberapa hari.
- Gunakan jaringan Wi-Fi yang memiliki akses internet.
- Alat tidak menyimpan data sementara (tanpa *buffer*). Data yang tidak terkirim saat Wi-Fi terputus tidak dikirim ulang.
- Jangan mengunggah sandi Wi-Fi atau Write API Key asli ke repositori. Jika sempat terunggah, buat ulang kuncinya di ThingSpeak.

## Informasi Proyek

- Mikrokontroler: ESP32 Dev Module
- Sensor: Sensor tanah NPK 7-in-1 (RS485 Modbus RTU)
- Aktuator: Pompa DC 5 V melalui relay 1 channel
- Bahasa pemrograman: C++ (Arduino) dan Python
- Komunikasi alat: Wi-Fi, HTTP ke ThingSpeak (`writeFields`)
- Eksperimen: HTTP POST dan MQTT (QoS 0 dan QoS 1) di Wokwi
- Interval kirim: 30 detik
- Anggota: Mokh Asyam Faiq Nabil (2311101009), Wasis Cahyono Aji (2311101018), Alfantyo Muhammad Juniawan (2311101030)
