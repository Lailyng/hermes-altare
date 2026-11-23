# Hermes + 9Router di Altare Sandbox dengan tmux

> Tutorial lengkap dari nol untuk pemula.
> Install Hermes Agent (Nous Research) dan 9Router di Altare Sandbox (Linux gratis), jalankan terus-menerus dengan `tmux`, sambungkan Hermes ke model AI lewat 9Router, lalu chat Hermes dari Telegram.

Tutorial ini dibuat untuk orang yang baru pertama kali pegang VPS / Sandbox. Semua perintah bisa copy-paste. Kalau ada langkah yang gagal, jangan lanjut dulu, baca bagian [Troubleshooting](#troubleshooting).

**Struktur tutorial ini:**
- **Bagian 1 — 9Router:** Langkah 0 sampai 10. Install 9Router, tambah provider AI, test endpoint.
- **Bagian 2 — Hermes:** Langkah 11 sampai 15. Install Hermes, sambungkan ke 9Router, buat bot Telegram, jalankan gateway di tmux, test dari Telegram.

Jangan loncat ke Bagian 2 sebelum Bagian 1 selesai dan `curl http://localhost:20128/v1/models` berhasil.

---

## Daftar Isi

- [Apa itu 9Router, Altare, dan tmux?](#apa-itu-9router-altare-dan-tmux)
- [Gambaran Alur](#gambaran-alur)
- [Yang Dibutuhkan](#yang-dibutuhkan)
- [0. Buat Altare Sandbox](#0-buat-altare-sandbox)
- [1. Masuk ke Sandbox lewat SSH dari Windows](#1-masuk-ke-sandbox-lewat-ssh-dari-windows)
- [2. Siapkan Dasar: Update, tmux, Node.js](#2-siapkan-dasar-update-tmux-nodejs)
- [3. Install 9Router](#3-install-9router)
- [4. Jalankan 9Router di tmux](#4-jalankan-9router-di-tmux)
- [5. Buka Dashboard 9Router dari Laptop](#5-buka-dashboard-9router-dari-laptop)
- [6. Config Awal di Dashboard](#6-config-awal-di-dashboard)
- [7. Tambah Provider AI](#7-tambah-provider-ai)
- [8. Combo: Perlu atau Tidak?](#8-combo-perlu-atau-tidak)
- [9. Test Endpoint 9Router](#9-test-endpoint-9router)
- [10. Sambungkan ke Hermes dan Tool Lain](#10-sambungkan-ke-hermes-dan-tool-lain)
- [11. Install Hermes Agent](#11-install-hermes-agent)
- [12. Sambungkan Hermes ke 9Router](#12-sambungkan-hermes-ke-9router)
- [13. Buat Bot Telegram dan Sambungkan ke Hermes](#13-buat-bot-telegram-dan-sambungkan-ke-hermes)
- [14. Jalankan Hermes Gateway di tmux](#14-jalankan-hermes-gateway-di-tmux)
- [15. Test Hermes dari Telegram dan Terminal](#15-test-hermes-dari-telegram-dan-terminal)
- [Perintah Harian tmux](#perintah-harian-tmux)
- [Troubleshooting](#troubleshooting)
- [Keamanan](#keamanan)
- [FAQ](#faq)
- [Cara Push Tutorial Ini ke GitHub](#cara-push-tutorial-ini-ke-github)

---

## Apa itu 9Router, Altare, dan tmux?

### 9Router

9Router adalah **router AI gratis dan open-source**. Dia berjalan di server kamu sendiri dan menjadi satu pintu untuk banyak provider AI.

Tanpa 9Router, kamu harus setting satu-satu:

```
Hermes -> Gemini
Hermes -> Kiro
Hermes -> Antigravity
```

Dengan 9Router, jadi satu saja:

```
Hermes -> 9Router -> Gemini / Kiro / Antigravity / dll
```

9Router menyediakan endpoint yang kompatibel dengan OpenAI:

```
http://localhost:20128/v1
```

Jadi semua tool yang bisa pakai endpoint OpenAI-compatible (Hermes, Claude Code, Cursor, Cline, OpenClaw, dll) bisa diarahkan ke 9Router, lalu 9Router yang meneruskan ke provider aslinya. Keuntungannya: satu API key, bisa ganti model tanpa ubah config tool, bisa fallback otomatis kalau satu provider kuotanya habis.

Dashboard 9Router ada di:

```
http://localhost:20128/dashboard
```

### Altare Sandbox

Altare Sandbox adalah **mesin Linux gratis di cloud** dari Altare (`console.altare.gg`). Cocok untuk menjalankan Hermes dan 9Router 24 jam tanpa harus menyalakan laptop terus.

Ciri-ciri Sandbox Altare:

- Linux, bukan Docker.
- **Tidak ada `systemd`**. Jadi tidak bisa pakai `systemctl enable` seperti di VPS biasa.
- Kalau SSH putus, proses yang jalan di terminal biasa akan mati.
- Karena itu kita pakai `tmux` (lihat di bawah).

> Penting: Sandbox bisa auto-pause kalau idle. Di Altare Console, set **Pause when idle ke `Never`** supaya 9Router dan Hermes tidak mati sendiri.

### tmux

`tmux` adalah terminal multiplexer. Bahasa sederhananya: terminal yang tetap hidup di server walau kamu tutup SSH.

Tanpa tmux:

```
SSH putus -> 9Router mati -> Hermes tidak bisa panggil model
```

Dengan tmux:

```
SSH putus -> tmux tetap jalan -> 9Router tetap jalan
```

Kamu akan buat 2 session tmux terpisah:

| Session | Isinya |
|---------|--------|
| `9router` | Server 9Router |
| `hermes` | Hermes Gateway (kalau kamu pakai Hermes) |

Jangan campur keduanya dalam satu session.

---

## Gambaran Alur

```
Laptop Windows (CMD)
    |
    | SSH
    v
Altare Sandbox (Linux)
    |
    +-- tmux session "9router"
    |       9Router jalan di http://localhost:20128
    |       Dashboard :20128/dashboard
    |       API       :20128/v1
    |
    +-- tmux session "hermes"
            Hermes Gateway
            -> panggil model lewat http://localhost:20128/v1
            -> Telegram / terminal
```

Dan dari laptop, untuk buka dashboard:

```
Browser laptop -- SSH Tunnel (-L 20128:localhost:20128) --> Sandbox :20128
```

---

## Yang Dibutuhkan

### Akun dan perangkat

- [ ] Akun Altare — daftar di `https://console.altare.gg`
- [ ] Laptop Windows dengan CMD (Command Prompt)
- [ ] Koneksi internet yang stabil
- [ ] Akun provider AI yang mau dipakai, pilih salah satu atau beberapa:
  - Gemini — API key dari Google AI Studio
  - Antigravity — akun Google, login OAuth
  - Kiro — akun Kiro gratis, 50 kredit/bulan
  - OpenCode Free — tanpa login, untuk coba-coba

Kamu **tidak wajib** punya semuanya. Satu provider saja sudah cukup untuk mulai. Sisanya bisa ditambah nanti.

### Software di Sandbox

Semua ini akan diinstall di tutorial ini, jadi tidak perlu disiapkan manual:

- `tmux` — supaya proses tetap jalan
- `curl` — untuk test endpoint
- `Node.js >= 20.18.1` — syarat mutlak 9Router
- `npm` — ikut terinstall bersama Node.js
- `9router` — diinstall lewat `npm install -g 9router`

### Pengetahuan dasar

Tidak perlu bisa Linux. Yang perlu kamu tahu hanya:

- Cara copy-paste perintah ke CMD / SSH
- Cara pencet `Ctrl+B` lalu `D` untuk keluar dari tmux tanpa mematikan proses (dijelaskan di bawah)
- Cara membaca error yang muncul di terminal, lalu cari di bagian Troubleshooting

---

## 0. Buat Altare Sandbox

1. Buka `https://console.altare.gg` di browser laptop, login.
2. Klik **Create / New Sandbox**.
3. Pilih region yang paling dekat / yang tersedia. Kalau satu region penuh, coba region lain.
4. Pilih paket **Free / Sandbox** (bukan Workstation berbayar).
5. Isi nama Sandbox, misal `9router-hermes`.
6. Klik **Create / Deploy**, tunggu sampai statusnya `Running`.

> Kalau muncul antrean seperti `Waiting for room on the node. You are number 14 in line`, itu normal saat Altare sedang ramai. Tunggu saja, jangan buat Sandbox baru berkali-kali.

> Kalau muncul error `No nodes satisfying the requirements` atau `500`, coba ganti region, lalu coba lagi beberapa menit kemudian. Cek juga `https://status.altare.gg` kalau Altare sedang ada insiden.

Setelah Sandbox `Running`:

1. Buka detail Sandbox > **Overview / Settings > SSH**.
2. Catat 3 hal ini (jangan dibagikan ke orang lain):
   - IP / Host Sandbox
   - Port SSH
   - Username, biasanya `root`
   - Perintah SSH lengkap yang ditampilkan Altare, contoh bentuknya:

```bash
ssh root@123.45.67.89 -p 22222
```

3. Di Console yang sama, cari setting **Pause when idle**, set ke **`Never`**.

---

## 1. Masuk ke Sandbox lewat SSH dari Windows

Di laptop Windows, kamu tidak butuh PuTTY atau aplikasi tambahan. CMD Windows sudah punya SSH bawaan.

1. Buka **CMD** di Windows.
2. Tempel perintah SSH dari Altare, contoh:

```bash
ssh root@IP_SANDBOX_KAMU -p PORT_SSH_KAMU
```

Ganti `IP_SANDBOX_KAMU` dan `PORT_SSH_KAMU` dengan yang ada di Console kamu.

3. Kalau pertama kali connect, akan muncul pertanyaan:

```
Are you sure you want to continue connecting (yes/no)?
```

Ketik `yes`, lalu Enter.

4. Masukkan password Sandbox kalau diminta. Saat mengetik password, tidak ada bintang `*` yang muncul, itu normal. Ketik saja lalu Enter.

Kalau berhasil, prompt terminal akan berubah, contoh:

```bash
root@altare-sandbox:~#
```

Artinya kamu sudah **di dalam Sandbox**, bukan lagi di laptop. Semua perintah setelah ini diketik di sini.

> Tips: Jangan tutup jendela CMD ini selama tutorial. Buka CMD baru kalau butuh terminal kedua (misalnya untuk SSH Tunnel di Langkah 5).

---

## 2. Siapkan Dasar: Update, tmux, Node.js

Jalankan perintah ini satu per satu di dalam SSH Sandbox. Tunggu tiap perintah selesai sebelum lanjut.

### 2.1 Update package dan install tmux + curl

```bash
apt update && apt install -y tmux curl nano
```

Cek tmux sudah terinstall:

```bash
tmux -V
```

Harus muncul versi, contoh `tmux 3.3a` atau lebih baru.

### 2.2 Cek Node.js

9Router butuh **Node.js >= 20.18.1**. Cek dulu versi yang sudah ada:

```bash
node -v
npm -v
```

- Kalau muncul `v20.18.1` ke atas, misal `v20.19.0` atau `v22.x`, lanjut ke Langkah 3.
- Kalau muncul `v18.x`, `v16.x`, atau `command not found`, install / update Node.js dulu dengan cara di bawah.

### 2.3 Install / Update Node.js ke versi 20+ (kalau belum)

Cara paling aman di Sandbox adalah pakai NodeSource. Jalankan:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs
```

Lalu cek lagi:

```bash
node -v
npm -v
```

Harus sudah `v20.x`.

> Alternatif lain: pakai `nvm` (Node Version Manager). Tapi untuk pemula, cara NodeSource di atas sudah cukup dan lebih simpel.

Kalau `node -v` masih versi lama setelah install, tutup SSH lalu masuk lagi dengan `ssh` seperti Langkah 1, lalu cek lagi.

---

## 3. Install 9Router

Di SSH Sandbox yang sama, jalankan:

```bash
npm install -g 9router
```

Tunggu sampai selesai. Proses ini mengunduh 9Router dari npm.

Cek instalasi:

```bash
9router --version
9router --help
```

Kalau muncul nomor versi, instalasi berhasil. Kalau muncul `command not found`, berarti folder npm global belum masuk ke PATH. Coba:

```bash
$(npm config get prefix)/bin/9router --version
```

Kalau perintah panjang itu berhasil, pakai path lengkap itu di Langkah 4, atau tambahkan ke PATH:

```bash
export PATH="$(npm config get prefix)/bin:$PATH"
```

Untuk update 9Router di kemudian hari, jalankan lagi:

```bash
npm install -g 9router@latest
```

---

## 4. Jalankan 9Router di tmux

Ini langkah kuncinya. **Jangan jalankan 9Router di terminal biasa tanpa tmux**, nanti mati saat SSH putus.

### 4.1 Cek apakah session 9router sudah ada

```bash
tmux ls
```

- Kalau muncul `no server running`, berarti belum ada session, lanjut ke 4.2.
- Kalau sudah ada `9router`, jangan buat baru. Langsung attach:

```bash
tmux attach -t 9router
```

### 4.2 Buat session baru untuk 9Router

Ada dua cara. Pilih salah satu.

**Cara A — Langsung jalan di background (disarankan untuk pemula):**

```bash
tmux new -d -s 9router '9router --host 127.0.0.1 --port 20128 --skip-update'
```

Penjelasan perintah:

- `tmux new -d -s 9router` = buat session tmux baru bernama `9router`, langsung di background (`-d`)
- `9router --host 127.0.0.1 --port 20128` = jalankan 9Router di localhost port 20128
- `--skip-update` = jangan cek update setiap start, lebih cepat dan stabil

**Cara B — Jalan di foreground supaya lognya kelihatan:**

```bash
tmux new -s 9router
```

Setelah masuk ke dalam tmux, ketik:

```bash
9router --host 127.0.0.1 --port 20128 --skip-update
```

Tunggu sampai log berhenti dan muncul tanda 9Router sudah listening / ready di port 20128.

Setelah itu, **jangan tekan `Ctrl+C`**. Untuk keluar dari tmux tanpa mematikan 9Router, pencet:

```
Ctrl+B lalu D
```

`Ctrl+B` ditekan dulu, lepas, lalu tekan `D`. Kamu akan kembali ke terminal biasa, tapi 9Router tetap jalan di belakang.

### 4.3 Verifikasi 9Router sudah jalan

Di terminal biasa (di luar tmux), jalankan:

```bash
tmux ls
```

Harus muncul minimal:

```
9router: 1 windows (created ...)
```

Lalu test portnya:

```bash
curl -s http://localhost:20128/v1/models | head -c 1000
```

Kalau keluar teks JSON berisi daftar model, berarti 9Router sudah jalan.

Kalau keluar `Failed to connect` / `Connection refused`, berarti 9Router belum jalan. Attach lagi untuk lihat lognya:

```bash
tmux attach -t 9router
```

Lihat error terakhir di layar, catat, lalu cek Troubleshooting.

---

## 5. Buka Dashboard 9Router dari Laptop

9Router di Sandbox hanya bisa diakses dari `localhost` Sandbox. Supaya bisa dibuka dari browser laptop, kita pakai **SSH Tunnel**.

### 5.1 Buat SSH Tunnel

Buka **CMD baru** di Windows (jangan tutup CMD SSH yang lama). Jalankan perintah SSH yang sama seperti Langkah 1, tapi tambah `-L`:

```bash
ssh -L 20128:localhost:20128 root@IP_SANDBOX_KAMU -p PORT_SSH_KAMU
```

Contoh arti `-L 20128:localhost:20128`:

- Port `20128` di laptop kamu diteruskan ke port `20128` di Sandbox.

Biarkan CMD tunnel ini **tetap terbuka** selama kamu mau buka dashboard. Kalau CMD ini ditutup, dashboard tidak bisa dibuka lagi.

### 5.2 Buka Dashboard di Browser

Di browser laptop, buka:

```
http://localhost:20128/dashboard
```

Kamu harus melihat halaman dashboard 9Router.

Kalau browser bilang `This site can't be reached`, cek:

1. Apakah 9Router sudah jalan? Cek `curl` di Langkah 4.3.
2. Apakah CMD Tunnel di 5.1 masih terbuka dan tidak ada error?
3. Apakah kamu buka `http://localhost:20128`, bukan IP Sandbox langsung?

> Jangan buka dashboard lewat IP publik Sandbox langsung (`http://IP_SANDBOX:20128`) kalau tidak perlu. Selain tidak aman, Altare biasanya tidak membuka port itu. Selalu pakai tunnel `localhost`.

---

## 6. Config Awal di Dashboard

Begitu pertama masuk dashboard, lakukan 3 hal ini **sebelum** tambah provider.

### 6.1 Ganti Password Default

Password default 9Router adalah:

```
123456
```

Ini password yang diketahui semua orang. Langsung ganti:

1. Masuk ke **Settings**.
2. Cari **Change Password / Dashboard Password**.
3. Masukkan password lama `123456`, lalu password baru yang kuat.
4. Simpan.

> Jangan pakai password yang sama dengan password Altare atau Gmail kamu.

### 6.2 Buat API Key 9Router

API Key ini yang nanti dipakai oleh Hermes dan tool lain untuk panggil 9Router. Bukan API key Gemini, bukan password dashboard.

1. Di dashboard, masuk ke menu **API Keys / Endpoint**.
2. Klik **Create / New Key**.
3. Beri nama, contoh `hermes-key`.
4. Copy key yang muncul. Key hanya ditampilkan sekali.

Simpan key itu di tempat aman, misalnya di password manager. **Jangan paste key asli ke chat, screenshot, atau repo GitHub.**

Kalau key hilang, tidak masalah: hapus key lama di dashboard, buat baru.

### 6.3 Aktifkan Require API Key

1. Masuk ke **Settings > Endpoint**.
2. Pastikan **Require API Key = ON**.

Dengan ini, orang yang tidak punya key tidak bisa memakai endpoint kamu, walau dia tahu alamatnya.

---

## 7. Tambah Provider AI

Provider adalah sumber model AI-nya. 9Router sendiri tidak punya model, dia hanya meneruskan ke provider.

Masuk ke dashboard:

```
Providers > Connect / Add Provider
```

Pilih salah satu atau beberapa di bawah ini. Kamu tidak wajib punya semuanya.

### 7.1 Gemini (API Key)

Cocok kalau kamu sudah punya Gemini API key.

1. Pilih **Gemini**.
2. Masukkan API key Gemini kamu.
   - API key Gemini didapat dari Google AI Studio.
   - Jangan tertukar dengan key 9Router.
3. Simpan / Connect.
4. Pastikan status provider jadi **Connected / Enabled**.

Model yang muncul biasanya:

```
gemini/gemini-3.8-flash
gemini/gemini-3.8-pro
```

> Catatan biaya: Gemini API dan langganan Gemini Pro (aplikasi Gemini) adalah billing yang berbeda. Punya Gemini Pro tidak otomatis berarti API gratis. Cek kuota API di Google AI Studio.

### 7.2 Antigravity

Cocok kalau kamu pakai akun Google untuk Antigravity.

1. Pilih **Antigravity**.
2. Klik **Login / Connect**, ikuti login OAuth Google.
3. Izinkan akses yang diminta.
4. Pastikan status jadi **Connected**.

Model yang muncul biasanya:

```
ag/gemini-3.8-flash
```

> Kalau Hermes tidak membalas padahal session Telegram sudah terbentuk, cek dulu provider ini. Penyebab paling umum: login Antigravity expired / disconnected, atau kuota habis. Login ulang di dashboard 9Router, lalu test lagi.

### 7.3 Kiro

Cocok untuk coba gratis Claude Sonnet.

1. Daftar / login akun Kiro di situs Kiro.
2. Di dashboard 9Router, pilih **Kiro**.
3. Login / Connect dengan akun Kiro kamu.
4. Pastikan status Connected.

Ketentuan gratis Kiro saat tutorial ini dibuat:

- Paket Free: **50 kredit per bulan, permanen**
- Bukan unlimited. Klaim lama "Kiro unlimited" sudah tidak berlaku.

Model yang muncul biasanya:

```
kr/claude-sonnet-4.5
```

### 7.4 OpenCode Free

Cocok untuk test cepat tanpa login.

1. Pilih **OpenCode Free**.
2. Connect, tanpa perlu API key.

Modelnya terbatas, tapi cukup untuk memastikan 9Router jalan.

### 7.5 Setelah Connect

Setelah provider connect:

1. Cek menu **Models** di dashboard, pastikan model yang kamu mau ada di daftar.
2. Aktifkan (Enable) model yang mau dipakai, nonaktifkan yang tidak dipakai.
3. Catat nama model persis seperti di dashboard, contoh `gemini/gemini-3.8-flash`. Nama ini yang dipakai di Hermes.

> Kamu tidak perlu restart 9Router setelah tambah provider. Perubahan provider langsung berlaku untuk panggilan berikutnya.

> Kamu juga tidak perlu mematikan Hermes sebelum edit provider. Edit di dashboard 9Router, lalu test lagi dari Hermes. Restart Hermes hanya perlu kalau kamu edit manual `~/.hermes/config.yaml` atau `~/.hermes/.env`, atau gateway stuck.

---

## 8. Combo: Perlu atau Tidak?

Combo adalah grup model untuk **fallback / routing otomatis**.

Contoh Combo bernama `my-fallback`:

```
1. gemini/gemini-3.8-flash
2. kr/claude-sonnet-4.5
```

Artinya: coba Gemini dulu. Kalau Gemini error / kuota habis, otomatis pindah ke Kiro.

### Kapan perlu buat Combo?

- Kamu punya 2 provider atau lebih, dan mau ada cadangan otomatis.
- Kamu mau satu nama model saja di Hermes, tapi di belakangnya ada beberapa provider.

### Kapan tidak perlu?

- Kamu cuma pakai 1 provider / 1 model. Langsung pakai nama modelnya saja.
- Kamu baru belajar. Lewati dulu bagian Combo, tambah nanti kalau sudah butuh.

### Cara buat Combo (opsional)

1. Di dashboard, masuk ke **Combo / Combos**.
2. Klik **Create Combo**.
3. Beri nama, contoh `fallback-flash`.
4. Tambahkan model satu per satu sesuai urutan prioritas.
5. Simpan.

Setelah itu, di Hermes kamu pakai nama Combo itu sebagai nama model.

---

## 9. Test Endpoint 9Router

Sebelum sambung ke Hermes, test dulu dari Sandbox supaya yakin 9Router dan provider sudah benar.

Di terminal SSH biasa (bukan di dalam tmux 9router), jalankan:

```bash
curl -s http://localhost:20128/v1/models | head -c 1000
```

Harus keluar JSON daftar model.

### Test dengan API Key

Ganti `KEY_9ROUTER_KAMU` dengan key yang kamu buat di Langkah 6.2. **Jangan paste key asli di chat atau di GitHub, hanya di terminal kamu sendiri.**

```bash
curl -s http://localhost:20128/v1/models -H "Authorization: Bearer KEY_9ROUTER_KAMU" | head -c 1000
```

### Test panggil model (opsional, lanjutan)

```bash
curl -s http://localhost:20128/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer KEY_9ROUTER_KAMU" \
  -d '{
    "model": "gemini/gemini-3.8-flash",
    "messages": [{"role": "user", "content": "Halo, jawab dengan satu kata: OK"}],
    "max_tokens": 20
  }'
```

Ganti `"model"` dengan model yang kamu punya. Kalau keluar balasan dari model, berarti provider dan routing sudah benar.

Kalau test ini gagal tapi `curl /v1/models` berhasil, masalahnya hampir pasti di provider (login expired, API key salah, kuota habis), bukan di 9Router-nya.

---

## 10. Sambungkan ke Hermes dan Tool Lain

Semua tool di bawah ini pakai pola yang sama:

```
Endpoint / Base URL: http://localhost:20128/v1
API Key: <key 9router dari dashboard>
Model: <nama model dari dashboard>
```

### 10.1 Hermes Agent

Hermes adalah agent yang jalan di Sandbox dan terhubung ke Telegram. Hermes memanggil model lewat 9Router.

Config Hermes ada di:

```
~/.hermes/config.yaml
```

Bagian model yang benar untuk 9Router:

```yaml
model:
  default: "gemini/gemini-3.8-flash"
  provider: "custom"
  base_url: "http://localhost:20128/v1"
  api_key: ${HERMES_CUSTOM_LOCALHOST_20128_API_KEY}
```

Penjelasan:

- `default` = nama model persis seperti di dashboard 9Router
- `provider: "custom"` = provider custom OpenAI-compatible
- `base_url` = endpoint 9Router
- `api_key` = key 9Router, disimpan sebagai environment variable, jangan tulis key mentah di file config

Jangan edit file ini manual kalau tidak perlu. Lebih aman setting lewat wizard Hermes (`hermes` / `hermes config`) kalau tersedia.

**Urutan start yang benar setiap Sandbox restart:**

1. Start 9Router dulu, test `curl /v1/models` sampai berhasil.
2. Baru start Hermes Gateway di tmux `hermes`.

```bash
tmux new -s hermes
hermes -p default gateway restart
```

Tunggu sampai log muncul:

```
[Telegram] Connected to Telegram (polling mode)
```

Lalu detach dengan `Ctrl+B` lalu `D`. Test dari Telegram dengan `/new` lalu kirim `halo`.

### 10.2 Claude Code / Cursor / Cline / OpenClaw

Di setting tool tersebut, pilih provider **OpenAI Compatible / Custom**, lalu isi:

```
Endpoint: http://localhost:20128/v1
API Key: <key 9router dari dashboard>
Model: gemini/gemini-3.8-flash
```

Nama model diganti sesuai yang kamu punya di dashboard.

---

---

# Bagian 2 — Hermes Agent

> Prasyarat: Bagian 1 sudah selesai. 9Router sudah jalan di tmux `9router`, dan `curl -s http://localhost:20128/v1/models` sudah keluar daftar model.
>
> Kalau prasyarat itu belum terpenuhi, jangan lanjut. Hermes tanpa model yang jalan hanya akan diam di Telegram.

## Apa itu Hermes Agent?

Hermes Agent adalah **AI agent open-source dari Nous Research**. Bedanya dengan chatbot biasa:

- Hermes bisa menjalankan perintah, membaca / menulis file, menjalankan terminal, dan memakai tools, bukan cuma menjawab chat.
- Hermes bisa dihubungi dari Telegram, jadi kamu bisa menyuruhnya dari HP, tanpa buka laptop.
- Hermes bisa memakai model apa saja yang kompatibel dengan OpenAI. Di tutorial ini, modelnya disuplai oleh 9Router.

Alurnya:

```
Kamu di Telegram
    |
    v
Bot Telegram
    |
    v
Hermes Gateway (di Sandbox, tmux "hermes")
    |
    v
9Router (di Sandbox, tmux "9router", http://localhost:20128/v1)
    |
    v
Gemini / Kiro / Antigravity
```

Dua file Hermes yang paling penting:

```
~/.hermes/config.yaml   # konfigurasi: model, provider, platform
~/.hermes/.env          # rahasia: API key, token bot (mode 0600, jangan dibagikan)
```

## 11. Install Hermes Agent

### 11.1 Syarat sebelum install

Pastikan ini sudah ada di Sandbox (sudah disiapkan di Bagian 1):

```bash
node -v
npm -v
curl --version
git --version
```

Hermes installer akan mengurus Python, Node.js, ripgrep, dan ffmpeg-nya sendiri. Yang kamu perlukan hanya `curl` dan `git`.

Install `git` dan paket dasar kalau belum ada:

```bash
apt update && apt install -y curl git ca-certificates python3
```

### 11.2 Jalankan installer resmi

Di SSH Sandbox, jalankan installer resmi Hermes:

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

> Kalau kamu tidak nyaman pipe script langsung ke `bash`, unduh dan baca dulu, baru jalankan:
>
> ```bash
> curl -fsSL https://hermes-agent.nousresearch.com/install.sh -o /tmp/hermes-install.sh
> less /tmp/hermes-install.sh
> bash /tmp/hermes-install.sh
> ```

Tunggu sampai installer selesai. Installer akan memilih Python, membuat virtual environment terisolasi, menginstall Hermes dan dependensinya, dan menaruh launcher CLI di `~/.local/bin`.

Setelah selesai, tutup SSH lalu masuk lagi, atau jalankan:

```bash
source ~/.bashrc
```

Lalu cek:

```bash
hermes --version
hermes doctor
```

Kalau `hermes --version` muncul nomor versi, instalasi berhasil.

Kalau muncul `command not found`, cek PATH:

```bash
ls ~/.local/bin/hermes
export PATH="$HOME/.local/bin:$PATH"
hermes --version
```

### 11.3 Install tools tambahan Hermes

Hermes punya package manager sendiri untuk tools yang dibutuhkannya (ffmpeg, node, ripgrep, chromium, dll). Jalankan:

```bash
hermes pm install
```

Hasil yang normal: sebagian besar berhasil, contoh `ffmpeg`, `node`, `npm`, `pnpm`, `python`, `ripgrep`, `venv`, `chromium` dengan tanda centang.

Kalau ada yang gagal, contoh:

```
✗ cua-driver: ... libxkbcommon.so.0: cannot open shared object file
⚠ optional cua-driver was not installed
```

Itu **boleh diabaikan dulu**. `cua-driver` hanya untuk computer-use / kontrol desktop (klik layar, buka browser). Untuk chat di Telegram, panggil model lewat 9Router, dan jalankan perintah terminal, `cua-driver` tidak dibutuhkan.

Kalau mau cek kesehatan instalasi:

```bash
hermes pm doctor
hermes config check
```

`hermes config check` harus tidak menampilkan error config. Kalau muncul error tentang `config.yaml`, perbaiki dulu sebelum lanjut, lihat Troubleshooting bagian config Hermes.

## 12. Sambungkan Hermes ke 9Router

Jangan pilih model di awal setup Hermes kalau wizard menawarkannya. Kita akan arahkan Hermes ke 9Router secara manual, supaya model bisa diganti nanti tanpa install ulang.

### 12.1 Pastikan 9Router jalan dulu

Di terminal biasa:

```bash
tmux ls
curl -s http://localhost:20128/v1/models | head -c 500
```

Harus ada session `9router` dan daftar model keluar. Kalau tidak, kembali ke Langkah 4 dan 9.

### 12.2 Siapkan API key 9Router

Ambil key 9Router yang kamu buat di Langkah 6.2. Key itu disimpan sebagai environment variable, **jangan tulis key mentah di `config.yaml`**.

Buka file `.env` Hermes:

```bash
nano ~/.hermes/.env
```

Tambahkan baris ini (ganti `KEY_9ROUTER_KAMU` dengan key kamu, hanya di terminal kamu sendiri, jangan paste key asli ke chat / GitHub):

```bash
HERMES_CUSTOM_LOCALHOST_20128_API_KEY=KEY_9ROUTER_KAMU
OPENAI_API_KEY=KEY_9ROUTER_KAMU
```

Kenapa ada dua nama? Karena contoh config Hermes lama memakai `${OPENAI_API_KEY}`, sedangkan setup 9Router baru memakai `${HERMES_CUSTOM_LOCALHOST_20128_API_KEY}`. Mengisi keduanya dengan key 9Router yang sama mencegah salah satu dari mereka kosong.

Simpan dengan `Ctrl+O`, Enter, lalu keluar dengan `Ctrl+X`.

> File `~/.hermes/.env` mode-nya `0600`, hanya bisa dibaca oleh kamu. Jangan pernah kirim isi file ini ke orang lain.

### 12.3 Setting model Hermes ke 9Router

Cara paling aman untuk pemula adalah lewat perintah `hermes config set`, bukan edit YAML manual:

```bash
hermes config set model.default "gemini/gemini-3.8-flash"
hermes config set model.provider "custom"
hermes config set model.base_url "http://localhost:20128/v1"
```

Ganti `gemini/gemini-3.8-flash` dengan nama model yang ada di dashboard 9Router kamu. Contoh lain:

- `ag/gemini-3.8-flash` — kalau modelnya dari Antigravity
- `kr/claude-sonnet-4.5` — kalau modelnya dari Kiro
- Nama Combo kamu — kalau kamu buat Combo di Langkah 8

Lalu cek hasilnya:

```bash
hermes config check
sed -n '70,95p' ~/.hermes/config.yaml
```

Harus terlihat blok seperti ini:

```yaml
model:
  default: "gemini/gemini-3.8-flash"
  provider: "custom"
  base_url: "http://localhost:20128/v1"
  api_key: ${OPENAI_API_KEY}
```

Penjelasan:

- `default` = nama model persis seperti di dashboard 9Router
- `provider: "custom"` = provider custom OpenAI-compatible
- `base_url` = endpoint 9Router
- `api_key` = key 9Router lewat environment variable

### 12.4 Cara ganti model nanti

Model bisa diganti kapan saja, tanpa install ulang. Ada dua cara:

**Cara A — Lewat wizard (paling mudah):**

```bash
hermes model
```

Pilih provider / model baru dari daftar.

**Cara B — Lewat perintah:**

```bash
hermes config set model.default "kr/claude-sonnet-4.5"
hermes -p default gateway restart
```

Contoh ganti dari Gemini Flash ke model lain, misal ke Kiro atau ke Gemini varian lain, caranya sama: ganti `model.default`, lalu restart gateway.

> Catatan: menghapus provider di dashboard 9Router **tidak otomatis** mengubah model di Hermes. Hermes menyimpan sendiri model default-nya di `~/.hermes/config.yaml`. Kalau kamu hapus Antigravity di 9Router tapi Hermes masih setting `ag/...`, Hermes akan tetap mencoba model lama itu dan gagal. Ganti dulu model di Hermes, baru hapus provider di 9Router, atau sebaliknya dengan urutan yang sadar.

### 12.5 Test Hermes di terminal dulu, sebelum Telegram

Ini langkah yang sering dilewati, padahal paling penting. Kalau Hermes tidak bisa jawab di terminal, Telegram hanya akan menambah tempat lain untuk melihat kegagalan yang sama.

Di terminal biasa (bukan di dalam tmux gateway), jalankan:

```bash
hermes chat -q "halo, jawab dengan satu kata: OK"
```

Atau masuk ke mode chat interaktif:

```bash
hermes chat
```

Ketik `halo`, tunggu balasan model. Kalau model membalas, berarti Hermes + 9Router + provider sudah benar. Baru lanjut ke Telegram.

Kalau tidak membalas, jangan lanjut ke Telegram dulu. Cek Troubleshooting Hermes di bawah, dan test 9Router langsung seperti Langkah 9.

## 13. Buat Bot Telegram dan Sambungkan ke Hermes

### 13.1 Buat bot di BotFather

1. Buka Telegram di HP / laptop, cari **@BotFather**.
2. Kirim `/newbot`.
3. BotFather akan tanya nama bot, contoh `Hermes Altare Bot`.
4. Lalu tanya username bot, harus diakhiri `bot`, contoh `hermes_altare_lai_bot`. Username harus unik, kalau sudah dipakai orang lain, coba nama lain.
5. BotFather akan membalas dengan **bot token**, bentuknya seperti:

```
123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
```

Copy token itu. Token ini sama rahasianya dengan password. **Jangan paste token asli ke chat, screenshot, atau GitHub.**

### 13.2 Dapatkan Telegram User ID kamu

1. Di Telegram, cari **@userinfobot**.
2. Kirim pesan apa saja ke bot itu.
3. Bot akan membalas dengan ID numerik kamu, contoh `123456789`.

ID ini dipakai untuk membatasi siapa yang boleh bicara dengan Hermes kamu.

### 13.3 Sambungkan bot ke Hermes

Ada dua cara. Pilih salah satu.

**Cara A — Lewat wizard gateway (disarankan):**

```bash
hermes gateway setup
```

Pilih **Telegram**, lalu ikuti wizard: paste token bot dari BotFather, masukkan User ID kamu saat diminta. Wizard akan menulis kredensial ke `~/.hermes/.env` secara otomatis.

**Cara B — Lewat `.env` manual:**

```bash
nano ~/.hermes/.env
```

Tambahkan:

```bash
TELEGRAM_BOT_TOKEN=TOKEN_BOT_KAMU
TELEGRAM_ALLOWED_USER_IDS=USER_ID_KAMU
```

Ganti `TOKEN_BOT_KAMU` dan `USER_ID_KAMU` dengan yang asli, hanya di terminal kamu. Simpan dengan `Ctrl+O`, Enter, `Ctrl+X`.

Nama variabel persisnya bisa berbeda tergantung versi Hermes dan wizard yang kamu pakai. Kalau wizard sudah menulisnya, jangan tambah manual yang duplikat. Cek isi `.env` dengan:

```bash
grep -i telegram ~/.hermes/.env
```

Pastikan token dan user ID sudah ada di sana, tanpa menampilkan isinya ke orang lain.

### 13.4 Keamanan Telegram — wajib dibaca

- **Batasi bot hanya untuk User ID kamu sendiri.** Jangan aktifkan allow-all / izinkan semua orang.
- Bot Hermes punya akses terminal dan file di Sandbox. Kalau orang asing bisa bicara dengannya, mereka bisa menyuruh Hermes menjalankan perintah di server kamu.
- Jangan bagikan username bot kamu ke grup publik kalau tidak perlu.
- Kalau token bot bocor, revoke di @BotFather dengan `/revoke`, lalu buat token baru dan update di `~/.hermes/.env`.

## 14. Jalankan Hermes Gateway di tmux

Gateway adalah proses panjang yang menghubungkan Hermes ke Telegram (dan platform lain). Gateway **harus tetap jalan** supaya bot membalas.

### 14.1 Kenapa tidak pakai systemd di Altare

Di VPS biasa, gateway dijalankan sebagai service dengan:

```bash
hermes gateway install --system
hermes gateway start
```

**Jangan jalankan itu di Altare Sandbox.** Sandbox tidak punya `systemd`, jadi perintah itu tidak akan bekerja dengan benar.

Di Sandbox, yang benar adalah: **gateway jalan manual di tmux**, dan `hermes gateway status` akan menampilkan:

```
Gateway is running (PID: ....) (Running manually, not as a system service)
```

Itu kondisi yang benar, bukan error.

Satu host hanya boleh ada **satu gateway**. Jangan start gateway kedua. Kalau sudah ada gateway yang jalan, perintah `hermes gateway run` akan bilang gateway sudah melayani profile `default`, dan menyuruhmu pakai `hermes -p default gateway restart`.

### 14.2 Start gateway di tmux `hermes`

Pastikan 9Router sudah jalan dulu (Langkah 12.1), lalu:

```bash
tmux new -s hermes
```

Di dalam tmux itu, jalankan:

```bash
hermes -p default gateway restart
```

Atau, kalau ini pertama kali dan belum ada gateway yang jalan:

```bash
hermes gateway run
```

Tunggu lognya. Yang normal akan muncul seperti ini:

```
Hermes Gateway Starting...
Messaging platforms + cron scheduler
Press Ctrl+C to stop
[Telegram] Connecting to Telegram (attempt 1/8)...
[Telegram] Connected to Telegram (polling mode)
```

Tanggal / jam log akan mengikuti waktu Sandbox kamu, contoh `2026-11-23 12:33:18`.

Setelah muncul `Connected to Telegram (polling mode)`, **jangan tekan `Ctrl+C` dan jangan tutup terminal itu**. Detach dengan:

```
Ctrl+B lalu D
```

Gateway tetap jalan di belakang.

### 14.3 Cek status gateway

Di terminal biasa (bukan di dalam tmux gateway), jalankan:

```bash
hermes gateway status
```

Hasil yang normal:

```
Gateway is running (PID: 7501) (Running manually, not as a system service)
```

Nomor PID akan berbeda di Sandbox kamu.

> **Jangan ketik perintah `hermes` di dalam jendela tmux yang sama dengan tempat gateway sedang jalan di foreground.** Perintah itu mau nyambung ke gateway lewat control socket, tapi akan ditolak. Keluar dulu dari tmux gateway dengan `Ctrl+B` lalu `D`, atau buka SSH baru, baru jalankan `hermes gateway status`.

### 14.4 Restart gateway yang benar

Kalau kamu edit `~/.hermes/config.yaml` atau `~/.hermes/.env` manual, ganti model, atau gateway stuck, restart dengan:

```bash
hermes -p default gateway restart
```

Jalankan itu dari terminal biasa, atau dari dalam tmux `hermes` setelah gateway lama berhenti. Jangan jalankan `hermes gateway run` saat gateway lama masih jalan, karena Hermes hanya mengizinkan satu gateway per host.

Kapan restart **tidak perlu**:

- Edit provider di dashboard 9Router (login ulang Antigravity, ganti model di 9Router, tambah provider). Perubahan itu langsung berlaku untuk panggilan berikutnya, Hermes / gateway boleh tetap jalan.

Kapan restart **perlu**:

- Edit manual `~/.hermes/config.yaml` atau `~/.hermes/.env`
- Key endpoint 9Router di config Hermes berubah
- Gateway stuck dengan `Agent Running: Yes` dan tidak membalas

### 14.5 Urutan start yang benar setiap Sandbox restart

Setiap kali Sandbox restart / stop lalu start lagi, jalankan urutan ini:

```bash
# 1. 9Router dulu
tmux new -d -s 9router '9router --host 127.0.0.1 --port 20128 --skip-update'
curl -s http://localhost:20128/v1/models | head -c 200

# 2. Baru Hermes gateway
tmux new -s hermes
hermes -p default gateway restart
# tunggu Connected to Telegram, lalu Ctrl+B lalu D

# 3. Cek keduanya
tmux ls
hermes gateway status
```

Harus ada 2 session di `tmux ls`: `9router` dan `hermes`.

## 15. Test Hermes dari Telegram dan Terminal

### 15.1 Test dari Telegram

1. Buka Telegram, cari bot kamu (username yang kamu buat di Langkah 13.1).
2. Kirim `/start`.
3. Kirim `/new`.
4. Kirim pesan biasa, contoh:

```
halo
```

5. Tunggu balasan Hermes.

Kalau berhasil, di Telegram akan muncul info session seperti model, provider, context, dan endpoint, lalu balasan dari model.

Contoh session yang sehat:

```
Model: gemini/gemini-3.8-flash (custom)
Provider: custom
Context: 0 / 1.048.576 (0%)
Connected Platforms: telegram, local
```

### 15.2 Perintah Telegram yang berguna

Setelah gateway jalan, kamu bisa kontrol Hermes sebagian dari Telegram:

| Perintah | Fungsi |
|----------|--------|
| `/new` | Mulai session baru |
| `/model` | Lihat / ganti model (hanya untuk provider yang sudah dikonfigurasi) |
| `/config` | Lihat config |
| `/skills` | Lihat skills |
| `/cron` | Lihat / kelola cron |
| `/profile` | Lihat profile |
| `/approve` | Setujui aksi yang menunggu persetujuan |
| `/deny` | Tolak aksi yang menunggu persetujuan |
| `/sethome` | Set home directory |
| `/restart` | Restart gateway |
| `/stop` | Stop gateway |

> Yang **tidak bisa** dilakukan hanya dari Telegram: tambah provider baru / API key pertama, edit manual `~/.hermes/config.yaml` / `~/.hermes/.env`, dan restart gateway setelah edit manual. Itu tetap harus dari terminal di Sandbox.

### 15.3 Test dari terminal

Di terminal biasa:

```bash
hermes gateway status
hermes chat -q "halo"
```

Kalau `hermes chat -q` membalas tapi Telegram diam, masalahnya di gateway / Telegram, bukan di model. Kalau keduanya diam, masalahnya di model / 9Router / provider, cek Langkah 9 dan Troubleshooting.

### 15.4 Checklist selesai

Hermes kamu bisa dibilang selesai kalau semua ini benar:

- [ ] `tmux ls` ada `9router` dan `hermes`
- [ ] `curl http://localhost:20128/v1/models` keluar daftar model
- [ ] `hermes config check` tidak ada error
- [ ] `hermes gateway status` bilang `Gateway is running (Running manually, not as a system service)`
- [ ] Log gateway ada `Connected to Telegram (polling mode)`
- [ ] Di Telegram, `/new` lalu `halo` dibalas oleh Hermes
- [ ] Session Telegram menampilkan model yang benar, contoh `gemini/gemini-3.8-flash (custom)`, bukan model lama yang sudah kamu hapus

---

## Perintah Harian tmux

Simpan bagian ini, ini yang paling sering dipakai.

```bash
# Lihat semua session tmux
tmux ls

# Masuk ke session 9router
tmux attach -t 9router

# Masuk ke session hermes
tmux attach -t hermes

# Pindah session saat sudah di dalam tmux
# Tekan Ctrl+B lalu S, pilih session dengan panah, Enter

# Pindah langsung dari dalam tmux
tmux switch-client -t 9router
tmux switch-client -t hermes

# Keluar dari tmux TANPA mematikan proses
# Ctrl+B lalu D

# Lihat log 9router tanpa masuk ke session
tmux capture-pane -t 9router -p | tail -n 100

# Restart 9router
tmux kill-session -t 9router
tmux new -d -s 9router '9router --host 127.0.0.1 --port 20128 --skip-update'

# Matikan session 9router sepenuhnya
tmux kill-session -t 9router

# Matikan session hermes sepenuhnya
tmux kill-session -t hermes

# Matikan semua session tmux (hati-hati, Hermes juga mati)
tmux kill-server

# Cek gateway Hermes (dari terminal biasa, bukan di dalam tmux gateway)
hermes gateway status

# Restart gateway Hermes
hermes -p default gateway restart

# Test Hermes di terminal
hermes chat -q "halo"
```

### Aturan tmux yang sering bikin bingung

1. **Jangan `tmux attach` dari dalam tmux.** Akan muncul warning `sessions should be nested with care`. Keluar dulu dengan `Ctrl+B` lalu `D`, atau pakai `tmux switch-client -t <nama>`.
2. **Jangan `Ctrl+C` di session 9router / hermes** kalau kamu tidak mau mematikan layanannya. `Ctrl+C` = stop proses. Detach yang benar adalah `Ctrl+B` lalu `D`.
3. **Satu session, satu layanan.** `9router` untuk 9Router, `hermes` untuk Hermes. Jangan jalankan keduanya di session yang sama.
4. **Cek `tmux ls` dulu sebelum buat session baru.** Session dobel dengan nama sama akan error.

---

## Troubleshooting

### 9Router tidak bisa start / langsung keluar

**Gejala:**

```bash
tmux attach -t 9router
# layar kosong, proses sudah mati, atau error Node
```

**Cek:**

```bash
node -v
9router --version
tmux capture-pane -t 9router -p | tail -n 100
```

**Penyebab umum:**

- Node.js di bawah 20.18.1. Update Node.js seperti Langkah 2.3.
- 9Router belum terinstall sempurna. Ulangi `npm install -g 9router`.
- Port sudah dipakai. Lihat bawah.

### Port 20128 sudah dipakai

**Gejala:**

```
Error: listen EADDRINUSE: address already in use 127.0.0.1:20128
```

**Artinya:** Sudah ada 9Router lain yang jalan.

**Solusi:**

```bash
tmux ls
curl -s http://localhost:20128/v1/models | head -c 200
```

Kalau `curl` berhasil, berarti 9Router lama masih jalan. Pakai itu saja, jangan start yang baru. Kalau mau restart:

```bash
tmux kill-session -t 9router
tmux new -d -s 9router '9router --host 127.0.0.1 --port 20128 --skip-update'
```

### `curl: Failed to connect / Connection refused`

**Gejala:**

```bash
curl -s http://localhost:20128/v1/models
# curl: (7) Failed to connect to localhost port 20128: Connection refused
```

**Artinya:** 9Router tidak jalan.

**Solusi:**

```bash
tmux ls
tmux attach -t 9router
```

Lihat log terakhir. Kalau session tidak ada, start ulang seperti Langkah 4.2.

### Dashboard tidak bisa dibuka dari laptop

**Cek berurutan:**

1. 9Router jalan? Test `curl` di Sandbox seperti Langkah 4.3.
2. SSH Tunnel jalan? CMD Tunnel di Langkah 5.1 masih terbuka? Coba tutup dan buat tunnel baru.
3. URL benar? Harus `http://localhost:20128/dashboard`, bukan `https`, bukan IP Sandbox.

### Hermes tidak membalas di Telegram

Ini biasanya **bukan** salah 9Router-nya, tapi provider di dalam 9Router.

**Cek berurutan:**

1. 9Router jalan? `curl http://localhost:20128/v1/models`
2. Provider di dashboard 9Router statusnya Connected? Cek menu Providers.
   - Antigravity: login expired? Login ulang.
   - Gemini: API key salah / kuota habis?
   - Kiro: kredit bulanan habis?
3. Model di Hermes sama dengan model di dashboard 9Router? Cek `~/.hermes/config.yaml` bagian `model.default`.
4. Gateway Hermes jalan? `tmux attach -t hermes`, lihat apakah ada log `Connected to Telegram (polling mode)`.
5. Test langsung ke 9Router seperti Langkah 9. Kalau test langsung berhasil tapi Hermes tidak balas, masalahnya di Hermes / Gateway, bukan di provider.
6. Cek session di Telegram: kalau session menampilkan `Lifetime tokens billed 0`, `Agent Running No`, dan ada error retry backoff, itu tanda panggilan model tidak pernah sampai / tidak pernah berhasil. Kembali ke langkah 1-3.

### Hermes gateway sudah jalan, `hermes gateway run` tidak start yang baru

**Gejala:**

```
✓ The host gateway already serves profile 'default' — nothing to start. PID 7501
One gateway per host serves every profile; manage it with `hermes -p default gateway restart`.
```

**Artinya:** Ini bukan error. Gateway lama masih hidup dan sudah melayani profile `default`. Hermes memang hanya boleh satu gateway per host.

**Solusi:**

- Jangan start gateway kedua.
- Kalau kamu mau gateway baca config baru, restart yang lama:

```bash
hermes -p default gateway restart
```

- Kalau gateway lama itu start sebelum config diperbaiki, dia masih pakai config lama / rusak. Restart wajib setelah perbaiki config.

### `hermes pm install` gagal di `cua-driver`

**Gejala:**

```
✗ cua-driver: staged entry failed verification, exited 127, libxkbcommon.so.0 cannot open shared object file
⚠ optional cua-driver was not installed
```

**Artinya:** `cua-driver` gagal install karena library sistem `libxkbcommon.so.0` tidak ada. Ini driver opsional untuk computer-use.

**Solusi:** Abaikan dulu untuk pemakaian Telegram / terminal. 8 dari 9 tools lain yang berhasil sudah cukup. Jangan ulangi `hermes pm install` berkali-kali hanya untuk `cua-driver`.

### Hermes tidak bisa install / `hermes: command not found`

**Cek:**

```bash
ls ~/.local/bin/hermes
source ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
hermes --version
```

Kalau masih gagal, ulangi installer Langkah 11.2 dan baca log error terakhirnya. Pastikan `curl` dan `git` sudah terinstall.

### `Gateway inaccessible: unsafe_control_peer`

**Artinya:** Perintah `hermes` di terminal tidak bisa nyambung ke gateway yang sedang jalan, karena identitas peer tidak cocok.

**Solusi:**

- Jangan ketik perintah `hermes` di dalam jendela tmux yang sama dengan tempat gateway sedang jalan (foreground).
- Keluar dulu dari tmux gateway dengan `Ctrl+B` lalu `D`, atau buka SSH baru, lalu jalankan perintah `hermes gateway status` dari terminal biasa.
- Kalau masih muncul, restart gateway:

```bash
hermes -p default gateway restart
```

### Config Hermes error: duplicate key `api_key` di line 300

**Gejala:**

```
Your settings file (/root/.hermes/config.yaml) has a formatting error at line 300.
found duplicate key "api_key"
Hermes is running on default settings until it is fixed
```

**Artinya:** Ada baris `api_key` nyasar di file config, di luar blok provider. Hermes jadi jalan pakai setting default, semua setting kamu diabaikan.

**Solusi:**

1. Lihat baris sekitar error:

```bash
sed -n '285,315p' /root/.hermes/config.yaml
grep -n "api_key" /root/.hermes/config.yaml
```

2. Hapus baris `api_key` yang nyasar itu saja (yang tidak berada di dalam blok `model:`). Jangan hapus `api_key` yang benar di dalam blok model.
3. Validasi:

```bash
hermes config check
```

4. Restart gateway:

```bash
hermes -p default gateway restart
```

> Jangan edit file config manual kalau tidak yakin. Backup dulu:
>
> ```bash
> cp /root/.hermes/config.yaml /root/.hermes/config.yaml.bak
> ```

### Sandbox `Stopped` / tidak bisa Start, antrean panjang

**Gejala di Console Altare:**

```
Your sandbox is stopped
Waiting for room on the node. You are number 14 in line.
```

**Artinya:** Altare sedang penuh / heavy demand, bukan karena setting kamu salah.

**Solusi:**

- Tunggu antrean turun.
- Jangan restart Sandbox berkali-kali, itu membuatmu antre dari belakang lagi.
- Cek `https://status.altare.gg` untuk lihat apakah ada insiden aktif.

### `tmux attach` bilang `sessions should be nested with care`

**Artinya:** Kamu mencoba attach tmux dari dalam tmux.

**Solusi:**

- Jangan pakai `unset $TMUX` untuk memaksa.
- Keluar dulu: `Ctrl+B` lalu `D`, lalu `tmux attach -t hermes` dari terminal biasa.
- Atau dari dalam tmux, pakai `tmux switch-client -t hermes` atau `Ctrl+B` lalu `S`.

---

## Keamanan

1. **Ganti password default `123456`** segera setelah install. Ini wajib.
2. **Aktifkan Require API Key** di dashboard.
3. **Jangan pernah share:**
   - API key 9Router
   - API key Gemini / provider lain
   - Password dashboard 9Router
   - Password SSH / Altare
   - Private key / seed phrase wallet
4. **Jangan commit file berisi key ke GitHub.** File ini (`README.md`) aman untuk di-push karena tidak berisi key asli. Jangan tambahkan file `.env`, `config.yaml`, atau `db.json` 9Router ke repo.
5. **Jangan buka port 20128 ke publik** tanpa autentikasi dan tanpa tahu risikonya. Selalu akses lewat SSH Tunnel `localhost`.
6. **Backup config Hermes** sebelum edit manual:

```bash
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak
```

7. **Jangan share token bot Telegram dan isi `~/.hermes/.env`.** Token bot sama rahasianya dengan password. Batasi bot hanya untuk User ID kamu sendiri, jangan aktifkan allow-all.
8. **Jangan commit `~/.hermes/.env`, `~/.hermes/config.yaml` yang berisi key mentah, atau `db.json` 9Router ke GitHub.** Repo ini hanya berisi `README.md` tutorial, tanpa key asli.

---

## FAQ

**Apakah 9Router gratis?**
Ya, 9Router-nya gratis dan open-source. Yang berbayar (kalau ada) adalah provider di belakangnya, tergantung kuota / API key yang kamu pakai. Misalnya Kiro gratis 50 kredit/bulan, Gemini API punya kuota gratis terbatas.

**Apakah harus buat Combo?**
Tidak. Combo hanya untuk fallback otomatis. Satu provider saja sudah cukup untuk mulai.

**Apakah 9Router harus jalan dulu sebelum Hermes?**
Ya. Urutannya selalu: 9Router dulu, test `curl`, baru Hermes.

**Apakah saya harus install 9Router-Go?**
Tidak wajib. 9Router biasa (npm) sudah cukup untuk kebanyakan pemula. 9Router-Go adalah versi yang lebih ringan, cocok kalau Sandbox kamu RAM-nya kecil dan 9Router npm terasa berat. Port default 9Router-Go berbeda (biasanya 20130), jadi jangan tertukar.

**Apakah Sandbox Altare bisa jalan terus 24 jam?**
Bisa, asalkan setting **Pause when idle = Never** dan Sandbox tidak kehabisan kuota free / tidak di-stop oleh Altare. Tetap cek Console Altare secara berkala.

**Saya sudah install 9Router, tapi mau ganti model dari Gemini ke Kiro, bagaimana?**
Tidak perlu install ulang. Tambah provider Kiro di dashboard 9Router, lalu ganti nama model di Hermes ke model Kiro, contoh `kr/claude-sonnet-4.5`. Restart gateway Hermes kalau perlu.

**Apakah tutorial ini bisa dipakai di VPS lain, bukan Altare?**
Bisa, dengan penyesuaian kecil. Di VPS biasa yang ada `systemd`, kamu bisa pakai `systemd service` sebagai pengganti tmux, tapi tmux tetap bisa dipakai dan lebih simpel untuk pemula. Langkah dashboard, provider, dan endpoint sama.

**Apakah Hermes bisa diganti modelnya nanti?**
Ya. Pakai `hermes model` di terminal, atau `hermes config set model.default "nama-model-baru"`, lalu `hermes -p default gateway restart`. Di Telegram, ganti model lewat `/model` hanya untuk provider yang sudah dikonfigurasi.

**Apakah Hermes bisa dikonfigurasi lewat Telegram?**
Sebagian bisa, setelah gateway jalan: `/new`, `/model`, `/config`, `/skills`, `/cron`, `/profile`, `/approve`, `/deny`, `/sethome`, `/restart`, `/stop`. Tambah provider baru / API key pertama, edit manual `~/.hermes/config.yaml` / `~/.hermes/.env`, dan restart gateway setelah edit manual tetap harus dari terminal.

**Apakah Hermes harus dimatikan sebelum edit provider di 9Router?**
Tidak. Hermes / gateway boleh tetap jalan, edit provider di dashboard 9Router langsung berlaku untuk panggilan berikutnya. Restart hanya perlu kalau kamu edit manual `~/.hermes/config.yaml` / `~/.hermes/.env`, key endpoint berubah, atau gateway stuck.

---

## Cara Push Tutorial Ini ke GitHub

Tutorial ini sengaja dibuat sebagai satu file `README.md` supaya mudah di-push ke repo baru.

### 1. Buat repo baru di GitHub

1. Buka `https://github.com/new`
2. Isi **Repository name**, contoh `hermes-altare`
3. Pilih **Public**
4. Jangan centang `Add a README file` (karena kita sudah punya file ini)
5. Klik **Create repository**

### 2. Push dari laptop kamu

Di laptop, masuk ke folder tempat file `README.md` ini disimpan. Contoh kalau filenya di `Documents/hermes-altare`:

```bash
cd Documents/hermes-altare
git init
git add README.md
git commit -m "Tutorial Hermes + 9Router di Altare Sandbox dengan tmux, dari nol untuk pemula"
git branch -M main
git remote add origin https://github.com/USERNAME_KAMU/hermes-altare.git
git push -u origin main
```

Ganti `USERNAME_KAMU` dengan username GitHub kamu, dan nama repo sesuai yang kamu buat di Langkah 1.

Kalau GitHub minta login, login lewat browser / GitHub CLI seperti biasa. Jangan paste token GitHub ke chat.

### 3. Update tutorial nanti

Kalau ada perbaikan:

```bash
git add README.md
git commit -m "Update tutorial: perbaiki langkah ..."
git push
```

---

## Referensi

- 9Router — GitHub: `https://github.com/decolua/9router`
- 9Router Dashboard: `http://localhost:20128/dashboard`
- 9Router API: `http://localhost:20128/v1`
- Altare Console: `https://console.altare.gg`
- Altare Status: `https://status.altare.gg`
- Hermes Agent (Nous Research): `https://github.com/NousResearch/hermes-agent` — cek repo resmi untuk perintah terbaru
- Hermes Agent installer: `https://hermes-agent.nousresearch.com/install.sh`
- Hermes Agent docs: `https://hermes-agent.nousresearch.com`

---

*Dibuat untuk pemula, dari nol. Kalau ada langkah yang sudah berubah di versi 9Router / Hermes terbaru, cek README resmi 9Router dan Hermes di GitHub sebagai acuan utama.*
