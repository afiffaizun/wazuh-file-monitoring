# File Integrity Monitoring dengan Wazuh: Setup, Konfigurasi, dan Testing

> Panduan lengkap memantau perubahan file sistem secara real-time menggunakan Wazuh FIM (File Integrity Monitoring)

---

## Apa itu Wazuh File Integrity Monitoring?

Wazuh **File Integrity Monitoring (FIM)** adalah fitur keamanan yang memantau perubahan pada file dan direktori di sistem operasi secara real-time maupun terjadwal. FIM bekerja dengan cara membuat **baseline checksum** (MD5, SHA1, SHA256) dari file yang dipantau, lalu membandingkannya secara berkala untuk mendeteksi perubahan yang tidak sah.

### Apa yang Dideteksi FIM?

FIM dapat mendeteksi berbagai perubahan, antara lain:

- **Modifikasi konten file** — perubahan isi, ukuran, atau hash
- **Pembuatan file baru** — file yang tidak ada sebelumnya tiba-tiba muncul
- **Penghapusan file** — file yang hilang dari direktori yang dipantau
- **Perubahan permission** — chmod, perubahan owner/group
- **Perubahan timestamp** — modifikasi waktu akses atau tulis

### Mengapa FIM Penting?

Dalam konteks keamanan siber, perubahan file yang tidak terotorisasi bisa menjadi indikator awal dari:

- **Malware atau backdoor** yang menyusup ke sistem
- **Insider threat** — perubahan konfigurasi tanpa izin
- **Compliance violation** — seperti standar PCI DSS 11.5, HIPAA 164.312.c, dan GDPR Article 5
- **Ransomware activity** — modifikasi massal pada file sistem

---

## Arsitektur Wazuh FIM

```
┌─────────────────────┐         ┌──────────────────────┐
│   Wazuh Agent       │ ──────▶ │   Wazuh Manager      │
│  (target_1)         │  Alert  │  (wazuh.manager)     │
│                     │         │                      │
│  /var/ossec/etc/    │         │  ┌────────────────┐  │
│  ossec.conf         │         │  │  OpenSearch /  │  │
│                     │         │  │  Elasticsearch │  │
│  [syscheck]         │         │  └────────────────┘  │
│  - Scan direktori   │         │                      │
│  - Hitung checksum  │         │  ┌────────────────┐  │
│  - Kirim alert      │         │  │  Kibana /      │  │
└─────────────────────┘         │  │  Dashboard     │  │
                                │  └────────────────┘  │
                                └──────────────────────┘
```

---

## Prerequisites

Sebelum memulai, pastikan environment berikut sudah tersedia:

| Komponen | Keterangan |
|---|---|
| Wazuh Manager | Sudah terinstall dan berjalan |
| Wazuh Agent | Akan diinstall di mesin target |
| OS Target | Ubuntu/Debian (panduan ini) |
| Koneksi | Agent bisa reach Manager (port 1514/1515) |

---

## Step 1 — Install dan Aktifkan Wazuh Agent

### 1.1 Install Wazuh Agent

Tambahkan repository Wazuh dan install agent:

```bash
# Tambahkan GPG key
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -

# Tambahkan repository
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" \
  | tee /etc/apt/sources.list.d/wazuh.list

# Install agent
apt-get update && apt-get install wazuh-agent -y
```

### 1.2 Daftarkan Agent ke Manager

Edit file konfigurasi untuk mengarahkan agent ke Wazuh Manager:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Pastikan bagian `<client>` sudah menunjuk ke IP Manager:

```xml
<ossec_config>
  <client>
    <server>
      <address>192.168.100.10</address>  <!-- IP Wazuh Manager -->
      <port>1514</port>
      <protocol>tcp</protocol>
    </server>
  </client>
</ossec_config>
```

### 1.3 Enable dan Start Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Verifikasi status agent:

```bash
sudo systemctl status wazuh-agent
```

Output yang diharapkan:
```
● wazuh-agent.service - Wazuh agent
   Active: active (running) since ...
```

---

## Step 2 — Konfigurasi File Integrity Monitoring

Buka file konfigurasi agent:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

### 2.1 Konfigurasi Dasar Syscheck

Tambahkan atau modifikasi blok `<syscheck>` berikut:

