---
title: "Berburu IDOR: Saat Server Terlalu Percaya pada Client"
dek: "IDOR adalah bug yang selamat dari setiap upgrade framework. Begini cara saya menemukannya, kenapa ia bersembunyi di depan mata, dan satu pertanyaan yang mengubah 200 OK menjadi sebuah temuan."
date: 2026-06-18
tags: [BugBounty, IDOR, Access Control, Web]
venue: BugBounty
reading_time: 9 mnt
draft: false
---

Insecure Direct Object Reference adalah bug yang sudah dibaca semua orang tapi
hampir semua orang masih merilisnya. Ia tidak eksotis. Tidak ada payload pintar,
tidak ada trik encoding, tidak ada celah race yang harus dijahit. Kamu mengubah
sebuah angka, dan server menyerahkan data yang tidak pernah jadi milikmu.
Alasan ia bertahan bukan karena kesulitan teknis. Melainkan karena otorisasi
adalah satu-satunya kontrol yang tidak bisa di-autopilot oleh framework
untukmu.

Begini cara saya menghadapi IDOR di target nyata, dari request pertama sampai
laporan yang lolos triage.

## Apa itu IDOR sebenarnya

Buang akronimnya dan ia tinggal satu kalimat: **aplikasi memakai identifier
yang dikirim client untuk mengambil sebuah objek, dan lupa mengecek apakah
pengguna saat ini berhak memilikinya.**

```http
GET /api/invoices/10423 HTTP/1.1
Authorization: Bearer <token-alice>
```

Server membaca `10423`, mencarinya, dan mengembalikannya. Ia mengautentikasi
Alice — ia tahu *siapa* dia. Ia hanya tidak pernah bertanya apakah invoice
`10423` *milik* dia. Autentikasi menjawab "siapa kamu." Otorisasi seharusnya
menjawab "apakah kamu boleh melihat ini," dan tidak ada yang bertanya.

Celah itulah keseluruhan bug-nya.

## Kenapa framework tidak menyelamatkanmu

Stack modern memberikan autentikasi nyaris gratis. Sebuah decorator, sebuah
middleware, sebuah session guard — pasang saja dan setiap route menuntut token
yang valid. Itulah jebakannya. Tim melihat gembok hijau di setiap endpoint lalu
menganggap access control sudah beres.

Tapi autentikasi itu global dan mekanis. Otorisasi itu per-objek dan
kontekstual. Tidak ada framework yang bisa tahu bahwa invoice `10423` milik org
`88` dan bahwa Alice hanya anggota org `42`. Logika itu hidup di domain *kamu*,
jadi *kamu* yang harus menulis pengecekannya — pada setiap pencarian objek.
Lewatkan satu saja, dan endpoint itu menjadi IDOR.

> Bug-nya bukan karena developer menulis kode yang buruk. Melainkan karena ia
> *tidak* menulis kode di tempat yang membutuhkan pengecekan, dan tidak ada
> apa pun di toolchain yang protes.

## Menemukannya: petakan, lalu mutasikan

Workflow saya membosankan, dan itu disengaja. Membosankan berarti bisa diulang.

### 1. Inventarisasi setiap identifier

Jelajahi aplikasi sebagai pengguna biasa dengan proxy menyala. Catat setiap
request yang membawa referensi objek:

- Path param numerik: `/orders/5512`, `/users/77/settings`
- UUID: `/files/3f9a-...` (ya, ini jauh lebih sering bisa ditebak daripada yang orang kira)
- ID di query string: `?account_id=42`
- ID tersembunyi di body JSON: `{"projectId": 19}`
- ID di header: `X-Account-Context: 42`

Yang sering dilupakan orang adalah body dan header. Hunter yang hanya mem-fuzz
URL melewatkan separuh permukaan serangan.

### 2. Selalu pakai dua akun

Ini kebiasaan paling penting. Daftarkan **dua** akun independen, idealnya di dua
org/tenant terpisah. Sebut saja *penyerang* (Alice) dan *korban* (Bob).

Tangkap sebuah request sebagai Bob yang mengembalikan salah satu objeknya:

```http
GET /api/invoices/20001 HTTP/1.1
Authorization: Bearer <token-bob>
```

Sekarang putar ulang dengan **token Alice** tapi **id objek Bob**:

```http
GET /api/invoices/20001 HTTP/1.1
Authorization: Bearer <token-alice>
```

