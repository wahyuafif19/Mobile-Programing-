# Analisis Strategi dan Pola Integrasi REST API pada Aplikasi Mobile

**Mata Kuliah:** Mobile Programming Lanjutan (INF2.62.6005)
**Pertemuan:** 11 — Strategi dan Pola Integrasi REST API pada Aplikasi Mobile Skala Produksi
**Nama:** Wahyu Abdil Afif
**NIM:** 23343085
**Tanggal:** 20 Juni 2026
**Studi Kasus:** BCA Mobile

---

# BAGIAN A — TUGAS DEKOMPOSISI

> Tahap *Dekomposisi* dalam model CT-DINAMO: memecah sistem integrasi API aplikasi mobile menjadi sub-topik yang lebih kecil dan dapat dipelajari secara mandiri.

## A.1 Lapisan (Layer) Sistem Integrasi API

Sistem integrasi API pada aplikasi Flutter dipecah menjadi 6 lapisan agar setiap bagian memiliki tanggung jawab yang jelas, mudah diuji, dan mudah diganti tanpa mengubah lapisan lain (separation of concerns) — sejalan dengan Repository Pattern yang dibahas pada Materi 1 modul ini.

| # | Nama Lapisan | Tanggung Jawab | Package Flutter yang Digunakan |
|---|---|---|---|
| 1 | **Presentation Layer (UI)** | Menampilkan widget/halaman, menangkap input pengguna, menampilkan state (loading, success, error, empty, refreshing). Tidak memanggil API secara langsung. | `flutter/material.dart`, `flutter_bloc` / `provider` / `riverpod` |
| 2 | **State Management Layer** | Mengatur alur state asinkron aplikasi, menjembatani UI dengan business logic, merepresentasikan 6 state data (Initial, Loading, Success, Error, Empty, Refreshing). | `flutter_bloc`, `provider`, `riverpod`, `get` (GetX) |
| 3 | **Domain / Use Case Layer** | Business logic murni: validasi input, aturan bisnis, orkestrasi pemanggilan repository. Tidak bergantung pada HTTP atau database. | Dart murni, `equatable`, `dartz` |
| 4 | **Repository Layer** | Mengorkestrasi Remote dan Local Data Source: memutuskan kapan mengambil dari network, kapan dari cache, dan menangani fallback saat gagal. | Abstract class/interface repository (Dart murni) |
| 5 | **Network / Service Layer** | Konfigurasi HTTP client terpusat (base URL, timeout, header), melakukan request HTTP, serta menangani interceptor (auth, logging, retry). | `dio`, `retrofit`, `http`, `json_serializable` / `freezed` |
| 6 | **Local Storage / Persistence Layer** | Menyimpan token autentikasi secara aman, cache response API, dan preferensi pengguna untuk mendukung offline-first. | `flutter_secure_storage`, `hive`, `shared_preferences`, `sqflite` |

**Alur data antar lapisan:**

```
UI (Presentation)
   ↓ event/action
State Management (Bloc/Provider/Riverpod)
   ↓ memanggil
Domain / Use Case
   ↓ memanggil
Repository
   ↓ memutuskan sumber data
Network Service  <—atau—>  Local Storage
   ↓ response
(kembali naik ke atas melalui lapisan yang sama)
```

## A.2 Dekomposisi Skenario "Login ke Aplikasi Banking"

Diuraikan menjadi 12 langkah teknis, dari pengguna menekan tombol login hingga dashboard berhasil tampil, merujuk pada alur autentikasi JWT komprehensif di Materi 2 modul.

