---
name: mcp-auth-wiring
description: Rancang autorisasi MCP 2026-07-28 dengan pendaftaran terikat issuer, CIMD, metadata sumber daya terlindungi, penyegaran JWKS, pinning audience, dan validasi per permintaan.
version: 2.0.0
phase: 13
lesson: 18
tags: [mcp, oauth, cimd, dcr, jwks, rfc8414, rfc7591, rfc8707, rfc7636, rfc9728, rfc9207]
---

Dengan sebuah konfigurasi server MCP dan sekumpulan kemampuan IdP, hasilkan permukaan autentikasi dan aturan penolakan yang membentuk lapisan autorisasi MCP produksi.

Masukan:

- `mcp_resource_url` — URL sumber daya kanonik (pengenal paling spesifik; simpan sebuah path hanya bila path itu membedakan server-server yang di-hosting bersama), dipakai sebagai `aud` dan sebagai nilai `resource` pada metadata sumber daya terlindungi.
- `idp_metadata_url` — URL `/.well-known/oauth-authorization-server` (atau OpenID Connect Discovery) milik IdP.
- `idp_capabilities`: nilai-nilai yang teramati untuk `issuer`, `code_challenge_methods_supported`, `grant_types_supported`, `client_id_metadata_document_supported`, `registration_endpoint` yang sudah usang, `response_types_supported`, dan `authorization_response_iss_parameter_supported`.
- `pre_registered_client_ids`: pemetaan opsional dari issuer ke client ID yang disediakan oleh operator server otorisasi. Utamakan identitas yang tercakup dalam issuer ini sebelum CIMD, lalu pakai DCR yang sudah usang hanya sebagai jalur kompatibilitas terakhir.
- `application_type`: `native` atau `web`, wajib bila kompatibilitas DCR yang sudah usang dipilih.
- `credential_store`: client ID dan kredensial pendaftaran yang dikunci menurut issuer server otorisasi, dengan token akses yang dikunci menurut `(issuer, mcp_resource_url)`.
- `tools`: daftar alat MCP dengan cakupan (scope) yang dibutuhkan masing-masing.

Hasilkan:

1. **Gerbang penolakan.** Tolak untuk merangkai dan berhenti bila ada kondisi keras yang gagal:
   - `S256` tidak ada di `code_challenge_methods_supported` (PKCE tidak punya mode degradasi).
   - `authorization_code` tidak ada di `grant_types_supported`.
   - `response_types_supported` berisi nilai apa pun selain tepat `["code"]`.
   - Tidak ada jalur pendaftaran: tidak tersedia `client_id` yang sudah terdaftar, `client_id_metadata_document_supported: true`, maupun endpoint kompatibilitas DCR yang sudah usang.
   - CIMD dipilih, tetapi `client_id`-nya bukan URL dokumen HTTPS absolut yang memiliki path, tidak cocok dengan URL dokumen, atau dokumen itu tidak memiliki `client_name` atau array `redirect_uris` yang tidak kosong. `application_type` bersifat opsional untuk CIMD.
   - Sebuah `iss` RFC 9207 yang dikembalikan berbeda dari issuer yang dicatat sebelum pengalihan, atau dihilangkan padahal server mengiklankan dukungannya.
   - DCR yang sudah usang tidak memiliki `application_type`, atau kebijakan redirect URI-nya bertentangan dengan `native` atau `web`.

2. **Dokumen metadata sumber daya terlindungi** (RFC 9728) untuk server MCP. Untuk sumber daya dengan path, sisipkan segmen well-known sebelum path tersebut: `https://host/team/mcp` dipetakan ke `https://host/.well-known/oauth-protected-resource/team/mcp`. Sertakan `resource`, `authorization_servers` (daftar izin issuer), `scopes_supported`, dan `bearer_methods_supported: ["header"]`.

3. **Endpoint HTTP.**
   - `GET /.well-known/oauth-protected-resource` — mengembalikan dokumen dari (2).
   - `POST /mcp` (transpor MCP stateless): memvalidasi token bearer untuk permintaan ini sebelum alat apa pun dipanggil.
   - Hanya kompatibilitas DCR: `POST /register`, dengan pemeriksaan application-type dan pemeriksaan pembatasan laju di depannya.

4. **Pekerjaan latar belakang + rutin.**
   - Penyegaran JWKS terjadwal yang mengambil ulang `jwks_uri` ke dalam tembolok `{keys, fetched_at}`. Idempoten; tidak pernah membuat kunci. AS yang melakukan rotasi; server sumber daya hanya menyegarkan. Default `0 */6 * * *`; perkecil menjadi `*/15 * * * *` untuk IdP dengan rotasi tinggi.
   - Rutin `validate` — memeriksa daftar izin `iss`, tanda tangan terhadap JWKS yang tembolok, `aud == mcp_resource_url`, `exp`, dan cakupan yang dibutuhkan.
   - Jalur penerbitan step-up — hanya bila daftar alat memuat operasi yang dibatasi oleh cakupan yang tidak diberikan pengguna pada awalnya.

