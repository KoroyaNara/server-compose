# 🏠 server-compose

Koleksi Docker Compose stack untuk home server pribadi. Semua service berjalan di atas Docker dan terhubung satu sama lain lewat shared network. (**BERDASARKAN PENGALAMAN**)

---

## 📋 Daftar Service

| Service | Deskripsi | Port |
|---|---|---|
| [AdGuard Home](#%EF%B8%8F-adguard-home) | DNS-level ad blocker & tracker blocker | `53`, `8080` |
| [Nginx Proxy Manager](#-nginx-proxy-manager-npm) | Reverse proxy + SSL otomatis | `80`, `443`, `81` |
| [Cloudflared](#%EF%B8%8F-cloudflared-tunnel) | Cloudflare Tunnel — expose service ke internet tanpa buka port router | — |
| [NetBird Client](#-netbird-client) | VPN mesh network peer-to-peer | — |
| [ownCloud](#%EF%B8%8F-owncloud) | Self-hosted cloud storage + file sharing | `8080` |
| [Minecraft Server](#-minecraft-server) | Paper MC server dengan voice chat & crack support | `25565` |
| [Playit Agent](#-playit-agent) | Tunnel Minecraft ke internet via playit.gg | — |

---

## ⚡ Prasyarat

Pastikan sudah terinstall sebelum mulai:

- [Docker Engine](https://docs.docker.com/engine/install/) (v24+)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2+)

---

## 🌐 Setup Network

Semua service menggunakan shared network. Buat network-nya dulu sebelum menjalankan stack manapun:

```bash
# Network untuk service utama (adguard, npm, cloudflared, owncloud)
docker network create private-net

# Network untuk game server (minecraft + playit)
docker network create game-net
```

---

## 🚀 Cara Menjalankan

Clone repo ini ke server kamu:

```bash
git clone https://github.com/KoroyaNara/server-compose.git
cd server-compose
```

Masuk ke folder service yang ingin dijalankan, lalu:

```bash
docker compose up -d
```

---

## 📦 Detail Service

### 🛡️ AdGuard Home

DNS server lokal yang memblokir iklan dan tracker di seluruh jaringan rumah.

**Setup:**

```bash
cd adguardhome
docker compose up -d
```

**Akses:** `http://<IP-SERVER>:3000` (setup wizard pertama kali), lalu `http://<IP-SERVER>:8080`

> Arahkan DNS router kamu ke IP server agar semua device di jaringan terproteksi secara otomatis.

---

### 🔁 Nginx Proxy Manager (NPM)

Reverse proxy dengan UI yang mudah digunakan. Bisa generate SSL certificate otomatis via Let's Encrypt.

**Setup:**

```bash
cd npm
docker compose up -d
```

**Akses:** `http://<IP-SERVER>:81`

**Kredensial default:**
- Email: `admin@example.com`
- Password: `changeme`

> Ganti kredensial default segera setelah login pertama.

---

## 🌍 Setup Domain & SSL

Setelah AdGuard Home dan NPM berjalan, lanjutkan ke setup domain agar semua service bisa diakses lewat subdomain yang rapi dengan HTTPS.

### 1. Daftarkan Domain ke Cloudflare

1. Login ke [Cloudflare Dashboard](https://dash.cloudflare.com/) → **Add a Site** → masukkan domain kamu
2. Cloudflare akan memberikan **2 nameserver**, contoh:
   ```
   anya.ns.cloudflare.com
   brad.ns.cloudflare.com
   ```
3. Buka panel registrar domain kamu → ganti nameserver lama dengan 2 nameserver dari Cloudflare
4. Tunggu propagasi — biasanya 5–30 menit, maksimal 24 jam. Cloudflare akan kirim email konfirmasi ketika domain sudah aktif

---

### 2. Arahkan Domain ke Home Server via AdGuard DNS Rewrite

Kuncinya ada di sini: home server dijadikan DNS server jaringan rumah lewat AdGuard, sehingga semua subdomain bisa diarahkan langsung ke IP server lokal — tanpa perlu buka port ke internet.

**Langkah 1 — Ubah primary DNS router ke IP server:**

Masuk ke setting router Wi-Fi → cari pengaturan **DNS Server** → ubah **Primary DNS** ke IP server kamu (contoh: `192.168.1.100`).

**Langkah 2 — Tambah DNS Rewrite di AdGuard Home:**

1. Buka dashboard AdGuard Home → menu **Filters** → **DNS Rewrites**
2. Klik **Add DNS Rewrite**, tambahkan dua entri:

   | Domain | IP |
   |---|---|
   | `domain.kamu.com` | `192.168.1.100` |
   | `*.domain.kamu.com` | `192.168.1.100` |

Sekarang semua request ke `*.domain.kamu.com` dari jaringan rumah akan langsung diarahkan ke server, dan NPM yang akan meneruskan ke masing-masing aplikasi.

---

### 3. Generate Wildcard SSL Certificate di NPM

Menggunakan **Cloudflare DNS Challenge** agar bisa dapat satu sertifikat SSL untuk semua subdomain sekaligus.

**Langkah 1 — Buat Cloudflare API Token:**

1. Buka [Cloudflare Dashboard](https://dash.cloudflare.com/) → **Profile** → **API Tokens** → **Create Token**
2. Pilih template **Edit zone DNS** → di bagian **Zone Resources** pilih domain kamu
3. Klik **Continue to summary** → **Create Token** → **salin token** (hanya tampil sekali)

**Langkah 2 — Tambah SSL Certificate di NPM:**

1. Buka NPM dashboard → menu **SSL Certificates** → **Add SSL Certificate**
2. Isi domain:
   ```
   domain.kamu.com
   *.domain.kamu.com
   ```
3. Aktifkan **Use a DNS Challenge** → pilih provider **Cloudflare**
4. Paste token Cloudflare yang tadi disalin
5. Klik **Save** — NPM akan otomatis request sertifikat wildcard ke Let's Encrypt

---

### 4. Tambah Proxy Host di NPM

Buat proxy host untuk setiap service — mulai dari AdGuard Home dan NPM itu sendiri.

1. Buka NPM dashboard → **Proxy Hosts** → **Add Proxy Host**
2. Isi konfigurasi:

   | Field | AdGuard Home | NPM |
   |---|---|---|
   | **Domain Names** | `adguard.domain.kamu.com` | `npm.domain.kamu.com` |
   | **Forward Hostname/IP** | `adguardhome` (nama container) | `npm` |
   | **Forward Port** | `3000` | `81` |
   | **Block Common Exploits** | ✅ | ✅ |

3. Tab **SSL** → pilih sertifikat wildcard yang tadi dibuat → aktifkan **Force SSL**
4. Klik **Save**

> Gunakan nama container sebagai hostname karena semua service berada di `private-net` yang sama, sehingga Docker bisa resolve nama container langsung.

---

### 5. Tutup Port Admin UI (Hardening)

Setelah bisa diakses lewat subdomain, port admin UI tidak perlu terbuka lagi ke jaringan. Comment port yang tidak diperlukan di masing-masing compose file:

**`adguardhome/docker-compose.yml`** — comment port setup wizard dan admin UI:

```yaml
ports:
  - "53:53/tcp"
  - "53:53/udp"
  # - "3000:3000/tcp"   # Setup wizard — matikan setelah setup selesai
  # - "8080:80/tcp"     # Admin UI — akses lewat subdomain saja
```

**`npm/docker-compose.yml`** — comment port admin UI:

```yaml
ports:
  - '80:80/tcp'
  - '443:443/tcp'
  # - '81:81/tcp'       # Admin UI — akses lewat subdomain saja
```

Setelah diedit, recreate container tanpa down:

```bash
# AdGuard
cd adguardhome && docker compose up -d

# NPM
cd ../npm && docker compose up -d
```

Docker akan otomatis recreate container dengan konfigurasi port yang baru.

---

### ☁️ Cloudflared Tunnel

Expose service ke internet tanpa perlu buka port di router (zero trust tunnel via Cloudflare).

**Setup:**

1. Buat tunnel di [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Salin token tunnel, lalu isi file `.env`:

```env
TUNNEL_TOKEN=token_kamu_di_sini
```

3. Jalankan:

```bash
cd cloudflared
docker compose up -d
```

---

### 🔒 NetBird Client

Koneksi VPN mesh antar device/server secara peer-to-peer menggunakan WireGuard di balik layar.

**Setup:**

1. Daftar di [NetBird](https://netbird.io/) dan buat Setup Key baru
2. Isi file `.env`:

```env
NB_SETUP_KEY=setup_key_kamu
```

3. Jalankan:

```bash
cd netbird-client
docker compose up -d
```

---

### 🔒 Setup NetBird Network Route

Setelah container NetBird berjalan, lakukan konfigurasi berikut di [NetBird Dashboard](https://app.netbird.io/) agar semua device yang terkoneksi ke NetBird bisa mengakses seluruh jaringan rumah — bukan hanya server itu sendiri.

#### Network Route — Expose LAN ke Semua Peer

1. Buka **Network Routes** → **Add Route**
2. Isi konfigurasi:

   | Field | Value |
   |---|---|
   | **Network Range** | `192.168.1.0/24` (sesuaikan dengan subnet LAN kamu) |
   | **Name** | `Home LAN Kamu` |
   | **Routing Peer** | `server-kamu` |
   | **Masquerade** | ✅ Aktifkan |

3. Tab **Settings** → pastikan **Enable Route** dan **Masquerade** keduanya aktif
4. Klik **Save Changes**

> **Masquerade** wajib diaktifkan agar device lain di LAN tidak perlu tahu soal NetBird — traffic dari peer akan di-NAT lewat server, sehingga reply packet bisa balik dengan benar.

#### DNS Nameserver — Arahkan DNS ke AdGuard

Agar peer yang connect via NetBird juga ikut menggunakan AdGuard Home sebagai DNS (dapat ad-blocking + bisa resolve subdomain lokal):

1. Buka **DNS** → **Nameservers** → **Add Nameserver**
2. Isi konfigurasi:

   | Field | Value |
   |---|---|
   | **Name** | `DNS Kamu` |
   | **IP** | `192.168.1.100` (IP server di LAN) |
   | **Port** | `53` |
   | **Distribution Groups** | `All` |

3. Aktifkan **Enable Nameserver** → **Save Changes**

Sekarang siapapun yang connect ke NetBird dari luar rumah akan otomatis bisa mengakses semua device di LAN dan memakai AdGuard sebagai DNS.

---

### 🗂️ ownCloud

Self-hosted cloud storage. Data file disimpan di HDD eksternal (`/mnt/hdd`), dengan backup folder Minecraft yang di-mount sekalian.

**Setup:**

1. Pastikan HDD sudah di-mount ke `/mnt/hdd` di server
2. Sesuaikan file `.env`:

```env
OWNCLOUD_VERSION=10.16
OWNCLOUD_DOMAIN=localhost:8080
OWNCLOUD_TRUSTED_DOMAINS=domain.kamu.com
ADMIN_USERNAME=username_admin
ADMIN_PASSWORD=password_yang_kuat
HTTP_PORT=8080
```

3. Jalankan:

```bash
cd owncloud
docker compose up -d
```

**Akses:** `http://<IP-SERVER>:8080`

> Stack ini menjalankan tiga container sekaligus: ownCloud, MariaDB, dan Redis.

---

### 🎮 Minecraft Server

Paper MC server versi 1.21.8 dengan konfigurasi untuk server crack (offline mode) + whitelist.

**Spesifikasi:**
- Type: PaperMC
- RAM: 6 GB
- Mode: Offline (crack support)
- Whitelist: Aktif
- Port suara (Simple Voice Chat): `18432/udp`
- Port Bedrock/Geyser: `19132/udp`

**Setup:**

```bash
cd minecraft
docker compose up -d
```

**Manajemen server via console:**

```bash
docker exec -it mc-server rcon-cli
```

Ini membuka RCON shell interaktif langsung ke server. Ketik `exit` untuk keluar tanpa mematikan server.

**Plugin yang dipakai:**

| Plugin | Fungsi |
|---|---|
| `AuthMe-5.6.0.jar` | Login/register password — wajib untuk server crack/offline mode |
| `Geyser-Spigot.jar` | Jembatan Bedrock ↔ Java — player Bedrock (HP/Console) bisa join |
| `floodgate-spigot.jar` | Companion Geyser — bypass auth untuk player Bedrock |
| `ViaVersion-5.6.0.jar` | Player dengan versi MC berbeda bisa join server yang sama |
| `voicechat-bukkit-2.6.7.jar` | Proximity voice chat in-game |
| `FastLeafDecay-Fabric-36.jar` | Daun pohon decay lebih cepat setelah ditebang |
| `RHLeafDecay-1.21_R1.jar` | Alternatif/pelengkap fast leaf decay untuk 1.21 |

> Untuk tambah plugin, letakkan file `.jar` di `minecraft/plugins/` lalu restart container.

**Command server penting (jalankan via `rcon-cli`):**

```bash
# Jadikan player sebagai OP (admin)
op NamaPlayer

# Cabut OP
deop NamaPlayer

# Tambah player ke whitelist
whitelist add NamaPlayer

# Hapus player dari whitelist
whitelist remove NamaPlayer

# Reload whitelist (setelah edit file whitelist.json manual)
whitelist reload

# Lihat daftar whitelist
whitelist list

# Kick player
kick NamaPlayer AlasanKick

# Ban player
ban NamaPlayer AlasanBan

# Lihat player yang sedang online
list

# Stop server dengan aman
stop
```

---

### 🎯 Playit Agent

Tunnel Minecraft ke internet via [playit.gg](https://playit.gg) — cocok jika tidak bisa buka port di router.

**Setup:**

1. Daftar di playit.gg dan buat agent baru
2. Salin secret key ke file `docker-compose.yml`:

```yaml
environment:
  - SECRET_KEY=secret_key_kamu
```

3. Jalankan:

```bash
cd playit
docker compose up -d
```

---

## 🗂️ Struktur Folder

```
server-compose/
├── adguardhome/
│   └── docker-compose.yml
├── cloudflared/
│   ├── docker-compose.yml
│   └── .env                  # TUNNEL_TOKEN
├── minecraft/
│   └── docker-compose.yml
├── netbird-client/
│   ├── docker-compose.yml
│   └── .env                  # NB_SETUP_KEY
├── npm/
│   └── docker-compose.yml
├── owncloud/
│   ├── docker-compose.yml
│   └── .env                  # Konfigurasi ownCloud
└── playit/
    └── docker-compose.yml
```

---

## 🔐 Catatan Keamanan

- Ganti semua password dan secret key default sebelum production
- Gunakan NPM untuk SSL termination agar semua traffic ke service dienkripsi
- Manfaatkan Cloudflare Tunnel + NetBird untuk akses remote yang aman tanpa expose port langsung

---

## 🛠️ Perintah Berguna

```bash
# Lihat semua container yang berjalan
docker ps

# Lihat log service tertentu
docker compose logs -f

# Stop semua container di satu stack
docker compose down

# Update image ke versi terbaru
docker compose pull && docker compose up -d

# Lihat penggunaan resource
docker stats
```