### 3. Ajukan satu pertanyaan

Saat request kedua itu kembali, tanyakan:

**Apakah Alice baru saja menerima data Bob?**

- `200 OK` dengan body invoice Bob → IDOR terkonfirmasi.
- `403 / 404` → pengecekannya ada, lanjut ke yang lain.
- `200 OK` dengan body kosong atau terfilter → sebagian; gali lebih dalam.

Itulah keseluruhan tesnya. Autentikasi utuh selama proses — Alice benar-benar
login. Kamu sedang menguji lapisan di atasnya.

## Variasi yang membayar

Begitu pembacaan dasar berhasil, kelemahan yang sama biasanya membusuk ke sisa
siklus hidup objek itu. Uji setiap verb, bukan hanya `GET`:

| Method   | Yang kamu uji                                   |
| -------- | ----------------------------------------------- |
| `GET`    | Membaca objek orang lain                         |
| `PUT`    | Menimpa objek orang lain                          |
| `DELETE` | Menghancurkan objek orang lain                    |
| `POST`   | Membuat objek *di dalam* resource orang lain      |

IDOR read-only adalah kebocoran. IDOR `DELETE` adalah outage. Tim triage
membayar untuk perbedaannya, jadi selalu demonstrasikan verb berdampak tertinggi
yang diterima endpoint itu.

Beberapa tempat lain ia bersembunyi:

- **Sepupu mass assignment.** `PATCH /users/me` dengan body
  `{"org_id": 42}` — bisakah kamu memindahkan dirimu ke org lain?
- **Referensi bersarang.** `/orgs/42/projects/19` mungkin mengecek org `42` tapi
  mempercayai `19` begitu saja. Tukar hanya id bagian dalam.
- **ID "acak" yang dapat ditebak.** UUIDv1 sekuensial, id berbasis timestamp,
  atau base64 dari sebuah integer bukanlah hal yang tak bisa ditebak. Dekode
  dulu sebelum kamu menganggapnya aman.
- **Endpoint export dan report.** Generator PDF/CSV terkenal sering melewatkan
  pengecekan kepemilikan yang ditegakkan oleh tampilan utama.

## Menulis laporan

IDOR terkonfirmasi dengan laporan yang lemah akan diturunkan severity-nya. Buat
triage menjadi mudah:

1. **Nyatakan dua identitas.** "Akun A (penyerang) dan Akun B (korban), tanpa
   hubungan di antara keduanya."
2. **Tunjukkan request-nya.** Token Alice, id objek Bob, header lengkap.
3. **Tunjukkan response-nya.** Data Bob, dengan field yang membuktikan
   kepemilikan (email-nya, nama org-nya).
4. **Nyatakan dampak dalam bahasa mereka.** "Setiap pengguna terautentikasi
   dapat membaca, mengubah, atau menghapus invoice milik tenant lain mana pun
   dengan menaikkan path parameter `id`."

Kalimat terakhir itulah yang menggerakkan severity. Kamu tidak melaporkan
`200 OK`. Kamu melaporkan akses data lintas-tenant tanpa batas otorisasi.

## Perbaikannya, agar kamu bisa menjelaskannya

Saat program bertanya cara remediasi — dan yang bagus pasti bertanya —
jawabannya selalu berbentuk sama: **scope-kan setiap pencarian objek ke
principal saat ini.**

```sql
-- rentan: hanya mempercayai id
SELECT * FROM invoices WHERE id = :id;

-- diperbaiki: id DAN pemilik harus cocok
SELECT * FROM invoices WHERE id = :id AND org_id = :current_org;
```

Lakukan pengecekan di lapisan data, bukan hanya controller, supaya tidak bisa
dilewati oleh route yang lupa guard-nya. Sentralisasikan. Filter kepemilikan
per-query yang diterapkan ORM secara otomatis lebih baik daripada `if` yang
ditulis tangan di setiap endpoint, karena bug-nya justru ada di endpoint yang
lupa menulis `if`-nya.

---

IDOR menghargai kesabaran di atas kepintaran. Dua akun, sebuah proxy, dan
disiplin untuk menukar satu identifier dalam satu waktu akan menemukan lebih
banyak bug nyata daripada scanner otomatis mana pun. Server terlalu percaya pada
client. Seluruh tugasmu adalah membuktikannya.
