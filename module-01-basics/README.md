# Module 01 — Volumes & Networking

## 🎯 Tujuan
Memahami 2 fondasi penting Docker:
1. **Volumes** — cara persist data
2. **Networking** — cara container saling komunikasi

---

## 📚 Bagian A — Volumes

### Masalah Dasar
Container itu **ephemeral** (sementara). Sekali dihapus → data hilang. Kalau jalanin database (Postgres/MySQL/Redis) di container, data ga akan persist.

### Analogi
- Container = Playstation
- Volume = Memory Card

PS rusak/diganti, save game di memory card aman.

### 3 Tipe Storage

| Tipe | Sintaks | Use Case |
|------|---------|----------|
| **Named Volume** ⭐ | `-v nama:/path` | Database, data persistent |
| **Bind Mount** | `-v /host/path:/container/path` | Source code (development) |
| **tmpfs** | `--tmpfs /path` | Data sensitif sementara (RAM) |

### Command Penting
```bash
docker volume create <nama>      # bikin volume
docker volume ls                 # list semua volume
docker volume inspect <nama>     # detail volume
docker volume rm <nama>          # hapus volume
docker volume prune              # hapus semua yang ga kepakai
```

---

## 📚 Bagian B — Networking

### Masalah Dasar
Container terisolasi by default. Container A mau ngomong ke Container B → butuh **network**.

### Analogi
- Container = Rumah
- Network = Perumahan
- Container Name = Alamat rumah

Rumah di perumahan sama → bisa kunjung-mengunjungi pakai alamat (nama). Beda perumahan → ga bisa.

### Tipe Network
| Tipe | Use Case |
|------|----------|
| **bridge** (default) | Default, container terisolasi tanpa custom DNS |
| **custom bridge** ⭐ | Container bisa connect pakai NAMA, bukan IP |
| **host** | Container pakai network host langsung |
| **none** | Isolasi total, no network |

### Command Penting
```bash
docker network create <nama>           # bikin custom network
docker network ls                      # list semua network
docker network inspect <nama>          # detail network
docker network rm <nama>               # hapus network
docker network connect <net> <container>     # tambah container ke network
docker network disconnect <net> <container>  # keluarin container dari network
```

### Kenapa Custom Network?
- Default `bridge` → cuma bisa connect pakai IP (susah, IP suka berubah)
- Custom `bridge` → otomatis ada **DNS** → connect pakai nama container

---

## 🧪 Praktik

Lihat folder ini untuk script praktik:
- `praktik-1-volume.sh` — Buktikan volume bikin data persist
- `praktik-2-network.sh` — 2 container saling ngobrol

---

## ✅ Checklist Selesai
- [ ] Paham bedanya named volume vs bind mount
- [ ] Bisa bikin Postgres dengan volume yang persist
- [ ] Paham kenapa custom network lebih baik dari default
- [ ] Berhasil bikin 2 container saling connect via nama
