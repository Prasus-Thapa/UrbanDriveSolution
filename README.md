# Urban Drive Solutions — Project Documentation

> A complete vehicle rental web application built with Java (Jakarta Servlets + JSP) and MySQL.
> This document explains the full system in plain language — no programming experience required.

---

## Table of Contents

1. [What Is Urban Drive Solutions?](#1-what-is-urban-drive-solutions)
2. [Who Uses It?](#2-who-uses-it)
3. [How User Accounts Work](#3-how-user-accounts-work)
4. [The Vehicle Fleet](#4-the-vehicle-fleet)
5. [How Booking Works — Step by Step](#5-how-booking-works--step-by-step)
6. [How Payment Works](#6-how-payment-works)
7. [Cancellation Policy & Refunds](#7-cancellation-policy--refunds)
8. [Automated Background Jobs](#8-automated-background-jobs)
9. [The Admin Panel](#9-the-admin-panel)
10. [Pages in the Application](#10-pages-in-the-application)
11. [Database Structure](#11-database-structure)
12. [Technology Stack](#12-technology-stack)
13. [Project Folder Structure](#13-project-folder-structure)

---

## 1. What Is Urban Drive Solutions?

Urban Drive Solutions is an online vehicle rental platform. Customers can browse a fleet of cars, pick dates, book a vehicle, and pay — all through a website. Admins can manage the fleet, view all bookings, and oversee payments from a separate dashboard.

Think of it like a hotel booking site, but for cars.

---

## 2. Who Uses It?

There are two types of users in the system:

| Role | What they can do |
|------|-----------------|
| **Customer** | Register, browse vehicles, book, pay, cancel, view booking history |
| **Admin** | Add/edit/delete vehicles, view all bookings, view all payments, manage fleet status |

An admin account is created directly in the database (not through the public registration page). Every other person who signs up through the website becomes a Customer by default.

---

## 3. How User Accounts Work

### Registration
When a new user signs up, they provide:
- Full name
- Email address (must be unique)
- Password
- Phone number (must be unique)
- Driver's license number (must be unique)

The password is **never stored as plain text**. It is converted into a scrambled string using a security method called BCrypt. Even if someone accessed the database, they could not read the actual passwords.

### Login & Sessions
When a user logs in, the system checks their email and password. If correct, a **session** is created — this is like a temporary ID card that the browser holds while you're logged in. When you log out or close the browser, the session is destroyed.

### Route Protection
Certain pages are protected. If you try to visit your dashboard or booking page without being logged in, an **Auth Filter** automatically redirects you to the login page. This check happens silently on every page request.

---

## 4. The Vehicle Fleet

### Vehicle Information
Each vehicle in the system has:
- Brand and model (e.g., Toyota Corolla)
- Vehicle type (e.g., Sedan, SUV)
- Registration number (unique plate)
- Color
- Number of seats
- Price per day (in Rupees)
- Availability status

### Availability Status
A vehicle can be in one of two states:

| Status | Meaning |
|--------|---------|
| **AVAILABLE** | Visible on the browse page; customers can book it |
| **MAINTENANCE** | Hidden from the browse page; cannot be booked |

Admins set the maintenance status manually when a vehicle needs servicing. The system automatically resets a vehicle back to **AVAILABLE** after its last confirmed booking's return date has passed (explained in [Section 8](#8-automated-background-jobs)).

> **Important note:** A vehicle stays marked AVAILABLE even while it has an active booking. Availability conflicts are checked against the actual booking dates in the database — not just the status label. So a vehicle showing as AVAILABLE on the browse page might still have certain date ranges blocked on its calendar.

---

## 5. How Booking Works — Step by Step

### Step 1 — Browse the Fleet
The customer visits the vehicle list page. Only AVAILABLE vehicles are shown. Vehicles can be filtered or sorted.

### Step 2 — Select Dates
The customer clicks "Book" on a vehicle and is taken to the booking page. A calendar (powered by Flatpickr) lets them choose a pickup date and return date. **Dates that are already booked by other customers are greyed out and cannot be selected.** The system fetches these blocked date ranges from the database in real time.

### Step 3 — Confirm Booking
The system calculates the total cost:

```
Total Cost = Price Per Day × Number of Days
```

The exact amount is shown to the customer before they confirm. Once confirmed, the booking is created with a status of **PENDING**.

### Booking Conflict Prevention
Before saving the booking, the system double-checks that no other PENDING or CONFIRMED booking overlaps with the chosen dates for the same vehicle. This prevents two customers from booking the same car on the same days.

### Booking Statuses

| Status | Meaning |
|--------|---------|
| **PENDING** | Booking created, awaiting payment |
| **CONFIRMED** | Payment received, rental is active |
| **CANCELLED** | Booking was cancelled (by customer or auto-expired) |
| **COMPLETED** | Return date has passed and the rental is finished |

---

## 6. How Payment Works

### Supported Payment Methods
- eSewa
- Khalti
- Card

### The 24-Hour Payment Window
After creating a booking, the customer has **24 hours** to complete payment. If they do not pay within 24 hours, the system automatically cancels the booking at no charge and the payment record is marked CANCELLED. The customer is free to rebook if the vehicle is still available.

A booking remains in **PENDING** status until payment is completed. Once payment is confirmed, the status moves to **CONFIRMED**.

### Payment Record
Every booking has a corresponding payment record that tracks:
- The amount owed
- Which payment method was used
- The payment status
- A transaction code (returned by the payment gateway)

### Payment Statuses

| Status | Meaning |
|--------|---------|
| **PENDING** | Payment not yet made |
| **PAID** | Full payment received |
| **PARTIALLY_REFUNDED** | Cancelled with a partial refund (20% or 50% fee applied) |
| **REFUNDED** | Full refund issued |
| **NON_REFUNDABLE** | Cancelled after pickup date — no refund |
| **CANCELLED** | Booking cancelled before payment was ever made |
| **FAILED** | Payment attempt failed |

---

## 7. Cancellation Policy & Refunds

Customers can cancel a PENDING or CONFIRMED booking. The cancellation fee depends on **how many days before the pickup date the customer cancels**.

### Cancellation Fee Tiers

| When you cancel | Fee | What you get back |
|-----------------|-----|-------------------|
| 2 or more days before pickup | **0%** — No fee | Full refund |
| The day before pickup | **20% fee** | 80% refunded |
| On the pickup date itself | **50% fee** | 50% refunded |
| After the pickup date | **100% — Non-refundable** | No refund |
| PENDING booking (never paid) | **0%** — No fee | Nothing charged (CANCELLED status) |

### How the Refund Logic Works
1. The system looks up the booking and checks who it belongs to.
2. It checks the current booking status (PENDING or CONFIRMED).
3. For PENDING bookings — the payment is simply marked CANCELLED (no money was exchanged).
4. For CONFIRMED bookings — it calculates the days remaining until pickup and applies the correct fee percentage.
5. The booking is marked CANCELLED with the fee amount recorded.
6. The payment record is updated to reflect the refund status.

Refunds are processed back to the original payment method and typically take 3–5 business days depending on the payment provider.

---

## 8. Automated Background Jobs

The system runs two automatic tasks every day at **midnight** (and once on startup in case the server was offline overnight). These tasks require no human action.

### Job 1 — Complete Expired Bookings
Every night, the system scans all CONFIRMED bookings. If a booking's return date has already passed, it:
1. Sets the vehicle's status back to **AVAILABLE** (unless it was set to MAINTENANCE by an admin)
2. Marks the booking as **COMPLETED**

This is done in that exact order intentionally: the vehicle must be reset first, then the booking status changes — otherwise the system would lose track of which vehicle to reset.

### Job 2 — Cancel Unpaid Bookings (24-Hour Expiry)
Every midnight, the system also scans all PENDING bookings. If a booking was created more than 24 hours ago and still has no payment, it:
1. Marks the payment record as **CANCELLED**
2. Marks the booking as **CANCELLED** with a cancellation fee of Rs. 0.00

Again, payments are cancelled first, then bookings — so the system can still identify which payments belong to those pending bookings when cancelling them.

Both jobs are independent — if one fails, the other still runs.

---

## 9. The Admin Panel

Admins have access to a separate dashboard with the following capabilities:

### Vehicle Management
- **Add a vehicle** — fill in all vehicle details; it appears on the browse page immediately
- **Edit a vehicle** — update any detail including price, color, or status
- **Delete a vehicle** — removes it from the fleet
- **Set to Maintenance** — hides the vehicle from customers without deleting it

### Booking Overview
Admins can see all bookings across all customers, including status, dates, amounts, and which vehicle was booked.

### Payment Overview
Admins can view all payment records, transaction codes, payment methods, and statuses.

### Search & Filter
The admin dashboard supports searching and filtering across bookings and vehicles.

---

## 10. Pages in the Application

| Page | Who sees it | Purpose |
|------|-------------|---------|
| Home | Everyone | Landing page with hero section and featured vehicles |
| Vehicle List | Everyone | Browse the full fleet with filters |
| Book Vehicle | Logged-in customers | Select dates and confirm a booking |
| Booking List | Logged-in customers | View all personal bookings and cancel if needed |
| Payment List | Logged-in customers | View payment history and transaction records |
| Dashboard | Logged-in customers | Personal overview with booking and payment summaries |
| Admin Dashboard | Admins only | Manage fleet, view all bookings and payments |
| Add Vehicle | Admins only | Form to add a new vehicle to the fleet |
| Edit Vehicle | Admins only | Form to update vehicle details |
| Login | Everyone | Sign in to an existing account |
| Register | Everyone | Create a new customer account |
| Policy | Everyone | Rental and cancellation policy details |
| Terms | Everyone | Full terms and conditions |
| Contact | Everyone | Contact form and support information |
| About | Everyone | Company information |

---

## 11. Database Structure

The system uses a MySQL database called `vehicle_rental_db` with four main tables.

### users
Stores account information for every registered user.

| Column | What it stores |
|--------|---------------|
| user_id | A unique number identifying each user |
| full_name | The user's full name |
| email | Login email (must be unique) |
| password_hash | Scrambled/encrypted password |
| phone | Contact phone number (must be unique) |
| license_number | Driver's license number (must be unique) |
| role | ADMIN or CUSTOMER |
| created_at | When the account was created |

### vehicles
Stores the car fleet.

| Column | What it stores |
|--------|---------------|
| vehicle_id | A unique number for each vehicle |
| brand | Car brand (e.g., Toyota) |
| model | Car model (e.g., Corolla) |
| vehicle_type | Category (Sedan, SUV, etc.) |
| registration_number | License plate (must be unique) |
| color | Vehicle color |
| seats | Number of seats |
| price_per_day | Daily rental rate in Rupees |
| availability_status | AVAILABLE or MAINTENANCE |
| created_at | When the vehicle was added |

### bookings
Stores every booking ever made.

| Column | What it stores |
|--------|---------------|
| booking_id | A unique number for each booking |
| user_id | Which customer made the booking |
| vehicle_id | Which vehicle was booked |
| pickup_date | When the rental starts |
| return_date | When the rental ends |
| total_amount | Total cost (price/day × days) |
| booking_status | PENDING / CONFIRMED / CANCELLED / COMPLETED |
| cancellation_fee | Fee charged if cancelled |
| cancelled_at | When it was cancelled (if applicable) |
| created_at | When the booking was created (used for 24-hour payment window) |

### payments
Stores one payment record per booking.

| Column | What it stores |
|--------|---------------|
| payment_id | A unique number for each payment |
| booking_id | Which booking this payment is for |
| amount | Amount to be paid or refunded |
| payment_method | CARD / ESEWA / KHALTI |
| payment_status | See [Section 6](#6-how-payment-works) for all statuses |
| transaction_code | Reference code from the payment gateway |
| payment_date | When the payment record was created |

---

## 12. Technology Stack

| Layer | Technology | What it does |
|-------|-----------|-------------|
| Backend Language | Java (Jakarta EE) | Handles all server-side logic |
| Web Framework | Servlets + JSP | Processes requests and renders HTML pages |
| Database | MySQL | Stores all data permanently |
| Password Security | BCrypt | Hashes passwords so they can never be read |
| Calendar UI | Flatpickr | The date picker used on the booking page |
| Fonts & Icons | Google Fonts (Space Grotesk) + Material Symbols | Visual design |
| Build Tool | Maven | Manages dependencies and builds the project |
| Server | Apache Tomcat | Hosts and runs the web application |

---

## 13. Project Folder Structure

```
UrbanDriveSolutions/
│
├── sql/
│   ├── schema.sql          ← Creates all database tables
│   └── seed.sql            ← Fills the database with sample vehicles and admin user
│
├── src/main/
│   ├── java/com/vehicle/urbandrivesolutions/
│   │   ├── controller/     ← Servlets: handle page requests and form submissions
│   │   ├── dao/            ← Database Access Objects: all SQL queries live here
│   │   ├── entity/         ← Data models: User, Vehicle, Booking, Payment
│   │   ├── service/        ← Business logic: cancellation fee calculations
│   │   ├── listener/       ← Background scheduler: midnight auto-jobs
│   │   ├── filter/         ← Auth filter: protects pages from unauthenticated access
│   │   └── utils/          ← Helpers: database connection, password hashing
│   │
│   └── webapp/
│       ├── WEB-INF/
│       │   ├── views/      ← JSP pages: the HTML that customers see
│       │   └── templates/  ← Shared components: nav bar, footer
│       └── static/
│           ├── css/        ← Stylesheets
│           ├── js/         ← JavaScript (navigation toggle)
│           └── img/        ← Vehicle images, hero background, video
│
└── pom.xml                 ← Maven build configuration and dependencies
```

---

## Key Business Rules Summary

| Rule | Detail |
|------|--------|
| Payment window | 24 hours from booking creation to pay, or booking auto-cancels |
| Cancellation (2+ days before) | 0% fee — full refund |
| Cancellation (1 day before) | 20% fee — 80% refunded |
| Cancellation (same day) | 50% fee — 50% refunded |
| Cancellation (after pickup) | 100% fee — no refund |
| Vehicle auto-reset | System resets vehicle to AVAILABLE the night after return date |
| Conflict detection | Checked against booking dates, not vehicle status label |
| License required | Valid driver's license must be presented at vehicle pickup |
| Fuel policy | Vehicle collected full, must be returned full |
| Subletting | Prohibited — only the registered customer may drive |

---

*Built by the Urban Drive Solutions development team.*