```xml
<syscheck>
  <!-- Aktifkan FIM -->
  <disabled>no</disabled>

  <!-- Interval scan terjadwal (setiap 12 jam) -->
  <frequency>43200</frequency>

  <!-- Scan saat agent pertama kali start -->
  <scan_on_start>yes</scan_on_start>

  <!-- ===== DIREKTORI YANG DIPANTAU ===== -->

  <!-- Pantau /etc dengan real-time monitoring -->
  <directories realtime="yes" check_all="yes">/etc</directories>

  <!-- Pantau binary sistem -->
  <directories check_all="yes">/usr/bin,/usr/sbin</directories>
  <directories check_all="yes">/bin,/sbin</directories>

  <!-- Pantau file kritis lainnya -->
  <directories check_all="yes">/boot</directories>

  <!-- ===== DIREKTORI YANG DIKECUALIKAN ===== -->
  <ignore>/etc/mtab</ignore>
  <ignore>/etc/hosts.deny</ignore>
  <ignore>/etc/mail/statistics</ignore>
  <ignore>/etc/random-seed</ignore>
  <ignore>/etc/random.seed</ignore>
  <ignore>/etc/adjtime</ignore>
  <ignore>/etc/httpd/logs</ignore>
  <ignore>/etc/utmpx</ignore>
  <ignore>/etc/wtmpx</ignore>
  <ignore>/etc/cups/certs</ignore>
  <ignore>/etc/dumpdates</ignore>
  <ignore>/etc/svc/volatile</ignore>

  <!-- Alert ketika file baru ditambahkan ke direktori yang dipantau -->
  <alert_new_files>yes</alert_new_files>

  <!-- Auto-ignore file yang sering berubah (false = laporkan semua) -->
  <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>
</syscheck>
```

### 2.2 Opsi Check yang Tersedia

| Opsi | Keterangan |
|---|---|
| `check_all` | Cek semua atribut |
| `check_md5sum` | Cek hash MD5 |
| `check_sha1sum` | Cek hash SHA1 |
| `check_sha256sum` | Cek hash SHA256 |
| `check_size` | Cek ukuran file |
| `check_owner` | Cek pemilik file |
| `check_group` | Cek grup file |
| `check_perm` | Cek permission file |
| `check_mtime` | Cek modification time |
| `realtime` | Monitor real-time (Linux inotify) |
| `whodata` | Rekam siapa yang mengubah (audit) |

### 2.3 Restart Agent Setelah Konfigurasi

```bash
sudo systemctl restart wazuh-agent

# Verifikasi tidak ada error di log
sudo tail -f /var/ossec/logs/ossec.log
```

---

## Step 3 — Testing File Monitoring

### 3.1 Simulasi Modifikasi File

Untuk menguji apakah FIM berjalan dengan benar, modifikasi file yang sedang dipantau:

```bash
# Tambahkan konten ke file /etc/alternatives/README
echo "# Test modifikasi FIM - $(date)" | sudo tee -a /etc/alternatives/README
```

### 3.2 Tunggu Hasil Scan

Jika menggunakan mode **scheduled**, tunggu hingga scan interval berikutnya. Untuk mempercepat, trigger scan manual:

```bash
# Kirim sinyal ke agent untuk melakukan re-scan
sudo kill -USR2 $(cat /var/ossec/var/run/wazuh-agentd.pid)
```

Atau untuk real-time monitoring, alert akan langsung muncul.

### 3.3 Verifikasi Alert di Log

```bash
# Cek alert di log manager (dari sisi manager)
sudo grep "README" /var/ossec/logs/alerts/alerts.log

# Atau lihat alert JSON
sudo grep "README" /var/ossec/logs/alerts/alerts.json | python3 -m json.tool
```

---

## Step 4 — Menganalisis Alert FIM

Berikut adalah contoh **alert nyata** yang dihasilkan dari pengujian FIM pada file `/etc/alternatives/README`:

```json
{
  "timestamp": "2026-03-12T07:30:34.827+0000",
  "rule": {
    "id": "550",
    "level": 7,
    "description": "Integrity checksum changed.",
    "groups": ["ossec", "syscheck", "syscheck_entry_modified", "syscheck_file"],
    "mitre": {
      "technique": ["Stored Data Manipulation"],
      "id": ["T1565.001"],
      "tactic": ["Impact"]
    }
  },
  "agent": {
    "id": "001",
    "name": "target_1",
    "ip": "192.168.100.115"
  },
  "syscheck": {
    "path": "/etc/alternatives/README",
    "event": "modified",
    "mode": "scheduled",
    "changed_attributes": ["size", "mtime", "md5", "sha1", "sha256"],
    "size_before": "154",
    "size_after": "156",
    "md5_before": "699344b79db702416c428a3bed0d9359",
    "md5_after": "61f2b414a9eb40f32def4699f624cd7a",
    "sha256_before": "91c567109894c2fa81d5a1f1503373ba6b4431861bf8b07e891c37f0e917cb23",
    "sha256_after": "7ed15c20c29c4becc77d028bb8e58e85e631be8c0ff23ce2ab29e305f2b6e637",
    "mtime_before": "2026-03-12T07:28:01",
    "mtime_after": "2026-03-12T07:30:28"
  }
}
```

### Membaca Alert FIM

Mari bedah informasi penting dari alert di atas:

#### 🔴 Rule & Severity
```
Rule ID   : 550 — "Integrity checksum changed"
Level     : 7 (Medium-High)
```
Rule level 7 mengindikasikan **perubahan yang signifikan** dan perlu investigasi.

#### 📋 Compliance Mapping
Alert ini otomatis di-mapping ke standar keamanan:

