# TOKET - Terbitan Online Kecoak Elektronik

```
------------------------------------------------------------------------
TOKET
Terbitan Online Kecoak Elektronik
------------------------------------------------------------------------
```

TOKET adalah e-zine Kecoak Elektronik untuk dokumentasi riset, catatan,
reverse engineering, exploit analysis, sistem operasi, network security,
cryptography, underground computing culture, dan sejarah komunitas.

E-zine ini ditulis untuk pembaca yang ingin memahami mekanisme. Setiap
artikel diharapkan membawa analisis yang dapat diuji, dibantah, dan
dikembangkan kembali oleh komunitas.

```
k-elektronik.org | underground since 1995
```

---

## 00 - Tentang TOKET

TOKET memuat tulisan teknikal dengan karakter berikut:

- Analitis, langsung, dan berbasis data/fakta.
- Mengutamakan mekanisme teknis, root cause, dan implikasi defensif.
- Menghindari klaim kosong, hype, dan bahasa marketing.
- Menjaga gaya zine klasik: tajam, rapi, berani, tetapi tetap akurat.
- Menulis untuk pembaca teknis.

TOKET menerima tulisan dalam bahasa Indonesia. Istilah teknis boleh tetap dalam
bahasa Inggris jika terjemahan membuat makna menjadi kabur.

---

## 01 - Ruang Lingkup Tulisan

Topik yang sesuai untuk TOKET:

| Kategori | Contoh |
|---|---|
| Vulnerability Analysis | Root cause CVE, patch diff, regression analysis |
| Reverse Engineering | Binary analysis, malware teardown, protocol reversing |
| Exploit Development | Primitive analysis, exploitability reasoning, mitigasi |
| Defensive Research | Detection logic, forensic artifact, hardening strategy |
| Network Security | SSH, VPN, BGP, DNS, routing security, firewall behavior |
| Operating System Security | Linux, BSD, kernel feature, privilege boundary |
| Cryptography | Protocol weakness, key exchange, downgrade, implementation flaw |
| Supply Chain Security | Build artifact, maintainer trust, dependency chain |
| Underground History | Arsip, kultur teknis, catatan komunitas, retrospektif |

Topik di luar scope dapat diterima jika memiliki nilai teknis yang jelas dan
ditulis dengan standar analisis yang kuat.

---

## 02 - Prinsip Editorial

Setiap artikel TOKET harus memenuhi prinsip berikut:

1. **Akurat**  
   Klaim teknis harus dapat dilacak ke advisory, source code, commit, paper,
   exploit note, hasil eksperimen, atau observasi yang dijelaskan dengan jelas.

2. **Terstruktur**  
   Artikel harus memiliki alur yang bisa diikuti: konteks, root cause,
   dampak, analisis, mitigasi, dan referensi.

3. **Tidak Overclaim**  
   Bedakan fakta, hipotesis, opini teknis, dan proyeksi. Jika suatu jalur RCE
   belum terbukti reliable, tulis sebagai exploitability analysis.

4. **Defensive Framing**  
   Tulisan boleh membahas exploitation primitive, tetapi tidak boleh berubah
   menjadi instruksi operasional untuk kompromi sistem nyata.

5. **Reproducible Thinking**  
   Pembaca harus dapat memahami cara berpikir penulis, batasan analisis, dan
   alasan teknis di balik kesimpulan.

6. **No Marketing Noise**  
   TOKET bukan press release, bukan vendor brochure, dan bukan konten SEO.

---

## 03 - Format Artikel

Format default artikel:

```text
Judul Artikel
Subjudul jika diperlukan

                                               by: handle/author, tahun

------------------------------------------------------------------------
00 - Daftar Menu
------------------------------------------------------------------------

  01  Pendahuluan
  02  Scope dan Threat Model
  03  Technical Background
  04  Root Cause Analysis
  05  Exploitability Analysis
  06  Detection dan Forensic Notes
  07  Mitigasi
  08  Kesimpulan
  09  Referensi

------------------------------------------------------------------------
01 - Pendahuluan
------------------------------------------------------------------------

  Isi artikel dimulai di sini.
```

Penomoran boleh memakai gaya klasik:

```text
00, 01, 02, 03, 04, 05, 06, 07, 08, 09, 0A, 0B, 0C
```

Untuk artikel panjang, sub-section dapat menggunakan format:

```text
05.1 - Heap State
05.2 - Signal Handler
05.3 - Distro Patch Delta
```

---

## 04 - Struktur Repository

Struktur repo yang disarankan:

```text
toket/
├── README.md
├── LICENSE
├── issues/
│   ├── 0001/
│   │   ├── README.md
│   │   ├── articles/
│   │   │   ├── 00-editorial.txt
│   │   │   ├── 01-openssh-modern-attack-surface.txt
│   │   │   └── 02-example.txt
│   │   └── assets/
│   │       └── diagrams/
│   └── 0002/
│       ├── README.md
│       ├── articles/
│       └── assets/
├── templates/
│   ├── article-template.txt
│   └── article-template.md
└── references/
    └── style-guide.md
```

---

## 05 - Penamaan File

Gunakan penamaan yang pendek, jelas, dan stabil.

