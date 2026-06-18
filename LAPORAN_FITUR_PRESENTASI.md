# Laporan Teknis Fitur MediKlik — Bahan Presentasi & Pertahanan (Defense)

> Dokumen ini menjelaskan cara kerja fitur (Frontend & Backend), keputusan desain di baliknya, contoh kasus nyata, dan **potensi blind spot / hal yang mungkin terlewat oleh developer**. Bahasa dibuat sederhana namun tetap teknis untuk audiens software engineer.

**Arsitektur singkat MediKlik:**
- **Frontend (FE):** React + TypeScript + Vite, data fetching pakai **SWR**, state global pakai **Zustand**.
- **Backend (BE):** Go (Gin), arsitektur berlapis **handler → usecase → repo (PostgreSQL)**. DB pakai **PostGIS** untuk perhitungan jarak geografis.
- **Algorithm service:** Python (FastAPI) — berisi optimizer *cheapest-nearest* (MILP/scipy), rekomendasi ML (TF-IDF + KNN), dan chatbot (Gemini).
- **Observability:** OpenTelemetry → Prometheus (metrik) + Loki (log) + Grafana (dashboard), Node Exporter (metrik server).

---

# BAGIAN A — ROLE USER

## A1. Seluruh Alur Keranjang (Cart Flow)

### Gambaran besar
Keranjang MediKlik bersifat **multi-apotek**: satu user bisa membeli banyak produk dari banyak apotek berbeda dalam satu keranjang. Karena itu data keranjang dikelompokkan **per apotek** (`CartGroup`), bukan daftar item datar.

### Hook-hook utama (Frontend)

#### 1) `useCart()` — [hooks/cart/useCart.ts](mediklik-frontend/src/hooks/cart/useCart.ts)
Ini adalah "otak" keranjang. Menggunakan SWR dengan key `["cart", userId]`.

```ts
const { data: groups = [], error, isLoading, mutate } = useSWR(
  userId ? ["cart", userId] : null,
  () => fetcher<CartResponse>(CART_ENDPOINT).then((res) => res.data),
  { revalidateOnFocus: false, dedupingInterval: 5000 },
);
```

Keputusan desain penting:
- **`userId ? ... : null`** → SWR tidak akan fetch kalau user belum login (conditional fetching). Aman dari request "liar".
- **`dedupingInterval: 5000`** → request `/cart` yang sama dalam 5 detik di-*dedupe* (digabung) supaya tidak spam backend.
- **`revalidateOnFocus: false`** → tidak refetch tiap pindah tab; keranjang dianggap cukup stabil.

Turunan (derived) `grandTotal` & `totalItems` dihitung pakai `useMemo` dari `groups` — bukan disimpan di state, jadi selalu konsisten dengan data:
```ts
const { grandTotal, totalItems } = useMemo(() => {
  const total = groups.reduce((acc, g) => acc + parseFloat(g.summary.subtotal || "0"), 0);
  const items = groups.reduce((acc, g) => acc + (g.summary.total_quantity || 0), 0);
  return { grandTotal: total, totalItems: items };
}, [groups]);
```

**Operasi mutasi keranjang** yang disediakan hook ini (semuanya `useCallback` agar referensinya stabil):
| Fungsi | Endpoint | Keterangan |
|---|---|---|
| `addItem` | `POST /cart` | Tambah produk |
| `updateQty` | `PATCH /cart/items/:id` | Ubah kuantitas (**optimistic update**) |
| `deleteItem` | `DELETE /cart/items/:id` | Hapus 1 item |
| `deleteItems` | `DELETE /cart/cleanitems` | Hapus semua |
| `switchPharmacy` | `PATCH /cart/items/batch` | Pindah/split item ke apotek lain |
| `batchUpdate` | `PATCH /cart/items/batch` | Terapkan rekomendasi apotek sekaligus |

**Optimistic update pada `updateQty`** adalah bagian terpenting untuk dipahami:
```ts
const previousData = groups;                  // simpan snapshot
const optimisticData = groups.map(...)        // hitung tampilan baru di sisi FE
await mutate(optimisticData, { revalidate: false }); // langsung update UI tanpa tunggu server
try {
  await fetcher(CART_ITEM_ENDPOINT(itemId), { method: "PATCH", data: { quantity } });
  await Promise.all([mutate(), mutateCartSummary()]); // sinkron ulang dgn server
} catch (e) {
  await mutate(previousData, { revalidate: false });  // ROLLBACK kalau gagal
}
```
**Kenapa begini?** Supaya tombol +/- terasa instan (UX bagus). Kalau server menolak (misal stok kurang), UI otomatis dikembalikan ke kondisi sebelumnya. Ini pola standar "optimistic UI with rollback".

#### 2) `useCartSummary()` — [hooks/cart/useCartSummary.ts](mediklik-frontend/src/hooks/cart/useCartSummary.ts)
Hanya mengambil **badge angka** di ikon keranjang navbar (`/cart/summary` → `total_item`). Dipisah dari `useCart` agar navbar tidak perlu memuat seluruh isi keranjang yang berat. Setiap operasi di `useCart` memanggil `mutateCartSummary()` agar badge ikut ter-update.

#### 3) `useCartSuggestion()` — [hooks/cart/useCartSuggestion.ts](mediklik-frontend/src/hooks/cart/useCartSuggestion.ts)
Memanggil `POST /cart/suggestions` untuk meminta rekomendasi "apotek termurah-terdekat" untuk seluruh isi keranjang sekaligus, berdasarkan koordinat alamat aktif. Mengirim `{ lat, lng, products:[{product_id,min_stock}], open_only }`.

#### 4) `useAddToCart()` — [hooks/useAddToCart.ts](mediklik-frontend/src/hooks/useAddToCart.ts)
Dipakai dari halaman **Product Detail**. Membungkus validasi sebelum `addItem`:
- Cek user login.
- Cek **batas 100 produk unik** (`CART_ITEM_LIMIT = 100`). Kalau produk sudah ada di keranjang, batas diabaikan (boleh menambah qty).
- Mendukung dua aksi: `handleAddToCart` dan `handleBuyNow`.

### Alur dari Product Detail → Cart
1. User di halaman detail memilih apotek (`pharmacyProductId`) + qty.
2. `useAddToCart.execute()` → validasi → `addItem(pharmacyProductId, quantity)`.
3. BE `POST /cart` → cek alamat aktif & limit → insert/update item → naikkan `carts.product_quantity`.
4. FE me-`mutate()` cart + summary, animasi terbang dijalankan.

### Backend Cart (BE)

