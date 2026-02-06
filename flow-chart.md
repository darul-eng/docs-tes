# Flowchart Tiga Proses Bisnis

Dokumen ini berisi flowchart untuk tiga alur proses: **SO→PO→GR**, **SO→DO→Invoice Penjualan**, dan **PO→Invoice Rekon**.

---

## 1. Alur SO → PO → GR (Pembelian / Barang Masuk)

**Catatan:** Di SO, tipe barang **bukan** pertimbangan. Tipe barang baru menjadi pertimbangan di **PO** (jika Tipe Barang = convenience, maka Nomor SO wajib dipilih).

```mermaid
flowchart TB
    subgraph SO["Sales Order"]
        A1[Buat SO] --> A2[SO selesai<br/>tanpa pertimbangan tipe barang]
    end

    subgraph PO["Purchase Order"]
        B1[Buat PO] --> B2[Pilih Vendor, Currency,<br/>Tipe Barang]
        B2 --> B3{Tipe Barang<br/>= convenience?}
        B3 -->|Ya| B4[Wajib pilih Nomor SO]
        B3 -->|Tidak| B5[PO biasa / dari PR]
        B4 --> B6[Detail PO pakai QTY SO]
        B5 --> B6
        B6 --> B7[Simpan PO<br/>cnmr_po]
    end

    subgraph GR["Goods Receipt"]
        C1[Buat GR] --> C2[Pilih Nomor PO]
        C2 --> C3[Detail otomatis dari PO]
        C3 --> C4{Qty GR ≤ Qty PO?<br/>Tgl GR ≥ Tgl PO?}
        C4 -->|Ya| C5[Simpan GR]
        C4 -->|Tidak| C6[Validasi gagal]
        C5 --> C7[Stok bertambah]
    end

    A2 -.->|SO tersedia<br/>jika PO convenience| B4
    B7 --> C2
    C7 --> D[(Persediaan siap<br/>untuk DO)]
```

---

## 2. Alur SO → DO → Invoice Penjualan

```mermaid
flowchart TB
    subgraph SO["Sales Order"]
        A1[SO sudah ada] --> A2[nqty_do dilacak<br/>per detail SO]
    end

    subgraph DO["Delivery Order"]
        B1[Buat DO, Referensi = SO] --> B2["Sumber 1: SO di Header"]
        B2 --> B2a["Lookup: find/sales-order<br/>View: vso_ready_gi<br/>Kondisi: nqty_sisa > 0"]
        B2a --> B3[Pilih 1 Nomor SO<br/>→ cnmr_ref1]
        B3 --> B4[Header terisi: pelanggan, ship-to, tgl]
        B4 --> B5["Sumber 2: SO per baris detail"]
        B5 --> B5a["Lookup: find/so<br/>View: vsales_order_ready_do<br/>Kondisi: SO sudah PO+GR, qty_pending > 0"]
        B5a --> B6[Per baris: pilih Nomor SO + GR<br/>→ detail.cnmr_so, cnmr_po, cnmr_mutasi]
        B6 --> B7[Material detail = sama dengan SO header?]
        B7 -->|Ya| B8[Qty DO ≤ sisa SO]
        B7 -->|Tidak| B9[Validasi gagal]
        B8 --> B10[Simpan DO]
        B10 --> B11[Backend update nqty_do di SO]
    end

    subgraph INV["Invoice Penjualan"]
        C1[Buat Invoice] --> C2[Pilih DO/SO ready to invoice]
        C2 --> C3[Detail dari material-so/details]
        C3 --> C4[Tgl Invoice ≥ Tgl DO?]
        C4 -->|Ya| C5[Simpan Invoice]
        C4 -->|Tidak| C6[Validasi gagal]
    end

    A1 --> B2
    B11 --> C2
```

---

## 3. Alur PO → Invoice Rekonsiliasi

```mermaid
flowchart TB
    subgraph PO["Purchase Order"]
        A1[PO sudah ada] --> A2[Harga per material: nprice]
    end

    subgraph INR["Invoice Rekonsiliasi"]
        B1[Buat Inv Rekon] --> B2[Lookup PO: find/po<br/>View: vpo_ready_inv_recon]
        B2 --> B3[Pilih 1 Nomor PO]
        B3 --> B4[Data PO terisi: supplier, material, nprice]
        B4 --> B5[Isi harga baru: nprice_new]
        B5 --> B6{Kategori: naik / turun?}
        B6 -->|Ya| B7[Cek stok: checkStockCondition]
        B6 -->|Tidak| B8[Lanjut simpan]
        B7 --> B9[Generate jurnal<br/>sesuai naik/turun]
        B9 --> B10[Simpan Inv Rekon]
        B8 --> B10
        B10 --> B11[Nilai persediaan<br/>ter-update]
    end

    A2 --> B4
```

