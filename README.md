# 🚀 Rudalpolo Core (`rudalpolo-core`)

> **Brain, Memory, Skills & Architecture Configuration for Rudalpolo (Senior/Staff Full-Stack Engineering Assistant)**

Repositori private ini menyimpan seluruh konfigurasi, memori permanen, kepribadian, skill koding/SEO/security, dan otomatisasi terjadwal dari agen **Rudalpolo**.

---

## 📂 Struktur Utama Repositori

```text
.
├── SOUL.md              # Karakter, prinsip kejujuran teknis, & persona Rudalpolo
├── config.yaml          # Konfigurasi runtime Hermes, routing model, & toolsets
├── memories/            # Memori permanen (identitas Bos, konvensi, & knowledge)
│   ├── MEMORY.md        # Catatan memori teknis & arsitektur server
│   └── USER.md          # Profil pengguna & standing rules
├── skills/              # Kumpulan skill spesialis produksi
│   ├── rudalpolo/       # Persona senior engineer & understand before implement
│   ├── seo-engineering/ # Master standard SEO, OpenGraph, JSON-LD, llms.txt
│   ├── frontend-crud-ux/# Standar UX aplikasi Sticky Notes & CRUD client
│   ├── vps-security-audit/ # Audit keamanan Linux, Nginx, & Cloudflare
│   └── ...              # 50+ skill development, research, & productivity
├── cron/
│   └── jobs.json        # Jadwal tugas otomatis (Daily IT Briefing 10:00 WIB)
├── .env.example         # Template environment variables & API tokens
└── .gitignore           # Proteksi ketat mencegah kebocoran credentials & database
```

---

## ⚡ Panduan Migrasi ke Server Baru (Quick Setup)

Jika berpindah ke VPS / server baru, cukup jalankan langkah berikut:

### 1. Pasang Runtime Hermes
```bash
curl -fsSL https://raw.githubusercontent.com/nousresearch/hermes-agent/main/install.sh | bash
```

### 2. Clone Repositori Ini ke `~/.hermes`
```bash
# Pastikan SSH key GitHub sudah terpasang
git clone git@github.com:mail298/rudalpolo-core.git ~/.hermes
```

### 3. Buat File Kredensial `.env`
Salin template dan masukkan token rahasia:
```bash
cp ~/.hermes/.env.example ~/.hermes/.env
nano ~/.hermes/.env
```

### 4. Jalankan Service Hermes Gateway
```bash
hermes gateway start
```

Rudalpolo akan langsung aktif di Telegram baru dengan seluruh memori, skill, dan SOP yang sama persis tanpa perlu setup dari nol!
