# 🧪 Praktik 1 — Volume (Data Persistent)

## Tujuan
Membuktikan bahwa:
- Tanpa volume → data hilang saat container dihapus
- Dengan volume → data tetap aman walaupun container dihapus

---

## SKENARIO A — Tanpa Volume (Data Hilang)

### Step 1: Jalankan Postgres tanpa volume

```bash
docker run -d \
  --name pg-tanpa-volume \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine
```

**Penjelasan baris per baris:**

| Baris | Arti | Wajib? |
|-------|------|--------|
| `docker run` | Command untuk bikin & jalanin container baru | ✅ |
| `-d` | **Detached** — jalan di background, prompt langsung kembali | Opsional (kalau ga, terminal tertahan) |
| `--name pg-tanpa-volume` | Kasih nama container biar gampang dipanggil | Opsional (kalau ga, Docker auto-generate nama random) |
| `-e POSTGRES_PASSWORD=secret` | Environment variable — Postgres butuh ini, kalau ga di-set container ga akan jalan | ✅ Wajib (Postgres requirement) |
| `postgres:16-alpine` | Image: nama=postgres, tag=16-alpine (versi 16, varian Alpine Linux yang ringan) | ✅ |

### Step 2: Cek container jalan

```bash
docker ps
```

Cari container `pg-tanpa-volume` di list. Status harus **Up**.

### Step 3: Tunggu 3-5 detik biar Postgres ready

Postgres butuh waktu startup. Cek log untuk verify:

```bash
docker logs pg-tanpa-volume
```

Cari baris **"database system is ready to accept connections"** — itu tandanya Postgres siap.

### Step 4: Masuk ke Postgres CLI & bikin data

```bash
docker exec -it pg-tanpa-volume psql -U postgres
```

**Penjelasan:**

| Bagian | Arti |
|--------|------|
| `docker exec` | Jalanin command di dalam container yang lagi running |
| `-it` | `-i` (interactive, keep stdin open) + `-t` (TTY, terminal mode) — wajib untuk shell |
| `pg-tanpa-volume` | Nama container target |
| `psql -U postgres` | Command yang dijalanin: `psql` (Postgres CLI), `-U postgres` (login sebagai user `postgres`) |

Setelah masuk, kamu bakal lihat prompt `postgres=#`. Sekarang bikin tabel & insert data:

```sql
CREATE TABLE belajar(nama TEXT);
INSERT INTO belajar VALUES ('Ridho'), ('Docker'), ('Volume');
SELECT * FROM belajar;
```

Output:
```
  nama
--------
 Ridho
 Docker
 Volume
(3 rows)
```

Keluar dari psql:
```sql
\q
```

### Step 5: HAPUS container (simulasi crash atau replace)

```bash
docker rm -f pg-tanpa-volume
```

**Penjelasan:**
- `docker rm` = hapus container
- `-f` = **force**, hapus walau container masih running (auto-stop dulu)

### Step 6: Bikin container baru, cek datanya

```bash
docker run -d \
  --name pg-tanpa-volume \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

# tunggu Postgres ready
sleep 5

# cek data
docker exec -it pg-tanpa-volume psql -U postgres -c "SELECT * FROM belajar;"
```

**Penjelasan flag `-c`:**
- `-c "QUERY"` di psql = jalanin query lalu langsung exit (ga interactive)

**Hasil:**
```
ERROR: relation "belajar" does not exist
```

🚨 **DATA HILANG!** Karena container baru = filesystem baru = data lama ga ada.

### Step 7: Cleanup

```bash
docker rm -f pg-tanpa-volume
```

---

## SKENARIO B — Dengan Volume (Data Persist)

### Step 1: Jalankan Postgres DENGAN volume

```bash
docker run -d \
  --name pg-dengan-volume \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine
```

**Yang BARU di sini:**

| Bagian | Arti |
|--------|------|
| `-v pgdata:/var/lib/postgresql/data` | Mount volume |
| ↳ `pgdata` | Nama volume di Docker (otomatis dibikin kalau belum ada) |
| ↳ `:/var/lib/postgresql/data` | Path di dalam container yang mau di-link ke volume |

**Kenapa path-nya `/var/lib/postgresql/data`?**
Itu lokasi default Postgres simpan datanya. **Path ini fix** — ditentukan oleh image Postgres, bukan kita.

### Step 2: Verify volume dibuat

```bash
docker volume ls
```

Cari `pgdata` di list. Itu volume yang baru dibikin Docker.

### Step 3: Tunggu Postgres ready, lalu bikin data

```bash
sleep 5

docker exec -it pg-dengan-volume psql -U postgres -c \
  "CREATE TABLE belajar(nama TEXT); INSERT INTO belajar VALUES ('Ridho'), ('Docker'), ('Volume');"

# verify data masuk
docker exec -it pg-dengan-volume psql -U postgres -c "SELECT * FROM belajar;"
```

### Step 4: HAPUS container

```bash
docker rm -f pg-dengan-volume
```

**PENTING:** kita hapus **container**, tapi **volume `pgdata` TETAP ADA**. Volume punya lifecycle terpisah dari container.

```bash
docker volume ls
# pgdata masih ada di list ✅
```

### Step 5: Bikin container baru DENGAN volume yang sama

```bash
docker run -d \
  --name pg-dengan-volume \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine

sleep 5

# cek data
docker exec -it pg-dengan-volume psql -U postgres -c "SELECT * FROM belajar;"
```

**Hasil:**
```
  nama
--------
 Ridho
 Docker
 Volume
(3 rows)
```

🎉 **DATA MASIH ADA!** Container baru, tapi data dari volume `pgdata` ke-mount kembali → Postgres bisa baca data lama.

### Step 6: Cleanup

```bash
docker rm -f pg-dengan-volume
docker volume rm pgdata
```

⚠️ **Volume tidak otomatis terhapus saat container dihapus** — kamu harus manual `docker volume rm`. Ini fitur, bukan bug — biar data ga hilang nyangkut.

---

## 💡 Insight Penting

### 1. Container vs Volume itu beda lifecycle

```
Container A ──── (delete) ──── Container B
       │                              │
       └─────── Volume ───────────────┘
            (tetap ada, di-share)
```

### 2. Postgres path itu critical

Setiap database punya **default data path**-nya sendiri:
- **Postgres:** `/var/lib/postgresql/data`
- **MySQL:** `/var/lib/mysql`
- **MongoDB:** `/data/db`
- **Redis:** `/data`

Cek dokumentasi image-nya di Docker Hub.

### 3. Anonymous volume (jangan kebiasaan!)

Kalau kamu cuma tulis `-v /var/lib/postgresql/data` (tanpa nama di depan), Docker bikin **anonymous volume** dengan nama random. Ini bahaya:
- Susah di-track
- Gampang ke-delete tanpa sengaja
- Ga bisa di-share

**Selalu pakai named volume:** `-v pgdata:/var/lib/postgresql/data`

---

## ✅ Checklist Module 1 — Bagian A (Volume)

- [ ] Berhasil jalanin Skenario A (tanpa volume → data hilang)
- [ ] Berhasil jalanin Skenario B (dengan volume → data persist)
- [ ] Paham bedanya container vs volume lifecycle
- [ ] Bisa list, inspect, dan delete volume
- [ ] Tahu kenapa harus pakai named volume, bukan anonymous

Lanjut ke `praktik-2-network.md` setelah checklist ini selesai.
