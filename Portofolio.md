# Portofolio Praktikum Sistem Operasi

## Identitas

- Nama: Dinda Azzahra
- NIM: 2510512035
- Kelompok: 6 (Kelas A)
- Judul proyek: Rancangan dan Implementasi Sistem Otomasi Backup Data Linux Berbasis Checksum dan Pembatasan Akses Docker
- Pilihan proyek: 5A Otomasi Backup dan Pemulihan Data
- Peran: Anggota 5, Locking, logging, dan Docker

## Asal Proyek

Proyek ini dikerjakan secara berkelompok. Repository ini adalah *fork* dari repository utama kelompok:
https://github.com/likaxheri/kelompok6_5A_SistemOperasi

| Nama | NIM | Komponen |
| :---- | :---- | :---- |
| Dimah Alika Cahyani | 2510512014 | Validasi sumber dan parameter |
| Audira Syifa Amalia Rasyid | 2510512011 | Pengarsipan data & pengamatan sumber daya |
| Lucky Ardillah | 2510512029 | Verifikasi integritas (checksum.sh) |
| Dinda Azzahra | 2510512035 | Pemulihan data (restore.sh) |
| Alya Dyah Sari | 2510512036 | Locking, Logging, & Docker |

Proyek ini bukan karya pribadi sepenuhnya. Bagian yang saya kerjakan dijelaskan di bawah.

## Kontribusi Pribadi

**Skrip**
- `scripts/backup-lock.sh`: mencegah dua proses backup berjalan bersamaan menggunakan mekanisme file locking (flock -n) pada file descriptor 9 (evidence/backup.lock), mengembalikan exit status 6 jika proses sedang terkunci (locked), menjalankan skrip utama ./backup.sh "$@", dan meneruskan exit status hasil eksekusi ke sistem.

**Logging, Laporan, dan Konfigurasi Isolasi Docker**
- Menerapkan pencatatan aktivitas locking ke evidence/locking.log menggunakan tee -a dengan penanda waktu (timestamp) dan status ([LOCK ACQUIRED], [LOCKED], [SUCCESS], [ERROR]).
- Menyusun ringkasan laporan operasional sistem ke dalam berkas output/laporan.txt yang memuat status locking, status backup, serta ringkasan riwayat dari evidence/locking.log dan evidence/backup.log.
- Mengonfigurasi pembatasan akses laporan menggunakan container Docker (debian:trixie-slim) dengan bind mount mode read-only (-v /opt/backup-sistem/output:/report:ro).

**Pengujian**
- Skenario Uji 5 (KU-5.1 sampai KU-5.5) meliputi pengujian locking tunggal, penolakan proses backup bersamaan (concurrent), pencatatan log berhasil dan gagal, akses baca laporan melalui Docker, penolakan tulis pada mount read-only Docker, serta verifikasi integritas SHA-256 pada output/laporan.txt.
- Pengamatan penggunaan sumber daya (CPU dan memori) pada pengujian beban besar (100M data acak) menggunakan /usr/bin/time -v bersama Audira Syifa Amalia Rasyid.

**Dokumentasi (laporan)**
- Bagian 1.4, 3.5, 3.6, 4.5, 5.5, serta 7.3.

## Bukti Kontribusi

| Pekerjaan | Tautan |
| :---- | :---- |
| `scripts/backup-lock.sh` | [ISI: link commit] |
| Log pengujian ke `evidence/` | [ISI: link commit] |
| Laporan sistem | [ISI: link commit atau berkas] |
| Commit akhir repository pribadi | [ISI: link commit] |

Cara mengambil link commit: buka tab *Commits* di GitHub, klik commit yang dimaksud, lalu salin URL halaman.

## Skenario Individu: Locking, Logging, dan Akses Docker Read-Only

**Tujuan.** Membuktikan bahwa `validasi.sh` menerima input yang valid dan menolak input yang salah dengan pesan jelas dan exit status non-zero sebelum backup berjalan.

**Kondisi awal.** Skrip scripts/backup-lock.sh memiliki izin eksekusi (-rwxr-xr-x), direktori evidence/ dan output/ tersedia di /opt/backup-sistem, serta image Docker debian:trixie-slim siap dijalankan.

| ID | Perintah | Diharapkan | Aktual | Status |
| :---- | :---- | :---- | :---- | :---- |
| KU-5.1 | `./scripts/backup-lock.sh ../dataSumber ../dataBackup` | exit 1 | exit 1 | Lulus |
| KU-5.2 | `./scripts/backup-lock.sh ../dataSumber ../dataBackup` | exit 2 | exit 2 | Lulus |
| KU-5.3 | `./scripts/backup-lock.sh dataSumber dataBackup` | exit 3 | exit 3 | Lulus |
| KU-5.4 | `docker run --rm -v /opt/backup-sistem/output:/report:ro debian:trixie-slim cat /report/laporan.txt` | exit 0 | exit 0 | Lulus |
| KU-5.5 | `docker run --rm -v /opt/backup-sistem/output:/report:ro debian:trixie-slim sh -c 'echo "TEST" >> /report/laporan.txt'` | exit 0 | exit 0 | Lulus |

**Verifikasi.** Status keluaran diperiksa melalui $?, riwayat pencatatan diperiksa menggunakan tail -n 5 evidence/locking.log dan tail -n 5 evidence/backup.log, isi ringkasan diverifikasi dengan cat output/laporan.txt, serta keutuhan berkas laporan diperiksa dengan sha256sum.

