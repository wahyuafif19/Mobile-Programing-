# Algoritma Penanganan Error dan Implementasi Repository Pattern pada Aplikasi Mobile

**Mata Kuliah:** Mobile Programming Lanjutan (INF2.62.6005)
**Pertemuan:** 11 — Strategi dan Pola Integrasi REST API pada Aplikasi Mobile Skala Produksi
**Nama:** Wahyu Abdil Afif
**NIM:** 23343085
**Tanggal:** 21 Juni 2026

---

> Tahap *Algoritma* dalam model CT-DINAMO: merancang urutan langkah belajar yang sistematis dan dapat diulang.

## 1. Flowchart Alur Checkout Aplikasi E-commerce

Flowchart berikut menggambarkan alur lengkap proses checkout, dari pengguna menekan tombol checkout hingga pesanan terkonfirmasi. Diagram mencakup 5 titik keputusan yang diminta: **autentikasi** (sudah login?), **koneksi jaringan** (ada koneksi?), **refresh token** (respons 401? → sub-alur refresh), **ketersediaan stok** (stok tersedia?), dan **pembayaran** (pembayaran berhasil?).

### 1.1 Alur Utama

![Flowchart alur utama checkout](flowchart_checkout.png)

### 1.2 Sub-alur: Refresh Token Otomatis

Ketika alur utama mendeteksi respons `401 Unauthorized` saat memanggil `GET /products/stock`, sistem masuk ke sub-alur berikut sebelum melanjutkan proses checkout:

![Sub-alur refresh token](flowchart_refresh_token.png)

**Penjelasan titik keputusan:**

| Titik Keputusan | Letak dalam Alur | Penanganan |
|---|---|---|
| **Autentikasi** | Setelah tombol checkout ditekan | Jika belum login, redirect ke halaman login sambil menyimpan isi cart sementara agar tidak hilang |
| **Koneksi jaringan** | Sebelum memanggil API stok | Jika tidak ada koneksi, tampilkan status offline dengan tombol retry manual (tidak retry otomatis karena melibatkan transaksi) |
| **Refresh token (401)** | Saat respons API stok mengembalikan 401 | Otomatis memanggil `/auth/refresh`; jika gagal, hapus token dan arahkan ke login; jika berhasil, ulangi request asli |
| **Ketersediaan stok** | Setelah data stok diterima | Jika stok tidak cukup, tampilkan pesan dan tawarkan pengguna menghapus/mengganti item sebelum melanjutkan |
| **Pembayaran** | Setelah `POST /orders/checkout` dikirim | Jika gagal, tampilkan pesan gagal bayar dan tawarkan metode pembayaran lain; jika berhasil, tampilkan konfirmasi pesanan |

---

## 2. Algoritma Penanganan Error

Algoritma berikut merinci kondisi, tindakan aplikasi, dan tampilan UI untuk 5 skenario kegagalan yang umum terjadi pada integrasi API, merujuk pada matriks error handling komprehensif Materi 5 modul.

### a) Tidak Ada Koneksi Internet

```
KONDISI:
  - Request API gagal dilempar dengan SocketException, atau
  - connectivity_plus melaporkan status ConnectivityResult.none

ALGORITMA:
  1. Tangkap exception pada lapisan Repository (try-catch di sekitar panggilan remote data source)
  2. JANGAN langsung melempar error ke UI -- cek dulu apakah ada data cache yang valid
  3. JIKA ada cache valid (Cache First/Network First dengan fallback):
       -> kembalikan data cache, tandai sebagai "data offline"
     SELESAI ke-> 5
  4. JIKA tidak ada cache:
       -> lempar custom exception NetworkException ke UI layer
  5. Mulai StreamSubscription pada connectivity_plus untuk memantau perubahan koneksi
  6. KETIKA koneksi kembali tersedia:
       -> trigger ulang request secara otomatis (untuk GET) ATAU
       -> aktifkan kembali tombol retry (untuk POST/transaksi)

TAMPILAN UI:
  - Banner non-intrusif di bagian atas: "Anda sedang offline"
  - Jika ada cache: tampilkan data dengan label kecil "Data terakhir tersimpan"
  - Jika tidak ada cache: empty state dengan ilustrasi + tombol "Coba lagi"
  - Nonaktifkan tombol aksi yang memerlukan koneksi (checkout, transfer, dll.)
```

### b) Server Error (HTTP 500)

