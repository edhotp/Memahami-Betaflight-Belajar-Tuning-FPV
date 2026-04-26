# Memahami PID Betaflight untuk Pemula

> Panduan **super sederhana** untuk memahami apa itu **PID** di Betaflight.
> Tanpa rumus matematika rumit — fokus pada **konsep & analogi**.
>
> **Referensi terpercaya:**
> - Oscar Liang — <https://oscarliang.com/pid/>
> - Oscar Liang PID/Filter Tuning — <https://oscarliang.com/pid-filter-tuning-blackbox/>
> - Dokumentasi resmi Betaflight — <https://betaflight.com>
> - Joshua Bardwell (YouTube) — referensi komunitas

---

## Daftar Isi

1. [Apa Itu PID?](#1-apa-itu-pid)
2. [Analogi: Pengemudi Mobil](#2-analogi-pengemudi-mobil)
3. [Penjelasan P, I, D Satu per Satu](#3-penjelasan-p-i-d-satu-per-satu)
4. [Bagaimana PID Bekerja di Drone?](#4-bagaimana-pid-bekerja-di-drone)
5. [Contoh Konkret: Drone Diserang Angin](#5-contoh-konkret-drone-diserang-angin)
6. [Apa Itu Slider PID di Betaflight?](#6-apa-itu-slider-pid-di-betaflight)
7. [Gejala PID Salah Setting](#7-gejala-pid-salah-setting)
8. [Cara Tuning PID untuk Pemula](#8-cara-tuning-pid-untuk-pemula)
9. [Kesalahan Umum Pemula](#9-kesalahan-umum-pemula)
10. [Ringkasan](#10-ringkasan)

---

## 1. Apa Itu PID?

**PID** singkatan dari:
- **P** = **Proportional**
- **I** = **Integral**
- **D** = **Derivative**

**PID adalah algoritma kontrol** yang membuat drone tetap **stabil** di udara — bahkan saat ada angin atau gangguan lain.

> 🤔 **Tanpa PID:** Drone akan jatuh begitu kamu lepas stick.
> ✅ **Dengan PID:** Drone tetap level/stabil sesuai perintah stick kamu.

```mermaid
flowchart LR
    Stick[🎮 Stick<br/>perintah pilot] --> Target[🎯 Target<br/>posisi drone]
    Gyro[📡 Gyro<br/>posisi aktual] --> Error[⚠️ Error<br/>selisih target & aktual]
    Target --> Error
    Error --> PID[🧠 PID<br/>hitung koreksi]
    PID --> Motor[⚙️ Motor<br/>koreksi posisi]
    Motor --> Gyro
```

**Inti PID:** Ukur selisih antara **target** (yang kamu mau) dan **aktual** (posisi drone sekarang), lalu **koreksi**. Loop ini terjadi **8000 kali per detik** di drone modern.

---

## 2. Analogi: Pengemudi Mobil

Bayangkan kamu mengemudi mobil dan ingin **tetap di tengah jalur**:

```mermaid
flowchart TB
    Drift[Mobil drift ke kanan<br/>karena angin]
    Drift --> P[P: Putar setir ke kiri<br/>seberapa kuat?]
    Drift --> I[I: Jika angin terus-menerus,<br/>tahan setir miring kiri]
    Drift --> D[D: Saat hampir kembali<br/>ke tengah, perlambat setir<br/>biar tidak overshoot]
    P --> Result[🚗 Mobil kembali stabil]
    I --> Result
    D --> Result
```

| Term | Peran Pengemudi |
|---|---|
| **P** | "Ada selisih posisi → koreksi sekarang" |
| **I** | "Selisih ini terus terjadi → tambah koreksi terus" |
| **D** | "Mendekati target → perlambat agar tidak melebihi" |

---

## 3. Penjelasan P, I, D Satu per Satu

### 3.1 P — Proportional (Reaksi)

**Fungsi:** **Seberapa kuat** drone melawan saat ada error.

> 💡 P **proporsional dengan error**: error besar → koreksi besar; error kecil → koreksi kecil.

| P Setting | Efek |
|---|---|
| **Terlalu rendah** | Drone lemot merespon, terasa "lembek" |
| **Pas** | Drone tegas, mengikuti stick dengan baik |
| **Terlalu tinggi** | Drone bergetar cepat (high-frequency oscillation) |

**Analogi:** P seperti **kekuatan otot pengemudi** memutar setir.

### 3.2 I — Integral (Memori)

**Fungsi:** **"Mengingat"** error yang terus-menerus terjadi.

> 💡 I bertambah seiring waktu jika error tidak hilang. Berguna saat ada **angin konstan** atau **drone tidak balance** (misal kamera lebih berat di satu sisi).

| I Setting | Efek |
|---|---|
| **Terlalu rendah** | Drone "drift" / melayang tidak presisi |
| **Pas** | Drone tetap di posisi walau ada angin |
| **Terlalu tinggi** | Drone bergetar pelan (low-frequency oscillation/wobble) |

**Analogi:** I seperti **memori pengemudi** — kalau angin terus mendorong ke kanan, ingat untuk terus tahan setir miring kiri.

### 3.3 D — Derivative (Peredam)

**Fungsi:** Mengurangi **overshoot** dan **getaran** dengan menebak masa depan.

> 💡 D melihat **seberapa cepat error berubah**. Jika error berkurang cepat (drone mendekati target), D akan **memperlambat** koreksi agar tidak melewati target.

| D Setting | Efek |
|---|---|
| **Terlalu rendah** | Drone overshoot (lewat target lalu balik) |
| **Pas** | Drone berhenti tepat di target |
| **Terlalu tinggi** | Motor **panas**, suara "gemerincing", noise tinggi |

**Analogi:** D seperti **rem** — saat hampir di target, dia memperlambat agar tidak kebablasan.

```mermaid
flowchart LR
    LowD[D Rendah<br/>━━━━━╱╲╱╲━━━<br/>Overshoot] -.-> GoodD[D Pas<br/>━━━━━╱──━━━━<br/>Stop di target]
    GoodD -.-> HighD[D Terlalu Tinggi<br/>━━━━━┳━━━━━━<br/>Motor panas]
```

---

## 4. Bagaimana PID Bekerja di Drone?

PID berjalan di **3 sumbu** secara terpisah:

```mermaid
flowchart TB
    Sticks[🎮 Stick Input] --> Roll[Roll PID<br/>Kiri-Kanan]
    Sticks --> Pitch[Pitch PID<br/>Depan-Belakang]
    Sticks --> Yaw[Yaw PID<br/>Putar Sumbu]
    Roll --> Mix[Motor Mixer]
    Pitch --> Mix
    Yaw --> Mix
    Mix --> M1[Motor 1]
    Mix --> M2[Motor 2]
    Mix --> M3[Motor 3]
    Mix --> M4[Motor 4]
```

Setiap sumbu punya nilai **P, I, D** sendiri-sendiri.

---

## 5. Contoh Konkret: Drone Diserang Angin

Skenario: Drone hover, tiba-tiba **angin kanan** mendorong drone ke kiri 5°.

```mermaid
sequenceDiagram
    participant Wind as 💨 Angin
    participant Gyro as 📡 Gyro
    participant PID as 🧠 PID
    participant Motor as ⚙️ Motor

    Wind->>Gyro: Drone miring 5° ke kiri
    Gyro->>PID: Error = 5°
    PID->>PID: P → koreksi kuat
    PID->>PID: I → ingat (angin masih ada)
    PID->>PID: D → siap-siap rem
    PID->>Motor: Motor kanan turun, kiri naik
    Motor->>Gyro: Drone kembali level
    Gyro->>PID: Error = 0
    PID->>Motor: Stop koreksi
```

**Apa yang terjadi tanpa PID?**
- Tanpa **P**: Drone lemot melawan, makin miring.
- Tanpa **I**: Saat angin terus-menerus, drone akan terus drift sedikit.
- Tanpa **D**: Drone melawan terlalu kuat, lewat target, balik lagi → goyang-goyang.

---

## 6. Apa Itu Slider PID di Betaflight?

Betaflight modern (4.3+) punya **Sliders** yang menyederhanakan tuning PID. **Pemula tidak perlu utak-atik angka P/I/D langsung** — cukup main slider.

```mermaid
flowchart TB
    PIDTab[PID Tuning Tab] --> Sliders[Simplified Sliders]
    Sliders --> S1[Master Multiplier<br/>Kalikan semua P, I, D]
    Sliders --> S2[P&D Gain Damping<br/>Rasio P vs D]
    Sliders --> S3[Stick Response FF<br/>Feedforward / responsivitas]
    Sliders --> S4[Drift Wobble I<br/>Kekuatan I-term]
    Sliders --> S5[Dynamic Damping D Max<br/>Boost D saat snap move]
```

### 4 Slider Utama yang Perlu Pemula Tahu:

| Slider | Fungsi | Default 5" |
|---|---|---|
| **Master Multiplier** | Kekuatan keseluruhan PID | 1.0 |
| **P&D Gain (Damping)** | Naikkan = D lebih kuat = motor lebih panas tapi anti-overshoot | 1.0 |
| **Stick Response (FF)** | Naikkan = drone makin "snappy" merespon stick | 1.0 |
| **Drift-Wobble (I)** | Naikkan = drone makin "kekunci" di posisi | 1.0 |

> 🍼 **Tips pemula:** Default Betaflight **sudah bagus** untuk drone 5". Jangan rusak dengan utak-atik tanpa pengalaman!

---

## 7. Gejala PID Salah Setting

### 7.1 Cara Identifikasi dari "Rasa" Terbang

```mermaid
flowchart TD
    Issue{Apa Gejala?}
    Issue -->|Drone bergetar tinggi<br/>suara nyaring| HighP[P atau D terlalu tinggi]
    Issue -->|Drone wobble pelan<br/>seperti goyang ombak| HighI[I terlalu tinggi]
    Issue -->|Drone lemot, drift| LowPI[P atau I terlalu rendah]
    Issue -->|Motor panas setelah terbang| HighD[D terlalu tinggi<br/>atau filter terlalu sedikit]
    Issue -->|Drone bouncy, overshoot| LowD[D terlalu rendah]
    Issue -->|Drone terasa "delay"| LowFF[Feedforward terlalu rendah]
```

### 7.2 Tabel Cepat Diagnosa

| Gejala | Penyebab | Solusi Slider |
|---|---|---|
| Suara motor "kresek-kresek" tinggi | D atau P terlalu tinggi | Turunkan **Master Multiplier** atau **Damping** |
| Goyang pelan saat hover | I terlalu tinggi | Turunkan **Drift-Wobble** |
| Drone "loyo", lambat respon | P atau FF terlalu rendah | Naikkan **Master Multiplier** atau **Stick Response** |
| Motor sangat panas (>60°C) | D terlalu tinggi / filter kurang | Turunkan **Damping**, periksa filter |
| Overshoot saat flip/roll | D terlalu rendah | Naikkan **Damping** |
| Drone drift saat hover | I terlalu rendah | Naikkan **Drift-Wobble** |

---

## 8. Cara Tuning PID untuk Pemula

### 8.1 Aturan Emas Pemula

> 🚨 **JANGAN TUNING PID DULU JIKA:**
> - Drone baru pertama dirakit (test default dulu!)
> - Masih belajar terbang acro
> - Hardware bermasalah (frame retak, prop bengkok)
> - Belum aktifkan **RPM Filter** & **Bi-directional DShot**

### 8.2 Urutan yang Benar

```mermaid
flowchart TD
    A[1. Build drone yang sehat] --> B[2. Setup hardware<br/>RPM filter, BiDir DShot]
    B --> C[3. Set Rate yang nyaman<br/>baca panduan Rate]
    C --> D[4. Terbang dengan PID DEFAULT<br/>minimal 10-20 baterai]
    D --> E{Drone terasa OK?}
    E -->|Ya, oke| F[✅ Selesai!<br/>Pemula tidak perlu tuning]
    E -->|Ada masalah| G[5. Identifikasi gejala<br/>lihat tabel diagnosa]
    G --> H[6. Geser SATU slider sedikit<br/>0.1-0.2 step]
    H --> I[7. Test fly]
    I --> E
```

### 8.3 Workflow Pemula Sederhana

1. **Backup CLI dulu:** Tab CLI → ketik `diff all` → simpan ke file.
2. **Catat slider awal:** Misal Master = 1.0, Damping = 1.0, dst.
3. **Geser SATU slider** ke arah yang sesuai gejala (lihat tabel di atas).
4. **Test fly 1 baterai.**
5. **Bandingkan rasanya.** Lebih baik? Lanjut. Lebih buruk? Balik ke nilai awal.
6. **Ulangi pelan-pelan**, jangan ubah banyak slider sekaligus.

> ⚠️ **Pemula HAMPIR TIDAK PERLU** mengetik angka P/I/D langsung. Slider sudah lebih dari cukup.

---

## 9. Kesalahan Umum Pemula

| Kesalahan | Akibat | Solusi |
|---|---|---|
| Tuning PID padahal hardware bermasalah | Tidak akan pernah dapat tune yang baik | Fix hardware dulu (frame, prop, gyro) |
| Geser banyak slider sekaligus | Tidak tahu mana yang ngaruh | Ubah 1 slider per test |
| Tidak backup CLI sebelum tuning | Susah balik ke setting awal | Selalu `diff all` → simpan dulu |
| Naikkan PID terlalu tinggi | Motor panas, motor terbakar | Cek suhu motor setelah terbang |
| Tuning tanpa RPM Filter aktif | Hasil tuning tidak optimal | Aktifkan RPM filter + BiDir DShot |
| Kira default PID jelek | Default Betaflight sudah sangat bagus untuk 5" | Terbangkan dulu default, baru tuning kalau ada masalah |
| Copy PID dari pilot lain | Setiap drone unik (motor, frame, prop beda) | Tune untuk drone kamu sendiri |

---

## 10. Ringkasan

```mermaid
mindmap
  root((PID))
    P - Proportional
      Reaksi seberapa kuat
      Terlalu rendah - lemot
      Terlalu tinggi - getar tinggi
    I - Integral
      Memori error berkelanjutan
      Lawan angin & imbalance
      Terlalu tinggi - wobble pelan
    D - Derivative
      Peredam overshoot
      Terlalu rendah - bouncy
      Terlalu tinggi - motor panas
    Slider Pemula
      Master Multiplier
      Damping
      Stick Response FF
      Drift-Wobble I
    Aturan Pemula
      Default sudah bagus
      Aktifkan RPM Filter dulu
      Geser 1 slider per test
      Backup CLI dulu
      Cek suhu motor
```

### 5 Hal Penting yang Harus Diingat:

1. **PID menjaga drone tetap stabil** — tanpa PID, drone jatuh.
2. **P = reaksi**, **I = memori**, **D = peredam**.
3. **Default Betaflight sudah bagus** untuk drone 5" yang dirakit dengan benar.
4. **Slider > angka** — pemula cukup main slider, tidak perlu sentuh angka P/I/D.
5. **Cek suhu motor** setelah setiap perubahan — motor panas = D terlalu tinggi atau filter kurang.

> 🚁 **Quote dari Oscar Liang:** *"Spend more time flying, that's how you know whether your quad flies good or not. Graphs are just graphs after all."*

Terbang lebih banyak > utak-atik PID. Jangan jatuh ke "tuning rabbit hole" terlalu dini!

---

## Bacaan Lanjut

- 📘 [Memahami Betaflight Rate untuk Pemula](MEMAHAMI_RATE.md) — pasangan PID, wajib dipahami juga!
- 📘 [Panduan Lengkap Tuning Betaflight (Bahasa Indonesia)](PANDUAN_TUNING_BETAFLIGHT.md) — untuk yang sudah siap tuning lanjut, termasuk Blackbox analysis.