| Langkah | Aksi Teknis | Komponen Flutter / API / Local Storage |
|---|---|---|
| 1 | Pengguna mengisi user ID dan PIN/password, lalu menekan tombol "Login". | `TextFormField`, `ElevatedButton.onPressed` (Presentation) |
| 2 | UI memvalidasi format input di sisi client (field kosong, panjang PIN) sebelum mengirim event. | `Form` + `GlobalKey<FormState>`, `validate()` (Presentation) |
| 3 | UI mengirim event ke state management, contoh: `context.read<AuthBloc>().add(LoginRequested(...))`. | `flutter_bloc` / `riverpod` (State Management) |
| 4 | State management memanggil `LoginUseCase.execute()` untuk menjalankan business rule (misal cek apakah akun terkunci). | Class `LoginUseCase` (Domain) |
| 5 | Use Case memanggil `AuthRepository.login(userId, password)`. | Interface `AuthRepository` (Repository) |
| 6 | Repository memanggil `dio.post('/auth/login', body)` ke backend bank melalui HTTPS. | `dio` (Network), endpoint `POST /v1/auth/login` |
| 7 | `AuthInterceptor` menambahkan header standar (Content-Type, device-id, app-version) sebelum request dikirim. | `dio` `Interceptor` (Network) |
| 8 | Backend memverifikasi kredensial; jika valid mengembalikan `access_token` (exp 15 menit), `refresh_token` (exp 7 hari), dan profil pengguna dalam JSON. | Model `LoginResponse` via `json_serializable`/`freezed` (Network) |
| 9 | Token disimpan secara terenkripsi di local storage — **bukan** `shared_preferences` biasa karena bersifat sensitif. | `flutter_secure_storage` (Local Storage) |
| 10 | Data profil ringan (nama, no. rekening tersamar) disimpan di cache lokal agar dashboard dapat dimuat cepat. | `hive` / `shared_preferences` (Local Storage) |
| 11 | State management mengubah state menjadi `Success`/`LoginSuccess`; UI mendengarkan lewat `BlocListener` untuk memicu navigasi. | `BlocListener`, `Navigator.pushReplacement` |
| 12 | Dashboard terbuka dan memanggil API tambahan (`GET /account/balance`, `GET /transactions/recent`) dengan `access_token` otomatis disisipkan via interceptor pada header `Authorization: Bearer <token>`. | `FutureBuilder`/`BlocBuilder`, `dio` Interceptor |

**Catatan keamanan untuk konteks banking:**
- Token wajib disimpan di `flutter_secure_storage` (Keychain di iOS, Keystore/EncryptedSharedPreferences di Android), bukan `shared_preferences` biasa — sesuai poin 2.4 modul tentang bahaya menyimpan token di tempat yang tidak terenkripsi.
- Refresh token otomatis melalui `AuthInterceptor` (lihat kode pada modul 2.4) menjaga sesi tetap aktif tanpa mengusik pengguna.
- Pertimbangkan `local_auth` (biometric lock) sebagai lapisan tambahan sebelum token dipakai ulang saat re-open app.

## A.3 Daftar Sub-topik Integrasi API yang Perlu Dipelajari

### Tingkat Dasar (bisa dikerjakan hari ini)
- Konsep REST API: method HTTP (GET, POST, PUT, DELETE, PATCH)
- Format data JSON dan proses parsing ke Dart object (`fromJson`/`toJson`)
- Penggunaan package `http` untuk request sederhana
- Menampilkan data API ke UI dengan `FutureBuilder`
- Penanganan status code dasar (200, 400, 401, 404, 500)
- Penanganan error sederhana (try-catch, timeout)
- Struktur folder dasar project Flutter (model, service, screen)

### Tingkat Menengah (perlu 1–2 minggu)
- Migrasi dari `http` ke `dio` untuk fitur lanjutan (interceptor, base options)
- State management untuk 6 state data asinkron (Bloc, Provider, Riverpod)
- Autentikasi token (Bearer Token, Basic Auth) dan struktur JWT
- Local storage aman untuk token (`flutter_secure_storage`, `shared_preferences`)
- Repository Pattern dan pemisahan layer (remote/local data source, repository)
- Model generation otomatis dengan `json_serializable` / `freezed`
- Strategi caching dasar (Cache First, Network First) dengan Hive
- Paginasi offset-based dan infinite scroll dengan `ScrollController`
- Upload file/gambar ke API (`multipart/form-data`)

### Tingkat Lanjutan (perlu 1 bulan lebih)
- Refresh token otomatis & antrian request (401 handling) via Dio Interceptor + `Completer`
- Implementasi Clean Architecture penuh (data, domain, presentation terpisah total)
- Cursor-based dan keyset pagination untuk dataset besar
- Stale While Revalidate dan strategi offline-first menyeluruh
- Retry logic dengan exponential backoff dan deteksi *retryable error*
- Keamanan API tingkat lanjut: SSL Pinning, enkripsi payload, anti-tampering
- Monitoring konektivitas real-time dengan `connectivity_plus`
- Dependency Injection untuk skalabilitas (`get_it`, `injectable`)
- Testing integrasi API (unit test repository, mock dengan `mockito`/`mocktail`)
- Monitoring & logging API call (Firebase Crashlytics, Sentry)

---

