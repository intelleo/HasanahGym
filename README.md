# HASANAH GYM
## UML & System Architecture | Concept v0.2
**Versi:** 0.2  |  **Tanggal:** 12 Oktober 2026  |  **Fokus Update:** Penambahan Absensi Staf, Shift Kasir, & Integrasi Hardware Scanner USB

---

## 01 | System Context Diagram
Diagram konteks sistem yang menggambarkan batas sistem HGMS dengan para aktor pengguna, perangkat keras scanner, dan layanan eksternal opsional.

```mermaid
flowchart LR
    subgraph Users["Aktor Pengguna"]
        Owner["Owner Gym"]
        Admin["Admin / Resepsionis"]
        Trainer["Personal Trainer"]
        Member["Member Gym"]
    end

    subgraph Hardware["Perangkat Keras Meja Resepsionis"]
        Scanner["USB 2D/QR Scanner (HID Keyboard)"]
    end

    subgraph System["HASANAH GYM MANAGEMENT SYSTEM (HGMS)"]
        CoreApp["HGMS Core Web Application & API"]
    end

    subgraph External["Layanan Pihak Ketiga (Opsional / Fase 3)"]
        PaymentGW["Payment Gateway (QRIS / VA)"]
        WA["WhatsApp Gateway API"]
    end

    Owner <-->|"Pantau dashboard, audit, staf & keuangan"| CoreApp
    Admin <-->|"Registrasi, pembayaran, POS, shift kerja"| CoreApp
    Trainer <-->|"Jadwal kelas, presensi sesi, absensi kerja"| CoreApp
    Member <-->|"Kartu QR digital, cek paket, booking kelas"| CoreApp

    Scanner -->|"Tembak token QR member / staf"| CoreApp
    CoreApp <-.->|"Webhook status pembayaran"| PaymentGW
    CoreApp -.->|"Notifikasi reminder jatuh tempo"| WA
```

---

## 02 | Use Case Overview Diagram
Pemetaan peran pengguna terhadap fitur-fitur fungsional sistem, termasuk modul baru absensi karyawan dan shift kasir.

```mermaid
flowchart LR
    subgraph MemberRole["Member"]
        UC_M1["Lihat Kartu QR & Status Paket"]
        UC_M2["Booking Kelas Latihan"]
        UC_M3["Lihat Riwayat Kunjungan & Invoice"]
    end

    subgraph TrainerRole["Trainer"]
        UC_T1["Lihat Jadwal Kelas & Sesi PT"]
        UC_T2["Presensi Kehadiran Member di Sesi"]
        UC_T3["Clock-In / Clock-Out Absensi Kerja"]
    end

    subgraph AdminRole["Admin / Resepsionis"]
        UC_A1["Pendaftaran Member & Kelola Paket"]
        UC_A2["Verifikasi Pembayaran & Invoice"]
        UC_A3["Scan QR Check-In / Out Member"]
        UC_A4["Transaksi POS Kasir (Minuman/Suplemen)"]
        UC_A5["Buka Shift Kasir & Clock-In Masuk"]
        UC_A6["Tutup Shift Kasir & Rekonsiliasi Kas"]
    end

    subgraph OwnerRole["Owner Gym"]
        UC_O1["Pantau Dashboard Analitik & Omzet"]
        UC_O2["Kelola Akun Staf & Hak Akses (RBAC)"]
        UC_O3["Pantau Rekap Absensi & Jam Kerja Staf"]
        UC_O4["Lihat Laporan Keuangan & Audit Log"]
        UC_O5["Persetujuan Refund / Void Transaksi"]
    end
```

---

## 03 | Activity Diagram: Registrasi & Aktivasi Membership
Alur registrasi member baru hingga penerbitan token QR dan status paket aktif.

