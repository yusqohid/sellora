# Sellora - Product Requirements Document

## 1. Product Overview

**Sellora** is a single-vendor digital product commerce platform for selling downloadable digital products such as:

* Software
* Source code
* Templates
* E-books
* Design assets
* Documents
* Other downloadable digital goods

The platform provides a custom storefront and customer account experience while delegating payment processing to an external payment provider such as LYNK.ID.

Sellora is designed to be **provider-agnostic** so that the payment provider can be replaced without redesigning the core application.

---

## 2. Product Goals

### Primary Goals

1. Provide a professional storefront for digital products.
2. Allow customers to browse and purchase digital products.
3. Allow customers to access their purchased products.
4. Allow administrators to manage products and orders.
5. Integrate with an external payment provider through a well-defined abstraction.
6. Process payment status updates through webhooks.
7. Keep the core application independent from a specific payment provider.
8. Provide a clean foundation for future features.

### Secondary Goals

1. Provide good SEO for public product pages.
2. Provide a responsive user interface.
3. Keep the application simple enough to deploy and maintain as a small project.
4. Demonstrate production-quality Go backend architecture.

---

## 3. Non-Goals

The MVP will NOT implement:

* Multi-vendor marketplace functionality
* Seller accounts
* Multiple stores
* Built-in payment processing
* Banking/payment gateway infrastructure
* Native mobile applications
* Complex recommendation systems
* Real-time chat
* Subscription billing
* Affiliate system
* Advanced tax calculation
* Multi-currency support
* Automated DRM

These may be considered in future versions.

---

# 4. User Roles

## 4.1 Admin

The admin owns and manages the store.

Admin capabilities:

* Login
* Logout
* View dashboard
* Create products
* Edit products
* Publish/unpublish products
* Delete products
* Upload product files
* Manage product images
* View orders
* View order details
* View customers
* View basic sales statistics

## 4.2 Customer

Customers can:

* Register
* Login
* Logout
* Browse products
* Search products
* View product details
* Create an order
* Proceed to payment
* View order history
* View order details
* Download purchased products

A customer cannot manage products or access the admin dashboard.

---

# 5. Core User Flows

## 5.1 Browse Product

```text
Customer
  ?
Homepage
  ?
Product Listing
  ?
Product Detail
```

Product detail should contain:

* Product name
* Description
* Price
* Product image
* File/version information where appropriate
* Purchase button

---

## 5.2 Purchase Product

```text
Customer
  ?
Product Detail
  ?
Buy Now
  ?
Create Order
  ?
External Checkout
  ?
Payment
  ?
Payment Provider
  ?
Webhook
  ?
Sellora
  ?
Order = PAID
  ?
Customer receives access
```

Sellora must not consider an order paid merely because the customer returns from the external checkout page.

The authoritative payment state must come from the provider's verified webhook or another trusted server-to-server mechanism.

---

## 5.3 Customer Download

```text
Customer
  ?
My Orders
  ?
Paid Order
  ?
Download
  ?
Authorization Check
  ?
Secure File Delivery
```

Only customers who are authorized to access a purchased product may download its files.

Product files must not be exposed directly through a publicly guessable URL.

---

# 6. Product Requirements

## 6.1 Product Catalog

Each product should support:

* ID
* Name
* Slug
* Short description
* Full description
* Price
* Thumbnail/image
* Status
* Created timestamp
* Updated timestamp

Product status:

```text
draft
published
archived
```

Only `published` products should be visible in the public storefront.

---

## 6.2 Product Files

A product may contain one or more downloadable files.

Each file should support:

* ID
* Product ID
* Original filename
* Storage path/key
* File size
* MIME type
* Version
* Checksum
* Created timestamp

The database should store file metadata rather than the file binary itself.

---

# 7. Order Requirements

An order should contain:

* ID
* Customer ID
* Order status
* Total amount
* Currency
* Payment provider
* Payment reference
* Created timestamp
* Updated timestamp

Order statuses:

```text
pending
paid
failed
cancelled
refunded
```

Order items should preserve the price at the time of purchase.

A later product price change must not modify historical orders.

---

# 8. Payment Requirements

Payment processing must use an abstraction.

Conceptually:

```go
type Provider interface {
    CreateCheckout(ctx context.Context, order Order) (string, error)
    VerifyWebhook(r *http.Request) (*Event, error)
}
```

The application must not directly depend on LYNK-specific concepts outside the payment integration package.

Example:

```text
payment/
+-- provider.go
+-- lynk.go
+-- webhook.go
```

Future providers may include:

```text
payment/
+-- provider.go
+-- lynk.go
+-- midtrans.go
+-- xendit.go
```

The exact provider implementation must follow the provider's current official API/webhook documentation.

---

# 9. Webhook Requirements

Webhook endpoint:

```text
POST /webhooks/{provider}
```

For LYNK:

```text
POST /webhooks/lynk
```

Webhook processing must:

1. Receive the request.
2. Validate authenticity/signature when supported.
3. Parse the provider event.
4. Validate required fields.
5. Identify the corresponding order.
6. Ensure the event is processed idempotently.
7. Update the order status.
8. Record the payment reference/event ID.
9. Return an appropriate HTTP response.