**Penambahan item** — [usecase/cart_mutation.go](mediklik-backend/usecase/cart_mutation.go):
```go
func (u *CartUsecase) AddCartItem(ctx, userID, req) error {
  if err := u.guardActiveAddress(ctx, userID); err != nil { return err } // wajib punya alamat aktif
  if err := u.guardCartLimit(ctx, userID, req.PharmacyProductID); err != nil { return err } // batas 100
  return u.txManager.WithTxM(ctx, func(txCtx) error {       // semua dalam 1 transaksi DB
    cartID, _ := u.getOrCreateCart(txCtx, userID)
    u.addOrUpdateCartItem(txCtx, cartID, req.PharmacyProductID, req.Quantity)
    return u.repo.IncrementCartQty(txCtx, cartID, req.Quantity)
  })
}
```
Keputusan desain:
- **Transaksi (`WithTxM`)** memastikan item & counter `product_quantity` selalu sinkron (atomik). Kalau salah satu gagal, semua di-rollback.
- **`addOrUpdateCartItem`** = pola *upsert*: kalau item sudah ada → `UpdateCartItemQty`, kalau belum → `InsertCartItem`. Menggunakan `sql.ErrNoRows` untuk membedakan.

**Update qty** memvalidasi stok di server (FE tidak dipercaya):
```go
if quantity > stock { return utils.ErrUnprocessableEntity("insufficient stock") }
```

**Ganti/split apotek (`BatchUpdateCartItemPharmacy`)**: memisahkan update yang punya `ItemID` (update existing) vs tanpa `ItemID` (insert baru), lalu menjalankan keduanya dalam satu transaksi. Untuk insert, ada cleanup `DeleteCartItemsByPharmacyProductIDs` dulu agar tidak terjadi duplikat.

### Query data keranjang — [repo/postgres/cart.go](mediklik-backend/repo/postgres/cart.go) `GetCartItems`
Query ini menggabungkan `cart_items + pharmacy_products + pharmacies + products + manufacturers + pharmacy_operational_hours`, dan menghitung **status operasional apotek saat ini**:
```sql
CASE
  WHEN poh.id IS NULL THEN 'TUTUP'
  WHEN (NOW() AT TIME ZONE 'Asia/Jakarta')::TIME BETWEEN poh.open_time AND poh.close_time THEN 'BUKA'
  ELSE 'TUTUP'
END AS operational_status
```
Koordinat apotek dikembalikan via `ST_Y/ST_X(ST_Transform(ph.location, 4326))` (PostGIS).

### Ganti Alamat (Change Address) di Cart — [pages/Cart.tsx](mediklik-frontend/src/pages/Cart.tsx)
Logika cukup pintar dengan beberapa `useRef`:
- `activeAddress` di-resolve dari `selectedAddress` (pilihan user di Zustand) → fallback ke alamat yang `is_active`.
- Saat **pertama kali** ada alamat & item, otomatis fetch saran apotek (`hasAutoFetched.current`).
- Saat **alamat berubah** (`prevAddressKey.current !== key`), bukan langsung fetch tetapi memunculkan **banner "Alamat berubah — Update rekomendasi"** agar user yang memutuskan (hemat panggilan algoritma).

### ⚠️ Blind spot Cart Flow
1. **`updateQty` tidak revalidasi limit unik**, hanya `addItem`. Wajar, karena update qty tidak menambah jumlah produk unik.
2. **Bug rename field di `handleConfirmSuggestions`** ([Cart.tsx](mediklik-frontend/src/pages/Cart.tsx)): payload memakai key `item_id`, tapi tipe `BatchUpdateItem` dan backend memakai `cart_item_id`. Perlu dicek apakah backend benar-benar membaca `item_id` atau `cart_item_id` — kalau tidak konsisten, update existing item bisa diperlakukan sebagai insert baru.
3. **`IncrementCartQty` di `UpdateCartItemQuantity`** dipanggil dengan parameter `userID` sebagai argumen pertama (`u.repo.IncrementCartQty(txCtx, userID, delta)`), padahal di `AddCartItem` dipanggil dengan `cartID`. Perlu dipastikan signature-nya memang konsisten (cartID vs userID) — ini rawan bug counter `product_quantity`.
4. **`flattenCartGroups` meng-overwrite `cartItems` Zustand setiap render** (`useEffect` depend ke `cartGroups`), berpotensi memicu fetch saran berulang jika referensi berubah.

---

## A2. Animasi "Fly to Cart" (dibuat AI) — [components/base/CartFlyAnimation.tsx](mediklik-frontend/src/components/base/CartFlyAnimation.tsx)

### Cara kerja
Saat user menambah produk, sebuah ikon keranjang kecil "terbang" dari tombol menuju ikon keranjang di navbar.

1. Komponen di-render via **`createPortal(..., document.body)`** — artinya elemen animasi dipasang langsung ke `<body>`, bukan di dalam komponen tombol. **Kenapa?** Agar tidak terpotong `overflow:hidden` parent dan posisinya bebas di atas semua elemen (`z-9999`).
2. Saat prop `trigger` berubah, `useEffect` menghitung:
   - **Titik awal** = tengah elemen tombol (`originRef.getBoundingClientRect()`).
   - **Titik akhir** = tengah ikon `#navbar-cart-icon` (`getTargetPosition()`), dengan fallback ke pojok kanan atas bila ikon tidak ditemukan.
   - **Titik tengah melengkung** (`midX`, `midY = startY - 22`) → membuat lintasan **melengkung ke atas**, bukan garis lurus (lebih natural).
3. Animasi pakai **Web Animations API** (`element.animate([...keyframes], options)`) dengan 3 keyframe (mulai → puncak lengkungan opacity 1 → tujuan mengecil & memudar), `duration: 1350ms`, easing `cubic-bezier(0.19, 1, 0.22, 1)` (efek "ease-out" cepat di awal).
4. Ikon navbar diberi animasi **"denyut" (scale 1→1.06→1)** dengan `delay: 980ms` agar bereaksi tepat saat paket "tiba".
5. `animation.onfinish` menyembunyikan elemen & memanggil `onComplete()`. Cleanup function membatalkan animasi bila komponen unmount (`animation.cancel()`).

### Keputusan desain
- Memakai **Web Animations API**, bukan animasi CSS/React state per-frame → animasi berjalan di compositor browser (lebih halus, tidak membebani React re-render).
- `transform: translate3d(...)` + `willChange: "transform, opacity"` → memaksa GPU acceleration.

### ⚠️ Blind spot Animasi
1. **Ketergantungan pada `getElementById("navbar-cart-icon")`** — kalau navbar belum ter-mount (mis. di halaman tanpa navbar atau saat lazy-load), animasi terbang ke pojok kanan atas (fallback), bukan ke ikon.
2. **Tidak ada penanganan `prefers-reduced-motion`** — user yang sensitif terhadap animasi tetap melihat animasi penuh (isu aksesibilitas).
3. **Tipe `originRef: React.RefObject<HTMLElement>`** mengharuskan `.current` non-null; bila tombol di-render kondisional, ref bisa null dan animasi dilewati diam-diam.

---

## A3. Kondisi-kondisi Keranjang (Cart Conditions)

