# Langkah 5 — Monozukuri Implementation & Pull Request

## Ringkasan eksekusi

Implementasi mengikuti Option A dari Langkah 4 dan mencakup API Rails serta UI React. Perubahan menutup batas otorisasi di backend, memvalidasi hasil AI sebelum menyimpan, membuat fit-gap generation idempotent, dan menegakkan kontrak API–UI saat runtime.

### Yang diimplementasikan

| Area | Implementasi |
|---|---|
| Tenant isolation & auth | Menambah membership user–organization dengan role dan status aktif. Login hanya menerbitkan JWT untuk membership assessor/admin aktif. Setiap request REST memuat ulang user dan membership dari database; role claim JWT tidak menjadi sumber otorisasi. Selector X-Tenant-Scheme yang berbeda dari tenant token ditolak. Portfolio, turunannya, fit-gap, vacancy, ekspor dan override dibatasi melalui tenant sesi. REST dan kedua WebSocket yang menerima JWT kini memerlukan membership dan role yang sesuai. |
| Data assessment | Menambah assessment_status dan assessment_reason pada skill portfolio. Level/confidence boleh null hanya pada not_assessed; check constraint database menjaga pasangan status/nilai. Skill lama yang sudah mempunyai skor dimigrasikan sebagai assessed karena histori bukti tidak cukup untuk merekonstruksi status lain. |
| AI evidence | Portfolios::EvidenceValidator hanya menerima rating integer 1–5, confidence enum, summary/quote dengan batas ukuran, dan sekurangnya dua kutipan yang cocok dengan dua turn kandidat tersimpan yang berbeda. Evidence disimpan sebagai {turn_id, quote}. Skor tidak lagi di-coerce/clamp. Skill tanpa output/bukti sah menjadi not_assessed; transkrip di atas 120.000 karakter gagal eksplisit dan tidak dipotong diam-diam. Isi transkrip disebut sebagai data tak tepercaya pada prompt. |
| Coverage | Menghapus promosi partial → covered berdasarkan probe count saja. Status coverage kini harus berasal dari update analyzer; jumlah probe sendiri tidak mengubah state menjadi covered. |
| Fit-gap | Menambahkan status, error code dan generation token pada report. Record dibuat sebelum job diantrekan; lock pada record menggabungkan polling/request bersamaan. Worker memeriksa tenant portfolio/vacancy dan token generation, sehingga job lama tidak dapat menimpa revision baru. Regenerate mempertahankan hasil valid sebelumnya; laporan lama dengan override ditandai perlu dibuat ulang. |
| API–UI | API dan TypeScript menggunakan expected_level, candidate_level nullable, result: not_assessed, serta is_override boolean. Zod memvalidasi response sebelum dipakai. UI membedakan skill belum dinilai, menampilkan bukti dan field persyaratan, menghentikan polling setelah state terminal, dan menyatakan error/retry tanpa mengklaim generation berhasil. |
| Test harness | Menambahkan RSpec unit tests untuk membership authorization, WebSocket coverage authorization, validasi evidence, idempotency service, dan hasil fit-gap; menambah test runner Node bawaan untuk validasi kontrak web tanpa dependency baru. |

## Bukti verifikasi yang sudah dijalankan

| Pemeriksaan | Hasil |
|---|---|
| Dari api/: PATH=/Users/asep/.rbenv/versions/3.3.2/bin:$PATH bundle exec rspec | **12 examples, 0 failures** |
| Dari api/: PATH=/Users/asep/.rbenv/versions/3.3.2/bin:$PATH bundle exec rails zeitwerk:check | **All is good** |
| Dari web/: npm test | **4 tests passed** |
| Dari web/: npm run build | **TypeScript check dan Vite production build berhasil** |
| git diff --check | Bersih |

Peringatan Vite hanya tentang posisi komentar tree-shaking pada dependency Zod; build tetap berhasil.

## Seeded Fault Test

Ini adalah fault injection deterministik memakai fixture sintetis, bukan kredensial kandidat. Tes memasukkan beberapa kegagalan yang mewakili output AI rusak:

- level: "3", boolean, pecahan, 0, dan 6 alih-alih integer 1–5.
- Kutipan yang tidak ada dalam transkrip sumber.
- Dua kutipan dari turn kandidat yang sama, sehingga tidak membentuk dua evidence independen.
- Kutipan melebihi batas ukuran.
- Pada kontrak web, field lama required_level, level berbentuk string, atau hilangnya is_override.