5. **Rencana tembolok.** Satu entri per issuer yang diterima, dikunci menurut `issuer`, berisi `{keys, fetched_at}`. Dokumentasikan pola baca: validator membaca tembolok dan beralih ke satu penyegaran sinkron saat `kid` tidak ditemukan (ambil ulang, bukan rotasi — pengambilan ulang bersifat idempoten dan tidak dapat diubah menjadi serangan DoS pembuatan kunci).

6. **Pemetaan cakupan.** Petakan setiap alat ke cakupan yang dibutuhkannya. Keluarkan sebuah tabel:
   `| tool | required_scope | rationale |`. Kelompokkan alat-alat destruktif di bawah cakupannya sendiri; jangan pernah memakai ulang cakupan baca untuk alat tulis.

7. **Aturan penolakan saat runtime** (validator harus mengodekan ini):
   - Tolak bila `aud != mcp_resource_url` → 401 `Bearer error="invalid_token", error_description="audience mismatch", resource_metadata="<prm_url>"`.
   - Tolak bila `iss` tidak ada di `authorization_servers`.
   - Tolak bila `kid` tidak ada di JWKS yang tembolok setelah satu pengambilan ulang cadangan.
   - Tolak bila cakupan yang dibutuhkan tidak ada → 403 `Bearer error="insufficient_scope", scope="<required>", resource_metadata="<prm_url>"`.
   - Tolak setiap permintaan otorisasi tanpa `code_challenge` S256, dan tolak setiap permintaan token yang `code_verifier`, klien, redirect URI, atau `resource`-nya tidak cocok dengan catatan kode otorisasi sekali pakai.
   - Tolak setiap kredensial atau token yang issuer-nya tidak cocok dengan kunci penyimpanan kredensialnya. Perubahan issuer memerlukan pendaftaran ulang.

Penolakan keras (jangan pernah merangkai salah satu dari ini — tolak permintaan dan dokumentasikan alasannya):

- Menyimpan `client_secret` dalam teks biasa. Klien publik memakai `token_endpoint_auth_method: none`; klien rahasia memakai `private_key_jwt`. Tidak boleh ada rahasia bersama dalam teks biasa, baik saat disimpan maupun di log respons pendaftaran.
- Melewatkan pemeriksaan `aud` pada validator. Pengikatan audience (pembatasan hak istimewa token akses) adalah alasan penuh di balik RFC 8707 + RFC 9728.
- Menghubungkan cadangan JWKS saat `kid` tidak ditemukan ke operasi rotasi-dan-pembuatan kunci, bukan ke pengambilan ulang. Operasi itu tidak pernah menghasilkan `kid` yang hilang dan membiarkan nilai `kid` yang dikendalikan penyerang memaksa pembuatan kunci tanpa batas. Cadangan harus berupa penyegaran yang idempoten.
- Mengizinkan permintaan kode otorisasi tanpa PKCE. OAuth 2.1 melarangnya; validator harus menolak setiap pertukaran `/token` yang catatan kode otorisasi tersimpannya tidak memiliki `code_challenge`.
- Menembolok JWKS tanpa pekerjaan penyegaran. Baik pekerjaan penyegaran dikirimkan, maupun permukaan autentikasi tidak boleh diterapkan.
- Mempercayai klaim `iss` tanpa daftar izin. Validator mana pun yang menerima token dari `iss` sembarangan membiarkan penyerang mendirikan IdP-nya sendiri dan memalsukan token.
- Meneruskan token MCP yang masuk ke API upstream (token passthrough). Bila server MCP memanggil API upstream, server itu WAJIB memperoleh token tersendiri yang terpisah; passthrough menimbulkan masalah deputy yang bingung.
- Menyimpan `registration_access_token` dalam teks biasa. Hash saat disimpan; minta teks terang pada setiap pembaruan.
- Memperlakukan metadata permintaan MCP atau sesi protokol yang telah dihapus sebagai keadaan otorisasi. Transpor 2026-07-28 bersifat stateless; autentikasi dan otorisasi setiap permintaan.

Hasil: rencana satu halaman berisi dokumen sumber daya terlindungi, tata letak pendaftaran yang dikunci menurut issuer, tata letak token yang dikunci menurut issuer dan sumber daya, jalur pendaftaran yang dipilih, endpoint HTTP, pekerjaan penyegaran JWKS, pemetaan cakupan, dan aturan penolakan saat runtime. Akhiri dengan gerbang penerapan pertama yang belum terpenuhi yang ditemukan pada metadata aktual server otorisasi.