Status setiap item dirender oleh [features/cart/CartItem.tsx](mediklik-frontend/src/features/cart/CartItem.tsx). Ada 4 kondisi visual utama:

| Kondisi | Pemicu | Tampilan / Aksi |
|---|---|---|
| **Produk dihapus apotek** | `item.is_deleted === true` | Banner merah "Produk tidak lagi tersedia", item **harus dihapus** sebelum checkout |
| **Apotek tutup** | `isClosed` (jam operasional) | Banner merah "Apotek sedang tutup", tombol **"Cari Alternatif"** |
| **Stok mentok/kurang** | `item.quantity >= item.stock` | Banner amber. Bila `quantity > stock` → "melebihi sisa stok"; bila `==` → "Butuh lebih banyak? Cari apotek lain" |
| **Normal** | — | Tampilan biasa |

**Input qty diproteksi berlapis:**
- `handleQtyChange` membuang non-digit, membersihkan leading-zero, dan **meng-clamp ke `item.stock`** (`if (numVal > item.stock) setLocalQty(item.stock)`).
- Perubahan qty di-**debounce 500ms** (`useDebounce`) sebelum benar-benar memanggil server — agar user yang menekan +/- berkali-kali tidak menembak banyak request.

**Kapan user TIDAK boleh menambah ke keranjang (server-side, otoritatif):**
1. **Tidak punya alamat aktif** → `guardActiveAddress` melempar `422 active shipping address is required`. (Tombol checkout & beberapa aksi juga disabled di FE via `useActiveAddress`.)
2. **Sudah 100 produk unik** → `guardCartLimit` melempar `422 cart limit reached`.
3. **Qty > stok** → `UpdateCartItemQuantity` melempar `422 insufficient stock`.

**Deteksi apotek tutup** ada dua sumber:
- **Backend** (`operational_status` di query, pakai zona waktu `Asia/Jakarta`).
- **Frontend** [utils/cart.ts](mediklik-frontend/src/utils/cart.ts) `isPharmacyClosed()` — bahkan menangani kasus **apotek buka melewati tengah malam** (`closeTotal < openTotal`).

### ⚠️ Blind spot Cart Conditions
1. **Dua sumber kebenaran "tutup"** (FE `isPharmacyClosed` vs BE `operational_status`). Bila jam server beda dengan jam device user, status bisa berbeda. Sebaiknya satu sumber (backend) jadi acuan final.
2. **Checkout menyaring item bermasalah secara diam-diam** (`buildCheckoutItems` membuang item `stock<=0 || qty<=0 || is_deleted`). Item yang melebihi stok (`qty > stock`) **tetap ikut dengan qty asli** → backend yang harus menolak/menyesuaikan, kalau tidak bisa over-sell.
3. **Apotek tutup tetap bisa di-checkout** (hanya `is_deleted`/stok yang difilter). Desainnya memang "dikirim saat apotek buka", tapi ini perlu ditegaskan saat presentasi agar tidak terlihat seperti bug.

---

## A4. Daftar Produk (Product List) & Logika Cheapest-Nearest

### Bagaimana sistem memilih produk yang ditampilkan?

Inti idenya: untuk setiap **produk**, sistem hanya menampilkan **SATU penawaran terbaik** (dari apotek termurah yang stoknya cukup & sedang buka di sekitar user). Jadi daftar produk = daftar "produk unik dengan harga terbaiknya", bukan daftar semua stok di semua apotek.

#### Hook FE — [hooks/product/useProducts.ts](mediklik-frontend/src/hooks/product/useProducts.ts)
- Mengambil filter dari `useProductFilterStore` (search, klasifikasi, bentuk sediaan, kategori, sort) dan lokasi dari `useLocationStore`.
- **`buildFilterKey`** membuat "sidik jari" filter (tanpa cursor). Bila berubah → reset akumulasi & cursor (search baru). Bila hanya cursor berubah → "load more" (infinite scroll).
- Search di-debounce 500ms.
- Akumulasi hasil di-dedup berdasarkan `id` agar tidak ada produk dobel saat load-more.
- Mengirim `lat/lng` **hanya bila status lokasi `success`**.

### Skenario: punya alamat vs tidak punya alamat (BE) — [usecase/product.go](mediklik-backend/usecase/product.go) `GetProducts`

```go
if q.HasLocation {
  nearby, _ := repo.CheckAnyPharmacyNearby(lat, lng)         // ada apotek dalam radius?
  if !nearby { return nil, "", true, nil }                   // → no_nearby = true
  anyOpen, _ := repo.CheckAnyPharmacyOpenNearby(lat, lng)    // ada yang buka?
  if !anyOpen { return ..., utils.ErrNoPharmacyOpen }        // → flag no_pharmacy_open
  q.RadiusMeter = 25000
  return repo.GetBestProducts(ctx, q)                        // cheapest-nearest dlm radius 25km
}
// tanpa lokasi → katalog global (tanpa filter jarak/buka)
return repo.GetBestProducts(ctx, q)
```

Tiga kemungkinan keluaran yang dibedakan FE:
1. **Punya lokasi + ada apotek buka** → daftar produk termurah-terdekat (radius 25 km).
2. **Punya lokasi tapi tidak ada apotek dalam radius** → `noNearbyPharmacies` (UI: "tidak ada apotek di sekitarmu").
3. **Punya lokasi, ada apotek tapi semuanya tutup** → `noPharmacyOpen` (UI: "semua apotek tutup").
4. **Tanpa lokasi** → tetap menampilkan **katalog global** (browsing tetap jalan walau belum set alamat).

### Logika "Cheapest-Nearest" di SQL — [repo/postgres/product_location_query.go](mediklik-backend/repo/postgres/product_location_query.go)
Polanya:
1. **CTE `nearby_open_pharmacies`**: filter apotek `is_active` + hari & jam buka sekarang (`Asia/Jakarta`) + `ST_DWithin(...radius)`. Sekaligus menghitung `distance_m`.
2. **`CROSS JOIN LATERAL`** per produk: ambil **1 penawaran termurah** (`ORDER BY pp.price ASC LIMIT 1`) yang stoknya ≥ 1, aktif, tidak terhapus.
3. Hasilnya bisa di-sort (`price` / `stock` / `nearest` / `name`) dan dipaginasi pakai **keyset/cursor pagination** `(sort_value, p.id) > (cursor)` — lebih scalable dari OFFSET.

> **Catatan teknis penting:** proyeksi titik memakai SRID **23868** (`ST_Transform(..., 23868)`) untuk perhitungan jarak metrik di wilayah Indonesia, sementara koordinat lat/lng ditampilkan dengan transform balik ke **4326** (WGS84).

### Kapan algoritma Python dipakai vs SQL Go?

Ini sering ditanya — penting dijelaskan jelas:

| Use case | Mesin | Alasan |
|---|---|---|
| **Daftar produk / produk populer** | **SQL (Go)** — `cheapest per product` | Hanya butuh 1 apotek termurah per produk → cukup query SQL, cepat, sederhana |
| **Saran keranjang (banyak produk, butuh kombinasi optimal & boleh split antar-apotek)** | **Python MILP** (`/nearest-cheapest`) | Ini masalah optimasi: minimalkan **total (harga × qty) + ongkir per apotek**, sambil memenuhi target qty & batas stok. SQL tidak bisa menyelesaikan optimasi gabungan seperti ini |

**Fallback:** bila service Python error/timeout, `GetCartSuggestedProducts` jatuh ke `fallbackSuggestions` yang memakai SQL `GetBestProductsByIDs` (cheapest-nearest per produk, tanpa split). Sistem tetap jalan walau algo mati — keputusan desain yang bagus untuk ketahanan.

### Algoritma Python (MILP) — [services/nearest_cheapest/](mediklik-algorithm/services/nearest_cheapest/)

**Alur:** `router.py` (POST `/nearest-cheapest`) → `processor.py` → `prelude.py` (ambil data DB) → `optimizer.py` (selesaikan MILP) → kembalikan split per produk.

`prelude.find_pharmacies` mengambil semua kandidat (produk × apotek) dalam radius 25 km, beserta `price`, `stock`, dan `distance` (km, dibulatkan ke atas). Data "diratakan" jadi vektor matriks (produk × apotek).

`optimizer.optimizer` menyusun **Mixed-Integer Linear Program** dengan `scipy.optimize.milp`:
- **Variabel keputusan:** jumlah unit tiap (produk, apotek) + variabel biner "apotek dipakai/tidak".
- **Fungsi tujuan (diminimalkan):** `Σ(harga × qty) + Σ(ongkir × apotek_dipakai)`.
- **Constraint target:** total qty tiap produk = target yang diminta (`constraint_limits`).
- **Constraint artifisial:** `Σ qty di apotek i − M·(biner_i) ≤ 0` → memastikan ongkir suatu apotek hanya dihitung **sekali** jika apotek itu dipakai (M = "angka besar").
- **Bounds:** qty dibatasi stok (`limits`), biner ∈ [0,1].

**Contoh kasus nyata:**
> User butuh **3 Paracetamol** dan **2 Vitamin C**.
> - Apotek A (1 km): Paracetamol Rp5.000 (stok 2), Vitamin C Rp10.000 (stok 5).
> - Apotek B (3 km): Paracetamol Rp4.500 (stok 10), Vitamin C Rp12.000 (stok 0).
>
> Optimizer akan mempertimbangkan: ambil semua dari B termurah, tapi kena ongkir B + Vitamin C tidak ada di B. Bisa jadi solusinya **split**: Paracetamol dari B (lebih murah, stok cukup), Vitamin C dari A — atau menumpuk di A saja agar **hemat ongkir** walau harga satuan sedikit lebih mahal. Inilah yang dihitung MILP dan **tidak bisa** ditebak dengan aturan sederhana "ambil termurah".

Hasil split inilah yang ditampilkan di dialog **"Rekomendasi Apotek"** ([features/cart/PharmacySuggestion.tsx](mediklik-frontend/src/features/cart/PharmacySuggestion.tsx)) — lengkap dengan badge "Dibagi ke N apotek", harga, jarak, dan jam buka. Mode **"Buka Sekarang"** (`open_only=true`) hanya mempertimbangkan apotek yang sedang buka (untuk kebutuhan mendesak).

### ⚠️ Blind spot Product List & Algoritma
1. **`objective_function.py` adalah dead code yang bahkan punya bug slicing.** `f` tidak pernah dipanggil (optimizer langsung `milp(np.hstack((prices, shipping_costs)), ...)`), dan slice `x[product_count*pharmacy_count : product_count+pharmacy_count]` salah indeks (seharusnya `... : product_count*pharmacy_count + pharmacy_count`). Tidak berdampak karena tidak terpakai, **tapi menyesatkan** dan sebaiknya dihapus.
2. **Konstanta `SHIPPING_COST = 2500` di `prelude.py` tidak dipakai.** Yang dipakai sebagai biaya per-apotek justru **jarak (km)**, bukan rupiah ongkir. Jadi "ongkir" dalam optimizer = penalti jarak, bukan biaya nyata. Perlu diluruskan agar tidak salah klaim saat presentasi.
3. **`print(query)` & `print(params)` di [cart_product.go](mediklik-backend/repo/postgres/cart_product.go) `GetBestProductByID`** masih ada — debug log yang membocorkan query ke stdout, sebaiknya dibuang di production.
4. **Radius 25 km hard-coded** di banyak tempat. Bila ingin kota lain dengan kepadatan apotek berbeda, perlu konfigurasi.
5. **`milp` bisa lambat / tidak feasible** untuk keranjang besar (banyak produk × banyak apotek). Tidak ada batas waktu eksplisit di optimizer; mengandalkan fallback hanya bila HTTP error, bukan bila optimizer lambat.

---

## A5. Homepage — "Paling Banyak Dibeli Hari Ini"

### Cara kerja
Section ini menampilkan produk terlaris. Sumber datanya tabel **`product_popularities (product_id, date, count)`**.

**BE — [repo/postgres/product_popular.go](mediklik-backend/repo/postgres/product_popular.go) `GetPopularProductIDs`:**
```sql
-- 1) cek apakah ada penjualan HARI INI
SELECT COALESCE(SUM(count),0) FROM product_popularities WHERE date::date = CURRENT_DATE;
-- 2) kalau ada → filter "hari ini saja"; kalau belum ada → pakai all-time (fallback)
SELECT product_id FROM product_popularities pop
JOIN pharmacy_products ... WHERE is_active AND NOT is_deleted [AND date = CURRENT_DATE]
GROUP BY product_id ORDER BY SUM(count) DESC LIMIT $1;
```
**Keputusan desain pintar:** kalau hari masih pagi dan belum ada transaksi, section tidak kosong — otomatis menampilkan terlaris sepanjang masa (fallback), lalu beralih ke "hari ini" begitu transaksi pertama masuk.

### Skenario punya alamat vs tidak — [usecase/product.go](mediklik-backend/usecase/product.go) `GetPopularProducts`
- **Punya lokasi:** ambil ID produk populer → resolve ke penawaran **termurah-terdekat dalam 25 km** (`GetBestProductsByIDs`). Bila tidak ada apotek dekat/buka → kembalikan `no_nearby = true`.
- **Tanpa lokasi:** resolve harga termurah secara global (tanpa jarak).

**FE — [hooks/product/usePopularProducts.ts](mediklik-frontend/src/hooks/product/usePopularProducts.ts):**
```ts
// minta versi berbasis lokasi dulu
const { data } = useSWR(`/products/popular${query}`, fetcher);
// kalau backend bilang no_nearby → minta versi GLOBAL sebagai cadangan
const { data: globalData } = useSWR(data?.no_nearby ? `/products/popular?per_page=${perPage}` : null, fetcher);
const products = (data?.no_nearby ? globalData?.data : data?.data) ?? [];
```
Jadi user yang berada di luar jangkauan apotek **tetap** melihat produk populer (versi global), bukan section kosong.