Jalankan dari folder api/:

    PATH=/Users/asep/.rbenv/versions/3.3.2/bin:$PATH bundle exec rspec spec/services/portfolios/evidence_validator_spec.rb

Jalankan fault kontrak dari folder web/ bersama seluruh suite:

    npm test

Hasil yang diharapkan: evidence/score yang gagal tidak menghasilkan rating tersimpan dan dipresentasikan sebagai not_assessed; contract parser menolak field atau tipe yang tidak sah. Suite yang dijalankan pada workspace ini menghasilkan 4 contoh API validator dan 4 kasus kontrak web lulus.

## AI Verification Moment

Verifikasi lokal memakai payload model berbentuk JSON dan dua turn **kandidat sintetis** yang telah ditanam pada fixture test. Kutipan yang muncul pada turn 11 dan 12 menghasilkan assessment_status: assessed, ai_level: 3, serta evidence dengan kedua turn_id. Varian yang mengirim "3" sebagai string atau mengarang kutipan diturunkan menjadi not_assessed dengan reason code, tanpa menyimpan score.

Ini membuktikan validator dan jalur bukti secara offline; **tidak ada panggilan Gemini langsung pada sesi ini** dan tidak ada transkrip kandidat nyata yang digunakan. Uji live provider, akurasi semantik skill, serta perilaku model saat prompt injection masih perlu dijalankan pada environment dengan kredensial dan data sintetis yang disetujui.

## Migrasi dan persiapan tenant

Migrasi baru: api/db/migrate/20260924000000_add_tenant_memberships_and_generation_state.rb. Migrasi **belum dijalankan** pada database lokal. Akses ke 127.0.0.1:5432 ditolak oleh sandbox (Operation not permitted), sehingga perubahan SQL/schema perlu diterapkan dan diperiksa pada database pengembangan/CI yang tersedia sebelum merge/deploy.

Membership user lama sengaja tidak di-backfill otomatis. Setelah migrasi, pemilik tenant harus memetakan user ke organisasi yang benar; user tanpa membership tidak dapat login/mengakses REST. Untuk admin lokal saja, seed menerima ID eksplisit:

    SEED_ADMIN_USER_ID=<id-admin-yang-sudah-ada> bundle exec rails db:seed

Jangan menjalankan pemetaan massal lintas organisasi. Untuk produksi, gunakan pemetaan tenant yang diverifikasi dan prosedur provisioning organisasi. Pastikan migrasi berjalan pada database kosong dan database upgrade, lalu verifikasi constraint, index unik, foreign key ke organisasi, serta regresi request dua tenant.

## Pull request

Pull request **belum dibuat**. Repo memiliki remote GitHub, tetapi gh auth status gagal (CLI tidak terautentikasi); jaringan lokal database juga dibatasi. Perubahan saat ini tersimpan sebagai working-tree changes pada branch feature/product-engineer-revamp; belum ada commit atau push yang dibuat.

Setelah autentikasi GitHub tersedia dan migrasi diuji pada database pengembangan/CI, perubahan siap ditinjau sebagai PR. Periksa hanya file Step 5 dan kode yang berubah; catatan Step 2–4 yang sudah ada di working tree tidak termasuk scope PR Step 5.

## Batas verifikasi dan follow-up

- RSpec yang dijalankan adalah unit/service tests dengan dependency database digandakan. Belum ada request integration test terhadap PostgreSQL nyata. Migrasi, constraint, transaksi unik, locking lintas koneksi, serta foreign key perlu diverifikasi pada CI/database test sebelum merge.
- Uji tenant REST/WebSocket memakai membership test double; lakukan uji black-box dua tenant setelah database aktif dan provisioning selesai.
- Live Gemini verification belum dilakukan; hasil provider tidak boleh diklaim tervalidasi secara semantik.
- Batas retensi/penghapusan data, notice kandidat, dasar pemrosesan, DPIA, wilayah/transfer Gemini, dan proses keberatan tetap keputusan Pengendali/Legal sesuai Constraint Signal Langkah 4.
- User/tenant provisioning produksi belum disediakan sebagai UI/API admin pada perubahan ini. Kegagalan tertutup disengaja sampai membership diberikan secara sah.