# BAGIAN B — TUGAS TERSTRUKTUR (Analisis Studi Kasus Individual)

**Aplikasi Studi Kasus:** BCA Mobile

## B.1 Identitas & Konteks

| Keterangan | Isi |
|---|---|
| Nama | Wahyu Abdil Afif |
| NIM | 23343085 |
| Tanggal | 20 Juni 2026 |
| Aplikasi Studi Kasus | BCA Mobile (aplikasi mobile banking PT Bank Central Asia) |

BCA Mobile dipilih karena merepresentasikan kelas aplikasi dengan kebutuhan integrasi API paling ketat: keamanan data finansial, akurasi saldo real-time, serta toleransi nol terhadap data yang stale pada transaksi uang. Analisis berikut bersifat **inferensial** — disusun berdasarkan pola arsitektur umum aplikasi mobile banking dan prinsip-prinsip pada modul ini, bukan hasil reverse-engineering kode sumber resmi BCA Mobile yang bersifat tertutup/proprietary.

## B.2 Diagram Alur Integrasi API

Diagram berikut menggambarkan alur komprehensif: autentikasi (login → simpan token), request terautentikasi, refresh token otomatis, dan fallback ke cache saat offline.

![Diagram alur autentikasi BCA Mobile](diagram-alur-autentikasi.png)

Diagram mencakup titik keputusan: kredensial valid/tidak, ada koneksi/tidak, token kadaluarsa (401)/tidak, dan refresh token berhasil/gagal — sesuai cakupan flowchart pada modul (autentikasi, request terautentikasi, refresh token, dan offline fallback). Versi lengkap dengan tata letak rapi juga tersedia pada file `.docx`.

## B.3 Analisis Pattern yang Digunakan

Aplikasi sekelas BCA Mobile secara umum mengadopsi kombinasi **Repository Pattern** dan **Service Layer**, sejalan dengan Materi 1 modul ini.

**Service Layer** bertindak sebagai satu titik konfigurasi untuk seluruh komunikasi HTTP — base URL endpoint perbankan, timeout yang ketat (mengingat transaksi finansial tidak boleh menggantung terlalu lama), header wajib seperti device fingerprint, dan rangkaian interceptor untuk autentikasi serta logging. Tanpa Service Layer terpusat, setiap fitur (transfer, cek saldo, pembayaran) akan mengonfigurasi koneksinya sendiri-sendiri, yang membuat audit keamanan dan pembaruan kebijakan TLS menjadi sangat sulit dilakukan secara konsisten di seluruh aplikasi.

**Repository Pattern** menjadi lapisan abstraksi yang memisahkan "bagaimana data saldo/transaksi diperoleh" dari "bagaimana data tersebut ditampilkan". Untuk fitur seperti cek saldo, `BalanceRepository` akan mengorkestrasi antara `BalanceRemoteDataSource` (memanggil endpoint saldo terkini) dan `BalanceLocalDataSource` (cache sementara untuk tampilan instan), lalu memutuskan strategi mana yang dipakai berdasarkan kekritisan data — pola yang identik dengan contoh `ProductRepositoryImpl` pada modul (Materi 1.3).

**Mengapa pattern ini dipilih untuk konteks banking:**
1. **Testability** — Logika bisnis (misalnya validasi limit transfer harian) dapat diuji secara terisolasi tanpa benar-benar memanggil API produksi bank.
2. **Auditabilitas** — Pemisahan layer membuat jalur data finansial dapat ditelusuri dan diaudit, sebuah kebutuhan kepatuhan (compliance) yang ketat di industri perbankan.
3. **Keamanan terpusat** — Semua request finansial melewati satu Service Layer yang sama, sehingga kebijakan keamanan (sertifikat SSL, header wajib) dapat diterapkan secara konsisten di satu tempat.
4. **Fleksibilitas data source** — Repository dapat membedakan data yang *boleh* di-cache (riwayat transaksi lama) dari data yang *tidak boleh* di-cache (saldo real-time, status OTP) tanpa mengubah kode di lapisan UI.

**Diagram struktur folder (gambaran umum):**