---

## 4. Hubungan Antar Tiga Proses (Overview)

```mermaid
flowchart LR
    subgraph Proses1["SO → PO → GR"]
        direction TB
        S1[SO] --> P1[PO]
        P1 --> G1[GR]
        G1 --> ST[(Stok)]
    end

    subgraph Proses2["SO → DO → Inv Penjualan"]
        direction TB
        S2[SO] --> D1[DO]
        D1 --> I1[Invoice]
        D1 -.->|pakai stok| ST
    end

    subgraph Proses3["PO → Inv Rekon"]
        direction TB
        P2[PO] --> IR[Inv Rekon]
        IR -.->|koreksi nilai| ST
    end

    S1 -.->|PO tipe convenience:<br/>wajib pilih SO| P1
    S2 -.->|sama SO| S1
```

---

## 5. Ringkasan Singkat

| Alur | Titik kunci |
|------|-------------|
| **SO→PO→GR** | Di PO: jika Tipe Barang = convenience → wajib pilih SO; GR hanya dari PO → stok naik. |
| **SO→DO→Inv** | DO: SO header (sisa qty) + SO per baris (sudah PO+GR, qty pending); Invoice dari DO/SO ready. |
| **PO→Inv Rekon** | Pilih PO → isi harga baru → jurnal naik/turun → nilai persediaan dikoreksi. |

File ini bisa dibuka di editor yang mendukung Mermaid (VS Code dengan ekstensi Mermaid, atau render di GitHub/GitLab).

---

## 6. Sequence Diagram: Proses Delivery Order (DO)

Diagram di bawah menggambarkan alur **Create** (dengan status Open dan Release) serta **Update** (Open→Open dan Open→Release) beserta validasi dan efek di backend (SO, stok, jurnal).

---

### Sumber Data: Stock, Qty SO, Qty, dan nprice di DO (penekanan)

Empat field utama di detail DO dan **dari mana datanya berasal** (hanya untuk referensi **SO / SOI**):

| Field | Sumber data | Keterangan |
|-------|-------------|------------|
| **Stock (nstock)** | **Create:** dari lookup **find/so** (view `vsales_order_ready_do`) → kolom **qty_pending** (qty pending untuk kombinasi SO+PO+GR yang dipilih).<br/>**Update (saat buka modal):** FE memanggil `invoice-reconciliation/stock-condition-by-so` (param `cnmr_so` + `ckd_material`) → **qty_pending** untuk stock; **fallback** ke `sales-order/details` → **nqty − nqty_do**.<br/>**Update (saat pilih ulang SO per baris):** dari `sales-order/details` → **nqty − nqty_do** (karena mode edit). | Menunjukkan “stok” yang boleh dikirim untuk baris itu. Validasi: `nstock ≥ 1` dan **detail.nqty ≤ detail.nstock** (Qty DO tidak melebihi stock). |
| **Qty SO (nqty_so)** | Diisi saat user **pilih SO per baris** di kolom Nomor SO.<br/>**nqty_so** = `qtyByMaterialFromHeader[ckd_material] ?? qtyPendingFromSO`.<br/>• **qtyByMaterialFromHeader:** dari SO **header** (`cnmr_ref1`) via **sales-order/details** → **(nqty − nqty_do)** per material.<br/>• **qtyPendingFromSO:** dari detail SO **yang dipilih di baris** → **(nqty − nqty_do)** untuk material itu. | Batas maksimal qty DO per baris (sisa SO). Validasi: **detail.nqty ≤ nqty_so** (“Qty DO tidak boleh melebihi Qty SO”). Tampil di kolom “Qty SO”. |
| **Qty (nqty)** | **Input user** (kolom QTY di tabel detail). | Qty yang akan dikirim. Validasi: > 0, ≤ nqty_so, ≤ nstock. |
| **nprice** | **Urutan prioritas:**<br/>1) **Invoice Reconciliation:** GET `invoice-reconciliation/nprice-by-so?cnmr_so=...` → **nprice_new** per material; jika ada dan ≥ 0 → dipakai.<br/>2) **Metode persediaan FIFO dan detail punya cnmr_po:** dari **PO** (purchaseOrderDetails) → jika PPN include (cjns_ppn = "0") → **nprice/1.11** (DPP), else **nprice**.<br/>3) **Lainnya:** dari **GR** (find/gr oleh cnmr_mutasi) → **nprice** detail GR untuk material+lokasi; bila tidak ada GR → **master material** (barang.detail per cloc) → **nhrg_1**. | Harga per satuan di DO (untuk total & jurnal). Validasi (bukan free): nprice ≥ 0. |