| Standar | Kontrol |
|---|---|
| **PCI DSS** | 11.5 — Deploy file integrity monitoring |
| **HIPAA** | 164.312.c.1, 164.312.c.2 — Integrity controls |
| **NIST 800-53** | SI.7 — Software, Firmware & Information Integrity |
| **GDPR** | II_5.1.f — Data integrity |
| **MITRE ATT&CK** | T1565.001 — Stored Data Manipulation |

#### 📊 Perubahan yang Terdeteksi

| Atribut | Sebelum | Sesudah |
|---|---|---|
| **Size** | 154 bytes | 156 bytes |
| **MD5** | `699344b7...` | `61f2b414...` |
| **SHA1** | `8474f858...` | `94bb67e7...` |
| **SHA256** | `91c56710...` | `7ed15c20...` |
| **mtime** | 07:28:01 | 07:30:28 |

> **Insight:** File bertambah 2 bytes, semua hash berubah, dan timestamp diperbarui — konsisten dengan penambahan konten.

---

## Step 5 — Melihat Alert di Wazuh Dashboard

### 5.1 Navigasi ke FIM Module

Buka Wazuh Dashboard → Pilih Agent → **Integrity Monitoring**

Tampilan yang tersedia:
- **Events** — daftar semua event FIM
- **Files** — inventory file yang dipantau
- **Registry** — monitoring Windows registry (khusus Windows agent)

### 5.2 Filter Alert Spesifik

Di Kibana/OpenSearch Dashboards, gunakan filter berikut untuk melihat alert FIM:

```
rule.groups: syscheck AND agent.name: target_1
```

Atau filter berdasarkan path file:
```
syscheck.path: "/etc/alternatives/README"
```

### 5.3 Index Pattern

Alert FIM tersimpan di index:
```
wazuh-alerts-4.x-YYYY.MM.DD
```

Contoh untuk 12 Maret 2026:
```
wazuh-alerts-4.x-2026.03.12
```

---

## Konfigurasi Lanjutan: Real-time Monitoring

Untuk monitoring yang lebih responsif (deteksi dalam hitungan detik), gunakan **realtime** mode:

```xml
<syscheck>
  <disabled>no</disabled>

  <!-- Real-time monitoring menggunakan inotify (Linux) -->
  <directories realtime="yes" check_all="yes">/etc</directories>
  <directories realtime="yes" check_all="yes">/var/www/html</directories>

  <!-- whodata: rekam user yang melakukan perubahan (butuh auditd) -->
  <directories whodata="yes" check_all="yes">/etc/passwd</directories>
  <directories whodata="yes" check_all="yes">/etc/shadow</directories>
</syscheck>
```

> **Catatan:** `whodata` membutuhkan `auditd` terinstall dan dikonfigurasi. Install dengan: `sudo apt install auditd -y`

---

## Troubleshooting Umum

### Agent tidak mengirim alert

```bash
# Cek koneksi ke manager
sudo /var/ossec/bin/agent_control -i 001

# Cek log agent
sudo tail -100 /var/ossec/logs/ossec.log | grep -i error

# Pastikan syscheck tidak disabled
sudo grep -A5 "<syscheck>" /var/ossec/etc/ossec.conf
```

### Scan tidak berjalan

```bash
# Cek apakah ada file database FIM
ls -la /var/ossec/queue/fim/db/

# Force re-scan
sudo /var/ossec/bin/wazuh-agentd --test
```

### Terlalu banyak false positive

Tambahkan pengecualian di `ossec.conf`:

```xml
<syscheck>
  <!-- Ignore file yang sering berubah -->
  <ignore>/var/log</ignore>
  <ignore type="sregex">.log$|.tmp$|.swp$</ignore>
</syscheck>
```

---

## Kesimpulan

Wazuh File Integrity Monitoring adalah komponen penting dalam strategi **defense in depth**. Dengan FIM yang dikonfigurasi dengan benar, kamu bisa:

✅ Mendeteksi perubahan file kritis secara real-time
✅ Memenuhi persyaratan compliance (PCI DSS, HIPAA, GDPR)
✅ Mendapatkan mapping otomatis ke MITRE ATT&CK framework
✅ Melihat siapa, kapan, dan apa yang diubah pada file sistem

Langkah selanjutnya yang bisa dikembangkan:

- **Active Response** — otomatis rollback file yang dimodifikasi
- **Custom Rules** — buat rule alert untuk path spesifik
- **Integration** — kirim alert ke Slack, PagerDuty, atau SIEM lain
- **whodata** — aktifkan audit trail lengkap siapa yang mengubah file

---

*Ditulis berdasarkan pengujian langsung di lingkungan Wazuh 4.x dengan Ubuntu target. Alert data diambil dari index `wazuh-alerts-4.x-2026.03.12`.*

---

**Tags:** `#Wazuh` `#CyberSecurity` `#FileIntegrityMonitoring` `#SIEM` `#BlueTeam` `#Linux` `#SOC`