### ⚠️ Blind spot Popular Section
1. **Inkonsistensi skema yang serius.** `init.sql` mendefinisikan `product_popularities.product_id` sebagai **FK ke `products(id)`**, tapi query `GetPopularProductIDs` melakukan `JOIN pharmacy_products ON pharmacy_products.id = pop.product_id` lalu `GROUP BY pharmacy_products.product_id`. Artinya kode memperlakukan kolom itu sebagai **`pharmacy_products.id`**, bertentangan dengan definisi FK. Ini wajib diklarifikasi — salah satunya pasti keliru, dan bisa menghasilkan data populer yang salah/missing.
2. **Tidak ada penambahan popularitas real-time saat checkout.** Pencarian di seluruh kode (`.go`/`.sql`) menunjukkan `product_popularities` hanya **di-seed** (cmd/seeder) dan **dibaca**, tidak pernah di-`INSERT/UPDATE` saat ada pembelian. Jadi "Paling Banyak Dibeli Hari Ini" sebenarnya **mencerminkan data seed**, bukan transaksi nyata berjalan. Ini poin paling rawan ditanya dosen/penguji — siapkan jawaban (mis. "untuk demo pakai seed; mekanisme increment by trigger/cron adalah pekerjaan lanjutan").

---

# BAGIAN B — ROLE PHARMACIST

## B1. Inventori Apotek (Pharmacy Inventory)

### Hook FE — [hooks/usePharmacyInventory.ts](mediklik-frontend/src/hooks/usePharmacyInventory.ts)
- Satu hook mengelola **list + filter + pagination + CRUD**.
- **List:** SWR key = `/pharmacist/pharmacy_inventory` + querystring (search, classification, product_form, manufacturer_id, is_active, sort). `keepPreviousData: true` agar tabel tidak "berkedip" saat ganti halaman.
- **Pagination berbasis cursor dua arah** dengan `cursorStack` (tumpukan cursor) sehingga tombol **Prev** bisa kembali ke halaman sebelumnya — sesuatu yang sulit dilakukan cursor pagination biasa.
- **CRUD** pakai `useSWRMutation` (`triggerCreate/Update/Delete`) ke `POST/PATCH/DELETE /pharmacist/pharmacy_inventory[/:id]`.
- Setelah mutasi sukses: `mutate()` (refresh list) **+ `revalidateRelated()`** yang mem-`globalMutate` semua key yang mengandung `/stats` dan `/product_catalog` — supaya kartu statistik & katalog ikut akurat.
- Search di-debounce 400ms.

`useInventoryStats()` → `/pharmacist/pharmacy_inventory/stats` (kartu ringkasan: total produk, aktif, stok rendah, dll).

### Backend — [usecase/pharmacy_inventory_mutation.go](mediklik-backend/usecase/pharmacy_inventory_mutation.go)

**Tambah produk (`CreateProduct`):**
1. Resolve apotek dari user (`GetPharmacyIDByUserID`).
2. Cek produk belum ada di apotek (`ProductExistsInPharmacy`) → kalau ada, `422 produk telah ada`.
3. Dalam **transaksi**: `CreateOrUpdatePharmacyProduct` (upsert) → **tulis stock mutation log** (reason `"tambah stok"`) → `IncrementProductPharmacyCount` (menaikkan penghitung berapa apotek menjual produk ini, dipakai katalog).

**Update produk (`UpdateProduct`):** menerapkan perubahan harga/aktif, dan **bila qty diubah**:
```go
lastMutatedAt, _ := repo.GetLastStockMutationAt(ctx, id)
if err := assertStockUpdateAllowed(lastMutatedAt); err != nil { return err } // ⛔ aturan 1x/hari
newStock, _ := resolveStock(existing.Stock, *req.Qty, *req.Operasi)          // ganti/tambah/kurang
mutationLog = buildStockMutationLog(...)
```
**Kondisi yang dicek:**
- **`assertStockUpdateAllowed`**: stok hanya boleh diubah **manual 1× per hari** (`isSameDay`). Ini mencegah apoteker mengubah-ubah stok terlalu sering (dan menjaga audit trail rapi). Hanya berlaku untuk reason `tambah stok`/`kurang stok` — mutasi akibat penjualan tidak menghalangi.
- **`resolveStock`** mendukung 3 operasi: `ganti` (set absolut), `tambah` (+qty), `kurang` (−qty, ditolak bila jadi negatif → `422 insufficient stock`).

**Hapus produk (`DeleteProduct`):**
- **Tidak bisa dihapus bila pernah dibeli** (`ProductHasBeenPurchased` mengecek `order_items` dengan order non-`cancelled`) → `422`. Menjaga integritas riwayat order.
- Penghapusan adalah **soft delete**: `is_deleted=TRUE, price=0, stock=0, is_active=FALSE` ([pharmacy_inventory_mutation.go](mediklik-backend/repo/postgres/pharmacy_inventory_mutation.go)). Lalu `DecrementProductPharmacyCount`.

### ⚠️ Blind spot Inventory
1. **`CreateOrUpdatePharmacyProduct` melakukan `ON CONFLICT DO UPDATE` dan me-reset `created_at = NOW()` serta `is_deleted=FALSE`.** Artinya menambahkan kembali produk yang dulu di-soft-delete akan "menghidupkan" baris lama. Ini bisa diinginkan, **tetapi** `created_at` ikut ter-reset — riwayat kapan produk pertama dibuat jadi hilang.
2. **Aturan "1× per hari" mudah dilewati lewat `Create`** (`CreateProduct` selalu menulis log tanpa cek `assertStockUpdateAllowed`). Jadi pembatasan hanya pada `Update`. Perlu ditegaskan apakah ini memang desain yang diinginkan.
3. **`GetPharmacyIDByUserID` error langsung dipetakan `500`** padahal bisa jadi user bukan apoteker (seharusnya `403/404`). Pesan error generik menyembunyikan penyebab.

---

## B2. Log Mutasi Stok (Stock Mutation Logs)

### Apa itu
Setiap perubahan stok dicatat ke tabel **`stock_mutation_logs (pharmacy_product_id, quantity_change, reason, created_by, created_at)`**. Ini adalah **jejak audit** lengkap: siapa, kapan, berapa, kenapa.

**Sumber penulisan log (ada 2 jalur):**
1. **Aksi apoteker** (tambah/kurang/hapus produk) — reason `"tambah stok"` / `"kurang stok"`.
2. **Transaksi order** ([repo/postgres/order.go](mediklik-backend/repo/postgres/order.go)) — `decreasePharmacyProductStock` saat penjualan & `IncrementStock` saat pembatalan, masing-masing menulis `CreateStockMutationLog`. `created_by` bisa `NULL` (sistem) → ditampilkan sebagai **"System"**.