```
KONDISI:
  - Response.statusCode berada pada rentang 500-599 (Internal Server Error,
    Bad Gateway, Service Unavailable)

ALGORITMA:
  1. Intersep response di DioInterceptor.onError() sebelum diteruskan ke UI
  2. Cek apakah operasi bersifat idempotent (GET, aman diulang) atau
     non-idempotent (POST transaksi, berisiko duplikasi jika diulang)
  3. JIKA idempotent (GET):
       -> jalankan retryWithBackoff() maksimal 3 kali
       -> delay: 1s -> 2s -> 4s (exponential backoff)
     JIKA gagal setelah 3 percobaan:
       -> lempar error ke UI, log ke Firebase Crashlytics
  4. JIKA non-idempotent (POST transaksi seperti checkout):
       -> JANGAN retry otomatis
       -> tampilkan status "tertunda" dan arahkan user mengecek riwayat
          sebelum mencoba ulang
  5. Log seluruh detail error (status code, endpoint, response body) untuk
     keperluan debugging, tanpa menampilkan detail teknis ke pengguna

TAMPILAN UI:
  - Untuk GET: loading indicator selama proses retry berlangsung
  - Setelah retry gagal: "Terjadi kesalahan pada server. Coba lagi nanti."
    dengan tombol "Coba Lagi"
  - Untuk POST/transaksi: dialog "Permintaan Anda sedang diproses. Silakan
    cek status sebelum mencoba lagi." (mencegah transaksi duplikat)
```

### c) Request Timeout

```
KONDISI:
  - DioException dengan type DioExceptionType.connectionTimeout,
    sendTimeout, atau receiveTimeout

ALGORITMA:
  1. Pastikan konfigurasi timeout sudah realistis di Service Layer:
       connectTimeout: 10 detik, receiveTimeout: 30 detik
     (timeout terlalu pendek menyebabkan false-positive error)
  2. Tangkap DioException di interceptor, cek field 'type'
  3. JIKA operasi idempotent (GET) dan belum mencapai batas retry:
       -> retry dengan exponential backoff (sama seperti skenario b)
  4. JIKA operasi non-idempotent (transaksi):
       -> JANGAN retry otomatis karena request asli mungkin sudah
          diterima server meski response belum kembali ke client
       -> arahkan pengguna mengecek status transaksi terlebih dahulu
  5. Tampilkan estimasi waktu tunggu jika memungkinkan (skeleton loading
     dengan placeholder, bukan spinner kosong saja)

TAMPILAN UI:
  - "Koneksi terlalu lambat. Coba lagi?" dengan tombol retry yang jelas
  - Untuk transaksi: "Koneksi lambat. Mohon cek status transaksi di
    Riwayat sebelum mencoba lagi."
```

### d) Token Kadaluarsa (HTTP 401)

```
KONDISI:
  - Response.statusCode == 401 Unauthorized

ALGORITMA:
  1. Intersep di AuthInterceptor.onError()
  2. Cek apakah endpoint termasuk kategori sensitif (contoh: /transfer,
     /otp) -- jika ya, JANGAN coba refresh, langsung teruskan error
     (mencegah retry otomatis pada operasi finansial kritis)
  3. JIKA bukan endpoint sensitif:
     a. JIKA sedang ada proses refresh berjalan (_isRefreshing == true):
          -> masukkan request ini ke dalam antrian (_pendingRequests)
          -> tunggu sampai proses refresh selesai
     b. JIKA belum ada proses refresh berjalan:
          -> set _isRefreshing = true
          -> panggil POST /auth/refresh dengan refresh_token tersimpan
          -> JIKA berhasil:
               * simpan access_token baru ke flutter_secure_storage
               * ulangi request asli dengan token baru
               * selesaikan seluruh request yang mengantri di langkah (a)
          -> JIKA gagal (refresh_token juga kadaluarsa):
               * hapus seluruh token dari secure storage
               * set state aplikasi ke "logged out"
               * arahkan pengguna ke halaman login
  4. Set _isRefreshing = false setelah proses selesai (baik berhasil
     atau gagal), kosongkan antrian

TAMPILAN UI:
  - Jika refresh berhasil: TIDAK ADA perubahan terlihat oleh pengguna
    (proses sepenuhnya transparan/seamless)
  - Jika refresh gagal: dialog informatif "Sesi Anda telah berakhir,
    silakan login kembali" lalu redirect, BUKAN crash diam-diam
```

### e) Format Data Tidak Sesuai