```
lib/
├── data/
│   ├── datasources/
│   │   ├── remote/
│   │   │   ├── auth_remote_data_source.dart
│   │   │   ├── balance_remote_data_source.dart
│   │   │   └── transaction_remote_data_source.dart
│   │   └── local/
│   │       ├── token_local_data_source.dart
│   │       └── transaction_cache_data_source.dart
│   ├── models/
│   │   ├── login_response.dart
│   │   ├── balance_model.dart
│   │   └── transaction_model.dart
│   └── repositories/
│       ├── auth_repository_impl.dart
│       ├── balance_repository_impl.dart
│       └── transaction_repository_impl.dart
├── domain/
│   ├── entities/
│   │   ├── user.dart
│   │   └── transaction.dart
│   ├── repositories/            # abstract interface
│   │   ├── auth_repository.dart
│   │   ├── balance_repository.dart
│   │   └── transaction_repository.dart
│   └── usecases/
│       ├── login_usecase.dart
│       └── get_balance_usecase.dart
├── network/
│   ├── dio_service.dart
│   ├── interceptors/
│   │   ├── auth_interceptor.dart
│   │   └── logging_interceptor.dart
└── presentation/
    ├── login/
    ├── dashboard/
    └── transaction_history/
```

*(Jumlah kata bagian analisis pattern: ±320 kata)*

## B.4 Strategi Caching

Untuk aplikasi mobile banking, pemilihan strategi caching per endpoint sangat krusial karena keseimbangan antara *kecepatan tampilan* dan *akurasi data finansial* — sesuai 5 strategi caching pada Materi 4 modul.

| Endpoint | Strategi Caching | Alasan |
|---|---|---|
| `GET /account/balance` (saldo rekening) | **Network First** | Saldo adalah data paling kritis dan tidak boleh stale. Selalu coba ambil dari server terlebih dahulu; cache hanya dipakai sebagai fallback darurat saat benar-benar offline, disertai label "data terakhir, mungkin tidak akurat". |
| `GET /transactions/history` (riwayat transaksi lama, > 24 jam) | **Cache First** | Transaksi yang sudah selesai dan tercatat tidak berubah lagi, sehingga aman ditampilkan dari cache untuk kecepatan, sambil disegarkan di background. |
| `GET /reference/banks` (daftar bank tujuan transfer) | **Cache Only** | Data referensi statis yang jarang berubah. Diambil sekali (prefetch) dan disimpan; invalidasi manual dilakukan saat ada update dari server (versioning). |
| `POST /transfer/verify-otp` (verifikasi OTP transfer) | **Network Only** | Data keamanan kritis dengan masa berlaku sangat singkat; tidak boleh ada fallback offline maupun cache sama sekali. |
| `GET /dashboard/promo` (banner promo di home) | **Stale While Revalidate** | Bukan data finansial sensitif. Cache lama ditampilkan instan agar dashboard terasa cepat, sambil data terbaru diambil di background dan UI diperbarui begitu tersedia. |

## B.5 Error Handling Plan

Rancangan penanganan error untuk skenario kegagalan, mengikuti matriks error handling komprehensif pada Materi 5 modul, dengan penekanan pada konteks transaksi finansial.

| Skenario Error | Kode Status / Exception | Tindakan yang Diambil | Tampilan UI |
|---|---|---|---|
| Tidak ada koneksi internet | `SocketException` / Network Error | Cek konektivitas via `connectivity_plus`; jangan kirim transaksi finansial dalam kondisi ini sama sekali. | Banner "Tidak ada koneksi internet" + tombol nonaktif sementara untuk transaksi |
| Server error saat transfer | HTTP 500/502/503 | **Tidak** melakukan retry otomatis untuk transaksi finansial (berbeda dari GET biasa) — risiko transaksi ganda. Tampilkan status "tertunda", arahkan cek riwayat transaksi. | Dialog "Transaksi sedang diproses, mohon cek riwayat sebelum mencoba lagi" |
| Token kadaluarsa (401) saat cek saldo | HTTP 401 Unauthorized | `AuthInterceptor` otomatis memanggil `/auth/refresh`; jika berhasil, ulangi request asli secara transparan. | Tidak ada perubahan terlihat oleh pengguna (seamless) |
| Refresh token juga kadaluarsa | HTTP 401 pada endpoint refresh | Hapus seluruh token dari `flutter_secure_storage`, arahkan ke halaman login. | Dialog "Sesi Anda telah berakhir, silakan login kembali" |
| PIN/OTP salah saat transfer | HTTP 400 Bad Request (`INVALID_OTP`) | Parse error code spesifik dari response; jangan retry otomatis demi keamanan (cegah brute force). | Pesan "Kode OTP salah, sisa percobaan: 2" |
| Saldo tidak cukup | HTTP 400 (`INSUFFICIENT_BALANCE`) | Tampilkan pesan spesifik berdasarkan error code dari server, bukan asumsi dari sisi client. | Dialog "Saldo Anda tidak cukup untuk transaksi ini" |
| Timeout saat transfer | `DioException` (timeout) | Set timeout realistis (connect 10s, receive 30s); **jangan** retry otomatis untuk transaksi yang sudah terkirim — arahkan cek status terlebih dahulu. | "Koneksi lambat. Mohon cek status transaksi di Riwayat sebelum mencoba lagi" |

