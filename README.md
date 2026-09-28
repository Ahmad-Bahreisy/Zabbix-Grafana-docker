# Install Zabbix + Grafana di Ubuntu dengan Docker Compose

Panduan langkah demi langkah untuk memasang **Zabbix Server 6.4** (MySQL + Web UI) dan **Grafana** dengan plugin Zabbix di Ubuntu menggunakan Docker Compose, lalu menghubungkannya untuk memonitor server lain lewat **Zabbix Agent 2**.

Diuji pada Ubuntu 22.04 / 24.04 dengan Zabbix 6.4.21.

## Daftar Isi

1. [Arsitektur](#arsitektur)
2. [Prasyarat](#prasyarat)
3. [Install Docker](#1-install-docker)
4. [Siapkan project](#2-siapkan-project)
5. [Jalankan stack](#3-jalankan-stack)
6. [Login pertama](#4-login-pertama-dan-ganti-password)
7. [Hubungkan Grafana ke Zabbix](#5-hubungkan-grafana-ke-zabbix)
8. [Monitor server lain (Zabbix Agent 2)](#6-monitor-server-lain-zabbix-agent-2)
9. [Monitor jaringan lokal dengan Zabbix Proxy (opsional)](#7-monitor-jaringan-lokal-dengan-zabbix-proxy-opsional)
10. [Notifikasi email (opsional)](#8-notifikasi-email-opsional)
11. [Troubleshooting](#troubleshooting)
12. [Catatan keamanan](#catatan-keamanan)

## Arsitektur

```
[Server A]  [Server B]  [Server C]        <- Zabbix Agent 2 di tiap server
      \          |          /
       \         |         /   (port 10050)
        v        v        v
   +-----------------------------+
   |  Zabbix Server (container)  |<---- Zabbix Web UI (port 8080)
   |  MySQL 8.0 (container)      |
   +-----------------------------+
                 ^
                 | Zabbix API
                 |
        Grafana + plugin Zabbix (port 3000)
```

## Prasyarat

- Ubuntu 22.04 atau 24.04 dengan akses `sudo`
- Minimal 2 GB RAM dan 20 GB disk
- Port yang akan dipakai belum terpakai aplikasi lain: `8080` (Zabbix Web), `3000` (Grafana), `10051` (Zabbix Server)

Cek port yang sedang dipakai:

```bash
sudo ss -tulpn | grep LISTEN
```

## 1. Install Docker

Lewati bagian ini jika Docker sudah terpasang (`docker --version`).

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

Logout lalu login ulang supaya perintah `docker` bisa dipakai tanpa `sudo`, kemudian cek:

```bash
docker --version
docker compose version
```

## 2. Siapkan project

```bash
mkdir -p ~/zabbix-grafana && cd ~/zabbix-grafana
```

### `docker-compose.yml`

```yaml
services:
  mysql-server:
    image: mysql:8.0
    container_name: zabbix-mysql
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
      - --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
    restart: unless-stopped
    networks:
      - zabbix-net

  zabbix-server:
    image: zabbix/zabbix-server-mysql:alpine-6.4-latest
    container_name: zabbix-server
    environment:
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    ports:
      - "10051:10051"
    depends_on:
      - mysql-server
    restart: unless-stopped
    networks:
      - zabbix-net

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:alpine-6.4-latest
    container_name: zabbix-web
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      PHP_TZ: Asia/Jakarta
    ports:
      - "${ZABBIX_WEB_PORT:-8080}:8080"
    depends_on:
      - zabbix-server
      - mysql-server
    restart: unless-stopped
    networks:
      - zabbix-net

  zabbix-agent:
    image: zabbix/zabbix-agent:alpine-6.4-latest
    container_name: zabbix-agent
    environment:
      ZBX_HOSTNAME: "Zabbix server"
      ZBX_SERVER_HOST: zabbix-server
    depends_on:
      - zabbix-server
    restart: unless-stopped
    networks:
      - zabbix-net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
      GF_INSTALL_PLUGINS: alexanderzobnin-zabbix-app
    ports:
      - "${GRAFANA_PORT:-3000}:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - zabbix-server
    restart: unless-stopped
    networks:
      - zabbix-net

networks:
  zabbix-net:
    driver: bridge

volumes:
  mysql-data:
  grafana-data:
```

### `.env`

Nama file harus persis `.env` (dengan titik di depan). Ganti semua password dengan milik Anda sendiri.

```bash
cat > .env << 'EOF'
MYSQL_DATABASE=zabbix
MYSQL_USER=zabbix
MYSQL_PASSWORD=ganti-dengan-password-kuat
MYSQL_ROOT_PASSWORD=ganti-dengan-root-password-kuat

GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=ganti-dengan-password-grafana

# Ubah jika port bentrok dengan aplikasi lain
ZABBIX_WEB_PORT=8080
GRAFANA_PORT=3000
EOF
```

## 3. Jalankan stack

```bash
docker compose up -d
docker compose ps
```

Run pertama butuh 1-2 menit karena MySQL menginisialisasi database. Selama itu log `zabbix-server` akan menampilkan "MySQL server is not available. Waiting 5 seconds..." dan itu normal. Pantau dengan:

```bash
docker compose logs -f zabbix-server
```

Stack siap ketika `zabbix-web` berstatus `healthy`.

## 4. Login pertama dan ganti password

| Aplikasi | URL | Login |
|---|---|---|
| Zabbix Web | `http://IP-SERVER:8080` | `Admin` / `zabbix` (huruf **A** besar) |
| Grafana | `http://IP-SERVER:3000` | sesuai `GRAFANA_ADMIN_*` di `.env` |

**Ganti password Zabbix default segera:** klik **User settings** (pojok kiri bawah) lalu **Change password**.

## 5. Hubungkan Grafana ke Zabbix

1. Grafana: **Administration → Plugins**, cari **Zabbix**, klik **Enable**.
2. **Connections → Data sources → Add data source → Zabbix**.
3. Isi:
   - **URL**: `http://zabbix-web:8080/api_jsonrpc.php` (pakai nama service, bukan `localhost` dan bukan IP publik)
   - **Authentication (bagian atas)**: `No Authentication`
   - **Zabbix Connection → Username / Password**: akun Zabbix Anda
4. Klik **Save & test**. Hasil yang benar: **"Zabbix API version 6.4.x"** dengan tanda centang hijau.

### Membuat panel pertama

1. **Dashboards → New → New dashboard → Add visualization**
2. Pilih data source Zabbix, lalu isi **Group → Host → Item**.
3. Contoh item: `Linux: CPU utilization`, `Linux: Memory utilization`, `FS [/]: Space: Used, in %`, `Interface ens18: Bits received` / `Bits sent`.
4. **Apply → Save**.

Tips supaya grafik terbaca stabil: set **Unit** `Percent (0-100)`, **Min** `0`, **Max** `100`, dan tambahkan function `movingAverage` (misal `5m`) pada query.

## 6. Monitor server lain (Zabbix Agent 2)

### 6.1 Install agent di server yang mau dimonitor

Pilih link sesuai versi Ubuntu (`lsb_release -a`):

| Ubuntu | Paket rilis |
|---|---|
| 22.04 | `zabbix-release_6.4-1+ubuntu22.04_all.deb` |
| 24.04 | `zabbix-release_6.4-1+ubuntu24.04_all.deb` |

Contoh untuk 24.04:

```bash
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo apt update
sudo apt install zabbix-agent2 -y
```

### 6.2 Konfigurasi agent

```bash
sudo nano /etc/zabbix/zabbix_agent2.conf
```

Ubah tiga baris ini:

```
Server=IP-ZABBIX-SERVER
ServerActive=IP-ZABBIX-SERVER
Hostname=nama-unik-server-ini
```

Restart dan aktifkan:

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl enable zabbix-agent2
sudo systemctl status zabbix-agent2
```

Buka port 10050 untuk Zabbix server (jika memakai `ufw`):

```bash
sudo ufw allow from IP-ZABBIX-SERVER to any port 10050 proto tcp
```

### 6.3 Daftarkan host di Zabbix Web

1. **Data collection → Host groups → Create host group** (misal `Production`).
2. **Data collection → Hosts → Create host**:
   - **Host name**: harus **persis sama** (termasuk huruf besar/kecil) dengan `Hostname` di config agent
   - **Host groups**: group tadi
   - **Interfaces**: Agent, IP server target, port `10050`
   - **Templates**: `Linux by Zabbix agent`
3. Klik **Add**. Dalam 1-2 menit ikon **ZBX** di kolom Availability berubah hijau.

Uji koneksi dari server Zabbix:

```bash
docker exec -it zabbix-server zabbix_get -s IP-SERVER-TARGET -p 10050 -k agent.ping
```

Hasil `1` berarti koneksi berhasil.

### 6.4 Memonitor server Docker host itu sendiri

Agent yang dipasang native di server yang sama dengan Zabbix diakses dari container lewat gateway jaringan Docker.

1. Cek subnet dan gateway:
   ```bash
   docker network inspect zabbix-grafana_zabbix-net | grep -E "Subnet|Gateway"
   ```
2. Di `zabbix_agent2.conf`, izinkan subnet itu (contoh subnet `172.20.0.0/16`):
   ```
   Server=127.0.0.1,172.20.0.0/16
   ```
3. Jika `ufw` aktif:
   ```bash
   sudo ufw allow from 172.20.0.0/16 to any port 10050 proto tcp
   sudo systemctl restart zabbix-agent2
   ```
4. Daftarkan host dengan IP interface = **gateway Docker** (misal `172.20.0.1`), bukan IP publik.

## 7. Monitor jaringan lokal dengan Zabbix Proxy (opsional)

Zabbix Server di cloud tidak bisa menjangkau IP lokal kantor (`192.168.x.x`). Solusinya memasang **Zabbix Proxy** di jaringan lokal. Proxy mengumpulkan data dari perangkat lokal lalu mengirimnya keluar ke Zabbix Server, sehingga tidak perlu VPN dan tidak perlu membuka port masuk di kantor.

```
[Perangkat LAN] --> [Zabbix Proxy di kantor] --(outbound 10051)--> [Zabbix Server di cloud]
```

### 7.1 Daftarkan proxy di Zabbix Web

**Administration → Proxies → Create proxy**, isi **Proxy name** (misal `Proxy-Kantor`) dan **Proxy mode: Active**.

### 7.2 Jalankan proxy di server lokal

`docker-compose-proxy.yml`:

```yaml
services:
  zabbix-proxy:
    image: zabbix/zabbix-proxy-sqlite3:alpine-6.4-latest
    container_name: zabbix-proxy
    environment:
      ZBX_HOSTNAME: "Proxy-Kantor"          # harus sama dengan nama proxy di web
      ZBX_SERVER_HOST: "IP-ZABBIX-SERVER"
      ZBX_PROXYMODE: 0                      # 0 = active
      ZBX_CONFIGFREQUENCY: 60
      ZBX_PROXYOFFLINEBUFFER: 1             # simpan data yang belum terkirim maksimal 1 jam
    restart: unless-stopped
    network_mode: host
    mem_limit: 128m
    cpus: 0.3
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "2"
```

```bash
docker compose -f docker-compose-proxy.yml up -d
docker logs zabbix-proxy --tail 30
```

Di **Administration → Proxies**, kolom **Last seen** harus menunjukkan waktu beberapa detik yang lalu.

### 7.3 Tambahkan host lokal

- Di agent perangkat lokal, arahkan `Server=` dan `ServerActive=` ke **IP proxy**, bukan IP cloud.
- Saat membuat host di Zabbix Web, isi **Monitored by proxy** dengan `Proxy-Kantor`.
- Jika proxy dan agent berada di mesin yang sama, `Server=` di agent harus mencakup IP yang dipakai proxy untuk menghubungi agent (misal `Server=127.0.0.1,IP-LAN-SERVER`).

## 8. Notifikasi email (opsional)

1. **Alerts → Media types → Email**, isi SMTP server, port (`587` + `STARTTLS`), alamat email, dan kredensial. Centang **Enabled**, lalu **Update**.
2. **Users → Users → (user Anda) → Media → Add**: pilih tipe `Email`, isi alamat penerima (bisa lebih dari satu), pilih level severity.
3. **Alerts → Actions → Trigger actions → Create action**:
   - **Conditions**: `Trigger severity` **is greater than or equals** `Average` (jangan pakai `equals`, karena level High dan Disaster tidak akan terkirim)
   - **Operations**: kirim ke user Anda lewat `Email`
4. Uji dari halaman Media types dengan tombol **Test**.

## Troubleshooting

| Gejala | Penyebab dan solusi |
|---|---|
| `The "MYSQL_USER" variable is not set` | File `.env` tidak terbaca. Pastikan namanya persis `.env`, bukan `env` atau `.env.txt`, lalu `docker compose down && docker compose up -d`. |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Port bentrok. Ubah `ZABBIX_WEB_PORT` di `.env`, cek dengan `sudo ss -tulpn \| grep LISTEN`. |
| Log: `MySQL server is not available. Waiting 5 seconds` | Normal saat inisialisasi pertama. Tunggu 1-2 menit. |
| Login Zabbix: "account is temporarily blocked" | Terlalu banyak salah login. Tunggu beberapa saat, lalu coba `Admin` / `zabbix`. |
| Login Grafana gagal dengan password dari `.env` | Variabel `GF_SECURITY_ADMIN_*` hanya berlaku saat volume masih kosong. Coba `admin`/`admin`, atau reset: `docker exec -it grafana grafana-cli admin reset-admin-password PasswordBaru`. |
| Data source Zabbix tidak muncul saat "Add data source" | Enable plugin di **Administration → Plugins**, lalu `docker compose restart grafana` dan hard refresh browser (`Ctrl+Shift+R`). |
| Error `Post "": unsupported protocol scheme` | Field URL data source kosong. Isi ulang `http://zabbix-web:8080/api_jsonrpc.php`, lalu **Save & test**. |
| `wget` ke `api_jsonrpc.php` mengembalikan `412 Precondition Failed` | Normal. Artinya jaringan antar container terhubung. Endpoint ini butuh request POST JSON. |
| Item `Interface ...` atau `FS [...]` tidak ada di dropdown | Discovery belum jalan (interval default 1 jam). **Data collection → Hosts → Discovery**, centang rule, klik **Execute now**. |
| Semua panel Grafana "No data" setelah VPS restart | `docker compose restart grafana`, hard refresh browser, cek **Save & test** di data source. |
| ZBX merah, `zabbix_get` menggantung (timeout) | Firewall memblokir. Izinkan port 10050 dari IP Zabbix Server (atau subnet Docker, lihat 6.4). |
| ZBX merah padahal `zabbix_get` berhasil | `Hostname` di agent tidak sama dengan host name di Zabbix Web, atau `Server=` tidak memuat IP yang menghubungi agent. |
| Host bawaan "Zabbix server" selalu merah | Interface default `127.0.0.1` tidak menemukan agent di container. Ubah interface-nya ke DNS `zabbix-agent` port `10050`, atau nonaktifkan host itu jika sudah memonitor host lewat agent native. |
| CentOS 7: `yum` gagal (`mirrorlist.centos.org`) | CentOS 7 sudah EOL. Arahkan repo ke `vault.centos.org` lalu install `zabbix-agent2` dari repo `rhel/7`. |

## Catatan keamanan

- Jangan meng-commit file `.env` ke Git. Tambahkan ke `.gitignore`:
  ```
  .env
  ```
- Ganti password bawaan Zabbix (`Admin` / `zabbix`) dan gunakan password yang kuat untuk MySQL dan Grafana.
- Batasi akses port 8080 dan 3000 dengan firewall atau reverse proxy HTTPS bila server terbuka ke internet.
- Buka port 10050 hanya untuk IP Zabbix Server, jangan ke `0.0.0.0/0`.
- Pertimbangkan mode active check atau Zabbix Proxy supaya server yang dimonitor tidak perlu membuka port masuk.

## Referensi

- [Dokumentasi Zabbix](https://www.zabbix.com/documentation/current/)
- [Zabbix Docker images](https://hub.docker.com/u/zabbix)
- [Plugin Zabbix untuk Grafana](https://grafana.com/grafana/plugins/alexanderzobnin-zabbix-app/)
