# BLH Scanner - Broken Link Hijacking: Social Media Scanner

Tool untuk mencari akun sosial media yang sudah mati/nonaktif yang masih di-link dari sebuah domain. Berguna untuk bug bounty (Broken Link Hijacking).

## Platform yang di-scan
- LinkedIn
- Twitter / X
- Instagram
- GitHub
- Facebook

## Install & Run

```bash
# Clone / copy file main.go ke folder kamu
mkdir blh-scanner && cd blh-scanner
# taruh main.go di sini

# Init module
go mod init blh-scanner

# Build
go build -o blh-scanner main.go

# Jalankan
./blh-scanner -u https://target.com
```

## Usage

```
./blh-scanner -u <URL> [options]

Options:
  -u  string   Target URL (wajib), contoh: https://nasa.gov
  -d  int      Kedalaman crawl subpage (default: 1, 0 = homepage saja)
  -w  int      Jumlah worker paralel (default: 5)
  -o  string   Simpan hasil ke file (opsional), contoh: -o hasil.txt
```

## Contoh

```bash
# Scan homepage saja
./blh-scanner -u https://nasa.gov

# Scan + crawl subpage depth 2
./blh-scanner -u https://nasa.gov -d 2

# Scan dengan 10 worker + simpan hasil
./blh-scanner -u https://nasa.gov -d 1 -w 10 -o hasil.txt
```

## Output

```
🔴 DEAD / HIJACKABLE ACCOUNTS (2 found)
  [DEAD] LinkedIn     | @john.doe.12345          | https://linkedin.com/in/john.doe.12345
        Found on: https://nasa.gov/team

🟢 ALIVE ACCOUNTS (5)
  [ALIVE] GitHub      | @nasaofficial           | https://github.com/nasaofficial

[SUMMARY] Dead: 2 | Alive: 5 | Unknown: 1 | Total: 8
```

## Tips Bug Bounty

- Kombinasikan dengan Google Dork dulu:
  ```
  site:target.com "linkedin.com/in" filetype:pdf
  ```
  Lalu scan URL PDF yang ditemukan satu per satu.

- Fokus ke **LinkedIn** dan **Twitter/X** karena paling sering diterima sebagai valid bug bounty finding.

- Selalu **minta izin** (dalam scope program bug bounty) sebelum scan.

## Legal

Tool ini hanya untuk digunakan pada target yang sudah memberikan izin (bug bounty program). Penggunaan di luar itu adalah tanggung jawab pengguna.