```mermaid
flowchart TD
    Start([Mulai Registrasi]) --> InputData[Admin Input Profil Member Baru]
    InputData --> SelectPkg[Pilih Paket Langganan]
    SelectPkg --> GenInvoice[Sistem Buat Invoice: Status PENDING]
    GenInvoice --> PayProcess[Member Melakukan Pembayaran]
    PayProcess --> VerifyPay{Pembayaran Tervalidasi?}
    
    VerifyPay -- Tidak --> StayPending[Status Tetap PENDING / Tindak Lanjut]
    
    VerifyPay -- Ya --> SetPaid[Update Status Invoice: PAID]
    SetPaid --> ActivateSub[Aktivasi Subscription: Set Tanggal Berlaku & Kuota]
    ActivateSub --> GenToken[Generate Token QR Acak Unik]
    GenToken --> StoreToken[Simpan Hash Token di Database]
    StoreToken --> DeliverQR[Terbitkan Kartu QR Digital / Cetak Kartu Fisik]
    DeliverQR --> Finish([Selesai])
```

---

## 04 | Activity Diagram: Check-In Member (Scan QR)
Alur validasi real-time saat member datang ke gym dan melakukan pemindaian token QR di meja resepsionis.

```mermaid
flowchart TD
    Start([Member Datang]) --> ShowQR[Member Tunjukkan Kartu QR]
    ShowQR --> ScanQR[Resepsionis Scan via Barcode Scanner USB]
    ScanQR --> SendServer[Kirim Token ke API Server HGMS]
    SendServer --> ValToken{Token Terdaftar & Valid?}
    
    ValToken -- Tidak --> Reject1[Tolak Akses: QR Tidak Dikenali]
    
    ValToken -- Ya --> CheckSub{Paket Aktif & Kuota Cukup?}
    CheckSub -- Tidak --> Reject2[Tolak Akses: Paket Habis / Expired]
    
    CheckSub -- Ya --> CheckDup{Sedang Aktif Check-in?}
    CheckDup -- Ya --> Reject3[Tolak Akses: Sudah Tercatat Check-in]
    
    CheckDup -- Tidak --> AtomicVisit[Catat Kunjungan: Visit checked_in_at = NOW]
    AtomicVisit --> DecrQuota[Kurangi Kuota Kunjungan bila paket tipe kuota]
    DecrQuota --> ShowSuccess[Layar Tampilkan Status SUKSES Hijau]
    ShowSuccess --> Done([Member Dipersilakan Masuk])

    Reject1 --> ShowErr[Layar Tampilkan Alasan Penolakan Merah]
    Reject2 --> ShowErr
    Reject3 --> ShowErr
    ShowErr --> FinishFail([Selesai / Arahkan ke Kasir])
```

---

## 05 | Activity Diagram: Absensi Staf & Shift Kasir (BARU)
Alur pencatatan jam kerja staf, pembukaan shift kasir, dan penutupan kasir di akhir hari.

```mermaid
flowchart TD
    StaffArrive([Staf Tiba di Gym]) --> Identify[Scan QR Staf atau Login Akun]
    Identify --> RoleCheck{Peran Staf?}

    RoleCheck -- Trainer / Staf Umum --> DirectIn[Sistem Catat Clock-In Masuk]
    
    RoleCheck -- Resepsionis / Kasir --> InputFloat[Input Saldo Kas Awal / Starting Float]
    InputFloat --> OpenShift[Buat Data Shift Kasir + Catat Clock-In Masuk]

    DirectIn --> WorkPeriod[Staf Menjalankan Aktivitas Kerja]
    OpenShift --> WorkPeriod

    WorkPeriod --> ShiftEnd([Waktu Pulang / Selesai Shift])
    ShiftEnd --> CheckOutRole{Peran Staf?}

    CheckOutRole -- Trainer / Staf Umum --> DirectOut[Scan QR Pulang / Catat Clock-Out]
    
    CheckOutRole -- Resepsionis / Kasir --> CountCash[Hitung Uang Fisik di Laci Kas]
    CountCash --> InputClosing[Input Total Uang Fisik ke Sistem]
    InputClosing --> SystemReconcile[Sistem Hitung Selisih: Transaksi vs Fisik]
    SystemReconcile --> CloseShiftRec[Tutup Shift Kasir + Catat Clock-Out Pulang]

    DirectOut --> Summary[Sistem Hitung Total Durasi Jam Kerja]
    CloseShiftRec --> Summary
    Summary --> EndStaff([Selesai])
```

