# File Integrity Monitoring dengan Wazuh: Setup Docker Single Node & Testing

> Panduan lengkap memantau perubahan file sistem secara real-time menggunakan Wazuh FIM (File Integrity Monitoring) — berbasis deployment Docker Single Node

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

## Arsitektur: Wazuh Docker Single Node

Pada setup ini, seluruh komponen Wazuh (Manager, Indexer, Dashboard) berjalan dalam **satu host** menggunakan Docker Compose. Agent diinstall secara langsung di mesin target (bare-metal atau VM).

```
┌──────────────────────────────────────────────┐
│              Docker Host                      │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │        docker-compose (single node)     │ │
│  │                                         │ │
│  │  ┌───────────────┐  ┌────────────────┐  │ │
│  │  │ wazuh.manager │  │ wazuh.indexer  │  │ │
│  │  │  :1514 (agent)│  │  (OpenSearch)  │  │ │
│  │  │  :1515 (enroll│  └────────────────┘  │ │
│  │  └───────────────┘                      │ │
│  │  ┌───────────────┐                      │ │
│  │  │wazuh.dashboard│  port :443           │ │
│  │  │  (Kibana-like)│◀──────── Browser     │ │
│  │  └───────────────┘                      │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
           ▲ Agent reports via port 1514
           │
┌──────────────────┐
│  Target Machine  │
│  (target_1)      │
│  192.168.100.115 │
│                  │
│  Wazuh Agent     │
│  /var/ossec/     │
└──────────────────┘
```

---

## Prerequisites

| Komponen | Keterangan |
|---|---|
| Docker & Docker Compose | Terinstall di host |
| Wazuh Docker Single Node | Repository resmi Wazuh |
| Target Machine | Ubuntu/Debian untuk install agent |
| Koneksi jaringan | Target bisa reach Docker host (port 1514, 1515) |
| RAM minimal | 4 GB untuk Docker host |

---

## Step 1 — Deploy Wazuh Docker Single Node

### 1.1 Clone Repository Wazuh Docker

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.2
cd wazuh-docker/single-node
```

### 1.2 Generate Sertifikat SSL

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

### 1.3 Jalankan Stack

```bash
docker compose up -d
```

Tunggu beberapa menit hingga semua container running:

```bash
docker compose ps
```

Output yang diharapkan:
```
NAME                              STATUS          PORTS
single-node-wazuh.manager-1    Up (healthy)    0.0.0.0:1514->1514/tcp
single-node-wazuh.indexer-1    Up (healthy)    9200/tcp
single-node-wazuh.dashboard-1  Up (healthy)    0.0.0.0:443->5601/tcp
```

### 1.4 Akses Dashboard

Buka browser dan akses:
```
https://<IP_DOCKER_HOST>
```

Login default:
- **Username:** `admin`
- **Password:** `SecretPassword` *(lihat & ubah di `docker-compose.yml`)*

---

## Step 2 — Install Wazuh Agent di Mesin Target

Jalankan semua perintah berikut di **mesin target** (bukan Docker host).

### 2.1 Tambahkan Repository dan Install

```bash
# Tambahkan GPG key Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -

# Tambahkan repository
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list

# Update dan install
sudo apt-get update
sudo apt-get install wazuh-agent -y
```

### 2.2 Konfigurasi Koneksi ke Manager

Edit file konfigurasi agent:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Arahkan ke IP Docker host:

```xml
<ossec_config>
  <client>
    <server>
      <address>192.168.100.10</address>  <!-- IP Docker Host -->
      <port>1514</port>
      <protocol>tcp</protocol>
    </server>
    <enrollment>
      <enabled>yes</enabled>
    </enrollment>
  </client>
</ossec_config>
```

### 2.3 Enable dan Start Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Verifikasi agent aktif:

```bash
sudo systemctl status wazuh-agent
```

### 2.4 Verifikasi Agent Terdaftar di Manager

Dari **Docker host**, masuk ke container manager dan cek daftar agent:

```bash
docker exec -it single-node-wazuh.manager-1 bash

/var/ossec/bin/agent_control -l
```

Output yang diharapkan:
```
Wazuh agent_control. List of available agents:
   ID: 000, Name: wazuh.manager, IP: 127.0.0.1, Active/Local
   ID: 001, Name: target_1, IP: 192.168.100.115, Active
