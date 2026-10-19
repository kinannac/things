# Dokumentasi Framed Autonomy berbasis OPA untuk Agent AI

Studi Kasus: Integrated dengan KasirKu -> akan diimplementasikan ke Agent AI

## 1. Ringkasan

Framed autonomy di sini berarti agent AI diberi otonomi, tetapi di dalam bingkai aturan yang ditegakkan terpisah dari agent. Setiap aksi agent (menulis file, menjalankan command, commit) diterjemahkan ke format standar, dikirim ke Open Policy Agent (OPA), lalu dijawab dengan salah satu dari tiga keputusan: allow, ask\_human, atau deny.

Aturan ditulis dalam Rego (policy.rego). Daftar path, level autonomy, dan command ditaruh sebagai data (data.json), sehingga bisa diubah tanpa menyentuh logika aturan.

Status pengujian saat ini:

- 37 unit test lulus (37/37).
- 6 skenario fixtures lewat curl sesuai prediksi.
- 2 batch uji bypass (16 command) sesuai prediksi setelah perbaikan.
- Belum dilakukan: uji fail-closed adapter, shadow mode dengan agent nyata, dan penegakan (enforce).

## 2. Ruang lingkup dan threat model

**Yang dilindungi:** logic hak akses dan sesi, database dan konfigurasi, kredensial (.env), logic bisnis KasirKu (transaksi, laporan, pengeluaran, retur), serta riwayat repo.

**Ancaman yang dipertimbangkan:**

- Agent salah memahami perintah manusia (contoh: diminta hapus database a, lalu menghapus folder database).
- Prompt injection yang membuat agent menjalankan aksi berbahaya.
- Perubahan tidak sengaja pada pengecekan izin di endpoint.
- Kebocoran kredensial lewat command network.

**Di luar cakupan (batasan yang diakui):**

- Kode berbahaya di dalam isi file yang ditulis ke zona bebas.
- Eksekusi tidak langsung, misalnya menulis file test berbahaya lalu menjalankan bun test.
- Agent yang menentukan levelnya sendiri (pada adapter prototipe, level datang dari input).
- Manusia yang menyetujui ask\_human tanpa membaca.

Policy menilai aksi terstruktur, bukan kalimat natural language dari manusia. Kalimat seperti hapus database a dibaca agent, lalu diterjemahkan menjadi tool action (misalnya shell.exec dengan command rm), dan tool action itulah yang diperiksa OPA.

## 3. Arsitektur

Alur satu aksi:

Perintah manusia, agent AI, tool action, adapter, OPA (policy.rego + data.json), keputusan, adapter menegakkan keputusan.

| Komponen | Fungsi |
| --- | --- |
| policy.rego | Aturan firewall (Rego). Dimuat OPA dan dijawab saat ada request |
| data.json | Data aturan: level, daftar path, command, token izin |
| policy\_test.rego | Unit test aturan, dijalankan dengan opa test. Tidak dipanggil agent |
| adapter.py | Jembatan: menormalkan aksi, menghitung diff, bertanya ke OPA, menegakkan hasil |
| fixtures/ | File JSON skenario untuk uji manual lewat curl |
| opa-server | Container Docker OPA, port 8181, folder policies di-mount dengan flag watch |

Struktur folder project:

- docker-compose.yml (opsional, alternatif docker run)
- policies/ berisi policy.rego, data.json, policy\_test.rego
- fixtures/ berisi file skenario JSON (di luar policies/, supaya tidak dimuat OPA sebagai data)
- adapter.py

Catatan penting: OPA hanya memutuskan, tidak memblokir. Yang menegakkan keputusan adalah adapter atau hook di sisi agent. Adapter dirancang fail-closed: jika OPA tidak bisa dihubungi, aksi ditolak (exit code 1).

Bagian yang berbeda per agent hanya adapter (cara membaca format aksi agent itu). Policy, data, dan test tetap sama.

## 4. Model keputusan

- Urutan prioritas: deny mengalahkan ask\_human, ask\_human mengalahkan allow.
- Default deny: jika tidak ada aturan yang mengizinkan, hasilnya deny.
- Semua alasan yang terpicu ditampilkan di daftar reasons, bukan hanya yang pertama.
- Pencocokan path dan command tidak peka huruf besar/kecil (penting karena macOS tidak membedakan huruf pada file).