Diagram alur sumber data (untuk referensi SOI):

```mermaid
flowchart LR
    subgraph Stock["nstock (Stock)"]
        S1[Create: find/so] --> S1a[qty_pending]
        S2[Update: sales-order/details] --> S2a[nqty - nqty_do]
    end

    subgraph QtySO["nqty_so (Qty SO)"]
        Q1[SO Header cnmr_ref1] --> Q1a[sales-order/details<br/>nqty - nqty_do per material]
        Q2[SO dipilih per baris] --> Q2a[nqty - nqty_do]
        Q1a --> Q3["nqty_so dari header atau baris"]
        Q2a --> Q3
    end

    subgraph Qty["nqty (Qty)"]
        U1[Input user]
    end

    subgraph Nprice["nprice"]
        P1[1. Inv Rekon nprice_new] --> P2{Pakai?}
        P2 -->|Ya| P3[nprice]
        P2 -->|Tidak| P4[2. FIFO + PO?]
        P4 -->|Ya| P5[PO detail: DPP/nprice]
        P4 -->|Tidak| P6[3. GR nprice / material nhrg_1]
        P5 --> P3
        P6 --> P3
    end
```

**Kapan diisi:** Stock, Qty SO, dan nprice per baris diisi saat user **memilih Nomor SO di kolom detail** (lookup `find/so` → `handleSelectSODetail`). Qty diisi user setelah baris ada. Saat **buka modal update** DO, FE juga melakukan pengayaan data: **nprice** dari `invoice-reconciliation/nprice-by-so` dan **nstock** dari `invoice-reconciliation/stock-condition-by-so` (fallback ke `sales-order/details`).

---

### 6.1 Create DO dengan Status Open

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant FE as Frontend DO Screen
    participant API as API Backend
    participant SO as Sales Order
    participant DB as Database

    User->>FE: Klik "Tambah"
    FE->>FE: Buka modal, reset payload (ctype_ref1, details kosong)
    User->>FE: Pilih Jenis Referensi = SO
    User->>FE: Klik lookup "Sales Order" (header)
    FE->>API: GET procurement/goods-issue/find/sales-order
    API->>DB: Query view vso_ready_gi (nqty_sisa > 0)
    DB-->>API: Daftar SO
    API-->>FE: Data SO
    FE->>FE: Tampilkan list SO
    User->>FE: Pilih 1 SO untuk header DO
    FE->>API: GET sales-order/details/{cnmr_so} untuk header
    API->>DB: Ambil SO header (details, dmodify, dtgl_so)
    API-->>FE: Header materials dan qty per material
    FE->>FE: Set cnmr_ref1, ship-to, tgl DO, header materials, qty header, validateModify
    FE->>FE: Tambah 1 baris detail kosong (material belum dipilih)
    User->>FE: Per baris: klik kolom "Nomor SO" → lookup find/so
    FE->>API: GET procurement/goods-issue/find/so
    API->>DB: Query view vsales_order_ready_do
    API-->>FE: Data SO+PO+GR (qty_pending untuk mengisi field stock)
    User->>FE: Pilih SO per baris
    FE->>API: GET sales-order/details/{cnmr_so} (SO baris ini)
    API->>DB: tsales_order_dtl (nqty, nqty_do per material)
    FE->>FE: Cek material SO baris sama dengan material SO header
    FE->>FE: Isi field qty so dari qty header (nqty dikurangi nqty_do)
    FE->>FE: Isi field stock
    Note over FE: Create mode, stock diisi dari qty_pending pada data find/so yang dipilih
    alt Proses mengisi nilai price
        opt Cek apakah ada harga dari Recon
            FE->>API: GET invoice-reconciliation/nprice-by-so (param cnmr_so)
            API-->>FE: Jika ada nprice_new per material maka dipakai
        end
        opt Jika tidak ada Recon dan metode FIFO dan ada nomor PO
            Note over FE: Ambil harga dari purchaseOrderDetails yang sudah ikut di sales-order/details
        end
        opt Jika tidak FIFO atau tidak ada PO
            opt Jika ada nomor GR
                FE->>API: GET procurement/goods-issue/find/gr/{cnmr_mutasi}
                API-->>FE: Ambil harga dari detail GR untuk material dan lokasi
            end
            opt Jika tidak ada GR
                Note over FE: Ambil harga dari master material (nhrg_1) per lokasi
            end
        end
    end
    FE->>FE: Isi detail row stock qtyso price dan set qty 0
    User->>FE: Isi Qty (nqty) tiap baris, Penerima, Tanggal DO
    User->>FE: Pilih Status = Open (0)
    User->>FE: Klik Simpan
    FE->>FE: Validasi tanggal DO tidak kurang dari tanggal SO
    FE->>FE: Validasi qty tidak nol
    FE->>FE: Validasi qty tidak lebih dari qty so
    FE->>FE: Validasi qty tidak lebih dari stock
    FE->>FE: Validasi nprice tidak negatif
    FE->>FE: Validasi daftar material detail sama dengan material SO header
    alt Validasi gagal
        FE-->>User: Swal error, tidak kirim
    end
    FE->>API: POST validate-modify (tsales_order_hdr, cnmr_so, dmodify)
    API-->>FE: OK / 409 (data dipakai)
    alt 409/400/499
        FE-->>User: "DO tidak tersimpan - Data sedang digunakan"
    end
    opt Detail punya cnmr_po
        FE->>API: GET procurement/purchase-order/details/{cnmr_po} (per PO unik)
        FE->>FE: dataToCheck (lock PO)
    end
    FE->>API: POST procurement/goods-issue/create (payload, cstatus: "0")
    API->>API: createGI: generate cnmr_mutasi (GIG-...)
    loop Setiap detail
        Note over API: cstatus bukan 1, skip update nqty_do
    end
    API->>DB: INSERT tmutasi_goods_hdr
    API->>DB: DELETE + INSERT tmutasi_goods_dtl
    Note over API: Tidak panggil journalGI / checkQty (karena Open)
    API-->>FE: 200 + data DO
    FE->>FE: Tutup modal, refresh list
    FE-->>User: DO tersimpan (status Open)