## B.6 Contoh Kode Dio Interceptor

Implementasi `AuthInterceptor` yang menangani penambahan auth header, deteksi 401, dan refresh token otomatis dengan antrian request — diadaptasi dari pola pada Materi 2.4 modul untuk konteks BCA Mobile.

```dart
class BcaAuthInterceptor extends Interceptor {
  final SecureTokenStorage _tokenStorage;
  final AuthRepository _authRepo;

  // Antrian request yang menunggu proses refresh token selesai
  final List<ErrorInterceptorHandler> _pendingRequests = [];
  bool _isRefreshing = false;

  // STEP 1: Sisipkan access token ke setiap request keluar
  @override
  void onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await _tokenStorage.getAccessToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    // Header tambahan khusus konteks banking
    options.headers['X-Device-Id'] = await _tokenStorage.getDeviceId();
    handler.next(options);
  }

  // STEP 2: Tangani error 401 dan lakukan refresh token
  @override
  void onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    final statusCode = err.response?.statusCode;

    // Jangan refresh untuk endpoint sensitif (OTP, transfer)
    // — biarkan error diteruskan langsung demi keamanan
    final isSensitiveEndpoint =
        err.requestOptions.path.contains('/transfer') ||
        err.requestOptions.path.contains('/otp');

    if (statusCode == 401 && !isSensitiveEndpoint) {
      if (_isRefreshing) {
        // Antrikan request ini sampai proses refresh selesai
        _pendingRequests.add(handler);
        return;
      }
      _isRefreshing = true;
      try {
        final newToken = await _authRepo.refreshToken();
        await _tokenStorage.saveAccessToken(newToken);

        // Ulangi request asli dengan token baru
        final retryResponse =
            await _retryRequest(err.requestOptions, newToken);
        handler.resolve(retryResponse);

        // Selesaikan semua request yang sempat mengantri
        for (final pending in _pendingRequests) {
          final resp = await _retryRequest(err.requestOptions, newToken);
          pending.resolve(resp);
        }
      } catch (_) {
        // Refresh token gagal — paksa logout demi keamanan
        await _tokenStorage.clearAll();
        handler.reject(err);
      } finally {
        _isRefreshing = false;
        _pendingRequests.clear();
      }
    } else if (statusCode == 403) {
      // 403 = otorisasi ditolak, bukan masalah token — jangan refresh
      handler.next(err);
    } else {
      handler.next(err);
    }
  }

  Future<Response> _retryRequest(
    RequestOptions requestOptions,
    String newToken,
  ) {
    final options = Options(
      method: requestOptions.method,
      headers: {
        ...requestOptions.headers,
        'Authorization': 'Bearer $newToken',
      },
    );
    return Dio().request(
      requestOptions.path,
      data: requestOptions.data,
      queryParameters: requestOptions.queryParameters,
      options: options,
    );
  }
}
```

**Catatan desain kode:**
- Endpoint sensitif (`/transfer`, `/otp`) **tidak** diikutsertakan dalam mekanisme auto-refresh dan retry — kegagalan pada endpoint ini dibiarkan diteruskan ke business logic agar pengguna mengulang transaksi secara sadar, mencegah risiko transaksi ganda akibat retry otomatis.
- Status 403 (Forbidden) sengaja dibedakan dari 401 (Unauthorized) — sejalan dengan poin Materi 5.1: 403 adalah masalah otorisasi, bukan token kadaluarsa, sehingga tidak perlu memicu refresh token.

---

## Pengumpulan

- **Platform:** GitHub README.md / Notion (format digital)
- **Format nama file/repo:** `Analisis_P11_23343085_Wahyu_Abdil_Afif`
- **Deadline:** 48 jam setelah pertemuan
- **Pengumpulan:** Link repositori GitHub atau Notion dikirimkan ke Google Drive kelas