---

## 06 | Sequence Diagram: Pembayaran ke Aktivasi QR
Interaksi antarkomponen perangkat lunak dalam memproses pembayaran hingga penerbitan token akses.

```mermaid
sequenceDiagram
    autonumber
    actor Member
    actor Admin as Resepsionis
    participant UI as Web Frontend
    participant API as API Controller
    participant PaySvc as Billing Service
    participant QRSvc as QR Token Service
    participant DB as MySQL Database

    Member->>Admin: Pilih paket langganan
    Admin->>UI: Input order membership
    UI->>API: POST /api/v1/invoices (create order)
    API->>DB: INSERT Invoice (status: PENDING)
    DB-->>API: invoice_id
    API-->>UI: Tampilkan invoice & total bayar
    
    Member->>Admin: Bayar tunai / transfer
    Admin->>UI: Klik "Konfirmasi Bayar"
    UI->>API: POST /api/v1/payments (confirm payment)
    
    API->>PaySvc: ProcessPayment(invoice_id, amount)
    PaySvc->>DB: BEGIN TRANSACTION
    PaySvc->>DB: UPDATE Invoice SET status = 'PAID'
    PaySvc->>DB: INSERT Payment record
    PaySvc->>DB: INSERT/UPDATE Subscription (status: ACTIVE)
    PaySvc->>QRSvc: GenerateMemberToken(member_id)
    QRSvc->>DB: INSERT QRToken (token_hash, status: ACTIVE)
    PaySvc->>DB: COMMIT TRANSACTION
    
    PaySvc-->>API: Aktivasi Sukses + QR Data
    API-->>UI: Response OK (Membership Aktif)
    UI-->>Admin: Tampilkan bukti bayar & kartu QR
    Admin-->>Member: Serahkan kartu fisik / QR digital aktif
```

---

## 07 | Sequence Diagram: Absensi Staf via Scanner USB
Interaksi cepat saat staf men-scan kartu tanda pengenal staf di meja resepsionis.

```mermaid
sequenceDiagram
    autonumber
    actor Staff as Karyawan Gym
    participant Scanner as USB QR Scanner
    participant PC as Browser Resepsionis
    participant API as API Server
    participant AttSvc as Staff Attendance Service
    participant DB as MySQL Database

    Staff->>Scanner: Arahkan kartu QR staf ke scanner
    Scanner->>PC: Input token otomatis + ketik ENTER (HID Emulation)
    PC->>API: POST /api/v1/staff/attendance/scan (token)
    API->>AttSvc: RecordAttendance(token)
    AttSvc->>DB: Cari User dari token QR
    
    alt Token Valid & Belum Clock-In Hari Ini
        AttSvc->>DB: INSERT StaffAttendance (user_id, clock_in: NOW, status: ON_TIME/LATE)
        DB-->>AttSvc: attendance_id
        AttSvc-->>API: Status: "CLOCK_IN_SUCCESS"
        API-->>PC: Pop-up Hijau: "Selamat Bekerja, [Nama Staf] (08:02)"
    else Sudah Clock-In (Sedang Bertugas)
        AttSvc->>DB: UPDATE StaffAttendance SET clock_out = NOW WHERE user_id = :id
        AttSvc-->>API: Status: "CLOCK_OUT_SUCCESS"
        API-->>PC: Pop-up Biru: "Terima Kasih, Absen Pulang Tercatat (17:05)"
    else Token Tidak Valid
        AttSvc-->>API: Error: Token Tidak Dikenali
        API-->>PC: Pop-up Merah: "Kartu Staf Tidak Terdaftar"
    end
```