```

### 6.2 Create DO dengan Status Release

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant FE as Frontend DO Screen
    participant API as API Backend
    participant SO as Sales Order (tsales_order_dtl)
    participant DB as Database
    participant Jurnal as Modul Jurnal
    participant Stok as Check Qty / Material Dtl

    User->>FE: Klik "Tambah"
    FE->>FE: Buka modal dan reset payload
    User->>FE: Pilih Jenis Referensi SO
    User->>FE: Klik lookup Sales Order untuk header
    FE->>API: GET procurement/goods-issue/find/sales-order
    API->>DB: Query view vso_ready_gi
    API-->>FE: Data SO
    User->>FE: Pilih 1 SO untuk header DO
    FE->>API: GET sales-order/details/{cnmr_so} untuk header
    API->>DB: Ambil SO header
    API-->>FE: Header materials qty dan dmodify
    FE->>FE: Set cnmr_ref1 ship-to tgl DO header materials qty header validateModify
    FE->>FE: Tambah 1 baris detail kosong

    User->>FE: Per baris pilih Nomor SO
    FE->>API: GET procurement/goods-issue/find/so
    API->>DB: Query view vsales_order_ready_do
    API-->>FE: Data SO PO GR dan qty_pending
    User->>FE: Pilih SO per baris
    FE->>API: GET sales-order/details/{cnmr_so} untuk baris
    API-->>FE: Detail SO baris
    FE->>FE: Cek material SO baris sama dengan material SO header
    FE->>FE: Isi qty so dari qty header
    FE->>FE: Isi stock dari qty_pending data find/so
    alt Proses mengisi price
        opt Cek Recon
            FE->>API: GET invoice-reconciliation/nprice-by-so (param cnmr_so)
            API-->>FE: Jika ada nprice_new maka dipakai
        end
        opt Jika tidak ada Recon dan metode FIFO dan ada nomor PO
            Note over FE: Ambil harga dari purchaseOrderDetails yang ikut di sales-order/details
        end
        opt Jika tidak FIFO atau tidak ada PO
            opt Jika ada nomor GR
                FE->>API: GET procurement/goods-issue/find/gr/{cnmr_mutasi}
                API-->>FE: Ambil harga dari detail GR
            end
            opt Jika tidak ada GR
                Note over FE: Ambil harga dari master material per lokasi
            end
        end
    end
    FE->>FE: Isi detail row dan set qty 0
    User->>FE: Isi Qty dan field header lain
    User->>FE: Pilih Status = Release (1)
    User->>FE: Klik Simpan
    FE->>FE: Validasi tanggal qty stock price dan material sama header
    FE->>API: POST validate-modify
    opt Detail punya cnmr_po
        FE->>API: GET procurement/purchase-order/details/{cnmr_po} untuk lock PO
    end
    FE->>API: POST procurement/goods-issue/create (payload, cstatus: "1")
    API->>API: createGI: generate cnmr_mutasi (GIG-...)
    loop Setiap detail (ref SOI)
        API->>SO: SELECT tsales_order_dtl (cnmr_so, ckd_material, cnmr_so_dtl)
        API->>SO: "UPDATE nqty_do tambah detail nqty"
    end
    API->>DB: INSERT tmutasi_goods_hdr (cstatus: 1)
    API->>DB: INSERT tmutasi_goods_dtl
    API->>API: journalGI(data, users, runner, giNo)
    API->>Jurnal: Buat jurnal pengeluaran barang (COA, debet/kredit)
    API->>Stok: checkQty(data, true, runner) → kurangi stok per material/lokasi
    API->>DB: UPDATE mmaterial_dtl (nstock_utama, nstock_stoll, dll)
    API->>API: commitTransaction()
    API-->>FE: 200 + data DO
    FE-->>User: DO tersimpan (status Release), stok keluar, SO nqty_do bertambah
```