Repeated webhook delivery must not create duplicate orders or duplicate fulfillment.

---

# 10. Authentication & Authorization

Authentication is required for:

* Customer account
* Admin dashboard

Authorization rules:

```text
Public
+-- Homepage
+-- Product listing
+-- Product detail

Customer
+-- Profile
+-- Orders
+-- Downloads

Admin
+-- Dashboard
+-- Products
+-- Product files
+-- Orders
+-- Customers
```

Admin routes must never be accessible to ordinary customers.

---

# 11. UI Requirements

The frontend will use:

* Server-rendered HTML
* Go `html/template`
* HTMX
* CSS

JavaScript should be used only when HTMX/server-rendered interactions cannot reasonably provide the required behavior.

The UI should prioritize:

* Simplicity
* Accessibility
* Responsive design
* Fast page loads
* Clear purchase flow

---

# 12. HTMX Usage

HTMX should be used for interactions such as:

* Form submission
* Product filtering
* Search
* Pagination
* Admin CRUD interactions
* Flash messages
* Modal/dialog interactions where appropriate

Avoid turning the application into a JavaScript-heavy SPA.

The server remains the source of truth for application state.

---

# 13. SEO

Public pages should be server-rendered and SEO-friendly.

Product pages should include:

* Meaningful `<title>`
* Meta description
* Canonical URL
* Semantic HTML
* Open Graph metadata
* Structured data where appropriate

URLs should be human-readable:

```text
/products/golang-rest-api-boilerplate
```

rather than:

```text
/products?id=123
```

---

# 14. Security Requirements

The application must:

* Hash passwords using a modern password hashing algorithm.
* Never store plaintext passwords.
* Protect authenticated routes.
* Protect admin routes.
* Validate user input.
* Prevent SQL injection through parameterized queries.
* Implement CSRF protection where applicable.
* Validate uploaded files.
* Prevent unauthorized file downloads.
* Avoid exposing secrets in source code.
* Store secrets in environment variables or a secure secret mechanism.
* Verify payment webhooks.
* Make webhook processing idempotent.
* Apply reasonable request limits to sensitive endpoints.

---

# 15. Data Model

Initial entities:

```text
User
Product
ProductFile
Order
OrderItem
PaymentEvent
```

Relationships:

```text
User
 ¦
 +--< Order
          ¦
          +--< OrderItem >-- Product
                                ¦
                                +--< ProductFile
```

`PaymentEvent` stores provider events required for auditing and idempotency.

---

# 16. Suggested Database Tables

```text
users
products
product_files
orders
order_items
payment_events
sessions
```

Potential future tables:

```text
coupons
downloads
licenses
audit_logs
```

These should not be implemented until their requirements are actually needed.

---

# 17. Architecture

Sellora should use a modular monolithic architecture.

```text
Browser
   ¦
   ?
HTTP Router
   ¦
   ?
Handlers
   ¦
   ?
Services
   ¦
   +-- Repository
   ¦      ?
   ¦   PostgreSQL
   ¦
   +-- Payment Provider
          ?
       LYNK.ID
```

The application should remain deployable as a single service.

---

# 18. Initial Project Structure

```text
sellora/
+-- cmd/
¦   +-- server/
¦       +-- main.go
¦
+-- internal/
¦   +-- config/
¦   +-- domain/
¦   +-- product/
¦   +-- order/
¦   +-- customer/
¦   +-- auth/
¦   +-- payment/
¦   +-- http/
¦
+-- web/
¦   +-- templates/
¦   ¦   +-- layouts/
¦   ¦   +-- pages/
¦   ¦   +-- components/
¦   +-- static/
¦
+-- migrations/
¦
+-- .env.example
+-- AGENTS.md
+-- PRD.md
+-- go.mod
+-- README.md
```

---

# 19. MVP Acceptance Criteria

The MVP is considered complete when:

* Admin can log in.
* Admin can create a product.
* Admin can upload a product file.
* Admin can publish/unpublish products.
* Customers can browse published products.
* Customers can register and log in.
* Customers can create an order.
* Customers can be redirected to the configured payment provider.
* Verified payment webhooks can mark an order as paid.
* Webhook processing is idempotent.
* Customers can see paid orders.
* Customers can download purchased products.
* Unauthorized users cannot download products.
* Admin can view orders.
* The application can run locally with documented setup instructions.
* The application can be deployed as a single service.

---

# 20. Future Roadmap

Potential future features:

### Phase 2

* Coupons
* Discount codes
* Product versioning
* License keys
* Download history
* Email notifications
* Sales analytics

### Phase 3

* Multiple payment providers
* Object storage
* CDN
* Automated email delivery
* Advanced analytics
* Customer reviews

### Phase 4

Only if there is a genuine business requirement:

* Multi-vendor sellers
* Seller dashboard
* Marketplace commission
* Seller payouts
* Multiple storefronts

---

# 21. Product Principle

The central architectural principle of Sellora is:

> **Own the storefront, own the product catalog, own the customer experience, but do not unnecessarily own the payment infrastructure.**

The application should be able to replace the payment provider without rewriting the product catalog, order system, or frontend.