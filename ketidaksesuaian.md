**Ketidaksesuaian yang ditemukan**

Penyimpanan file. Sebagian besar dokumen sudah memakai keputusan "file disimpan di PostgreSQL", tetapi masih ada sisa desain lama yang bertentangan:



System-Architecture.md:330 menulis "Isi file biner tidak disimpan dalam PostgreSQL".

Tabel di baris 557 menyatakan PostgreSQL "tidak boleh menyimpan file biner".

Diagram di baris 159 menggambarkan browser mengakses file langsung lewat signed URL.

"URL bertanda tangan" muncul di NFR baris 36 dan Document-Generation baris 192 pada kedua varian, padahal aturan lain mewajibkan semua file dialirkan lewat NestJS.

ERD varian Konsultan belum mengikuti arsitekturnya:



Arsitektur meminta tabel penghubung consultation\_request\_payments (satu request bisa punya beberapa percobaan bayar). ERD hanya punya satu kolom payment\_id.

Belum ada tabel riwayat status komentar, delivery Penyusunan Penuh, atau relasi delivery-komentar yang disebut perlu untuk menjamin delivery tidak bisa diubah.

Tabel consultation\_assignments belum punya kolom assignment sebelumnya dan alasan revoke (diminta di bagian 9.1 arsitektur).

Tabel consultation\_workspaces belum punya kolom baseline dan promoted\_document\_version\_id.

Varian Konsultan masih menulis tiga aktor di beberapa tempat:



00-Business-Flow-Overview.md:5 masih menyebut "tiga aktor".

System Context Diagram belum memuat Konsultan.

Acceptance criteria PRD menyebut MFA hanya untuk User Induk dan Administrator, padahal Konsultan juga wajib MFA.

Format dan tautan:



Baris FR-12 terlepas dari tabelnya dan tercetak di bawah Acceptance Rules (Functional-Requirements.md:62).

PRD menautkan Use-Case-Diagram.drawio, tetapi file itu tidak ada.