### 6.3 Update DO dari Status Open ke Open

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant FE as Frontend DO Screen
    participant API as API Backend
    participant SO as Sales Order
    participant DB as Database

    User->>FE: Pilih DO (cstatus = 0) di list, klik "Ubah"
    FE->>API: GET procurement/goods-issue/details/{cnmr_mutasi}
    API->>DB: Query tmutasi_goods_hdr + details
    API-->>FE: Data DO (header dan details)
    FE->>FE: dispatch(changePayload), setStatusCurrent(0), isEditable=true
    opt Jika referensi SOI
        FE->>API: GET sales-order/details/{cnmr_ref1} untuk header
        API-->>FE: Header materials dan qty header per material
        FE->>FE: Set tgl SO untuk validasi tanggal dan set validateModify
        FE->>FE: Ambil daftar cnmr_so unik dari detail DO
        loop Untuk setiap cnmr_so unik
            FE->>API: GET invoice-reconciliation/nprice-by-so (param cnmr_so)
            API-->>FE: Map nprice_new per material
            FE->>API: GET sales-order/details/{cnmr_so}
            API-->>FE: Qty pending per material dari SO (nqty dikurangi nqty_do)
        end
        loop Untuk setiap detail DO
            FE->>API: GET invoice-reconciliation/stock-condition-by-so (param cnmr_so dan ckd_material)
            API-->>FE: qty_pending untuk stock
            FE->>FE: Set nstock dan nqty_ref dari qty_pending endpoint atau fallback qty pending SO
            FE->>FE: Set nqty_so dari qty header per material
            FE->>FE: Set nprice dari map nprice_new jika tersedia
        end
    end
    opt Jika user pilih ulang SO per baris
        Note over FE: handleSelectSODetail dipanggil lagi, create stock dari nqty minus nqty_do, qtyso dari header, price dari Recon atau FIFO PO atau GR atau master
    end
    User->>FE: Ubah field (tanggal, nqty, penerima, dll), Status tetap Open (0)
    User->>FE: Klik Simpan
    FE->>API: GET invoice-reconciliation/nprice-by-so untuk refresh nprice (per cnmr_so unik)
    FE->>FE: Validasi tanggal qty stock price dan material sama header
    FE->>API: POST validate-modify (SO dmodify)
    FE->>API: PUT procurement/goods-issue/update/{cnmr_mutasi} (payload, cstatus: "0")
    API->>API: updateGI(giNo, data)
    loop Setiap detail
        Note over API: cstatus bukan 1, tidak update nqty_do di SO
    end
    API->>DB: UPDATE tmutasi_goods_hdr (dtgl_mutasi, cstatus, ntotal, ...)
    API->>DB: DELETE tmutasi_goods_dtl WHERE cnmr_mutasi = giNo
    API->>DB: INSERT tmutasi_goods_dtl (detail baru)
    Note over API: Tidak journalGI, tidak checkQty
    API->>API: commitTransaction()
    API-->>FE: 200 + data DO
    FE->>FE: Tutup modal, refresh list
    FE-->>User: DO berhasil diubah (tetap Open), nqty_do SO tidak berubah
