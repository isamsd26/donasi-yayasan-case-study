# Donasi Yayasan — Production Crowdfunding & Donation Platform (Engineering Case Study)

![PHP](https://img.shields.io/badge/PHP-8.2%2B-777BB4?logo=php&logoColor=white)
![Laravel](https://img.shields.io/badge/Laravel-12.x-FF2D20?logo=laravel&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-v4-06B6D4?logo=tailwindcss&logoColor=white)
![Midtrans](https://img.shields.io/badge/Payment-Midtrans_API-002855)
![Meta WhatsApp API](https://img.shields.io/badge/Notification-Meta_WA_Cloud_API-25D366?logo=whatsapp&logoColor=white)
![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub_Actions-2088FF?logo=githubactions&logoColor=white)

> **Live Production:** [donasi.yarish-tegal.com](https://donasi.yarish-tegal.com)  
> **Repository Note:** This repository is a **public engineering case study** documenting the architecture, database design, payment/notification workflows, and CI/CD pipeline of a live client production system built for **Yayasan Riyadhush Sholihin (Tegal)**. Proprietary client source code is maintained in a private repository.

---

## Business Problem & Engineering Impact

Previously, donation collection relied on manual chat confirmations, bank mutation checks, and manual spreadsheet recaps. I designed and shipped an end-to-end production web platform that automates the full lifecycle from donor checkout to financial reporting:

- **End-to-End Donation Lifecycle:** Donors browse campaigns, donate online, track status by reference code without login, and download **PDF invoices** (`DomPDF`) directly.
- **Dual Payment Architecture:** Supports **Midtrans Payment Gateway** (real-time webhook verification) and **Manual Bank Transfer** with automated unique transfer codes.
- **Automated WhatsApp Notifications:** Integrated with **Meta WhatsApp Cloud API (v19.0)** to send structured Indonesian templates for pending instructions, admin alerts, and payment confirmations.
- **Self-Service Admin CMS & Reporting:** Full back-office panel for managing campaigns, categories, updates, banners, FAQs, testimonials, static pages, WA reply templates, and **Excel financial exports** (`maatwebsite/excel`).
- **Automated CI/CD:** Zero-manual-step production deployment via **GitHub Actions** over SSH with safe maintenance-mode rollback (`trap`), automated migrations, and Laravel production caching.

---

## Live Production Screenshots

| Public Home & Hero | Campaign Directory & Live Feed |
| :---: | :---: |
| ![Home Page](docs/screenshots/home.png) | ![Campaign Directory](docs/screenshots/campaigns.png) |

---

## System Architecture

```mermaid
flowchart LR
    Donor["Donor / Public User"] -->|"Browse & Donate"| App["Laravel 12 Application"]
    Admin["Foundation Admin"] -->|"CMS & Verification (/admin)"| App

    App -->|"Eloquent ORM"| DB[("MariaDB / MySQL")]
    App -->|"Snap / Payment API"| Midtrans["Midtrans Payment Gateway"]
    Midtrans -->|"POST /api/midtrans/notification"| App

    App -->|"MetaWaService (Graph API v19.0)"| MetaWA["Meta WhatsApp Cloud API"]
    MetaWA -->|"Pending & Paid Templates"| Donor
    MetaWA -->|"New Donation Alert"| Admin

    App -->|"DomPDF"| Invoice["PDF Donor Invoice"]
    App -->|"Maatwebsite Excel"| Excel["Financial Report (.xlsx)"]

    GitHub["GitHub (main branch)"] -->|"GitHub Actions SSH Deploy"| App
```

---

## Donation & Payment Sequence Flow

```mermaid
sequenceDiagram
    actor Donor
    participant App as Laravel App
    participant DB as Database
    participant Midtrans as Midtrans API
    participant MetaWA as Meta WA Cloud API

    Donor->>App: Submit Donation Form (Amount, Phone, Prayer)
    App->>DB: DB::transaction() -> Create Donor, Donation (pending), PaymentTransaction
    App->>MetaWA: Send Pending Instruction / Admin Alert Template
    App-->>Donor: Redirect to Donation Status & Payment Details

    alt Midtrans Automated Payment
        Donor->>Midtrans: Complete Payment
        Midtrans->>App: POST /api/midtrans/notification (Webhook)
        App->>DB: Verify status (capture/settlement) -> Update Donation & Transaction = paid
    else Manual Bank Transfer
        Donor->>App: Transfer Amount + Unique Code
        Note over App,DB: Admin verifies transfer in /admin/donations
        App->>DB: PATCH /admin/donations/{id}/status -> paid
    end

    App->>MetaWA: Send Payment Confirmation Template (sendDonationPaidTemplate)
    Donor->>App: GET /donation/{reference}/invoice
    App-->>Donor: Download Generated PDF Invoice
```

---

## Core Database Schema (ERD)

```mermaid
erDiagram
    CAMPAIGN_CATEGORIES ||--o{ CAMPAIGNS : categorizes
    CAMPAIGNS ||--o{ CAMPAIGN_UPDATES : has
    CAMPAIGNS ||--o{ DONATIONS : receives
    DONORS ||--o{ DONATIONS : makes
    DONATIONS ||--o{ PAYMENT_TRANSACTIONS : tracks

    CAMPAIGNS {
        bigint id PK
        bigint campaign_category_id FK
        string title
        string slug
        decimal target_amount
        string status
        timestamp deleted_at
    }

    DONORS {
        bigint id PK
        string name
        string email
        string phone
    }

    DONATIONS {
        bigint id PK
        bigint campaign_id FK
        bigint donor_id FK
        string reference
        decimal amount
        decimal base_amount
        int unique_code
        string status
        timestamp paid_at
        timestamp expires_at
    }

    PAYMENT_TRANSACTIONS {
        bigint id PK
        bigint donation_id FK
        string external_id
        string payment_method
        string status
        json payload
        timestamp paid_at
    }
```

---

## Key Engineering Highlights

1. **Atomic Donation Creation:** Donor lookup/creation, donation record generation, unique transfer code calculation, and payment transaction logging are wrapped inside `DB::transaction()` to guarantee data consistency.
2. **Idempotent Midtrans Webhook Handling:** `MidtransWebhookController` maps Midtrans `transaction_status` (`capture`, `settlement`, `pending`, `cancel`, `deny`, `expire`) and `fraud_status` (`challenge`, `accept`) into clean internal states while persisting raw gateway payloads as JSON for auditability.
3. **Resilient Meta WhatsApp Cloud API Wrapper (`MetaWaService`):** Normalizes Indonesian phone formats (`08...` -> `628...`), reads credentials from `.env` with dynamic database `Settings` fallback, enforces a 5-second HTTP timeout, and logs failures gracefully without blocking user checkout.
4. **Safe SSH CI/CD Deployment:** GitHub Actions workflow uses `php artisan down` with a bash `trap 'php artisan up || true' EXIT` safeguard so the production site never stays stuck in maintenance mode if a deployment step fails.

---

## Tech Stack

| Layer | Technologies |
| :--- | :--- |
| **Backend** | PHP 8.2+, Laravel 12, Eloquent ORM, OOP / MVC |
| **Frontend** | Blade Templates, Tailwind CSS v4, Vite 7, Axios |
| **Database** | MariaDB / MySQL (Production), SQLite (Local Development) |
| **Integrations** | Midtrans PHP SDK (`^2.6`), Meta WhatsApp Cloud API (`v19.0`) |
| **Reporting** | `barryvdh/laravel-dompdf` (PDF Invoice), `maatwebsite/excel` (Excel Export) |
| **DevOps & QA** | GitHub Actions (SSH CI/CD), PHPUnit 11, Laravel Pint, Playwright |

---

## Author

**Isa Mas'ud** — Full-Stack Web Developer  
- Portfolio: [port.private-isa.my.id](https://port.private-isa.my.id)  
- LinkedIn: [linkedin.com/in/isamasud](https://linkedin.com/in/isamasud)  
- GitHub: [github.com/isamsd26](https://github.com/isamsd26)