```
KONDISI:
  - JSON parsing melempar FormatException atau TypeError
  - Field yang diharapkan null/missing pada response API
  - Struktur response berubah dari kontrak yang disepakati (API versioning)

ALGORITMA:
  1. Bungkus SETIAP proses deserialisasi JSON (fromJson) dalam try-catch
     di lapisan Remote Data Source -- jangan biarkan exception
     menjalar tak terkendali ke Repository atau UI
  2. Tangkap exception spesifik:
       try {
         return Model.fromJson(response.data);
       } catch (e) {
         throw DataParsingException(
           message: 'Format data tidak sesuai',
           rawResponse: response.data, // untuk keperluan log, bukan ke user
         );
       }
  3. Log raw response (request URL, response body, stack trace) ke
     crash reporting (Firebase Crashlytics/Sentry) untuk investigasi tim
     backend -- ini biasanya indikasi API berubah tanpa pemberitahuan
  4. Tentukan fallback di Repository:
       JIKA ada cache valid -> tampilkan cache dengan peringatan
       JIKA tidak ada cache -> tampilkan error generik ke pengguna
  5. JANGAN PERNAH menampilkan detail teknis (nama field yang error,
     stack trace) langsung ke pengguna akhir

TAMPILAN UI:
  - Pesan generik dan tidak teknis: "Data tidak dapat dimuat saat ini.
    Tim kami sedang menangani masalah ini."
  - Tombol "Coba Lagi" untuk memicu request ulang
  - TIDAK menampilkan pesan seperti "FormatException: Unexpected
    character" ke pengguna
```

---

## 3. Urutan Langkah Implementasi Repository Pattern dari Nol

Berikut adalah urutan implementasi Repository Pattern secara bertahap pada proyek Flutter baru, dari pembuatan abstract class hingga terhubung ke UI layer — merujuk pada Materi 1 modul (komponen Repository Pattern) dengan studi kasus fitur "daftar produk".

### Langkah 1 — Definisikan Entity / Model Domain

Buat kelas data murni yang merepresentasikan domain object, terpisah dari struktur JSON API.

```dart
// lib/domain/entities/product.dart
class Product {
  final String id;
  final String name;
  final double price;
  final int stock;

  const Product({
    required this.id,
    required this.name,
    required this.price,
    required this.stock,
  });
}
```

### Langkah 2 — Buat Abstract Class (Repository Interface)

Definisikan kontrak operasi data. UI dan Use Case hanya akan bergantung pada interface ini.

```dart
// lib/domain/repositories/product_repository.dart
abstract class ProductRepository {
  Future<List<Product>> getProducts({int page = 1, int limit = 20});
  Future<Product> getProductById(String id);
}
```

### Langkah 3 — Implementasi Remote Data Source

Buat kelas yang mengambil data dari REST API menggunakan `dio`, termasuk model DTO dan konversi `fromJson`.

```dart
// lib/data/models/product_model.dart
class ProductModel {
  final String id;
  final String name;
  final double price;
  final int stock;

  ProductModel({required this.id, required this.name, required this.price, required this.stock});

  factory ProductModel.fromJson(Map<String, dynamic> json) => ProductModel(
        id: json['id'],
        name: json['name'],
        price: (json['price'] as num).toDouble(),
        stock: json['stock'],
      );

  Product toEntity() => Product(id: id, name: name, price: price, stock: stock);
}

// lib/data/datasources/remote/product_remote_data_source.dart
class ProductRemoteDataSource {
  final Dio _dio;
  ProductRemoteDataSource(this._dio);

  Future<List<ProductModel>> fetchProducts({int page = 1, int limit = 20}) async {
    final response = await _dio.get('/products', queryParameters: {'page': page, 'limit': limit});
    final List data = response.data['data'];
    return data.map((json) => ProductModel.fromJson(json)).toList();
  }
}
```

### Langkah 4 — Implementasi Local Data Source

Buat kelas untuk membaca/menulis data dari penyimpanan lokal (Hive) untuk keperluan caching.

```dart
// lib/data/datasources/local/product_local_data_source.dart
class ProductLocalDataSource {
  static const String _boxName = 'products_cache';

  Future<void> cacheProducts(List<ProductModel> products) async {
    final box = Hive.box(_boxName);
    await box.put('products', {
      'data': products.map((p) => p.toJson()).toList(),
      'cached_at': DateTime.now().toIso8601String(),
    });
  }

  Future<List<ProductModel>?> getCachedProducts() async {
    final box = Hive.box(_boxName);
    final cached = box.get('products');
    if (cached == null) return null;
    final List data = cached['data'];
    return data.map((json) => ProductModel.fromJson(json)).toList();
  }
}
```

### Langkah 5 — Implementasi Repository (Orkestrasi Remote + Local)

Hubungkan abstract class dengan kedua data source, tentukan strategi (Network First/Cache First, dsb.).