Contoh:

```text
01-openssh-modern-attack-surface.txt
02-xz-utils-backdoor-analysis.txt
03-ssh-agent-forwarding-risk.txt
04-bgp-route-leak-forensics.txt
```

Aturan:

- Gunakan huruf kecil.
- Gunakan tanda `-` sebagai pemisah kata.
- Hindari spasi.
- Hindari nama file yang terlalu panjang.
- Gunakan `.txt` untuk gaya zine klasik.
- Gunakan `.md` jika artikel membutuhkan tabel Markdown atau link yang lebih rapi.

---

## 06 - Submission Guideline

Pull request atau submission artikel harus menyertakan:

| Field | Keterangan |
|---|---|
| Judul | Judul artikel |
| Author | Nama, handle, atau staff identity |
| Ringkasan | 3-5 kalimat tentang isi artikel |
| Kategori | Vulnerability, reversing, defense, network, crypto, dll |
| Status | Draft, review, accepted, published |
| Referensi | Link advisory, paper, source code, atau bukti teknis |
| Risk Note | Jelaskan apakah artikel memuat offensive detail |

Checklist sebelum submit:

- [ ] Artikel memiliki daftar menu.
- [ ] Artikel membedakan fakta dan analisis.
- [ ] Artikel menyertakan referensi.
- [ ] Artikel tidak memuat instruksi kompromi sistem nyata.
- [ ] Artikel tidak mengandung credential, target nyata, atau data sensitif.
- [ ] Artikel sudah diperiksa ulang untuk klaim CVE, versi, dan affected product.
- [ ] Artikel sudah dibaca ulang untuk typo dan konsistensi istilah.

---

## 07 - Gaya Penulisan

Gaya TOKET:

```text
teknikal
jelas
tidak basa-basi
tidak menjual ketakutan
tidak menutupi kompleksitas
```

Gunakan bahasa yang langsung:

```text
Root cause berada pada error handling.
Patch distro menambah attack surface.
Exploitability bergantung pada hardening.
Detection harus memeriksa runtime dependency.
```

Hindari gaya berikut:

```text
Ini bukan sekadar X, tetapi Y.
Bukan X, bukan Y, bukan Z.
Serangan ini bersembunyi dalam senyap.
Ancaman ini menghantui dunia digital.
```

TOKET menghargai tulisan yang kuat karena akurat, bukan karena dramatis.

---

## 08 - Referensi

Referensi harus jelas dan dapat diverifikasi.

Sumber yang disarankan:

- Vendor security advisory.
- Upstream release notes.
- Mailing list disclosure.
- CVE record.
- Academic paper.
- Conference paper.
- Source code commit.
- Patch diff.
- Reproducible test note.
- Forensic artifact yang dijelaskan dengan aman.

Format referensi:

```text
[1] Author/Organization. "Title." Year.
    https://example.org/reference
```

Untuk klaim teknis penting, jangan hanya memakai blog sekunder jika advisory
primer tersedia.

---

## 09 - Safety dan Etika Publikasi

TOKET membahas keamanan komputer untuk riset, edukasi, audit, dan pertahanan.

Artikel tidak boleh memuat:

- Credential nyata.
- Target nyata tanpa izin.
- Instruksi step-by-step untuk kompromi sistem aktif.
- Payload siap pakai untuk penyalahgunaan.
- Data pribadi atau data organisasi yang tidak boleh dipublikasikan.
- Materi yang mendorong pemerasan, pencurian, sabotase, atau akses ilegal.

Analisis exploitability boleh dibahas pada level primitive, constraint,
affected condition, dan mitigation. Detail yang membuat penyalahgunaan menjadi
langsung operasional harus dihapus atau diabstraksikan.

---

## 0A - Template Header Artikel

```text
Judul Artikel
Subjudul Artikel

                                               by: Author, Year

------------------------------------------------------------------------
00 - Daftar Menu
------------------------------------------------------------------------

  01  Pendahuluan
  02  Scope dan Threat Model
  03  Technical Background
  04  Root Cause Analysis
  05  Impact Analysis
  06  Detection
  07  Mitigasi
  08  Kesimpulan
  09  Referensi

------------------------------------------------------------------------
01 - Pendahuluan
------------------------------------------------------------------------

  Tulis pembuka artikel di sini.
```

---

## 0B - Status Publikasi

Status artikel:

```text
DRAFT       - masih ditulis
REVIEW      - sedang diperiksa editor
TECH-REVIEW - sedang diperiksa akurasi teknis
ACCEPTED    - diterima untuk issue berikutnya
PUBLISHED   - sudah terbit
REJECTED    - ditolak atau dikembalikan total
```

---

## 0C - Catatan untuk Pembaca

TOKET ditulis untuk pembaca yang masih percaya bahwa detail teknis penting.
Jika menemukan kesalahan, kirim patch, issue, atau catatan koreksi dengan
referensi yang jelas.

Koreksi teknis lebih berharga daripada tepuk tangan.

```
------------------------------------------------------------------------
TOKET - Terbitan Online Kecoak Elektronik
k-elektronik.org | underground since 1995
EOF
------------------------------------------------------------------------
```