```

---

## Step 3 — Konfigurasi File Integrity Monitoring

Edit `ossec.conf` di **mesin target**:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

### 3.1 Tambahkan Blok Syscheck

```xml
<syscheck>
  <!-- Aktifkan FIM -->
  <disabled>no</disabled>

  <!-- Interval scan terjadwal (detik): 43200 = 12 jam -->
  <frequency>43200</frequency>

  <!-- Scan saat agent pertama kali start -->
  <scan_on_start>yes</scan_on_start>

  <!-- ===== DIREKTORI YANG DIPANTAU ===== -->

  <!-- Real-time monitoring /etc -->
  <directories realtime="yes" check_all="yes">/etc</directories>

  <!-- Scheduled monitoring binary sistem -->
  <directories check_all="yes">/usr/bin,/usr/sbin</directories>
  <directories check_all="yes">/bin,/sbin</directories>
  <directories check_all="yes">/boot</directories>

  <!-- ===== DIREKTORI YANG DIKECUALIKAN ===== -->
  <ignore>/etc/mtab</ignore>
  <ignore>/etc/hosts.deny</ignore>
  <ignore>/etc/adjtime</ignore>
  <ignore>/etc/cups/certs</ignore>
  <ignore type="sregex">.log$|.tmp$|.swp$</ignore>

  <!-- Laporkan penambahan file baru -->
  <alert_new_files>yes</alert_new_files>

  <!-- Jangan abaikan file yang sering berubah -->
  <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>
</syscheck>
```

### 3.2 Tabel Opsi Check

| Opsi | Fungsi |
|---|---|
| `check_all` | Cek semua atribut sekaligus |
| `check_md5sum` | Hash MD5 |
| `check_sha1sum` | Hash SHA1 |
| `check_sha256sum` | Hash SHA256 |
| `check_size` | Ukuran file |
| `check_owner` | Pemilik file (uid) |
| `check_group` | Grup file (gid) |
| `check_perm` | Permission (rwxrwxrwx) |
| `check_mtime` | Modification time |
| `realtime` | Monitor real-time via inotify |
| `whodata` | Rekam siapa yang mengubah (butuh auditd) |

### 3.3 Restart Agent

```bash
sudo systemctl restart wazuh-agent

# Monitor log untuk memastikan tidak ada error
sudo tail -f /var/ossec/logs/ossec.log
```

---

## Step 4 — Testing File Monitoring

### 4.1 Simulasi Modifikasi File

Tambahkan konten ke file `/etc/alternatives/README` yang sedang dipantau:

```bash
echo "# Test FIM Wazuh - $(date)" | sudo tee -a /etc/alternatives/README
```

### 4.2 Trigger Re-Scan (Mode Scheduled)

Jika menggunakan mode scheduled (bukan realtime), percepat deteksi dengan restart agent:

```bash
sudo systemctl restart wazuh-agent
```

Atau kirim sinyal re-scan manual:

```bash
sudo kill -USR2 $(sudo cat /var/ossec/var/run/wazuh-agentd.pid)
```

### 4.3 Cek Alert dari Container Manager

```bash
# Masuk ke container Wazuh Manager
docker exec -it single-node-wazuh.manager-1 bash

# Cek alert yang masuk
grep "README" /var/ossec/logs/alerts/alerts.log

# Format JSON lebih lengkap
grep "README" /var/ossec/logs/alerts/alerts.json | python3 -m json.tool | head -80
```

---

## Step 5 — Menganalisis Alert FIM

Berikut adalah **alert nyata** yang dihasilkan dari pengujian FIM pada file `/etc/alternatives/README`:

```
File '/etc/alternatives/README' modified
Mode: scheduled
Changed attributes: size, mtime, md5, sha1, sha256
Size changed from '154' to '156'
Old md5sum was  : 699344b79db702416c428a3bed0d9359
New md5sum is   : 61f2b414a9eb40f32def4699f624cd7a
Old sha256sum   : 91c567109894c2fa81d5a1f1503373ba6b4431861bf8b07e891c37f0e917cb23
New sha256sum   : 7ed15c20c29c4becc77d028bb8e58e85e631be8c0ff23ce2ab29e305f2b6e637
```

### Breakdown Alert

#### 🔴 Rule & Severity

| Field | Value |
|---|---|
| **Rule ID** | 550 |
| **Deskripsi** | Integrity checksum changed |
| **Level** | 7 (Medium-High) |
| **Index** | `wazuh-alerts-4.x-2026.03.12` |

#### 👤 Agent Info

| Field | Value |
|---|---|
| **Agent Name** | target_1 |
| **Agent ID** | 001 |
| **IP** | 192.168.100.115 |

#### 📊 Perubahan yang Terdeteksi

| Atribut | Sebelum | Sesudah |
|---|---|---|
| **Size** | 154 bytes | 156 bytes |
| **MD5** | `699344b7...` | `61f2b414...` |
| **SHA1** | `8474f858...` | `94bb67e7...` |
| **SHA256** | `91c56710...` | `7ed15c20...` |
| **mtime** | 07:28:01 | 07:30:28 |

> **Insight:** File bertambah 2 bytes dan semua hash berubah — konsisten dengan penambahan konten teks baru.

#### 📋 Compliance Mapping (Otomatis)

| Standar | Kontrol |
|---|---|
| **PCI DSS** | 11.5 — Deploy file integrity monitoring |
| **HIPAA** | 164.312.c.1, 164.312.c.2 — Integrity controls |
| **NIST 800-53** | SI.7 — Software & Information Integrity |
| **GDPR** | II_5.1.f — Data integrity |
| **MITRE ATT&CK** | T1565.001 — Stored Data Manipulation |

---

## Step 6 — Melihat Alert di Wazuh Dashboard

### 6.1 Navigasi ke Modul FIM

```
Menu → Modules → Integrity Monitoring
```

Pilih agent **target_1**, lalu eksplorasi tab:
- **Events** — semua event FIM (modified, added, deleted)
- **Files** — inventori file yang sedang dipantau

### 6.2 Query di OpenSearch Dashboards

Filter alert FIM di index pattern `wazuh-alerts-*`:

```
rule.groups: "syscheck" AND agent.name: "target_1"
```

Filter spesifik per file:
```
syscheck.path: "/etc/alternatives/README"
```

### 6.3 Index Pattern

```
wazuh-alerts-4.x-YYYY.MM.DD
```

Contoh untuk tanggal 12 Maret 2026:
```
wazuh-alerts-4.x-2026.03.12
```

---

## Tips: Konfigurasi Lanjutan

### Real-time Monitoring

Untuk deteksi dalam hitungan detik (menggunakan Linux inotify):

```xml
<syscheck>
  <directories realtime="yes" check_all="yes">/etc</directories>
  <directories realtime="yes" check_all="yes">/var/www/html</directories>