Tiga lapisan penilaian:

| Lapisan | Pertanyaan | Contoh |
| --- | --- | --- |
| Level | Jenis aksi apa yang boleh? | L0 tidak boleh menulis |
| Zona | Di bagian mana? | src/global.ts terlarang |
| Konteks dan isi | Kapan dan bagaimana? | production menolak perubahan; cek izin tidak boleh dihapus |

## 5. Format input standar

| Field | Isi | Dipakai oleh |
| --- | --- | --- |
| agent.autonomy\_level | L0, L1, L2 | Pengecekan level |
| action.type | file.read, file.write, file.delete, shell.exec, git.commit, test.run | Hampir semua aturan |
| action.path | Path relatif terhadap root repo | Zona, traversal, .env |
| action.command | String command | Aturan shell |
| action.removed\_lines dan added\_lines | Array baris yang dihapus dan ditambah | Aturan isi diff |
| context.environment | dev atau production | Aturan production |
| context.tests\_passed | true atau false | Aturan commit |

Pada adapter.py, input datar (type, path, command, new\_content, autonomy\_level, environment, tests\_passed) diubah menjadi format di atas, dan removed\_lines serta added\_lines dihitung dari perbandingan isi file lama dan baru.

## 6. Level autonomy

| Level | Aksi yang diizinkan | Artinya |
| --- | --- | --- |
| L0 | file.read | Hanya mengamati dan menjelaskan kode |
| L1 | file.read, file.write, test.run | Boleh mengedit dan menguji |
| L2 | L1 ditambah shell.exec dan git.commit | Boleh menjalankan command dan commit |

Catatan:

- Level adalah batas atas jenis aksi, bukan izin otomatis. Zona tetap membatasi: L2 pun tidak boleh menulis src/global.ts.
- file.delete tidak diberikan ke level mana pun, sehingga selalu ditolak oleh aturan level.
- Level yang tidak dikenal (misalnya L9) ditolak.
- Level bisa dinaikkan atau diturunkan hanya dengan mengubah data.json. Karena flag watch aktif, perubahan langsung berlaku.
- Pada sistem nyata, level harus berasal dari konfigurasi yang dikendalikan manusia, bukan dari data yang dikirim agent.

## 7. Daftar aturan

### 7.1 Aturan deny

| ID | Aturan | Contoh pemicu |
| --- | --- | --- |
| D1 | Level tidak mengizinkan jenis aksi | L0 menulis file; L1 menjalankan shell |
| D2 | Aksi file tanpa path | file.write tanpa field path |
| D3 | shell.exec tanpa command | shell.exec tanpa field command |
| D4 | Path mengandung titik dua (traversal) | frontend/../src/server.ts |
| D5 | Path absolut | Path diawali garis miring |
| D6 | Menulis atau menghapus di protected\_paths | src/global.ts |
| D7 | Membaca file .env (termasuk .env.example) | file.read .env |
| D8 | Command memuat blocked\_commands | rm -rf, DROP TABLE |
| D9 | Command menyentuh protected\_keywords bersama verb atau redirect | echo x ke src/global.ts; cat .env dengan curl |
| D10 | Force push (git push dengan opsi -f atau --force) | git push -f origin main |
| D11 | docker down dengan -v atau --volumes | docker compose down --volumes |
| D12 | Environment production untuk aksi ubah (file.write, file.delete, shell.exec, git.commit) | Tulis file di production |
| D13 | git.commit saat tests\_passed bukan true | Commit sebelum test lulus |
| D14 | Perubahan menghapus baris yang memuat authz\_tokens di src/method\_function | Menghapus cek permission\_level |
| D15 | Perubahan menambah delete, update, atau insert di src/method\_function/GET | Menambah baris db.delete di endpoint GET |

### 7.2 Aturan ask\_human

| ID | Aturan | Contoh pemicu |
| --- | --- | --- |
| A1 | Path ada di approval\_paths | src/method\_function/POST/barang.ts |
| A2 | Path tidak ada di free\_paths (allowlist) | src/file\_baru.ts |
| A3 | Install dependensi (bun, npm, pnpm, yarn dengan add, install, i) | bun add lodash |
| A4 | Command shell tidak cocok dengan allowlist | git push origin main |