```dart
// lib/data/repositories/product_repository_impl.dart
class ProductRepositoryImpl implements ProductRepository {
  final ProductRemoteDataSource _remote;
  final ProductLocalDataSource _local;

  ProductRepositoryImpl({required ProductRemoteDataSource remote, required ProductLocalDataSource local})
      : _remote = remote, _local = local;

  @override
  Future<List<Product>> getProducts({int page = 1, int limit = 20}) async {
    try {
      final products = await _remote.fetchProducts(page: page, limit: limit);
      if (page == 1) await _local.cacheProducts(products);
      return products.map((p) => p.toEntity()).toList();
    } catch (e) {
      if (page == 1) {
        final cached = await _local.getCachedProducts();
        if (cached != null) return cached.map((p) => p.toEntity()).toList();
      }
      rethrow;
    }
  }

  @override
  Future<Product> getProductById(String id) async {
    final model = await _remote.fetchProductById(id);
    return model.toEntity();
  }
}
```

### Langkah 6 — Dependency Injection

Daftarkan seluruh dependensi (Dio, data source, repository) agar dapat di-inject ke Use Case/ViewModel tanpa coupling langsung ke implementasi konkret.

```dart
// lib/injection/injection_container.dart (menggunakan get_it)
final getIt = GetIt.instance;

void setupDependencies() {
  // Network
  getIt.registerLazySingleton<Dio>(() => DioService.createDio(
        baseUrl: 'https://api.tokopedia-clone.com',
        authInterceptor: getIt<AuthInterceptor>(),
      ));

  // Data sources
  getIt.registerLazySingleton<ProductRemoteDataSource>(
      () => ProductRemoteDataSource(getIt<Dio>()));
  getIt.registerLazySingleton<ProductLocalDataSource>(
      () => ProductLocalDataSource());

  // Repository
  getIt.registerLazySingleton<ProductRepository>(() => ProductRepositoryImpl(
        remote: getIt<ProductRemoteDataSource>(),
        local: getIt<ProductLocalDataSource>(),
      ));

  // Use case
  getIt.registerLazySingleton<GetProductsUseCase>(
      () => GetProductsUseCase(getIt<ProductRepository>()));
}
```

### Langkah 7 — Hubungkan ke UI Layer

State management (Bloc/Provider/Riverpod) memanggil Use Case, lalu UI mendengarkan perubahan state dan merender hasilnya.

```dart
// lib/presentation/product/product_bloc.dart
class ProductBloc extends Bloc<ProductEvent, ApiState<List<Product>>> {
  final GetProductsUseCase _getProductsUseCase;

  ProductBloc(this._getProductsUseCase) : super(const Initial()) {
    on<LoadProducts>((event, emit) async {
      emit(const Loading());
      try {
        final products = await _getProductsUseCase.execute();
        emit(products.isEmpty ? const Empty() : Success(products));
      } catch (e) {
        emit(Error(e.toString()));
      }
    });
  }
}

// lib/presentation/product/product_screen.dart
class ProductScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => getIt<ProductBloc>()..add(LoadProducts()),
      child: BlocBuilder<ProductBloc, ApiState<List<Product>>>(
        builder: (context, state) => switch (state) {
          Loading() => const Center(child: CircularProgressIndicator()),
          Success(:final data) => ProductListView(products: data),
          Error(:final message) => ErrorView(message: message),
          Empty() => const EmptyStateView(),
          _ => const SizedBox.shrink(),
        },
      ),
    );
  }
}
```

**Ringkasan urutan implementasi:**

```
1. Entity / Model domain    -> apa yang direpresentasikan
2. Abstract class           -> kontrak apa yang bisa dilakukan
3. Remote Data Source       -> bagaimana mengambil dari API
4. Local Data Source        -> bagaimana mengambil dari cache lokal
5. Repository Implementation -> bagaimana mengorkestrasi keduanya
6. Dependency Injection     -> bagaimana semua komponen terhubung
7. UI Layer (Bloc + Widget) -> bagaimana hasil ditampilkan ke pengguna
```

Urutan ini sengaja dimulai dari lapisan domain (paling independen) menuju UI (paling tergantung), sehingga setiap lapisan dapat diuji secara terisolasi sebelum lapisan berikutnya dibangun di atasnya.

---

## Pengumpulan

- **Format nama:** `Algoritma_P11_23343085_Wahyu_Abdil_Afif`
- **Deadline:** 48 jam setelah pertemuan
- **Pengumpulan:** Link dokumen + link diagram dikumpulkan di kolom submission