### Hook FE — [hooks/useStockLog.ts](mediklik-frontend/src/hooks/useStockLog.ts)
- List log via `/pharmacist/pharmacy_inventory/stockmutationlog[/:pharmacyProductId]` + filter `reason`, `search` (minimal **3 karakter** baru dikirim), cursor.
- Daftar **reason unik** untuk dropdown filter dari `/stockmutationlog/reasons`.
- Pagination dua arah (`cursorStack`) sama seperti inventory.
- **Export CSV** (`exportCSV`): fetch manual dengan header `Authorization: Bearer <token>` ke endpoint `....csv`, lalu unduh sebagai Blob (`createObjectURL` → klik `<a download>` → revoke). Mendukung filter aktif (product_id, search, reason).

### Backend — [repo/postgres/pharmacy_inventory_mutation.go](mediklik-backend/repo/postgres/pharmacy_inventory_mutation.go)
`GetAllStockMutationLogs` membangun query dinamis (filter optional digabung), dengan **keyset pagination**:
```sql
(sml.created_at, sml.id) < ($cursor_time::timestamptz, $cursor_id::bigint)
ORDER BY sml.created_at DESC
```
Ambil `perPage + 1` baris untuk menentukan apakah ada `next_cursor` ([usecase/pharmacy_inventory_mutation.go](mediklik-backend/usecase/pharmacy_inventory_mutation.go) `listStockMutationLogs`):
```go
req.PerPage = perPage + 1
if len(logs) > perPage {
  last := logs[perPage-1]
  c := last.CreatedAt.UTC().Format(time.RFC3339Nano) + "," + strconv.FormatInt(last.ID, 10)
  nextCursor = &c
  logs = logs[:perPage]
}
```
Setiap baris diperkaya dengan nama/foto produk, `created_by_name`/`email` (LEFT JOIN users → "System" bila null), `selling_unit`, dan `current_stock`. `ExportStockMutationLogs` sama tapi tanpa limit (ambil semua untuk CSV).

### ⚠️ Blind spot Stock Mutation Logs
1. **Export CSV tanpa LIMIT** → bila apotek punya jutaan baris log, satu request bisa membebani DB & memori (tidak ada streaming/chunking).
2. **`current_stock` pada baris log = stok SAAT INI**, bukan stok historis setelah mutasi itu terjadi. Jadi kolom ini sama untuk semua baris produk yang sama → bisa membingungkan pembaca yang mengira itu "stok setelah transaksi".
3. **Filter `search` butuh ≥3 karakter di FE**, tapi BE tidak memaksakan aturan ini → bila ada pemanggil lain (mis. export), pencarian pendek tetap diproses.

---

# BAGIAN C — DOCKER & OBSERVABILITY STACK

## C1. Semua File yml/config/nginx Terkait Stack Observability

Stack observability MediKlik mengikuti pola standar: **aplikasi mengirim telemetry → OTel Collector → backend penyimpanan (Prometheus/Loki/Jaeger) → divisualisasikan Grafana.**

| File | Fungsi | Kenapa diperlukan & letaknya |
|---|---|---|
| **otel-collector-config.yaml** | Konfigurasi **OpenTelemetry Collector** | Titik pusat pengumpulan. Menerima OTLP (gRPC `4317`, HTTP `4318`), juga membaca **file log container Docker** (`filelog/docker`). Diletakkan di root agar di-mount read-only ke container otel |
| **prometheus.yml** / **prometheus.local.yml** | Target scraping Prometheus | Memberi tahu Prometheus dari mana mengambil metrik: `otel-collector:8889`, `prometheus` sendiri, dan `node_exporter`. Dua versi untuk host-network (VM) vs bridge-network (local) |
| **loki/config.yml** | Konfigurasi **Loki** (penyimpanan log) | Menyimpan log dalam format TSDB di filesystem (`/loki/chunks`), `auth_enabled:false`, port `3100`, `allow_structured_metadata` |
| **grafana/provisioning/datasources/prometheus.yaml** | Auto-provision datasource Prometheus | Agar Grafana langsung tersambung ke Prometheus tanpa setup manual. URL `http://prometheus:9090/vm1/prometheus`, jadi datasource default |
| **grafana/provisioning/datasources/loki.yaml** | Auto-provision datasource Loki | Menyambungkan Grafana ke Loki (`http://loki:3100`) untuk panel log |
| **grafana/provisioning/dashboards/dashboard.yaml** | Provider dashboard | Memuat semua file dashboard JSON dari folder otomatis (`updateIntervalSeconds:10`, boleh di-edit di UI) |
| **grafana/provisioning/dashboards/*.json** | Definisi dashboard | 5 dashboard: api-requests, latency-metrics, rate-metrics, traffic-level, node-exporter |
| **nginx.conf** / **nginx.local.conf** | Reverse proxy | Menyatukan semua layanan di balik satu domain + sub-path `/vm1/...` (grafana, prometheus, pgadmin, api, algo, frontend) |
| **proxy.Dockerfile** / **proxy.local.Dockerfile** | Build image nginx | Menyalin nginx.conf yang sesuai (VM vs local) ke image nginx |

### Pipeline OTel Collector ([otel-collector-config.yaml](otel-collector-config.yaml))
- **Receivers:** `otlp` (metrik/log/trace dari backend Go) + `filelog/docker` (membaca `/var/lib/docker/containers/*/*-json.log`, mem-parse JSON, memetakan label `service_name` → `resource["service.name"]`).
- **Exporters:** `prometheus` (expose di `:8889` untuk di-scrape), `otlphttp/loki` (kirim log ke Loki), `otlp/jaeger` (kirim trace ke Jaeger).
- **Pipelines:** `metrics` (otlp→prometheus), `logs` (otlp+filelog→loki), `traces` (otlp→jaeger).

### Kenapa nginx sub-path `/vm1/`?
Semua tool diakses lewat satu host: `/vm1/grafana`, `/vm1/prometheus`, `/vm1/pgadmin`, `/vm1/api`, `/vm1/api/v1/algo`. Aset statis (`.css/.js/...`) di-cache (`proxy_cache ncache`, valid 1 hari), sedangkan `/vm1/api/` di-set `proxy_no_cache` (data dinamis tidak boleh di-cache). Grafana diberi `GF_SERVER_SERVE_FROM_SUB_PATH=true` agar berfungsi di bawah sub-path.

## C2. Perbedaan Deployment Local vs VM

Sumber: `diff docker-compose.yaml docker-compose.local.yaml` dan `diff nginx.conf nginx.local.conf`.

| Aspek | **VM** (`docker-compose.yaml`, `nginx.conf`) | **Local** (`docker-compose.local.yaml`, `nginx.local.conf`) |
|---|---|---|
| **Jaringan** | `network_mode: "host"` — semua container memakai network host, saling sapa via `localhost` | Bridge network `mediklik_network` — container saling sapa via **nama service** (DNS internal Docker) |
| **DB host (backend/algo)** | `DATABASE_HOST=localhost` | `DATABASE_HOST=database` (nama service) |
| **Target nginx** | `proxy_pass http://localhost:8080/...` | `proxy_pass http://backend:8080`, `http://frontend:8088`, `http://grafana:3000`, dst (nama service) |
| **Target Prometheus** | `localhost:8889/9090/9909` | `otel-collector:8889`, `prometheus:9090`, `node_exporter:9909` |
| **Port mapping proxy** | Tidak perlu (`host` langsung pakai port 80) | Eksplisit `ports: ["80:80"]` |
| **healthcheck DB** | `pg_isready -U mediklik` (hard-coded) | `pg_isready -U ${DOCKER_DATABASE_NAME}` (dari env) |
| **shm_size DB** | `1g` (untuk beban produksi/seed besar) | tidak diset |
| **Dependensi backend** | hanya `database` | juga menunggu `prometheus`, `loki`, `otel-collector` siap |