---

## 08 | Domain Class Diagram (Entity-Relationship)
Struktur entitas basis data relasional sistem dengan penambahan tabel `StaffAttendance` dan `CashShift`.

```mermaid
classDiagram
    class Role {
        +int id
        +string name
    }

    class User {
        +int id
        +int role_id
        +string name
        +string email
        +string password_hash
        +string staff_code
        +string status
    }

    class StaffAttendance {
        +int id
        +int user_id
        +date shift_date
        +datetime clock_in
        +datetime clock_out
        +string status
        +string notes
    }

    class CashShift {
        +int id
        +int user_id
        +datetime opened_at
        +datetime closed_at
        +decimal starting_float
        +decimal expected_cash
        +decimal actual_cash
        +decimal variance
        +string status
    }

    class AuditLog {
        +int id
        +int actor_id
        +string action
        +string entity
        +int entity_id
        +json changes
        +datetime created_at
    }

    class Member {
        +int id
        +string member_code
        +string name
        +string phone
        +string status
        +datetime created_at
    }

    class QRToken {
        +int id
        +int member_id
        +int user_id
        +string token_hash
        +string token_type
        +datetime issued_at
        +datetime revoked_at
    }

    class Package {
        +int id
        +string name
        +string package_type
        +int duration_days
        +int visit_limit
        +decimal price
    }

    class Subscription {
        +int id
        +int member_id
        +int package_id
        +date start_at
        +date end_at
        +int remaining_visits
        +string status
    }

    class Visit {
        +int id
        +int member_id
        +datetime checked_in_at
        +datetime checked_out_at
        +string checkin_method
    }

    class Invoice {
        +int id
        +int member_id
        +string invoice_number
        +decimal total_amount
        +string status
        +datetime created_at
    }

    class Payment {
        +int id
        +int invoice_id
        +string method
        +decimal amount
        +string status
        +datetime paid_at
    }

    class Trainer {
        +int id
        +int user_id
        +string name
        +string specialty
    }

    class ClassSession {
        +int id
        +int trainer_id
        +string class_name
        +datetime start_at
        +datetime end_at
        +int capacity
    }

    class Booking {
        +int id
        +int member_id
        +int session_id
        +string status
        +datetime booked_at
    }

    class Product {
        +int id
        +string sku
        +string name
        +decimal price
        +int stock
    }

    class Sale {
        +int id
        +int cashier_id
        +int member_id
        +int cash_shift_id
        +decimal total
        +datetime created_at
    }

    class SaleItem {
        +int id
        +int sale_id
        +int product_id
        +int qty
        +decimal price
    }

    class Expense {
        +int id
        +int cashier_id
        +string category
        +decimal amount
        +string description
        +datetime created_at
    }

    %% Relasi Entitas
    Role "1" -- "N" User : assigns
    User "1" -- "N" AuditLog : performs
    User "1" -- "N" StaffAttendance : logs
    User "1" -- "N" CashShift : operates
    User "1" -- "0..1" Trainer : profiles
    User "1" -- "0..1" QRToken : identifies_staff

    Member "1" -- "N" Subscription : owns
    Package "1" -- "N" Subscription : defines
    Member "1" -- "N" QRToken : possesses
    Member "1" -- "N" Visit : makes
    Member "1" -- "N" Invoice : billed
    Invoice "1" -- "N" Payment : pays
    Member "1" -- "N" Booking : reserves

    Trainer "1" -- "N" ClassSession : leads
    ClassSession "1" -- "N" Booking : contains

    Product "1" -- "N" SaleItem : included_in
    Sale "1" -- "N" SaleItem : consists_of
    CashShift "1" -- "N" Sale : collects
```

---