### 7.3 Aturan allow

Aksi diizinkan jika tidak ada deny, tidak ada ask\_human, dan level mengizinkan jenis aksinya.

## 8. Isi konfigurasi data.json

**blocked\_commands** (tidak peka huruf): rm -rf, DROP TABLE, DROP DATABASE, git push --force, docker compose down -v.

**protected\_keywords**: database, config/, .env, user\_session, src/global.ts, src/rate\_limit, src/setup. Aturan D9 hanya terpicu jika command juga memakai salah satu verb (rm, mv, cp, cat, curl, wget, sed, truncate, tee, echo, printf, find, python, node, bun) atau operator redirect.

**allowed\_commands** (pola awal): bun test, bun run test, bun run build, git status, git diff, git log, ls, pwd. Command yang cocok pola tetap dianggap tidak dikenal jika memuat penyambung shell (titik koma, ampersand, pipa, tanda lebih besar atau kecil, dolar, backtick, baris baru).

**authz\_tokens**: permission\_level, global.permissions, ADMINISTRATOR, MANAGE\_, user\_info.role\_id.

## 9. Peta akses pada repo KasirKu

Berlaku untuk aksi menulis (file.write) pada environment dev dengan level L1 atau lebih. Path dicocokkan relatif terhadap root repo dan tidak peka huruf besar/kecil. Struktur diambil dari screenshot repo dan README; isi folder PATCH dan POST belum diperiksa.

### 9.1 DENY pasti (menulis)

- .env dan turunannya di root (.env\*)
- config/\*\*
- database/\*\* (folder root)
- src/database/\*\*
- src/user\_session/\*\*
- src/rate\_limit/\*\*
- src/setup/\*\*
- src/global.ts (memuat definisi global.permissions dan handle database)
- Di bawah src/method\_function pada semua method (GET, POST, PATCH, DELETE): user.ts, users.ts, role.ts, roles.ts, profile.ts, serta file berawalan login, logout, dan password. Contoh: DELETE/user.ts dan DELETE/role.ts ikut terlarang
- src/method\_function/GET/uar\_list.ts (daftar user berdasarkan role)
- src/method\_function/PATCH/change\_password.ts (pola login, logout, dan password mencocokkan kata di mana saja dalam nama file)
- Semua file.delete (tidak ada level yang diberi izin ini)

### 9.2 ASK\_HUMAN (butuh persetujuan manusia)