**Inti perbedaan:** VM memakai **host networking** (lebih sederhana, performa native, cocok 1 VM) sementara Local memakai **bridge network dengan service discovery** (lebih portable & terisolasi, standar Docker Compose di mesin developer). Karena `host` mode tidak mendukung DNS nama-service, VM harus pakai `localhost` + port unik.

## C3. Formula Dashboard & Alur Observability (dari DB hingga ke Dashboard)

### Alur lengkap "dari nol sampai tampil di Grafana"
1. **Request masuk** ke backend Go. Middleware [middleware/metrics.go](mediklik-backend/middleware/metrics.go) memanggil `RequestCount.Add(1, {method, route, status_code})` → menambah **counter OTel** `http.server.requests`.
2. **Log** ditulis [middleware/logger.go](mediklik-backend/middleware/logger.go) (Zap) berisi method, uri, code, latency, IP → keluar sebagai JSON ke stdout container.
3. **OTel SDK** ([utils/telemetry/metrics.go](mediklik-backend/utils/telemetry/metrics.go)) mengekspor metrik via OTLP/HTTP setiap **15 detik** ke OTel Collector (`APP_OTLPENDPOINT`).
4. **OTel Collector** mengubah metrik menjadi format Prometheus (namespace `otel` → metric jadi `otel_http_server_requests_total`) dan mengekspos di `:8889`. Log container dibaca lewat `filelog/docker` dan diteruskan ke **Loki**.
5. **Prometheus** scrape `:8889` tiap 1 menit dan menyimpan time-series.
6. **Grafana** meng-query Prometheus (metrik) & Loki (log) memakai datasource yang sudah di-provision, lalu merender panel.

### Formula PromQL per dashboard (dan alasannya)

**rate-metrics.json (Rate Mediklik):**
| Panel | Formula | Arti |
|---|---|---|
| Total Requests (5m) | `sum(increase(otel_http_server_requests_total[5m]))` | Total request dalam 5 menit. `increase()` = pertambahan counter pada rentang waktu |
| Success Rate | `sum(increase(otel_http_server_requests_total{status_code=~"2.."[5m]))` | Jumlah respons sukses (2xx) |
| Error Rate | `sum(increase(...{status_code=~"5.."}[5m]))` | Jumlah error server (5xx) |

**traffic-level.json (Traffic Level) — pakai `rate()` (per-detik):**
| Panel | Formula |
|---|---|
| General Traffic | `sum(rate(otel_http_server_requests_total[5m]))` |
| Per Endpoint | `sum(rate(...[5m])) by (route, method)` |
| Per Status Code | `sum(rate(...[5m])) by (status_code)` |

> **Kenapa `rate` vs `increase`?** `increase()` memberi **total tambahan** pada jendela (cocok untuk "berapa banyak"); `rate()` memberi **laju per detik** (cocok untuk "seberapa sibuk/throughput"). Keduanya dihitung dari counter yang sama.

**api-requests.json (API Requests):** memakai `sum(increase(otel_http_server_requests_total{route=~"..."}[...]))` untuk Total/Success/Warning(4xx)/Error(5xx), `topk(10, sum by (route) (...))` untuk **Top Endpoints**, dan panel **log** dari Loki (`{service_name="mediklik-backend"}` difilter Error/Cron/Success).

**latency-metrics.json (Latency Mediklik):**
| Panel | Formula | Arti |
|---|---|---|
| Latency P99 | `histogram_quantile(0.99, sum(rate(otel_http_server_request_duration_seconds_bucket[5m])) by (le))` | Persentil ke-99 (1% request paling lambat) |
| Latency P95 | `histogram_quantile(0.95, ...)` | Persentil ke-95 |
| Average Latency | `sum(rate(..._sum[5m])) / sum(rate(..._count[5m]))` | Rata-rata = total durasi ÷ jumlah request |
| Per Endpoint | versi di atas `by (http_route, http_method)` | Breakdown per endpoint |

> `histogram_quantile` bekerja pada metrik **histogram** yang punya bucket `_bucket{le=...}`, plus `_sum` dan `_count`. `le` = "less-or-equal", batas atas bucket.

**node-exporter.json:** metrik server (CPU, RAM, disk, network) dari Node Exporter — kesehatan mesin host.

### ⚠️ Blind spot Observability (PALING PENTING untuk diketahui sebelum presentasi)
1. **Dashboard Latency kemungkinan KOSONG / no-data.** Histogram durasi **dimatikan** di kode: pada [middleware/metrics.go](mediklik-backend/middleware/metrics.go), seluruh bagian `Duration` (Float64Histogram) dan `m.Duration.Record(...)` **di-comment**. Akibatnya metric `otel_http_server_request_duration_seconds_*` tidak pernah dihasilkan, sehingga panel P95/P99/Average Latency tidak punya data. **Siapkan jawaban** kalau panel itu kosong saat demo.
2. **Inkonsistensi label.** Counter memakai label `route`, `method`, `status_code`, sedangkan beberapa formula latency memakai `http_route`, `http_method`. Ini memperkuat dugaan bahwa metrik latency belum pernah benar-benar mengalir.
3. **Trace ke Jaeger dikonfigurasi di OTel (`otlp/jaeger`) tetapi service Jaeger tidak ada di docker-compose** — exporter trace akan gagal connect. Pipeline trace praktis tidak berfungsi.
4. **`scrape_interval: 1m`** cukup kasar; lonjakan singkat bisa terlewat. Wajar untuk demo, tapi perlu disebut bila ditanya soal resolusi.
5. **Loki memakai filesystem storage + inmemory ring** (single-node). Tidak untuk skala besar/HA — cukup untuk demo.

---

# BAGIAN D — RINGKASAN TINGKAT TINGGI (High-Level Summary)