## 09 | Membership State Diagram
Siklus hidup status paket langganan member (*Subscription Lifecycle*).

```mermaid
stateDiagram-v2
    [*] --> Pending : Buat Order / Tagihan Terbit
    Pending --> Active : Pembayaran Terverifikasi (PAID)
    Pending --> Cancelled : Dibatalkan / Timeout

    Active --> Frozen : Cuti Disetujui (Pause Masa Aktif)
    Frozen --> Active : Masa Cuti Selesai / Resume
    Frozen --> Expired : Batas Cuti Habis (Sesuai Aturan)

    Active --> Expired : Masa Berlaku Habis / Kuota 0
    Active --> Cancelled : Pembatalan Disetujui (Void/Refund)

    Expired --> Active : Perpanjang Paket (Subscription Baru)
    Cancelled --> [*]
```

---

## 10 | Staff Shift & Attendance State Diagram (BARU)
Siklus status sesi shift kerja staf resepsionis dan pencatatan presensi.

```mermaid
stateDiagram-v2
    [*] --> OffDuty : Belum Masuk Jam Kerja
    OffDuty --> OnDuty : Clock-In (Scan QR / Buka Shift)
    
    state OnDuty {
        [*] --> ShiftOpen : Input Starting Float Kasir
        ShiftOpen --> Serving : Melayani Member & Transaksi Kasir
        Serving --> ShiftClosing : Selesai Jam Kerja (Tutup Shift)
        ShiftClosing --> Reconciled : Input Fisik Kas & Cek Selisih
        Reconciled --> [*]
    }

    OnDuty --> OffDuty : Clock-Out Berhasil
```

---

## 11 | Logical System Architecture
Arsitektur logis aplikasi web terintegrasi dengan modul-modul backend dan database terpusat.

```mermaid
flowchart TD
    subgraph ClientLayers["Front-End Web Apps (Responsive Multi-Role)"]
        OwnerUI["Owner Dashboard Portal"]
        AdminUI["Reception & POS Cashier Portal"]
        TrainerUI["Trainer Class/PT Portal"]
        MemberUI["Member Self-Service & QR Portal"]
    end

    subgraph HardwareLayer["Meja Resepsionis"]
        USBScanner["USB QR / Barcode Scanner (HID Input)"]
    end

    subgraph APILayer["API Gateway & Security Layer"]
        Router["REST API Router / Controllers"]
        AuthMid["JWT Auth & Session Middleware"]
        RBAC["Role-Based Access Control (RBAC)"]
    end

    subgraph ServiceModules["Core Business Services"]
        MemberSvc["Member & Subscription Service"]
        QRSvc["QR Token & Generation Service"]
        AttMemberSvc["Member Attendance & Check-In Service"]
        AttStaffSvc["Staff Attendance & Shift Service (BARU)"]
        BillingSvc["Billing, Payment & Invoice Service"]
        ClassSvc["Class Schedule & PT Booking Service"]
        POSSvc["POS & Inventory Stock Service"]
        FinanceSvc["Cash Shift Drawer & Finance Service"]
        AuditSvc["Audit Trail & System Log Service"]
    end

    subgraph DataLayer["Storage Layer"]
        MySQL[(MySQL Relational Database)]
    end

    USBScanner --> AdminUI
    OwnerUI & AdminUI & TrainerUI & MemberUI --> Router
    Router --> AuthMid --> RBAC
    
    RBAC --> MemberSvc
    RBAC --> QRSvc
    RBAC --> AttMemberSvc
    RBAC --> AttStaffSvc
    RBAC --> BillingSvc
    RBAC --> ClassSvc
    RBAC --> POSSvc
    RBAC --> FinanceSvc
    RBAC --> AuditSvc

    MemberSvc & QRSvc & AttMemberSvc & AttStaffSvc & BillingSvc & ClassSvc & POSSvc & FinanceSvc & AuditSvc --> MySQL
```