```

### 6.4 Update DO dari Status Open ke Release

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant FE as Frontend DO Screen
    participant API as API Backend
    participant SO as Sales Order (tsales_order_dtl)
    participant DB as Database
    participant Jurnal as Modul Jurnal
    participant Stok as Check Qty / Material Dtl

    User->>FE: Pilih DO (cstatus = 0), klik "Ubah"
    FE->>API: GET procurement/goods-issue/details/{cnmr_mutasi}
    API-->>FE: Data DO (header dan details)
    opt Jika referensi SOI
        FE->>API: GET sales-order/details/{cnmr_ref1} untuk header
        API-->>FE: Header materials dan qty header per material
        FE->>FE: Set tgl SO untuk validasi tanggal dan set validateModify
        FE->>FE: Ambil daftar cnmr_so unik dari detail DO
        loop Untuk setiap cnmr_so unik
            FE->>API: GET invoice-reconciliation/nprice-by-so (param cnmr_so)
            API-->>FE: Map nprice_new per material
            FE->>API: GET sales-order/details/{cnmr_so}
            API-->>FE: Qty pending per material dari SO
        end
        loop Untuk setiap detail DO
            FE->>API: GET invoice-reconciliation/stock-condition-by-so (param cnmr_so dan ckd_material)
            API-->>FE: qty_pending untuk stock
            FE->>FE: Set nstock dan nqty_ref dari qty_pending endpoint atau fallback qty pending SO
            FE->>FE: Set nqty_so dari qty header per material
            FE->>FE: Set nprice dari map nprice_new jika tersedia
        end
    end
    opt Jika user pilih ulang SO per baris
        Note over FE: handleSelectSODetail dipanggil lagi, stock dari nqty minus nqty_do, qtyso dari header, price dari Recon atau FIFO PO atau GR atau master
    end
    User->>FE: Ubah field jika perlu (nqty, dll), Pilih Status = Release (1)
    User->>FE: Klik Simpan
    FE->>API: GET invoice-reconciliation/nprice-by-so untuk refresh nprice (per cnmr_so unik)
    FE->>FE: Validasi tanggal qty stock price dan material sama header
    FE->>API: POST validate-modify
    FE->>API: PUT procurement/goods-issue/update/{cnmr_mutasi} (payload, cstatus: "1")
    API->>API: updateGI(giNo, data)
    loop Setiap detail (ref SOI)
        API->>SO: SELECT tsales_order_dtl (cnmr_so, ckd_material, cnmr_so_dtl)
        API->>SO: "UPDATE nqty_do tambah detail nqty"
    end
    API->>DB: UPDATE tmutasi_goods_hdr (cstatus: "1", dtgl_mutasi, ntotal, ...)
    API->>DB: DELETE + INSERT tmutasi_goods_dtl
    API->>API: journalGI(data, users, runner, giNo)
    API->>Jurnal: Buat jurnal pengeluaran barang
    API->>Stok: checkQty(data, true, runner) → kurangi stok
    API->>DB: UPDATE mmaterial_dtl (stok)
    API->>API: commitTransaction()
    API-->>FE: 200 + data DO
    FE-->>User: DO berubah ke Release, SO nqty_do bertambah, stok keluar, jurnal terbentuk
```

### Ringkasan Perbedaan Empat Skenario

| Skenario | Status dikirim | Update nqty_do di SO? | Jurnal GI? | Update stok? |
|----------|----------------|------------------------|------------|--------------|
| Create + Open | cstatus: "0" | Tidak | Tidak | Tidak |
| Create + Release | cstatus: "1" | Ya (tambah per detail) | Ya | Ya (keluar) |
| Update Open→Open | cstatus: "0" | Tidak | Tidak | Tidak |
| Update Open→Release | cstatus: "1" | Ya (tambah per detail) | Ya | Ya (keluar) |

**Sumber field di sequence Create/Update:**  
- **nstock:** Create = find/so qty_pending; Update (buka modal) = stock-condition-by-so qty_pending (fallback sales-order/details nqty minus nqty_do); Update (pilih ulang SO) = sales-order/details nqty minus nqty_do.  
- **nqty_so:** sales-order/details (header + per SO) → nqty − nqty_do.  
- **nqty:** input user; validasi ≤ nqty_so, ≤ nstock.  
- **nprice:** Inv Rekon (nprice-by-so) → PO (FIFO) → GR/material; saat Update sebelum Simpan: refresh via invoice-reconciliation/nprice-by-so.
