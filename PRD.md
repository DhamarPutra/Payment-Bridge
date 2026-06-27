# Product Requirement Document (PRD)

# Project

**Platform SaaS Payment Gateway Aggregator (Xendit Integration)**

Aplikasi SaaS yang memungkinkan merchant menerima pembayaran melalui Xendit tanpa perlu mengurus proses registrasi, validasi, maupun integrasi teknis secara mandiri. Platform bertindak sebagai middleware yang mengelola pembuatan invoice, callback, rekonsiliasi pembayaran, dan dashboard transaksi.

---

# 1. Project Overview & Objectives

Dokumen ini merinci kebutuhan pembangunan aplikasi monolitik modern menggunakan Laravel 12 dan React dalam satu repository (Single Repository).

Frontend menggunakan React yang dihubungkan dengan backend Laravel melalui Inertia.js sehingga tidak memerlukan REST API maupun autentikasi JWT.

Platform akan menyediakan layanan:

* Manajemen Merchant
* Manajemen Produk
* Pembuatan Invoice
* Integrasi Payment Gateway Xendit
* Dashboard Monitoring Pembayaran
* Webhook Automation
* Email Notification

Target aplikasi adalah menjadi SaaS yang dapat digunakan banyak merchant (multi-tenant).

---

# 2. Tech Stack Architecture

## Backend

* Laravel 12

## Frontend

* React
* Inertia.js
* Vite

## Styling

* TailwindCSS

## Database

* MySQL

## Queue

* Database Queue (MVP)
* Redis (Production Recommended)

## Cache

* Redis (Optional)

## Payment Gateway

* Xendit Invoice API

## Authentication

* Laravel Breeze (Inertia + React)

---

# 3. Architecture

```text
React

↓

Inertia.js

↓

Laravel Controller

↓

Service Layer

↓

Repository (Optional)

↓

MySQL

↓

Xendit API
```

Controller hanya bertugas menerima request.

Seluruh business logic berada pada Service Layer.

---

# 4. Core Features

## A. Authentication

Menggunakan Laravel Breeze.

Fitur:

* Register
* Login
* Logout
* Forgot Password
* Reset Password
* Email Verification

Menggunakan Session Authentication bawaan Laravel.

Tidak menggunakan JWT.

---

## B. Merchant Management

Merchant merupakan tenant dalam sistem.

Setiap merchant memiliki:

* nama
* slug
* status
* subscription plan

Semua transaksi wajib memiliki merchant_id.

---

## C. Product Management

Merchant dapat membuat produk.

Field minimal:

* name
* description
* price
* status

Harga hanya boleh divalidasi dari database.

Frontend tidak boleh mengirim nominal pembayaran sebagai sumber kebenaran.

---

## D. Checkout Flow

Flow pembayaran:

```text
User

↓

Klik Bayar

↓

POST /checkout

↓

Laravel

↓

Validasi User

↓

Validasi Merchant

↓

Validasi Product

↓

Ambil Harga dari Database

↓

Create Order

↓

Call Xendit Invoice API

↓

Update Invoice ID

↓

Redirect ke invoice_url
```

Redirect menggunakan:

```php
return Inertia::location($invoiceUrl);
```

---

## E. Order Creation

Order dibuat sebelum request ke Xendit.

Semua proses menggunakan:

```php
DB::transaction()
```

Status awal:

```
PENDING
```

---

## F. Payment Callback

Webhook Endpoint

```
POST /api/webhooks/xendit
```

Endpoint tidak menggunakan CSRF.

Wajib memvalidasi:

* x-callback-token

Status yang harus ditangani:

* PAID
* PENDING
* EXPIRED
* FAILED

Jika status PAID:

* Update Order
* Simpan Payment
* Dispatch Queue
* Kirim Email

Webhook harus bersifat idempotent.

Jika Order sudah PAID maka callback berikutnya harus diabaikan.

---

## G. Queue

Semua proses berat dijalankan menggunakan Queue.

Contoh:

* Email
* Notification
* Invoice Processing
* Webhook Processing

---

## H. Retry Mechanism

Apabila webhook gagal diproses:

* Retry maksimal 3x
* Simpan log error

---

## I. Dashboard

Merchant dapat melihat:

* Total Revenue
* Total Order
* Pending Order
* Paid Order
* Failed Order
* Recent Transactions

---

# 5. Database Schema

## merchants

```sql
id
name
slug
plan
status
created_at
updated_at
```

---

## products

```sql
id
merchant_id
name
description
price
status
created_at
updated_at
```

---

## orders

```sql
id
merchant_id
user_id
product_id

external_id

amount

status

payment_method

xendit_invoice_id

paid_at

created_at

updated_at
```

Status:

```
PENDING
PAID
FAILED
EXPIRED
```

---

## payments

Menyimpan hasil pembayaran.

```sql
id

order_id

invoice_id

payment_method

paid_amount

paid_at

callback_payload

created_at
```

---

## webhook_logs

Digunakan untuk debugging callback.

```sql
id

provider

headers

payload

status

created_at
```

---

# 6. Security Requirements

## Session Authentication

Menggunakan Session Laravel.

Tidak menggunakan JWT.

---

## Callback Validation

Seluruh webhook wajib memvalidasi:

```
x-callback-token
```

---

## Server-side Validation

Harga tidak boleh berasal dari frontend.

Frontend hanya mengirim:

```
product_id
```

Backend mengambil harga dari database.

---

## Idempotency

Webhook yang sama boleh datang berkali-kali.

Order hanya boleh berubah menjadi PAID satu kali.

---

## Environment Variables

```env
APP_URL=

APP_ENV=

XENDIT_SECRET_KEY=

XENDIT_CALLBACK_TOKEN=

XENDIT_BASE_URL=
```

---

## Payment Config

Seluruh konfigurasi payment berada pada:

```
config/payment.php
```

Controller tidak boleh membaca .env secara langsung.

---

# 7. Folder Structure

```text
app/

    Http/

    Services/

        Payment/

            XenditService.php

        Order/

            OrderService.php

    Jobs/

    Events/

    Listeners/

    Mail/

resources/

    js/

        Pages/

        Components/

        Layouts/

routes/

    web.php

    auth.php

    api.php

    payment.php
```

---

# 8. Payment Workflow

```text
User

↓

Checkout

↓

Laravel

↓

Validate Product

↓

Validate Price

↓

Create Order

↓

Create Xendit Invoice

↓

Redirect User

↓

User Pays

↓

Xendit Callback

↓

Validate Token

↓

Update Order

↓

Save Payment

↓

Dispatch Queue

↓

Send Email

↓

Finish
```

---

# 9. Non Functional Requirements

Deployment:

* Ubuntu Server
* Nginx
* PHP 8.4
* MySQL
* Supervisor (Queue Worker)
* Redis (Optional)

Build Pipeline:

```bash
composer install --no-dev

npm install

npm run build

php artisan migrate --force

php artisan queue:work
```

---

# 10. Development Environment

Saat development lokal wajib menggunakan:

* Ngrok
* Expose

Agar webhook dari Xendit dapat mengakses endpoint lokal.

---

# 11. Future Roadmap

Fitur yang tidak termasuk MVP namun dipersiapkan:

* Subscription Plan
* Multi Payment Gateway
* Midtrans Integration
* Tripay Integration
* Manual Transfer
* Refund Management
* Settlement Report
* Export Excel/PDF
* Role & Permission
* Audit Log
* API untuk Merchant
* Multi Currency
* Multi Language

---

# 12. Success Criteria

MVP dianggap selesai apabila:

* User dapat login
* Merchant dapat membuat produk
* User dapat checkout
* Invoice berhasil dibuat di Xendit
* User diarahkan ke halaman pembayaran
* Webhook berhasil diterima
* Status order otomatis berubah
* Email berhasil dikirim
* Dashboard menampilkan transaksi secara real-time