## D1. Fitur yang Sudah Berjalan
- **Multi-peran (RBAC):** User, Apoteker (Pharmacist), Admin, Kurir (Courier) — middleware auth + JWT, hook `useAuthZ`/`useAuthN`. Onboarding apoteker/kurir lewat email berisi password.
- **Katalog & Kategori:** pencarian + filter (klasifikasi obat, bentuk sediaan, kategori), sort (harga/nama), infinite scroll cursor-based, klasifikasi obat (bebas/bebas terbatas/keras/non-obat).
- **Keranjang multi-apotek:** pengelompokan per apotek, optimistic update, batas 100 produk unik, deteksi tutup/stok/dihapus, rekomendasi & split apotek.
- **Checkout & Pembayaran:** membangun item per apotek, integrasi payment gateway (`PAY_URL` SLP / AirSIPP), ongkir via **RajaOngkir/Komerce**.
- **Pelacakan Pesanan:** status order (pending → paid → processed → shipped → delivered/cancelled), riwayat & detail.
- **Inventori Apoteker:** CRUD produk apotek, statistik, batas update stok 1×/hari, soft-delete dengan proteksi "pernah dibeli", log mutasi stok + export CSV.

## D2. Fitur Unik (Pembeda)
- **Cheapest-Nearest (MILP):** optimasi gabungan harga + jarak/ongkir dengan kemampuan **split antar-apotek**, plus fallback SQL bila service algoritma mati. (Lihat A4.)
- **AI Chatbot "Doto"** ([chatbot.py](mediklik-algorithm/services/recommendation/chatbot.py)): pakai **Google Gemini** (`gemini-3.1-flash-lite`). Menjawab keluhan user dalam Bahasa Indonesia singkat **dan** mengembalikan **filter terstruktur** (`category` + `classification`) dalam JSON, sehingga jawaban chatbot bisa langsung dipakai memfilter katalog. FE: [hooks/useChat.ts](mediklik-frontend/src/hooks/useChat.ts) + `ChatbotWidget`.
- **Rekomendasi ML (Produk Serupa)** ([recommender.py](mediklik-algorithm/services/recommendation/recommender.py)): **TF-IDF** (n-gram 1–2 atas nama+generic+kategori) + **KNN cosine** untuk mencari produk mirip dari sebuah query (`/similar`). Model di-fit saat startup.
- **Produk Komplementer (Cross-sell)** ([rules.py](mediklik-algorithm/services/recommendation/rules.py) + `recommend_complementary`): aturan gejala→kategori (mis. "demam" → demam_nyeri, vitamin, batuk-flu). **Menghindari** merekomendasikan obat dengan generic_name sama (bukan substitusi, tapi pelengkap), lalu memprioritaskan yang **tersedia di apotek terkait**.
- **Role Kurir & Penugasan Kurir** ([usecase/courier.go](mediklik-backend/usecase/courier.go)): apoteker/admin menugaskan kurir ke order (`AssignCourier`/`UnassignCourier`), kurir melihat order, menyelesaikan pengiriman dengan **bukti foto** (`CompleteDelivery` upload ke Supabase Storage). Pengecekan unik (NIK/plat) via `CheckUnique`.

## D3. Fitur Otomatisasi (Cron & Email)

### Cron Jobs (di balik layar) — [cron/scheduler.go](mediklik-backend/cron/scheduler.go)
Memakai library `gocron`. 4 job (jadwal dari `.env`):
| Job | Jadwal (default) | Fungsi |
|---|---|---|
| `cancel-expired-orders` | `0 0,12 * * *` (00:00 & 12:00) | Batalkan order yang tidak dibayar/diproses tepat waktu |
| `confirm-expired-orders` | `0 1,13 * * *` | Auto-konfirmasi order yang lewat batas |
| `deliver-shipped-orders` | `0 2,14 * * *` | Tandai "delivered" untuk order yang sudah dikirim |
| `handle-expired-payment` | `0 * * * *` (tiap jam) | Tangani pembayaran kedaluwarsa |

**Keandalan job dibungkus berlapis** (decorator pattern):
- `safeJob` — **panic recovery** + timeout konteks (`JobTimeout=180s`).
- `withRetry` — retry sampai `RetryMax=3` dengan **exponential backoff** (`RetryBaseWait × 2^n`).
- `withLock` — **distributed lock via DB** (`AcquireLock`) supaya bila ada >1 instance backend, job hanya jalan di satu tempat (mencegah double-processing).
- `withMetrics` — catat durasi, jumlah baris terdampak, sukses/gagal ke channel `jobHistory`.
- `RunMissedJobs()` saat startup — menjalankan job yang mungkin terlewat saat server mati.

### Pengiriman Email — [utils/email.go](mediklik-backend/utils/email.go)
- Library **`go-mail`**, transport **SMTP Gmail** (`smtp.gmail.com:587`) dengan kredensial dari env (`EMAIL_SMTPUSER/PASS`).
- Jenis email (template HTML, [utils/email_template.go](mediklik-backend/utils/email_template.go)): **verifikasi email** (kedaluwarsa 30 menit), **reset password** (10 menit), **akun apoteker baru** (kirim password), **akun kurir baru**, **upgrade role**.
- Tiap jenis punya judul/konten/expiry/link sendiri (`emailContents` map) dan tautan diarahkan ke `EMAIL_FRONTENDURL`.

### ⚠️ Blind spot Otomatisasi
1. **Job cron tidak menyentuh `product_popularities`** — tidak ada job yang meng-agregasi penjualan harian ke tabel popularitas (konsisten dengan temuan A5: data populer tidak ter-update otomatis).
2. **Email dikirim sinkron** di alur request (mis. registrasi). Bila SMTP lambat, response API ikut tertahan. Idealnya async/queue.
3. **Kredensial SMTP Gmail App Password** di env — perlu dijaga; rotasi kunci & rate-limit Gmail bisa jadi kendala saat banyak user mendaftar.

---

## Lampiran — Ringkasan Blind Spot Paling Kritikal (untuk diantisipasi penguji)
1. **Latency dashboard kemungkinan kosong** — histogram durasi di-comment di [middleware/metrics.go](mediklik-backend/middleware/metrics.go).
2. **Popularitas tidak real-time** — `product_popularities` hanya di-seed & dibaca; ada pula **konflik skema** (FK ke `products` vs join ke `pharmacy_products`).
3. **Pipeline trace Jaeger tidak lengkap** — exporter ada, service Jaeger tidak.
4. **Dead code & konstanta menyesatkan** di optimizer Python (`objective_function.py`, `SHIPPING_COST` tak terpakai, "ongkir" = jarak km).
5. **Potensi inkonsistensi key** `item_id` vs `cart_item_id` pada batch update keranjang, dan argumen `userID` vs `cartID` di `IncrementCartQty`.
6. **Debug `print`/`fmt.Println` query** masih ada di jalur produksi ([cart_product.go](mediklik-backend/repo/postgres/cart_product.go)).
