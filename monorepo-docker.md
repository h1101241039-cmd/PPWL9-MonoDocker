# Monorepo - Docker
Kenapa Docker?
- Dedicated Environment: membungkus seluruh dependency  & config sesuai dengan aplikasi.
- Karena monorepo memiliki berbagai kelompok stack workspace, kita bisa membungkus tiap workspace dengan docker agar lingkungan khusus yang sesuai dengan kebutuhan tiap workspace.

Referensi yang berharga untuk mulai belajar docker:
- [Docker Tutorial You Need To Get Started](https://youtu.be/DQdB7wFEygo?si=WKrwiZ1hT_G2EogT)
- [The detailed intro to Docker started](https://youtu.be/Ud7Npgi6x8E?si=pntk0m4QedQfVJFB)

## 0. Prerequisite
- Selesaikan Fitur API Google Classroom yang ada di `monorepo-3.md`
- **Install Docker CLI**: disarankan pakai Docker CLI tanpa Docker Desktop agar lebih ringan, tapi jika sudah install Docker Desktop tidak apa-apa karena sudah include Docker CLI. 
    - Bagi pengguna windows yang bermasalah penyimpanan dan ingin setup Docker yang ringan, bisa [install WSL](https://learn.microsoft.com/en-us/windows/wsl/install) (disarankan pakai distro Debian karena paling ringan), lalu di dalam WSL bisa [install Docker CLI](https://docs.docker.com/engine/install/) (Docker Engine).

## 1. Setup File
Struktur file baru:
```
monorepo/
├── docker-compose.yml    ← action untuk memanggil dockerfile
├── apps/             
│   ├── backend/
│   │   ├── Dockerfile    ← Setup docker (dependency, file transfer, & script command)
│   │   └── prisma
│   │       └──db.ts      ← modifikasi! perbaiki path
│   └── frontend/
│       ├── Dockerfile    ← Setup docker (dependency, file transfer, & script command) 
│       └── nginx.conf    ← konfigurasi web untuk opsi static nginx (production)
└── packages/
    └── shared/
```

### 1.1. **docker-compose.yml**
Konfigurasi linkungan utama docker project, berisi **name**, **services**, & **volumes**.
```yml
name: monorepo

services:
  backend:
    build:
      context: .                        # root = context supaya bisa akses packages/shared
      dockerfile: apps/backend/Dockerfile
    ports:
      - "3000:3000"
    volumes:
      - db_data:/app/apps/backend/prisma
    restart: unless-stopped

  frontend:
    build:
      context: .
      dockerfile: apps/frontend/Dockerfile
    ports:
      - "5173:80"
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  db_data:
```

### 1.2. **apps/backend/Dockerfile**
- Build menggunakan template dari `oven/bun:1`
- Copy file project ke dalam lingkungan virtual dari Docker, lalu run script 
- Run script instalasi, Genereate, & runtime (sekarang masih dari src/, anda bisa edit dengan run build dulu lalu pakai dist/)

```Dockerfile
FROM oven/bun:1 AS base
WORKDIR /app

# Copy workspace manifests dulu (layer caching)
COPY package.json bun.lock ./
COPY apps/backend/package.json ./apps/backend/
COPY packages/shared/package.json ./packages/shared/

RUN bun install

# Copy source
COPY apps/backend ./apps/backend
COPY packages/shared ./packages/shared

WORKDIR /app/apps/backend

# Generate Prisma client
RUN bunx prisma generate

EXPOSE 3000
CMD ["bun", "run", "src/index.ts"]
```

### 1.3. **apps/backend/prisma/db.ts**
Modifikasi path file database agar realatif dengan file kode, jika tidak didefinisi di env.
```ts
import { PrismaClient } from "../src/generated/prisma/client";
import { PrismaLibSql } from "@prisma/adapter-libsql";
import path from "path";

const dbUrl = process.env.DATABASE_URL || `file:${path.resolve(__dirname, "../dev.db")}`;

const adapter = new PrismaLibSql({ url: dbUrl, authToken: process.env.DB_AUTH_TOKEN });
export const prisma = new PrismaClient({ adapter });
```

### 1.4. **apps/frontend/Dockerfile**
- Build menggunakan template dari `oven/bun:1`
- Copy file project ke dalam lingkungan virtual dari Docker, lalu run script 
- Run script instalasi, Build, & runtime khusus.
- runtime pakai `nginx:alpine` 

```Dockerfile
# Stage 1: build
FROM oven/bun:1 AS builder
WORKDIR /app

COPY package.json bun.lock ./
COPY apps/frontend/package.json ./apps/frontend/
COPY packages/shared/package.json ./packages/shared/

RUN bun install

COPY apps/frontend ./apps/frontend
COPY packages/shared ./packages/shared

WORKDIR /app/apps/frontend

RUN bun run build

# Stage 2: serve dengan nginx
FROM nginx:alpine
COPY --from=builder /app/apps/frontend/dist /usr/share/nginx/html
COPY apps/frontend/nginx.conf /etc/nginx/conf.d/default.conf

CMD sh -c "echo '🦊 Frontend → http://localhost:5173' && nginx -g 'daemon off;'"

EXPOSE 80
```

### 1.5. **apps/frontend/nginx.conf**
konfigurasi untuk runtime `nginx` 
```conf
server {
    listen 80;

    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls ke backend container
    location /api/ {
        proxy_pass http://backend:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 2. CLI
```bash
# Jika pakai window wsl, hapus file bun.lock, lalu jalankan `bun install --lockfile-only` di WSL (menghasilkan file bun.lock). supaya path di file lock pakai forward slash "/", bukan backslash "\".

# test dulu development (jika di wsl pakai )
bun dev

# jika dev berhasil, sekarang test Build & Start
docker compose build
docker compose up

# ----
# Jika dapat error, lalu kamu buat perubahan tapi ketika `compose up` kode tidak berubah, coba build tanpa cache
docker compose build --no-cache

# jika dapat error port is already allocated, run ini untuk stop all docker process
docker ps -q | xargs docker stop

# cek jika ada port yang running
docker ps --format "table {{.Names}}\t{{.Ports}}"
``` 

### Fix error `bun dev`

Jika dapat error `Port 5173 is already in use` walau sudah pakai `kill-port`, cukup ganti port nya jadi `5174`, dst. Atau pakai Powershell (run as administrator):
```bash
# lihat siapa yang memakai port 5173: 
netstat -ano | findstr :5173

# Jika muncul angka di kolom paling kanan (misal: 1234), matikan prosesnya:
taskkill /F /PID 1234
```

---

## Final
Kumpulkan:
1. **Link**: repo **`https://github.com/<username>/ppwl9-mono-docker`**
2. **rekaman layar**:
    - menampilkan test web & perlihatkan command line hasil build docker nya, tutup terminal yang lain.
    - Memastikan bahwa web yang terlihat adalah hasil build docker, bukan `bun dev` biasa.
    - Test halaman root `/` (tampilkan tabel data) dan `/classroom` (grid data API Course) di web frontend.

[Contoh Submisi Video](#) akan di update belakangan.