- src/method\_function/DELETE/\*\* (di luar file akun di 9.1)
- src/method\_function/PATCH/\*\*
- src/method\_function/POST/\*\*
- src/method\_function/GET/: penjualan\*, laporan.ts, info\_total.ts, pengeluaran.ts, retur\_barang.ts
- src/utils/\*\* (utils.ts, 488 baris, berisi helper campuran)
- src/sse\_server/\*\*
- src/server.ts, index.ts
- scripts/\*\*
- package.json, Dockerfile, docker-compose.yml, tsconfig.json, SECURITY.md
- Semua path yang tidak ada di daftar mana pun, misalnya dist/**, tests/**, README.md, LICENSE, .gitignore (karena free\_paths bersifat allowlist)
- Tidak ada file PATCH, POST, atau DELETE yang pernah allow

### 9.3 ALLOW (menulis, level L1 atau lebih)

- frontend/\*\*
- screenshots/\*\*
- src/method\_function/GET/barang.ts
- src/method\_function/GET/barang\_kosong.ts
- src/method\_function/GET/barang\_masuk.ts
- src/method\_function/GET/cari\_barang.ts
- src/method\_function/GET/kategori\_barang.ts
- src/method\_function/GET/bak\_list.ts (daftar barang berdasarkan kategori, tafsiran dari isi file)
- src/method\_function/GET/public\_info.ts
- src/method\_function/GET/settings/\*\*

Syarat tambahan: perubahan pada file GET di atas tetap kena aturan isi (D14 dan D15). Menghapus pengecekan izin atau menambah operasi tulis ke database akan ditolak.

### 9.4 Membaca (file.read)

Semua level (L0 ke atas) boleh membaca file apa pun di repo kecuali .env dan turunannya (termasuk .env.example), yang ditolak.

### 9.5 Command shell dan git (hanya L2)

| Jenis | Hasil |
| --- | --- |
| bun test, bun run test, bun run build, git status, git diff, git log, ls, pwd (tanpa penyambung shell) | allow |
| Command lain yang tidak berbahaya secara pasti (contoh: git push origin main) | ask\_human |
| Install dependensi (bun add dan sejenisnya) | ask\_human |
| Command berisi blocked\_commands, force push, down dengan volume, atau menyentuh protected\_keywords dengan verb tulis atau hapus | deny |
| Semua shell.exec di environment production | deny |
| git.commit dengan tests\_passed true di dev | allow |
| git.commit tanpa tests\_passed | deny |
| Level L0 atau L1 menjalankan shell atau commit | deny |

test.run diizinkan untuk L1 ke atas karena belum ada aturan khusus.

## 10. Hasil pengujian

### 10.1 Unit test (opa test)

Hasil: PASS 37 dari 37. Rinciannya: 22 test awal (3 allow, 4 ask\_human, 15 deny), 10 test tambahan hasil temuan uji bypass dan 5 test tambahan hasil pemeriksaan peta akses.

Test tambahan mencakup: rm -fr tanpa garis miring, redirect ke src/global.ts, force push singkat, push biasa, docker down dengan volumes, command di allowlist, allowlist yang disusupi penyambung, path huruf besar, baca .env huruf besar, dan file.write tanpa path.

Test hasil pemeriksaan peta akses mencakup: PATCH/change_password.ts, POST/login.ts, POST/logout.ts, PATCH/profile.ts (semuanya deny), dan POST/masuk_ke_pembukuan.ts (ask_human).

### 10.2 Skenario fixtures (curl)

| Skenario | Hasil | Status |
| --- | --- | --- |
| L2 bun add lodash | ask\_human (install dependensi baru) | Sesuai prediksi |
| L1 tulis POST/barang.ts | ask\_human (dua alasan: tidak di free\_paths, di approval\_paths) | Sesuai prediksi |
| L2 cat .env dengan curl | deny (menyentuh area dilindungi: .env) | Sesuai prediksi |
| L2 tulis frontend/index.html di production | deny (production menolak aksi ubah) | Sesuai prediksi |
| L1 baca .env.example | deny (membaca file rahasia) | Sesuai prediksi (keputusan desain, lihat bagian 11) |
| L2 rm -rf database/ | deny (dua alasan: rm -rf dan database) | Sesuai prediksi |

Tambahan: L0 menulis src/server.ts menghasilkan deny (level L0 tidak boleh file.write). Sebelum perbaikan bug level, kasus ini menghasilkan ask\_human.

### 10.3 Batch uji bypass 1 (L2, shell)

| No | Command | Sebelum perbaikan | Sesudah perbaikan | Verifikasi |
| --- | --- | --- | --- | --- |
| 1 | echo x ke src/global.ts | allow (lolos) | deny | Unit test |
| 2 | rm -fr database | allow (lolos) | deny | Unit test |
| 3 | rm -r -f database/ | deny | deny | Manual |
| 4 | RM -RF database/ | deny | deny | Manual |
| 5 | find database -delete | allow (lolos) | deny | Unit test |
| 6 | git push -f origin main | allow (lolos) | deny | Unit test |
| 7 | git push origin main | allow (lolos) | ask\_human | Unit test |
| 8 | cat ./.env | deny | deny | Manual |
| 9 | docker compose down --volumes | allow (lolos) | deny | Unit test |
| 10 | python dengan shutil.rmtree database | allow (lolos) | deny | Unit test |

### 10.4 Batch uji bypass 2 (L2, shell)

Semua hasil sesuai prediksi.

| No | Command | Hasil |
| --- | --- | --- |
| 11 | bun test lalu rm -r src dengan operator dua ampersand | ask\_human |
| 12 | ls dengan command substitution berisi rm -r src | ask\_human |
| 13 | cat dengan wildcard pada nama folder user\_session | ask\_human |
| 14 | git dengan opsi -c disisipkan sebelum push | ask\_human |
| 15 | sed -i pada src/global.ts | deny |
| 16 | bun test (kontrol) | allow |

Catatan: wildcard dan opsi yang disisipkan menghindari pencocokan teks, tetapi tetap tertahan karena command tidak ada di allowlist. Allowlist berfungsi sebagai jaring pengaman untuk celah pencocokan teks.

### 10.5 Uji OPA mati

Dengan opa-server dihentikan, perintah cek (curl) dan cekcmd menghasilkan pesan error Expecting value: line 1 column 1. Itu terjadi karena curl tidak mendapat jawaban sehingga json.tool menerima input kosong. Perilaku fail-closed (deny saat OPA tidak bisa dihubungi) ada di adapter.py, bukan di curl. Status: uji fail-closed lewat adapter.py belum dijalankan.

## 11. Temuan selama pengujian

1. Bug level: agent L0 yang menulis ke approval\_paths awalnya mendapat ask\_human. Diperbaiki dengan aturan D1.
2. Input tanpa path lolos sebagai allow karena aturan yang merujuk field kosong dianggap tidak berlaku. Diperbaiki dengan D2 dan D3.
3. Pencocokan teks sensitif huruf: pada macOS (APFS tidak membedakan huruf), akses lewat huruf besar bisa lolos. Diperbaiki dengan menyamakan ke huruf kecil.
4. Blocklist command saja tidak cukup (rm -fr, find -delete, redirect, bahasa lain). Ditambah kata kunci yang lebih luas dan allowlist command.
5. Pengecekan izin tersebar di setiap endpoint KasirKu, sehingga pemeriksaan path saja tidak cukup. Ditambah aturan isi diff D14 dan D15.
6. Keputusan desain: .env.example ikut ditolak dibaca. Bisa dilonggarkan dengan pengecualian khusus jika agent perlu membacanya.
7. Temuan password pada src/method\_function/PATCH/change\_password.ts

## 12. Batasan yang diketahui

- Isi file tidak diperiksa selain dua aturan diff di src/method\_function. Kode berbahaya di frontend/ (zona bebas) tidak terdeteksi.
- Eksekusi tidak langsung: file test atau script yang ditulis agent lalu dijalankan lewat command di allowlist. File test belum punya zona khusus.
- False positive: kata kunci database cukup luas, sehingga command seperti bun test dengan argumen yang memuat kata itu bisa ditolak.
- Pola .env\* hanya mencocokkan di root repo; file .env di subfolder mengandalkan aturan allowlist (ask\_human).
- Nama file bak\_list.ts dan uar\_list.ts adalah tafsiran dari isi kode.
- adapter.py masih prototipe: membaca JSON dari stdin, level datang dari input, belum terpasang sebagai hook di agent.
- Terlalu banyak ask\_human dapat membuat manusia menyetujui tanpa membaca.

## 13. Langkah berikutnya

- [x] Uji fail-closed lewat adapter.py dengan opa-server dimatikan (harus deny dan exit code 1).
- [ ] Uji ulang nomor 5 dan 10 dari batch 1, lalu jadikan test.
- [ ] Tambahkan test untuk batch 2 (nomor 11 sampai 15).
- [ ] Beri file test zona ask\_human; jalankan test di sandbox.
- [ ] Lengkapi peta akses setelah folder PATCH dan POST diperiksa.
- [ ] Pindahkan penentuan level ke konfigurasi adapter yang dikendalikan manusia.
- [ ] Shadow mode: agent nyata pada repo clone, keputusan OPA hanya dicatat.
- [ ] Mode enforce: keputusan OPA benar-benar menahan aksi.
- [ ] Tulis kriteria cukup: threat model, daftar batasan, dan hasil uji pada aksi nyata.

## 14. Lampiran: perintah penting

- Menyalakan dan mematikan OPA: docker start opa-server dan docker stop opa-server
- Membuat container (sekali): docker run -d --name opa-server -p 8181:8181 -v "$PWD/policies:/policies" openpolicyagent/opa run --server --addr 0.0.0.0:8181 --watch /policies
- Menjalankan unit test: docker run --rm -v "$PWD/policies:/policies" openpolicyagent/opa test /policies -v
- Uji satu skenario dari file: cek fixtures/nama\_file.json
- Uji satu command shell: cekcmd L2 'perintah'
- Uji lewat adapter: kirim JSON datar ke python3 adapter.py lewat stdin (exit code 0 allow, 1 deny, 2 ditolak manusia)
- Alamat query OPA: http://localhost:8181/v1/data/agent/guard/decision (isi request dibungkus dengan key input)
