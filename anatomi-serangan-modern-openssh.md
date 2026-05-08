# Anatomi Serangan Modern pada OpenSSH

**Dari Backdoor Manual ke Supply Chain Compromise**

**by:** Kecoak Staff, 2026

## 00 - Daftar Menu

- [01 - Pendahuluan](#01---pendahuluan)
- [02 - Scope, Threat Model, dan Batasan Analisis](#02---scope-threat-model-dan-batasan-analisis)
- [03 - Arsitektur OpenSSH dan Pergeseran Attack Surface](#03---arsitektur-openssh-dan-pergeseran-attack-surface)
- [04 - Era Lama: Backdoor Manual via Source Patch](#04---era-lama-backdoor-manual-via-source-patch)
- [05 - regreSSHion: CVE-2024-6387](#05---regresshion-cve-2024-6387)
- [06 - XZ Utils Backdoor: CVE-2024-3094](#06---xz-utils-backdoor-cve-2024-3094)
- [07 - ProxyCommand Injection: CVE-2025-61984](#07---proxycommand-injection-cve-2025-61984)
- [08 - Downstream Patch sebagai Attack Surface](#08---downstream-patch-sebagai-attack-surface)
- [08.1 - GSSAPI KEX Heap/Memory Corruption: CVE-2026-3497](#081---gssapi-kex-heapmemory-corruption-cve-2026-3497)
- [08.2 - DisableForwarding Logic Error: CVE-2025-32728](#082---disableforwarding-logic-error-cve-2025-32728)
- [09 - OpenSSH 10.x dan Transisi Post-Quantum KEX](#09---openssh-10x-dan-transisi-post-quantum-kex)
- [0A - Perbandingan Attack Surface: Lama, Modern, dan Emerging](#0a---perbandingan-attack-surface-lama-modern-dan-emerging)
- [0B - Detection dan Forensic Analysis](#0b---detection-dan-forensic-analysis)
- [0C - Mitigasi dan Hardening Strategy](#0c---mitigasi-dan-hardening-strategy)
- [0D - Defensive Research Methodology](#0d---defensive-research-methodology)
- [0E - Kesimpulan](#0e---kesimpulan)
- [0F - Referensi](#0f---referensi)

## 01 - Pendahuluan

OpenSSH sudah lama menjadi komponen standar di hampir semua lingkungan
Unix-like. Ia dipakai untuk administrasi server, automation, file transfer,
tunneling, break-glass access, deployment pipeline, dan remote operations.
Ketika OpenSSH bermasalah, dampaknya tidak berhenti pada satu service.
Dampaknya masuk ke jalur administrasi seluruh infrastruktur.

Pada era lama, kompromi OpenSSH sering dibayangkan sebagai perubahan
langsung terhadap source code atau binary. Attacker yang sudah memiliki
root dapat memodifikasi jalur autentikasi, menanam magic password, atau
mencatat credential. Teknik seperti itu lebih tepat dipahami sebagai
persistence mechanism. Ia membutuhkan akses awal yang tinggi, meninggalkan
artefak yang relatif jelas, dan terbatas pada host yang sudah dikuasai.

Serangan modern lebih sering memanfaatkan area yang sulit dipantau: signal handler,
dependency chain, build artifact, distro-specific patch, client-side expansion,
forwarding path, dan transisi kriptografi baru. Target serangan tidak lagi
terbatas pada file sshd, tetapi juga mencakup komponen pendukung yang berjalan
di sekitar sshd.

Paper ini membahas evolusi teknikal tersebut melalui beberapa kasus utama:

- regreSSHion, CVE-2024-6387
- XZ Utils backdoor, CVE-2024-3094
- ProxyCommand injection, CVE-2025-61984
- GSSAPI KEX distro patch issue, CVE-2026-3497
- DisableForwarding logic error, CVE-2025-32728
- Post-quantum KEX transition dan attack surface baru

Tujuan paper ini adalah menjelaskan anatomi, batasan, dan konsekuensi
defensif dari masing-masing serangan. Bagian yang bersifat forward-looking
diposisikan sebagai analisis attack surface, bukan klaim adanya exploit
publik yang reliable.

Pembaca diasumsikan familiar dengan C, privilege separation, dynamic linking,
glibc allocator, PAM, systemd integration, dan SSH protocol.

## 02 - Scope, Threat Model, dan Batasan Analisis

Scope artikel ini:

    1. Membahas OpenSSH sebagai rangkaian daemon, runtime dependency, build chain, patchset distro, client workflow, forwarding mechanism, dan cryptographic policy.
    2. Membedakan upstream vulnerability, downstream patch issue, dan
       client-side misconfiguration path.
    3. Menjelaskan root cause pada level arsitektur dan primitive keamanan.
    4. Memberikan detection dan mitigation yang dapat dipakai defender.
    5. Menghindari instruksi eksploitasi operasional dan post-exploitation.

Threat model yang digunakan:

- Remote unauthenticated attacker terhadap sshd.
- Local atau semi-local attacker yang memengaruhi input ssh client.
- Supply chain actor yang memengaruhi release artifact atau package build.
- Compromised intermediate host yang berinteraksi dengan forwarded agent.
- Attacker yang mengejar long-term cryptographic exposure melalui downgrade
      atau weak KEX selection.

### Klasifikasi penting

| Kategori | Contoh | Catatan |
|---|---|---|
| Upstream server bug | CVE-2024-6387 | Bug ada di OpenSSH upstream |
| Supply chain | CVE-2024-3094 | Backdoor berada di liblzma |
| Client-side injection | CVE-2025-61984 | Butuh untrusted username dan ProxyCommand |
| Downstream patch bug | CVE-2026-3497 | Terkait distro GSSAPI delta |
| Policy logic error | CVE-2025-32728 | DisableForwarding tidak menutup semua forwarding |

Artikel ini sengaja memisahkan confirmed facts dari exploitability analysis.
Confirmed facts berasal dari advisory, release notes, atau disclosure teknis.
Exploitability analysis menjelaskan kemungkinan jalur riset berdasarkan
primitive, hardening, dan kondisi lingkungan. Keduanya tidak boleh dicampur.

## 03 - Arsitektur OpenSSH dan Pergeseran Attack Surface

### Privilege Separation Architecture

OpenSSH menggunakan privilege separation untuk mengurangi dampak bug pada
proses yang berinteraksi langsung dengan network.

    sshd listener
      |
      +-- per-connection process
            |
            +-- monitor/root-side logic
            |     - key material handling
            |     - PAM interaction
            |     - account/session decision
            |
            +-- restricted child/session-side logic
- packet parsing
- crypto negotiation
- user authentication flow
- channel handling

Pada rilis modern, OpenSSH semakin memisahkan pre-authentication attack
surface. OpenSSH 10.0 memperkenalkan pemisahan tambahan dengan memindahkan
user authentication phase ke binary terpisah, sehingga address space untuk
jalur pre-auth lebih terisolasi dari sisa connection handling.

### Dependency Chain

Paket OpenSSH pada Linux distribution sering berbeda dari upstream murni.
Distro dapat menambahkan patch, link tambahan, build flag, dan integrasi
service manager. Pada beberapa distro, systemd notification path membuat sshd
memiliki dependency tidak langsung ke libsystemd, lalu libsystemd dapat memiliki
dependency ke liblzma.

    sshd
      +-- libcrypto
      +-- libz
      +-- libpam
      +-- distro integration patch
             |
             +-- libsystemd
                    |
                    +-- liblzma

Kasus XZ Utils menunjukkan bahwa dependency yang tidak tampak relevan bagi
admin dapat masuk ke jalur pre-authentication melalui integrasi distro.

### Authentication Flow

    Client -> TCP connection
    Server -> protocol banner
    Client/Server -> KEX negotiation
    Server -> host key presentation
    Client -> user authentication request
       +-- publickey
       +-- password
       +-- keyboard-interactive
       +-- GSSAPI, jika enabled dan didukung distro/package

Attack surface modern muncul pada beberapa titik:

- signal handling saat LoginGraceTime habis
- packet parser sebelum user terautentikasi
- dynamic linking sebelum proses melayani request
- token expansion pada client-side configuration
- forwarding mechanism
- downstream patch yang tidak sama dengan upstream
- KEX algorithm negotiation, termasuk hybrid post-quantum KEX

## 04 - Era Lama: Backdoor Manual via Source Patch

Teknik lama yang sering muncul pada periode awal komunitas underground adalah
patch langsung terhadap jalur autentikasi OpenSSH. Secara konseptual,
attacker yang sudah memiliki root dapat melakukan beberapa perubahan:

- menambahkan secret authentication branch
- mencatat credential password-based authentication
- mengganti binary sshd
- me-restart service
- mempertahankan akses melalui jalur yang terlihat seperti login normal

Teknik ini punya limitasi besar.

### Prerequisite

- Attacker sudah memiliki root atau kontrol build host.
- Attacker dapat mengubah source, binary, atau package.
- Attacker dapat memaksa service restart.

### Detection Surface

- hash binary berubah
- package integrity check gagal
- mtime/ctime file berubah
- build path atau build timestamp anomali
- service restart terlihat di log
- artefak credential logging muncul di filesystem

### Scope

- Terbatas pada host yang sudah dikompromikan.
- Tidak menghasilkan initial access remote.
- Efektif sebagai persistence, bukan sebagai entry vector.

Teknik ini berbeda kelas dari supply chain compromise modern. Nilai historisnya
berada pada perubahan target: dari jalur autentikasi langsung ke dependency,
build pipeline, dan edge behavior.

## 05 - regreSSHion: CVE-2024-6387

### Background

CVE-2024-6387, dikenal sebagai regreSSHion, adalah signal handler race
condition pada OpenSSH server. Bug ini merupakan regresi dari flaw lama
CVE-2006-5051. Perubahan yang masuk pada OpenSSH 8.5p1 membuat jalur yang
sebelumnya aman kembali membuka kondisi unsafe. Qualys TRU melaporkan bug ini
pada 2024.

### Affected Versions

    Vulnerable:
- OpenSSH sebelum 4.4p1, kecuali sudah mendapat patch untuk flaw lama
- OpenSSH 8.5p1 sampai sebelum 9.8p1

    Not vulnerable:
- OpenSSH 4.4p1 sampai 8.4p1
- OpenSSH 9.8p1 ke atas

### Root Cause

Ketika client tidak menyelesaikan autentikasi sampai LoginGraceTime habis,
sshd menerima SIGALRM. Handler untuk kondisi ini masuk ke jalur cleanup yang
memanggil fungsi yang tidak async-signal-safe.

Masalah inti:

- signal handler dapat berjalan di tengah operasi lain
- fungsi seperti syslog() tidak dijamin aman dipanggil dari signal handler
- jalur tersebut dapat menyentuh internal lock, allocator, atau state yang
      sedang tidak konsisten
- hasil akhirnya dapat berupa crash, heap corruption, atau primitive lain
      tergantung platform dan hardening

### Exploitability

Risiko regreSSHion tinggi karena berada pada server-side pre-authentication path.
Attacker tidak perlu akun valid. Namun exploitability sangat bergantung pada:

- arsitektur CPU
- ASLR behavior
- PIE/non-PIE build
- glibc version
- MaxStartups
- LoginGraceTime
- network timing
- heap layout stability
- distro hardening

Demonstrasi publik awal paling kuat pada kondisi lab tertentu, terutama
glibc-based Linux dengan kondisi memory layout yang memungkinkan race window
dan address prediction lebih feasible. Untuk 64-bit hardened systems, reliable
RCE jauh lebih sulit dan tidak boleh dipresentasikan sebagai universal.

### Defensive Reading

Tiga implikasi teknis dari regreSSHion:

    1. Bug yang sudah ditutup dapat kembali melalui regresi.
    2. Signal handler masih menjadi area audit kritis pada daemon network.
    3. Hardening build dan runtime policy dapat mengubah exploitability secara
       signifikan, meskipun root cause-nya sama.

## 06 - XZ Utils Backdoor: CVE-2024-3094

### Background

CVE-2024-3094 adalah supply chain compromise terhadap XZ Utils/liblzma.
Kasus ini ditemukan oleh Andres Freund setelah ia melihat anomali performa
pada sshd di lingkungan Debian sid. Anomali sekitar ratusan milidetik pada
login SSH memicu investigasi yang kemudian membuka backdoor dalam release
artifact XZ 5.6.0 dan 5.6.1.

### Kenapa sshd Terlibat

Upstream OpenSSH tidak langsung menggunakan liblzma. Pada beberapa distro,
sshd membawa integration path ke systemd notification. Jalur ini dapat membuat
sshd me-load libsystemd, dan libsystemd dapat me-load liblzma.

    sshd
      -> distro systemd integration
        -> libsystemd
          -> liblzma

Backdoor berada pada dependency yang tampak jauh dari SSH, tetapi masuk ke
address space sshd melalui dynamic linking.

### Layering Backdoor

Backdoor XZ menggunakan beberapa lapis penyamaran:

    1. Payload disisipkan di test files.
    2. Injection logic berada pada release tarball, bukan git repository biasa.
    3. Build-time script aktif hanya pada kondisi tertentu.
    4. Target utama adalah x86-64 Linux dengan glibc dan build environment
       tertentu, terutama Debian/RPM packaging context.
    5. Hooking dilakukan melalui dynamic loader dan IFUNC-related behavior.
    6. Jalur publickey authentication menjadi titik trigger karena sshd dapat
       memanggil fungsi crypto yang sudah dialihkan.

### Operational Sophistication

Perbedaan XZ dengan backdoor manual sangat besar:

- tidak perlu mengubah OpenSSH source
- tidak perlu menyentuh file sshd secara langsung
- malicious logic muncul saat build dan runtime linking
- code review terhadap git repository saja tidak cukup
- trigger path berada pada pre-authentication context
- hanya pihak yang memiliki material kriptografi tertentu yang dapat
      memicu fungsi berbahaya secara valid

### Defensive Reading

Kasus XZ memperluas baseline evaluasi supply chain security:

- verifikasi source repository tidak cukup
- release tarball harus dibandingkan dengan source tree
- build artifact perlu reproducible verification
- dependency chain daemon kritikal harus dipetakan
- anomaly detection sederhana seperti latency dan CPU spike tetap bernilai

## 07 - ProxyCommand Injection: CVE-2025-61984

### Classification

CVE-2025-61984 bukan server-side pre-authentication RCE pada sshd. Ini adalah
client-side command injection condition pada ssh(1) ketika username yang
berasal dari sumber tidak tepercaya dipakai bersama ProxyCommand yang
menggunakan expansion tertentu.

Klasifikasi yang tepat:

- affected component : ssh client
- affected condition : untrusted username atau URI-derived input
- required feature   : ProxyCommand dengan username expansion
- impact             : possible command execution pada client side
- severity context   : rendah sampai serius, tergantung workflow

### Root Cause

OpenSSH sebelum 10.1 mengizinkan control characters pada username yang berasal
dari command line atau %-sequence expansion configuration. Jika nilai ini
dipakai dalam ProxyCommand, shell expression dapat terbentuk di luar struktur
command yang diharapkan.

OpenSSH 10.1 memperbaiki ini dengan menolak control characters pada username
dari sumber tidak tepercaya dan menolak NUL character pada ssh:// URI.

### Attack Scenario

Skenario yang masuk akal bukan attacker langsung menyerang sshd server.
Skenario yang lebih tepat:

- user atau automation membentuk ssh command dari input tidak tepercaya
- configuration menggunakan ProxyCommand
- username atau URI berisi control character
- client memulai ProxyCommand
- ekspansi command menghasilkan perilaku shell yang tidak dimaksudkan

Contoh lingkungan terdampak:

- automation script yang membangun ssh command dari inventory eksternal
- wrapper CLI internal
- deployment tooling
- repository atau config material yang mengandung host/user value tidak
      tepercaya
- bastion workflow dengan ProxyCommand agresif

### Defensive Reading

ProxyCommand injection menunjukkan bahwa attack surface OpenSSH mencakup
daemon dan client workflow. ssh client configuration, token expansion, URI
parser, dan automation wrapper perlu diperlakukan sebagai bagian dari security
boundary.

## 08 - Downstream Patch sebagai Attack Surface

Upstream OpenSSH memiliki proses security engineering yang ketat. Namun paket
yang dipakai organisasi sering bukan upstream murni. Distro menambahkan patch
untuk integrasi systemd, GSSAPI, SELinux, packaging behavior, crypto policy,
dan compatibility layer.

Patch ini berguna secara operasional dan memperluas attack surface.

### 08.1 - GSSAPI KEX Heap/Memory Corruption: CVE-2026-3497

### Classification

CVE-2026-3497 memengaruhi GSSAPI delta yang ditambahkan oleh sejumlah Linux
distribution. Vulnerability ini tidak memengaruhi upstream OpenSSH murni.

Kondisi penting:

- terkait GSSAPI Key Exchange patch
- GSSAPIKeyExchange harus relevan/enabled pada konfigurasi terdampak
- root cause berada pada error handling yang tidak menghentikan proses
- dampak sangat bergantung pada compiler hardening dan build flags

### Root Cause

Pada error path GSSAPI key exchange, penggunaan fungsi disconnect yang hanya
mengantre pesan disconnect tetapi tidak menghentikan proses membuat eksekusi
berlanjut. Setelah itu, variabel koneksi tertentu belum diinisialisasi dengan
benar. Akses terhadap variabel tersebut dapat menghasilkan undefined behavior.

Dampak yang dinyatakan advisory:

- crash atau denial of service
- possible arbitrary code execution pada kondisi tertentu
- exploitability sangat bergantung pada hardening compiler

### Defensive Reading

Nilai teknis utama CVE ini berada pada pola downstream patch risk:

- downstream patch menambah state machine baru
- error handling berbeda sedikit dari upstream expectation
- disconnect path tidak selalu berarti process termination
- uninitialized memory di jalur pre-auth dapat memiliki dampak besar

Untuk pembacaan defensif, CVE-2026-3497 harus dilihat sebagai bukti bahwa
audit OpenSSH tidak cukup jika hanya membaca upstream source. Patchset distro
perlu masuk ke security review.

### 08.2 - DisableForwarding Logic Error: CVE-2025-32728

### Classification

CVE-2025-32728 adalah logic error pada sshd sebelum OpenSSH 10.0. Opsi
DisableForwarding tidak sepenuhnya mematuhi dokumentasi karena gagal menutup
X11 forwarding dan agent forwarding sesuai ekspektasi.

Ini bukan memory corruption dan bukan remote pre-authentication RCE.
Dampaknya muncul ketika admin mengandalkan DisableForwarding sebagai policy
control yang dianggap menutup semua forwarding surface.

### Defensive Reading

Dampak praktis:

- admin dapat memiliki false sense of enforcement
- agent forwarding tetap menjadi risiko pada environment tertentu
- X11 forwarding dan agent forwarding perlu diverifikasi secara eksplisit
- policy hardening harus diuji melalui test case dan tidak cukup dengan penulisan config

OpenSSH 10.0 memperbaiki behavior ini. Release notes juga mengingatkan bahwa
X11 forwarding default-nya disabled pada server dan agent forwarding tidak
diminta secara default oleh client.

## 09 - OpenSSH 10.x dan Transisi Post-Quantum KEX

OpenSSH sudah menawarkan post-quantum key agreement secara default sejak
OpenSSH 9.0 melalui sntrup761x25519-sha512. OpenSSH 9.9 menambahkan
mlkem768x25519-sha256, dan OpenSSH 10.0 menjadikannya default. OpenSSH 10.1
menambahkan warning ketika koneksi tidak menggunakan post-quantum key exchange.

### Why It Matters

Transisi post-quantum perlu dianalisis sebagai isu kriptografi dan operasional.
SSH sering dipakai untuk administrasi sistem bernilai tinggi. Data yang direkam
hari ini dapat bernilai di masa depan jika masa kerahasiaannya panjang.

Risiko utama:

- downgrade ke non-PQ KEX
- misconfiguration karena compatibility pressure
- middlebox atau legacy device yang memaksa fallback
- implementasi algorithm baru yang belum memiliki umur audit panjang
- side-channel pada primitive baru

### Defensive Reading

Area yang perlu dipantau:

- KexAlgorithms policy pada client dan server
- warning non-PQ KEX di OpenSSH 10.1+
- inventory server yang masih memaksa classical-only KEX
- compatibility exception yang tidak pernah dicabut
- update cadence terhadap crypto library dan OpenSSH release

Post-quantum transition tidak otomatis melemahkan OpenSSH. Setiap transisi
kriptografi besar menambah parser baru, negotiation behavior baru,
compatibility branch baru, dan downgrade surface baru.

## 0A - Perbandingan Attack Surface: Lama, Modern, dan Emerging

| Dimensi | Era Lama | Modern/Emerging |
|---|---|---|
| Target utama | OpenSSH source/binary | Dependency, build, patchset |
| Initial access | Root sudah ada | Bisa remote, supply chain, atau client-side input path |
| Detection | Hash/package check | Build artifact diff, runtime maps, anomaly telemetry |
| Scope | Single host | Distro/package population |
| Trigger | Login credential | Pre-auth signal, dynamic link, token expansion, KEX behavior |
| Key risk | Persistence | Initial access, supply chain, policy bypass, downgrade |

### Threat Landscape Matrix

| Attack Surface | Pre-auth? | Severity | Likelihood 2026-2028 |
|---|---:|---|---|
| Signal handler regression | Yes | Critical | Medium |
| Supply chain artifact | Yes | Critical | Medium-High |
| Downstream distro patch | Possible | High | High |
| ProxyCommand/token expansion | Client-side | Medium | Medium |
| Agent forwarding abuse | No | High | High |
| ControlMaster misuse | Local path | Medium | Medium |
| PQ downgrade/weak KEX policy | No | Long-term high | High |
| New parser/feature bugs | Possible | Medium | High |

Attack surface yang paling immediate berada pada downstream patches,
forwarding behavior, dan automation wrapper. Attack surface yang paling besar
dampaknya berada pada supply chain dan pre-authentication daemon path.

## 0B - Detection dan Forensic Analysis

Detection harus dipisahkan berdasarkan jenis serangan.

### XZ / liblzma Runtime Exposure

    Objective:
      Menentukan apakah sshd me-load liblzma melalui dependency chain.

    Defensive checks:
- periksa dependency sshd
- periksa dependency libsystemd
- periksa memory maps proses sshd yang sedang berjalan
- periksa versi xz/liblzma
- validasi package advisory distro
- bandingkan build artifact dengan source tree bila memungkinkan

    Signal yang perlu dicari:
- sshd me-load liblzma pada distro yang tidak diharapkan
- xz/liblzma 5.6.0 atau 5.6.1 pada environment terdampak
- latency login SSH meningkat tidak wajar
- CPU spike pada sshd sebelum autentikasi selesai
- child process anomali dari sshd

### regreSSHion Attempt Pattern

    Objective:
      Mendeteksi pola koneksi pre-authentication dalam volume tinggi.

    Signal:
- banyak koneksi tidak menyelesaikan identifikasi atau autentikasi
- banyak preauth connection closed
- MaxStartups pressure meningkat
- koneksi berumur mendekati LoginGraceTime
- sumber koneksi terdistribusi dengan timing pattern berulang

    Telemetry source:
- sshd logs
- journald/syslog
- firewall session logs
- NetFlow
- EDR process telemetry
- auditd untuk child process execution dari sshd

### ProxyCommand Injection Exposure

    Objective:
      Menentukan apakah client-side workflow memakai ProxyCommand dengan input
      username/URI yang tidak tepercaya.

    Area audit:
- ~/.ssh/config dan /etc/ssh/ssh_config
- automation script
- CI/CD deployment wrapper
- inventory generator
- bastion/jump host configuration
- Git submodule atau repository metadata yang memengaruhi SSH invocation

    Signal:
- ProxyCommand memakai %r atau user-derived expansion
- ssh command dibentuk dari string tanpa validasi
- username berasal dari repository, ticket, inventory, URL, atau API
- control characters tidak difilter sebelum command construction

### Downstream Patch Exposure

    Objective:
      Memahami perbedaan antara upstream OpenSSH dan paket distro.

    Area audit:
- changelog package distro
- distro patch directory
- build flags
- GSSAPIKeyExchange setting
- systemd-notify integration
- SELinux/GSSAPI/PAM-related patchset

### Forwarding Exposure

    Objective:
      Memastikan policy forwarding sesuai ekspektasi.

    Area audit:
- AllowAgentForwarding
- AllowTcpForwarding
- X11Forwarding
- PermitOpen
- PermitListen
- DisableForwarding
- Match blocks
- forced command behavior
- SSH certificate extensions

## 0C - Mitigasi dan Hardening Strategy

### Patch Baseline

- regreSSHion: upgrade ke OpenSSH 9.8p1 atau versi lebih baru.
- ProxyCommand injection: upgrade ke OpenSSH 10.1 atau versi lebih baru.
- DisableForwarding logic error: upgrade ke OpenSSH 10.0 atau versi lebih
      baru.
- XZ backdoor: pastikan xz/liblzma bukan 5.6.0 atau 5.6.1 pada distro
      terdampak; ikuti advisory distro.
- CVE-2026-3497: patch sesuai advisory distro; evaluasi GSSAPIKeyExchange.

### Runtime Hardening

- Batasi exposure port 22 dengan network ACL.
- Gunakan MaxStartups yang konservatif untuk menekan pre-auth flood.
- Hindari LoginGraceTime terlalu panjang.
- Gunakan AllowUsers/AllowGroups untuk memperkecil akses.
- Disable password authentication jika feasible.
- Gunakan SSH certificates dengan TTL pendek untuk environment besar.
- Batasi agent forwarding secara eksplisit.
- Audit X11 forwarding dan TCP forwarding.
- Gunakan FIDO2/U2F untuk private key protection.

### Supply Chain Controls

- Terapkan reproducible build verification untuk komponen kritikal.
- Bandingkan release tarball dengan repository state.
- Monitor dependency baru pada daemon kritikal.
- Wajibkan SBOM untuk image dan package internal.
- Pin versi package pada golden image.
- Subscribe ke advisory upstream dan distro.
- Audit maintainer change pada dependency kritikal.
- Jalankan canary environment sebelum update masuk production.

### Client-Side Controls

- Jangan bentuk ssh command dari input tidak tepercaya tanpa validasi.
- Hindari ProxyCommand yang mengevaluasi username secara langsung.
- Gunakan ProxyJump jika cukup.
- Filter control characters pada username, hostname, dan URI.
- Audit wrapper internal yang memanggil ssh.
- Treat repository metadata sebagai untrusted input.

### Forwarding Controls

- Set AllowAgentForwarding no untuk akun yang tidak membutuhkan.
- Set X11Forwarding no kecuali ada justifikasi eksplisit.
- Gunakan Match block untuk policy per-role.
- Uji policy forwarding dengan test case. Jangan berhenti pada pembacaan config.
- Hindari ControlPath di directory shared atau world-writable.
- Pastikan directory ControlPath mode 700.

## 0D - Defensive Research Methodology

Bagian ini menjelaskan pendekatan riset defensif untuk mencari bug class,
bukan instruksi eksploitasi.

### 1. Patchset Diff Review

- Bandingkan upstream OpenSSH dengan paket distro.
- Fokus pada patch yang menyentuh pre-authentication path.
- Audit semua patch yang menambah state machine, callback, IPC, atau
      dynamic linking.
- Tandai penggunaan fungsi disconnect, cleanup, dan error handling yang
      perilakunya tidak menghentikan proses.

### 2. Signal Safety Review

- Identifikasi semua signal handler.
- Tandai pemanggilan fungsi non-async-signal-safe.
- Fokus pada logging, allocation, environment access, dan library call.
- Verifikasi apakah handler hanya mengubah flag sederhana atau masuk ke
      jalur kompleks.

### 3. Parser and Token Expansion Review

- Audit hostname, username, URI, dan config expansion.
- Anggap command line, repository metadata, API inventory, dan generated
      config sebagai untrusted.
- Validasi control characters, NUL, newline, shell metacharacter, dan
      boundary condition.

### 4. Privilege Boundary Review

- Petakan data yang dikirim dari unprivileged child ke monitor/root side.
- Tandai uninitialized memory, size mismatch, dan ownership confusion.
- Audit lifecycle object yang dikirim lewat socketpair atau IPC internal.

### 5. Build Artifact Review

- Jangan hanya review git repository.
- Audit release tarball, generated m4/autotools files, configure script,
      test fixtures, dan object yang muncul saat build.
- Bandingkan hasil build reproducible dengan artifact distribusi.

### 6. Crypto Negotiation Review

- Audit KEX downgrade behavior.
- Periksa legacy algorithm exception.
- Pastikan warning non-PQ KEX direspons oleh policy, bukan diabaikan.
- Petakan host yang masih membutuhkan classical-only KEX.

## 0E - Kesimpulan

- **Target serangan terhadap OpenSSH tidak lagi terbatas pada daemon utama.** Area kritikal sekarang mencakup signal handler, dependency chain, build artifact, downstream patch, client-side expansion, forwarding behavior, dan cryptographic negotiation.

- **regreSSHion menunjukkan pentingnya regression testing untuk security bug lama.** Bug yang sudah diperbaiki dapat muncul kembali ketika invariant lama tidak ikut dijaga pada perubahan code path.

- **Kasus XZ Utils menunjukkan bahwa source review saja tidak cukup.** Release artifact, generated build files, maintainer trust, dan dependency loading path harus masuk ke threat model.

- **ProxyCommand injection menunjukkan bahwa ssh client juga menjadi attack surface.** Automation yang membangun ssh command dari input tidak tepercaya dapat menghasilkan command execution dalam workflow tertentu.

- **CVE-2026-3497 dan CVE-2025-32728 menunjukkan bahwa distro patch dan policy directive perlu diuji sebagai security boundary.** Kekuatan upstream OpenSSH tidak otomatis menghilangkan risiko pada paket downstream.

- **Transisi post-quantum KEX menambah kebutuhan audit pada policy dan compatibility path.** Organisasi perlu memantau downgrade, compatibility exception, dan implementasi algoritma baru.

- **OpenSSH perlu dianalisis sebagai rangkaian komponen.** Audit harus mencakup daemon, runtime dependency, build chain, distro patchset, client workflow, forwarding mechanism, dan cryptographic policy.

## 0F - Referensi

[1] Qualys TRU. "regreSSHion: Remote Unauthenticated Code Execution
      Vulnerability in OpenSSH Server (CVE-2024-6387)." 2024.
      https://www.qualys.com/regresshion-cve-2024-6387

[2] OpenSSH. "OpenSSH 9.8 Release Notes." 2024.
      https://www.openssh.com/txt/release-9.8

[3] Andres Freund. "backdoor in upstream xz/liblzma leading to ssh
      server compromise." oss-security mailing list. March 29, 2024.
      https://www.openwall.com/lists/oss-security/2024/03/29/4

[4] Rapid7. "Backdoored XZ Utils (CVE-2024-3094)." 2024.
      https://www.rapid7.com/blog/post/2024/04/01/etr-backdoored-xz-utils-cve-2024-3094/

[5] OpenSSH. "OpenSSH 10.1 Release Notes." 2025.
      https://www.openssh.com/txt/release-10.1

[6] OpenSSH. "OpenSSH 10.0 Release Notes." 2025.
      https://www.openssh.com/txt/release-10.0

[7] OpenSSH. "OpenSSH Security." 2025.
      https://www.openssh.com/security.html

[8] Ubuntu Security. "CVE-2026-3497." 2026.
      https://ubuntu.com/security/CVE-2026-3497

[9] Ubuntu Security Notice. "USN-8090-1: OpenSSH vulnerabilities." 2026.
      https://ubuntu.com/security/notices/USN-8090-1

[10] IBM Security Bulletin. "CVE-2025-61984, CVE-2025-61985 affect Power HMC." 2026.
       https://www.ibm.com/support/pages/security-bulletin-vulnerabilities-openssh-library-cve-2025-61984-cve-2025-61985-affect-power-hmc

[11] OpenSSH. "Post-Quantum Cryptography in OpenSSH."
       https://www.openssh.com/pq.html

[12] Qualys TRU. "CVE-2023-38408: Remote Code Execution in OpenSSH's
       Forwarded ssh-agent." 2023.
       https://blog.qualys.com/vulnerabilities-threat-research/2023/07/19/cve-2023-38408-remote-code-execution-in-opensshs-forwarded-ssh-agent

[13] Bäumer et al. "Terrapin Attack: Breaking SSH Channel Integrity by
       Sequence Number Manipulation." USENIX Security 2024.
       https://www.usenix.org/conference/usenixsecurity24/presentation/baumer

[14] NIST. "FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism
       Standard." 2024.
       https://csrc.nist.gov/pubs/fips/203/final

------------------------------------------------------------------------

k-elektronik.org  |  underground since 1995

EOF

------------------------------------------------------------------------
