# 🐳 Docker Theory & Command Reference (Lengkap)

Dokumen ini adalah catatan lengkap tentang Docker — dari konsep paling dasar sampai breakdown setiap command line by line.

---

## DAFTAR ISI

1. [Apa itu Docker?](#1-apa-itu-docker)
2. [Arsitektur Docker](#2-arsitektur-docker)
3. [Konsep Inti (Image, Container, Volume, Network)](#3-konsep-inti)
4. [Anatomy Setiap Command](#4-anatomy-setiap-command)
5. [Volumes — Deep Dive](#5-volumes--deep-dive)
6. [Networking — Deep Dive](#6-networking--deep-dive)
7. [Glossary Istilah](#7-glossary-istilah)

---

## 1. APA ITU DOCKER?

### Definisi Formal
Docker adalah **platform open-source** untuk membungkus aplikasi beserta semua dependensinya ke dalam unit standar bernama **container**, sehingga aplikasi bisa berjalan konsisten di environment manapun.

### Sejarah Singkat
- **2013** — Docker dirilis oleh Solomon Hykes (perusahaan dotCloud)
- **2014** — Docker 1.0 release, jadi standar industri
- **2017** — Docker Inc. fokus ke enterprise, container runtime jadi open standard (OCI)

### Masalah yang Diselesaikan

| Masalah | Solusi Docker |
|---------|---------------|
| "Di laptop saya jalan!" | Container = environment identik di semua mesin |
| Setup ribet (install DB, Redis, dll) | `docker run postgres` → 5 detik jalan |
| VM boros resource | Container ringan, share kernel host |
| Deploy lambat & error-prone | Build sekali, deploy dimana saja |
| Konflik dependency antar app | Tiap app di container terisolasi |

### Container vs Virtual Machine (VM)

```
┌─────────────────────────────────────┐  ┌─────────────────────────────────────┐
│ VIRTUAL MACHINE                     │  │ DOCKER CONTAINER                    │
├─────────────────────────────────────┤  ├─────────────────────────────────────┤
│  ┌─────┐ ┌─────┐ ┌─────┐            │  │  ┌─────┐ ┌─────┐ ┌─────┐            │
│  │App A│ │App B│ │App C│            │  │  │App A│ │App B│ │App C│            │
│  ├─────┤ ├─────┤ ├─────┤            │  │  ├─────┤ ├─────┤ ├─────┤            │
│  │ OS  │ │ OS  │ │ OS  │ (1-5 GB)   │  │  │ Bin │ │ Bin │ │ Bin │ (50-200 MB)│
│  └─────┘ └─────┘ └─────┘            │  │  └─────┴─┴─────┴─┴─────┘            │
│  ┌──────────────────────┐           │  │  ┌──────────────────────┐           │
│  │     HYPERVISOR        │           │  │  │   DOCKER ENGINE      │           │
│  └──────────────────────┘           │  │  └──────────────────────┘           │
│  ┌──────────────────────┐           │  │  ┌──────────────────────┐           │
│  │     HOST OS           │           │  │  │     HOST OS          │           │
│  └──────────────────────┘           │  │  └──────────────────────┘           │
│  ┌──────────────────────┐           │  │  ┌──────────────────────┐           │
│  │     HARDWARE          │           │  │  │     HARDWARE         │           │
│  └──────────────────────┘           │  │  └──────────────────────┘           │
└─────────────────────────────────────┘  └─────────────────────────────────────┘

VM:        tiap aplikasi punya OS lengkap → BERAT
Container: share OS kernel host → RINGAN
```

| Aspek | VM | Container |
|-------|----|-----------|
| Ukuran | 1-20 GB | 50-200 MB |
| Startup | Menit | Detik |
| Isolasi | Hardware-level | Process-level |
| Resource | Boros | Hemat |
| Per host muat | 5-20 VM | 100-1000 container |

---

## 2. ARSITEKTUR DOCKER

### Komponen Utama

```
┌──────────────────────────────────────────────────────────┐
│                    USER (Anda)                           │
│                                                          │
│   $ docker run nginx                                     │
│         │                                                │
│         ▼                                                │
│   ┌──────────────┐                                       │
│   │  Docker CLI  │  ← command line tool                  │
│   └──────┬───────┘                                       │
│          │ REST API                                      │
│          ▼                                               │
│   ┌──────────────────────────────────────────┐           │
│   │      DOCKER DAEMON (dockerd)             │           │
│   │  ┌────────────┬──────────┬────────────┐  │           │
│   │  │ Container  │  Image   │  Volume    │  │           │
│   │  │ Manager    │  Builder │  Manager   │  │           │
│   │  ├────────────┼──────────┼────────────┤  │           │
│   │  │ Network    │  Storage │  Plugin    │  │           │
│   │  │ Manager    │  Driver  │  System    │  │           │
│   │  └────────────┴──────────┴────────────┘  │           │
│   └──────────────────┬───────────────────────┘           │
│                      │                                   │
│                      ▼                                   │
│   ┌──────────────────────────────────────────┐           │
│   │    REGISTRY (Docker Hub / GHCR / dll)    │           │
│   │       (tempat image disimpan)            │           │
│   └──────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────┘
```

### Penjelasan Tiap Komponen

#### A. Docker CLI (`docker`)
- Command line yang kamu pakai (`docker run`, `docker ps`, dll)
- **Bukan** yang menjalankan container — cuma jembatan ke daemon
- Komunikasi dengan daemon via REST API

#### B. Docker Daemon (`dockerd`)
- **Otak Docker** — proses background yang sebenarnya menjalankan container
- Manage: container, image, volume, network, plugin
- Di Mac & Windows → jalan di dalam **Docker Desktop** (which runs a tiny Linux VM)
- Di Linux → jalan native sebagai service

#### C. Registry
- Tempat menyimpan image (kayak GitHub untuk code)
- **Docker Hub** = registry default & paling besar
- Alternatif: GitHub Container Registry (GHCR), AWS ECR, Google GCR, Harbor

### Flow saat `docker run nginx`

```
1. CLI kirim request ke daemon
2. Daemon cek: image "nginx" ada di local?
   - Ada  → langsung jalan
   - Ga ada → pull dari Docker Hub
3. Daemon download semua layer image
4. Daemon bikin container baru dari image
5. Container jalan
6. Daemon return container ID ke CLI
7. CLI tampilkan ke user
```

---

## 3. KONSEP INTI

### A. IMAGE 📦

**Definisi:** Template/blueprint read-only untuk membuat container.

**Karakteristik:**
- **Immutable** (ga bisa diubah)
- **Layered** (terdiri dari banyak layer)
- Disimpan di registry
- Bisa di-share & versioned

**Analogi:** Image = **resep kue** (read-only). Container = kue yang dibuat dari resep itu.

#### Struktur Layer Image

```
┌─────────────────────────────────────┐
│  Layer 5: CMD ["node", "app.js"]    │  ← top layer (latest)
├─────────────────────────────────────┤
│  Layer 4: COPY app.js .             │
├─────────────────────────────────────┤
│  Layer 3: RUN npm install           │
├─────────────────────────────────────┤
│  Layer 2: COPY package.json .       │
├─────────────────────────────────────┤
│  Layer 1: FROM node:20-alpine       │  ← base layer
└─────────────────────────────────────┘
```

**Tiap instruksi di Dockerfile = 1 layer.** Layer di-cache → kalau cuma layer atas yang berubah, layer bawah tetap pakai cache (build kedua jauh lebih cepat).

#### Image Tag

Format: `nama:tag`

```
nginx                    # nama=nginx, tag=latest (default)
nginx:1.25               # tag spesifik versi
nginx:1.25-alpine        # variant Alpine (lebih kecil)
node:20-alpine
postgres:16
mycompany/myapp:v1.2.3   # custom image
```

**⚠️ Best practice:** JANGAN pakai `:latest` di production — versi bisa berubah tanpa kamu sadar. Selalu pin ke versi spesifik.

### B. CONTAINER 🚢

**Definisi:** Image yang lagi "hidup" — process yang sedang berjalan.

**Karakteristik:**
- **Mutable** (bisa diubah, tapi perubahan hilang saat dihapus)
- **Ephemeral** (sementara — by design)
- Punya filesystem, network, process tree sendiri
- 1 image bisa bikin banyak container

**Analogi:** Container = **kue jadi**. 1 resep (image) → bisa bikin banyak kue.

#### Lifecycle Container

```
   ┌──────────┐
   │ CREATED  │ ← docker create (belum jalan)
   └────┬─────┘
        │ docker start
        ▼
   ┌──────────┐
   │ RUNNING  │ ← docker run (= create + start)
   └────┬─────┘
        │
   ┌────┴────┬──────────┬───────────┐
   │         │          │           │
   ▼         ▼          ▼           ▼
PAUSED   STOPPED    KILLED    EXITED (process selesai)
   │         │          │           │
   │         │          │           │
   └─────────┴──────────┴───────────┘
                  │
                  ▼
              ┌──────────┐
              │ REMOVED  │ ← docker rm
              └──────────┘
```

#### Container vs Process

Container itu sebenernya **process di host**, tapi terisolasi pakai Linux features:
- **Namespaces** — isolasi PID, network, mount, user, IPC
- **Cgroups** — batasi CPU, memory, disk I/O
- **Union Filesystem** — efisien layer image

### C. VOLUME 💾

**Definisi:** Storage persistent yang di-manage Docker, terpisah dari container.

**Kenapa butuh volume?**
- Container ephemeral → hapus container = hapus data
- Database harus persist
- Share data antar container

(Detail lengkap di section 5)

### D. NETWORK 🌐

**Definisi:** Cara container saling berkomunikasi.

**Kenapa butuh network?**
- Container terisolasi by default
- Microservices → 5+ container saling ngobrol
- App ↔ Database → harus connect

(Detail lengkap di section 6)

### E. DOCKERFILE 📝

**Definisi:** Text file berisi resep cara build image.

```dockerfile
# Comment dimulai dengan #

FROM node:20-alpine          # base image
WORKDIR /app                 # working directory di container
COPY package.json .          # copy file dari host ke container
RUN npm install              # jalanin command saat build
EXPOSE 3000                  # dokumentasi: container listen di port ini
CMD ["node", "app.js"]       # default command saat container run
```

**Setiap baris = 1 instruksi = 1 layer.**

### F. DOCKER COMPOSE 🎼

**Definisi:** Tool untuk define & run multi-container apps via 1 file YAML.

Bukannya tulis 5 command `docker run` panjang, cukup tulis `docker-compose.yml` lalu `docker compose up`.

(Detail di Module 02)

---

## 4. ANATOMY SETIAP COMMAND

### `docker run` — Jalanin Container Baru

```bash
docker run [FLAGS] IMAGE [COMMAND] [ARGS]
```

**Format umum:**
```bash
docker run -d --name myapp -p 3000:3000 -v data:/app/data myimage:tag
```

#### Flag-Flag yang Sering Dipakai

| Flag | Fungsi | Contoh |
|------|--------|--------|
| `-d` | Detached (background) | `docker run -d nginx` |
| `--name <nama>` | Kasih nama container | `--name webku` |
| `-p <host>:<container>` | Port mapping | `-p 8080:80` |
| `-e KEY=VALUE` | Set environment variable | `-e DB_PASS=secret` |
| `-v <vol>:<path>` | Mount volume | `-v data:/var/lib/data` |
| `-it` | Interactive + terminal | `docker run -it ubuntu bash` |
| `--rm` | Auto-delete saat container stop | `docker run --rm nginx` |
| `--network <nama>` | Connect ke network | `--network mynet` |
| `--restart <policy>` | Auto-restart policy | `--restart unless-stopped` |
| `-w <path>` | Working directory di container | `-w /app` |
| `--user <user>` | Jalan sebagai user (security) | `--user 1000` |
| `-m <size>` | Limit memory | `-m 512m` |
| `--cpus <num>` | Limit CPU | `--cpus 0.5` |

#### Contoh Lengkap dengan Penjelasan Per-Baris

```bash
docker run \
  -d \
  --name pg-belajar \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_USER=ridho \
  -e POSTGRES_DB=mydb \
  -v pgdata:/var/lib/postgresql/data \
  --restart unless-stopped \
  postgres:16-alpine
```

**Baris per baris:**

| Baris | Arti |
|-------|------|
| `docker run \` | Mulai command, `\` = lanjut ke baris bawah |
| `-d \` | Jalan di background (kembali ke prompt setelah container start) |
| `--name pg-belajar \` | Nama container = "pg-belajar" |
| `-p 5432:5432 \` | Forward port 5432 host → 5432 container (Postgres default port) |
| `-e POSTGRES_PASSWORD=secret \` | Set env var: password Postgres |
| `-e POSTGRES_USER=ridho \` | Set env var: username (default `postgres` kalau ga di-set) |
| `-e POSTGRES_DB=mydb \` | Set env var: nama database awal |
| `-v pgdata:/var/lib/postgresql/data \` | Mount volume `pgdata` ke folder data Postgres |
| `--restart unless-stopped \` | Auto-restart kecuali kamu stop manual |
| `postgres:16-alpine` | Image yang dipakai |

**Restart Policy:**
- `no` (default) — ga auto-restart
- `on-failure` — restart kalau exit error
- `always` — selalu restart, bahkan kalau di-stop manual
- `unless-stopped` — restart, kecuali kamu stop manual ⭐

---

### `docker ps` — List Container

```bash
docker ps           # cuma yang lagi jalan
docker ps -a        # SEMUA (termasuk yang stopped)
docker ps -q        # cuma container ID (untuk scripting)
docker ps --format "table {{.Names}}\t{{.Status}}"  # custom format
```

**Output kolom:**
```
CONTAINER ID    IMAGE              COMMAND                 CREATED         STATUS         PORTS                    NAMES
abc123def       postgres:16        "docker-entrypoint…"    2 hours ago     Up 2 hours     0.0.0.0:5432->5432/tcp   pg-belajar
```

| Kolom | Arti |
|-------|------|
| CONTAINER ID | ID unik container (bisa dipakai pengganti nama) |
| IMAGE | Image dasar container |
| COMMAND | Command yang dijalanin (singkat) |
| CREATED | Kapan container dibuat |
| STATUS | Up/Exited/Restarting/Paused |
| PORTS | Port mapping aktif |
| NAMES | Nama container (auto-generated kalau ga set) |

---

### `docker exec` — Jalanin Command di Container yang Lagi Jalan

```bash
docker exec [FLAGS] CONTAINER COMMAND
```

**Use case paling umum:**

```bash
# Masuk ke shell container
docker exec -it myapp bash
docker exec -it myapp sh    # kalau alpine (no bash)

# Jalanin SQL di Postgres
docker exec -it pg-belajar psql -U postgres

# Cek file di dalam container
docker exec myapp ls /app

# Jalanin script
docker exec myapp /app/migrate.sh
```

**Flag penting:**
- `-i` — interactive (keep stdin open)
- `-t` — TTY (terminal mode)
- `-it` — kombinasi keduanya (untuk shell access)

---

### `docker logs` — Lihat Output Container

```bash
docker logs <container>            # semua log
docker logs -f <container>         # follow (streaming)
docker logs --tail 100 <container> # 100 baris terakhir
docker logs --since 10m <container>  # 10 menit terakhir
docker logs -t <container>         # dengan timestamp
```

**`-f` itu wajib hafal** — sama kayak `tail -f` Linux.

---

### `docker stop` & `docker start` & `docker restart`

```bash
docker stop <container>     # graceful stop (kasih 10 detik untuk shutdown)
docker stop -t 30 <c>       # kasih 30 detik
docker kill <container>     # force kill (signal SIGKILL)

docker start <container>    # nyalain container yang udah dibuat tapi stopped
docker restart <container>  # stop lalu start lagi
```

**Bedanya stop vs kill:**
- `stop` → kirim SIGTERM (graceful, app bisa cleanup)
- `kill` → kirim SIGKILL (paksa, app langsung mati)

---

### `docker rm` — Hapus Container

```bash
docker rm <container>      # cuma yang udah stopped
docker rm -f <container>   # FORCE (auto-stop dulu)
docker rm -v <container>   # hapus + delete anonymous volumes
docker rm $(docker ps -aq) # hapus SEMUA container (hati-hati!)
```

---

### `docker images` & `docker rmi` — Manage Image

```bash
docker images              # list semua image
docker images -q           # cuma image ID
docker rmi <image>         # hapus image
docker rmi -f <image>      # force hapus
docker image prune         # hapus image dangling (no tag)
docker image prune -a      # hapus SEMUA image yg ga dipakai container
```

---

### `docker pull` & `docker push`

```bash
docker pull nginx:1.25     # download image dari registry
docker push myuser/myapp   # upload image ke registry (perlu login dulu)
docker login               # login ke Docker Hub
```

---

### `docker build` — Build Image dari Dockerfile

```bash
docker build [FLAGS] PATH
```

**Contoh:**
```bash
docker build -t myapp:v1 .       # build dari Dockerfile di current dir
docker build -t myapp:v1 -f Dockerfile.prod .  # custom Dockerfile name
docker build --no-cache .         # ga pake cache
docker build --build-arg ENV=prod . # pass build argument
```

| Flag | Fungsi |
|------|--------|
| `-t <nama:tag>` | Kasih tag ke image hasil build |
| `-f <path>` | Path ke Dockerfile (default: `Dockerfile`) |
| `--no-cache` | Force rebuild semua layer |
| `--build-arg KEY=VAL` | Pass build-time variable |
| `--target <stage>` | Build sampai stage tertentu (multi-stage) |

---

### `docker inspect` — Lihat Detail Object

```bash
docker inspect <container>   # detail container (JSON)
docker inspect <image>       # detail image
docker inspect <volume>      # detail volume
docker inspect <network>     # detail network

# Filter output (pakai Go template)
docker inspect -f '{{.NetworkSettings.IPAddress}}' myapp
docker inspect -f '{{.State.Status}}' myapp
```

---

### `docker stats` — Monitor Resource Usage

```bash
docker stats              # monitor real-time (CPU, memory, network)
docker stats myapp        # cuma 1 container
docker stats --no-stream  # 1x snapshot doang
```

---

### `docker system` — Cleanup

```bash
docker system df           # lihat disk usage Docker
docker system prune        # hapus: stopped containers, dangling images, unused networks
docker system prune -a     # PLUS image yang ga dipakai container manapun
docker system prune -a --volumes  # PLUS volume (HATI-HATI: data hilang!)
```

---

## 5. VOLUMES — DEEP DIVE

### Mengapa Volume Penting?

Container itu **ephemeral** — saat dihapus, semua perubahan filesystem hilang. Ini bagus untuk **stateless app** (web server, API), tapi MASALAH untuk:
- Database (Postgres, MySQL, MongoDB)
- File upload user
- Cache yang harus persist
- Log yang harus disimpan

### 3 Jenis Storage di Docker

#### A. Named Volume (RECOMMENDED untuk database)

```bash
docker run -v myvolume:/data nginx
```

**Karakteristik:**
- Di-manage Docker (lokasi disembunyikan)
- Lokasi default: `/var/lib/docker/volumes/<nama>/_data`
- Portable (bisa backup, restore)
- Bisa di-share antar container
- Performa tinggi (native filesystem)

**Kapan pakai?**
- ✅ Data database
- ✅ Data persistent yang ga perlu kamu akses langsung dari host
- ✅ Production

#### B. Bind Mount (untuk development source code)

```bash
docker run -v /home/user/code:/app nginx
docker run -v "$(pwd)":/app nginx     # current directory
```

**Karakteristik:**
- Mount folder host **langsung** ke container
- Edit file di host → langsung ke-reflect di container
- Lokasi terserah kamu (path host eksplisit)
- Bisa baca-tulis file dari kedua sisi

**Kapan pakai?**
- ✅ Development (hot reload source code)
- ✅ Akses log dari host
- ❌ JANGAN untuk production (security & portability issue)

#### C. tmpfs Mount (jarang dipakai)

```bash
docker run --tmpfs /tmp nginx
```

**Karakteristik:**
- Disimpan di **RAM**, bukan disk
- Hilang saat container stop
- Sangat cepat
- Untuk data sensitif sementara

### Anatomy `-v` Flag

```bash
-v <SOURCE>:<TARGET>:<OPTIONS>
```

**Contoh:**
```bash
-v pgdata:/var/lib/postgresql/data            # named volume
-v /host/path:/container/path                 # bind mount
-v /host/path:/container/path:ro              # read-only
-v "$(pwd)":/app                              # current directory
-v ~/data:/app/data                           # home directory
```

**Options:**
- `ro` — read-only (container ga bisa write)
- `rw` — read-write (default)
- `Z` — SELinux private label (Linux)

### Atau Pakai `--mount` (Modern Syntax)

```bash
docker run --mount type=volume,source=pgdata,target=/var/lib/postgresql/data postgres
docker run --mount type=bind,source=/host/path,target=/container/path nginx
docker run --mount type=tmpfs,target=/tmp nginx
```

`--mount` lebih verbose tapi **lebih jelas & explicit** — direkomendasikan untuk script production.

### Command Volume Lengkap

```bash
docker volume create myvolume                 # bikin volume manual
docker volume ls                              # list semua volume
docker volume ls -f dangling=true             # cuma yang ga dipakai
docker volume inspect myvolume                # detail volume
docker volume rm myvolume                     # hapus volume
docker volume prune                           # hapus semua yg ga dipakai
docker volume rm $(docker volume ls -q)       # hapus SEMUA (hati-hati!)
```

### Backup & Restore Volume

```bash
# Backup volume ke tar.gz
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/myvolume.tar.gz -C /data .

# Restore volume dari tar.gz
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/myvolume.tar.gz -C /data
```

---

## 6. NETWORKING — DEEP DIVE

### Mengapa Networking Penting?

Container terisolasi by default. Untuk microservices, web app + database, dll → butuh network.

### Tipe Network

#### A. Bridge Network (Default)

Container baru otomatis masuk ke `bridge` network default.

```bash
docker network ls
# bridge    ← default
# host
# none
```

**Karakteristik default bridge:**
- Container dapat IP private (172.17.0.x)
- Container bisa akses internet (NAT)
- ❌ Container TIDAK bisa connect pakai NAMA (cuma IP)
- ❌ Tidak ada DNS

#### B. Custom Bridge Network ⭐ RECOMMENDED

```bash
docker network create mynet
```

**Karakteristik:**
- Container dapat IP private (subnet beda)
- ✅ Container BISA connect pakai NAMA (built-in DNS)
- ✅ Lebih aman (terisolasi dari default bridge)
- ✅ Bisa di-attach/detach container

**Untuk microservices, SELALU pakai custom bridge.**

#### C. Host Network

```bash
docker run --network host nginx
```

**Karakteristik:**
- Container pakai network stack host LANGSUNG
- Tidak ada isolasi network
- Performa tertinggi
- ❌ Cuma di Linux (tidak full-supported di Mac/Windows)

#### D. None Network

```bash
docker run --network none alpine
```

Container ga punya network sama sekali. Untuk isolasi total (misal: scheduled task yang ga butuh internet).

### Port Mapping vs Network

⚠️ **JANGAN BINGUNG** dua hal ini:

#### Port Mapping (`-p`)
**Untuk akses dari HOST → container**

```bash
docker run -p 8080:80 nginx
# host port 8080 → container port 80
# buka http://localhost:8080 di laptop
```

#### Network (`--network`)
**Untuk akses dari CONTAINER → container lain**

```bash
docker network create mynet
docker run -d --name web --network mynet nginx
docker run -d --name api --network mynet myapi
# api bisa akses web pakai: http://web (built-in DNS)
```

### Anatomy Port Mapping

```bash
-p <HOST_PORT>:<CONTAINER_PORT>
```

```bash
-p 8080:80              # host:container
-p 127.0.0.1:8080:80    # bind ke localhost only (security)
-p 8080:80/udp          # protocol UDP
-p 8080-8090:80-90      # range
-P                      # publish ALL exposed ports random ke host
```

### Command Network Lengkap

```bash
docker network create mynet                       # bikin network
docker network create --driver bridge mynet       # spesifik driver
docker network create --subnet 172.20.0.0/16 mynet  # custom subnet
docker network ls                                 # list semua network
docker network inspect mynet                      # detail (lihat container)
docker network rm mynet                           # hapus
docker network prune                              # hapus yang ga dipakai
docker network connect mynet myapp                # tambah container
docker network disconnect mynet myapp             # keluarin container
```

### DNS di Custom Network

Container bisa di-resolve pakai:
1. **Nama container** (`--name`)
2. **Network alias** (`--network-alias`)

```bash
docker network create mynet

docker run -d --name redis --network mynet redis
docker run -d --name api --network mynet \
  -e REDIS_HOST=redis \      # ← langsung pakai nama container
  myapi
```

Di dalam container `api`, hostname `redis` otomatis resolve ke IP container redis.

---

## 7. GLOSSARY ISTILAH

| Istilah | Arti |
|---------|------|
| **Image** | Template read-only untuk bikin container |
| **Container** | Image yang sedang berjalan (running instance) |
| **Volume** | Storage persistent yang di-manage Docker |
| **Network** | Jaringan virtual untuk container saling konek |
| **Registry** | Tempat menyimpan image (Docker Hub, GHCR, dll) |
| **Repository** | Koleksi image dengan nama sama, beda tag |
| **Tag** | Versi/variant image (`:1.25`, `:alpine`, `:latest`) |
| **Layer** | Slice dari image (1 instruksi Dockerfile = 1 layer) |
| **Dockerfile** | Resep cara build image |
| **Build context** | Folder yang di-copy saat `docker build` |
| **Daemon** | Process Docker yang jalan di background (`dockerd`) |
| **CLI** | Command line tool (`docker`) |
| **Compose** | Tool untuk multi-container app |
| **Bind mount** | Mount folder host ke container |
| **Named volume** | Volume di-manage Docker |
| **tmpfs** | Mount in-memory (RAM) |
| **Bridge** | Default network type (virtual switch) |
| **Port mapping** | Forward port host ke container |
| **Detached** | Container jalan di background (`-d`) |
| **Ephemeral** | Sementara, tidak persistent |
| **Stateless** | App tanpa data persistent (bisa di-restart bebas) |
| **Stateful** | App dengan data persistent (database, dll) |
| **Orchestration** | Manage banyak container di banyak server (K8s, Swarm) |
| **Microservices** | Arsitektur app: pecah jadi banyak service kecil |
| **Healthcheck** | Cara Docker tau container sehat atau ga |
| **Restart policy** | Aturan auto-restart container |
| **OCI** | Open Container Initiative — standar container industri |

---

**End of theory document.** Buka file ini kapan aja kalau lupa command/konsep!