</syscheck>
```

### whodata — Rekam Siapa yang Mengubah

```bash
# Install auditd di target machine
sudo apt install auditd -y
sudo systemctl enable auditd --now
```

Kemudian di `ossec.conf`:

```xml
<directories whodata="yes" check_all="yes">/etc/passwd</directories>
<directories whodata="yes" check_all="yes">/etc/sudoers</directories>
```

### Manajemen Konfigurasi Manager via Docker

Untuk edit konfigurasi Manager tanpa masuk ke dalam container:

```bash
# Copy konfigurasi keluar dari container
docker cp single-node-wazuh.manager-1:/var/ossec/etc/ossec.conf ./ossec-manager.conf

# Edit file
nano ossec-manager.conf

# Copy kembali ke container
docker cp ossec-manager.conf single-node-wazuh.manager-1:/var/ossec/etc/ossec.conf

# Restart manager di dalam container
docker exec single-node-wazuh.manager-1 /var/ossec/bin/wazuh-control restart
```

---

## Troubleshooting

### Agent tidak muncul di Manager

```bash
# Cek log enrollment di agent
sudo tail -50 /var/ossec/logs/ossec.log | grep -i "enroll\|error\|connect"

# Tes konektivitas port dari target ke Docker host
nc -zv <IP_DOCKER_HOST> 1514
nc -zv <IP_DOCKER_HOST> 1515
```

### Alert FIM tidak muncul

```bash
# Cek apakah syscheck aktif
sudo grep -A3 "<syscheck>" /var/ossec/etc/ossec.conf

# Cek log syscheck di agent
sudo grep "syscheck" /var/ossec/logs/ossec.log | tail -20

# Pastikan database FIM ada
ls -la /var/ossec/queue/fim/db/
```

### Container Manager tidak healthy

```bash
# Cek log container
docker logs single-node-wazuh.manager-1 --tail 50

# Cek resource usage (RAM sering jadi masalah di single node)
docker stats --no-stream
```

### Terlalu banyak alert false positive

```xml
<syscheck>
  <ignore>/var/log</ignore>
  <ignore>/tmp</ignore>
  <ignore type="sregex">.log$|.tmp$|.pid$|.swp$</ignore>
</syscheck>
```

---

## Kesimpulan

Dengan Wazuh Docker Single Node, kamu bisa mendeploy stack monitoring keamanan lengkap hanya dengan beberapa perintah. Wazuh FIM memberikan visibilitas penuh terhadap perubahan file sistem, lengkap dengan:

✅ Deteksi real-time maupun scheduled scan
✅ Hash comparison (MD5, SHA1, SHA256) untuk verifikasi integritas
✅ Compliance mapping otomatis ke PCI DSS, HIPAA, NIST, GDPR
✅ MITRE ATT&CK tagging untuk threat intelligence
✅ Dashboard terpusat via Wazuh web interface

**Langkah selanjutnya yang bisa dikembangkan:**

- **Active Response** — otomatis block atau rollback saat file dimodifikasi
- **Custom Rules** — tambah alert untuk direktori atau pattern spesifik
- **Multi-agent** — tambah lebih banyak target ke satu Manager
- **Integration** — kirim alert ke Slack, PagerDuty, atau TheHive

---

*Ditulis berdasarkan pengujian langsung di lingkungan Wazuh 4.x Docker Single Node dengan Ubuntu sebagai mesin target. Alert data dari index `wazuh-alerts-4.x-2026.03.12`, agent `target_1` (192.168.100.115).*

---

**Tags:** `#Wazuh` `#Docker` `#CyberSecurity` `#FileIntegrityMonitoring` `#SIEM` `#BlueTeam` `#Linux` `#SOC` `#DevSecOps`