**Pemulihan.** Menghapus direktori uji beban sementara di /tmp/uji-beban-besar dan memastikan tidak ada file lock aktif yang menahan proses backup berikutnya.

## Analisis Konsep Sistem Operasi

- **Critical section dan sinkronisasi proses.** Bagian pembuatan arsip pada direktori sumber dan tujuan yang sama merupakan critical section yang rentan mengalami race condition jika dua proses backup berjalan bersamaan. Perintah flock -n 9 meminta exclusive lock secara non-blocking sehingga proses kedua langsung ditolak tanpa saling menimpa arsip maupun log.
- **File descriptor.** Perintah exec 9>"$LOCK_FILE" membuka evidence/backup.lock dan mengaitkannya dengan file descriptor nomor 9 milik proses shell agar kunci tetap aktif selama skrip berjalan dan otomatis dilepas oleh kernel ketika proses selesai.
- **Manajemen berkas dan pencatatan log (audit trail).** Penggabungan tee -a dan operator append (>>) memungkinkan pesan status ditampilkan di layar sekaligus ditambahkan ke akhir evidence/locking.log dan evidence/backup.log tanpa menghapus riwayat eksekusi sebelumnya.
- **Isolasi container dan kontrol akses I/O.** Docker menjalankan proses pada namespace dan filesystem yang terisolasi dari host. Opsi :ro pada -v /opt/backup-sistem/output:/report:ro memaksa kernel menolak setiap system call penulisan ke mount tersebut (Read-only file system).
- **Utilisasi sumber daya.** Pada pengujian beban besar (100M data acak) menggunakan /usr/bin/time -v di VM 1 vCPU dan RAM 2048 MB, kompresi gzip menyerap utilisasi CPU hingga 94% dengan puncak memori proses 2868 kB ($\approx 2,80\text{ MB}$) dan 0 swap.

Keterbatasan yang saya sadari: skrip backup-lock.sh berpindah direktori kerja ke scripts/ sebelum memanggil ./backup.sh "$@", sehingga argumen jalur relatif dari direktori induk proyek harus disesuaikan menjadi ../dataSumber dan ../dataBackup. Perbaikannya dapat dilakukan dengan mengonversi argumen jalur input menjadi jalur absolut (realpath) sebelum berpindah direktori.

## Kendala dan Penyelesaian

| Kendala | Penyelesaian |
| :---- | :---- |
| Menjalankan ./scripts/backup-lock.sh dataSumber dataBackup dari /opt/backup-sistem gagal dengan exit code 2 karena skrip melakukan cd "$SCRIPT_DIR" sebelum memanggil ./backup.sh. | Menyesuaikan parameter jalur relatif menjadi ./scripts/backup-lock.sh ../dataSumber ../dataBackup agar mengarah tepat ke direktori sumber dan tujuan dari dalam folder scripts/. |
| Format tanggal pada perintah pembuatan output/laporan.txt sempat salah ketik (%:%H%:M%S) sehingga menghasilkan tampilan waktu yang keliru. | Memperbaiki baris tanggal secara langsung di dalam berkas menggunakan perintah sed -i "s/^Tanggal laporan.*/Tanggal laporan : $(date '+%Y-%m-%d %H:%M:%S')/" output/laporan.txt. |
| Utilitas /usr/bin/time belum terpasang di VM Debian sehingga pengukuran metrik detail CPU dan memori pada beban besar sempat gagal dijalankan. | Memasang paket time terlebih dahulu dengan apt update && apt install -y time, lalu menjalankan ulang pengukuran /usr/bin/time -v tar -czf ... hingga selesai (Exit status: 0). |
| Terdapat berkas sementara editor (scripts/.jalankan-backup.sh.swp) yang terdeteksi sebagai untracked file pada git status. | Memeriksa status dengan git status --short serta mengecualikan/membersihkan berkas .swp agar tidak ikut terunggah ke repository GitHub. |
| Berkas log atau laporan pada folder evidence/ atau output/ dapat terabaikan oleh aturan .gitignore kelompok saat hendak diunggah ke GitHub. | Memeriksa aturan abaian dengan git check-ignore -v dan menggunakan git add -f pada berkas bukti pengujian terpilih yang diperlukan untuk dokumentasi kelompok. |

## Refleksi

**Yang sudah saya kuasai.** [ara kerja sinkronisasi proses dan file locking di Linux menggunakan flock -n dan file descriptor (exec 9>), penanganan exit status ($?), pencatatan log menggunakan tee -a, pembatasan akses bind mount read-only (:ro) pada container Docker, serta pengukuran penggunaan CPU dan memori proses menggunakan /usr/bin/time -v.]

**Yang masih perlu saya pelajari.** [Automasi penjadwalan tugas backup secara berkala menggunakan cron atau systemd timer, penanganan normalisasi jalur (path resolution) otomatis pada skrip Bash agar mendukung jalur relatif maupun absolut dari mana saja, serta analisis perbandingan bottleneck I/O dan kompresi pada berbagai tingkat beban data.]

## Sumber dan Atribusi

Proyek dikerjakan berkelompok. Referensi: manual Linux (man flock, man bash, man tar, man sha256sum, man time), dokumentasi resmi Docker (docker run bind mounts read-only), serta materi kuliah Sistem Operasi. Menggunakan bantuan AI Gemini untuk membantu penyusunan logika skrip backup-lock.sh, pengujian skenario di Debian, serta perapian dokumentasi laporan dan portofolio.
