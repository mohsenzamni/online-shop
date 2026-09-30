# PLAN.md — Instagram Seller Storefront Builder

## Goal
A web app where each seller registers, gets a public mobile-first storefront at `/{slug}`, lists products, and receives orders via Telegram / WhatsApp / Instagram DM. Target market: Iran (Persian, RTL, Jalali dates, prices in Toman).

## Instructions for Copilot
- Work on ONE small task at a time from the roadmap below. Do not generate the whole app at once.
- Follow the stack, structure and rules in this file. Ask before adding new dependencies.
- After each task: write or update tests, and keep the app runnable.
- Every endpoint that touches shop/product/order data MUST verify the resource belongs to the authenticated seller.

## Stack
- **Backend:** Java 21, Spring Boot 3 (Web, Security, Data JPA, Validation), PostgreSQL, Flyway, BCrypt, JWT, Bucket4j (rate limiting), Thymeleaf (only for the public shop page shell)
- **Frontend:** React (JavaScript, no TypeScript) + Vite, Tailwind CSS, React Router, TanStack Query
- **Storage:** `StorageService` interface; local disk first, later S3-compatible (MinIO)
- **Repo layout:**
  ```
  /backend
  /frontend
  PLAN.md
  ```

## Localization rules
- RTL layout everywhere, Persian font (Vazirmatn), mobile-first
- Persian digits in UI, thousand separators for prices, prices in Toman
- Jalali dates in UI (store UTC in DB)
- Slugs: lowercase latin letters, digits and hyphens only

## Data model
| Table | Fields |
|---|---|
| `seller` | id, email (unique), password_hash, name, mobile (nullable), created_at |
| `shop` | id, seller_id (unique FK), slug (unique), title, bio, telegram, whatsapp, instagram, created_at |
| `product` | id, shop_id, title, description, price, stock, is_active, created_at |
| `product_image` | id, product_id, url, sort_order |
| `orders` | id, shop_id, customer_name, customer_phone, address, note, status (NEW, CONFIRMED, SHIPPED, DONE, CANCELED), created_at |
| `order_item` | id, order_id, product_id, quantity, unit_price |
| `subscription` | id, seller_id, plan (FREE, PRO), starts_at, ends_at |

All schema changes go through Flyway migrations (`V1__init.sql`, ...).

## Auth (no OTP in v1)
- `POST /api/auth/register` — body: email, password, shopTitle, slug. Creates seller + shop in one transaction, returns JWT.
- `POST /api/auth/login` — email + password, returns JWT.
- Password: min 8 chars, stored only as BCrypt hash.
- Reserved slugs (cannot be registered): `admin, api, login, register, dashboard, assets, static, app, help, about`.
- Rate limit login attempts (per IP + per email).
- Later (not v1): email verification, password reset (add soon), Google login, OTP.

## API (v1)
Seller (authenticated):
- `GET/PUT /api/shop` — my shop settings
- `GET/POST /api/products`, `GET/PUT/DELETE /api/products/{id}`
- `POST /api/products/{id}/images`, `DELETE /api/products/{id}/images/{imageId}`
- `GET /api/orders`, `GET /api/orders/{id}`, `PATCH /api/orders/{id}/status`

Public (no auth):
- `GET /api/public/shops/{slug}` — shop + active products
- `POST /api/public/shops/{slug}/orders` — customer places an order

## Public page and link previews (important)
Telegram / WhatsApp link previews need `og:title`, `og:description`, `og:image` in the server-rendered HTML. A pure React SPA cannot provide this.
- Spring serves `GET /{slug}` with a Thymeleaf shell that injects OG tags for that shop, then loads the built React bundle.
- Unknown slug returns 404 page.

## Order flow (v1, no online payment)
1. Customer adds products to a simple cart on the public page.
2. Customer enters name, phone, address, note and submits.
3. Order is saved (status NEW) and the customer sees a button that opens a prefilled message to the seller via Telegram / WhatsApp / Instagram DM (`https://t.me/...`, `https://wa.me/...`).
4. Seller sees the order in the dashboard and updates its status.

## Plans and limits
- FREE: up to 10 active products, app branding shown on the shop page
- PRO (later): unlimited products, no branding, custom domain, analytics
- Enforce limits in the service layer, not only in the UI.

## Roadmap
**Week 1** — Project setup (backend + frontend), Docker Compose for Postgres, Flyway V1, register/login with JWT, shop creation, basic dashboard shell
**Week 2** — Product CRUD, image upload via `StorageService`, dashboard product screens
**Week 3** — Public shop page (mobile, RTL), cart, order creation, message-to-seller buttons, OG tags via Thymeleaf shell
**Week 4** — Orders list + status changes, FREE plan limits, security review (ownership checks), deployment, test with real sellers

## Security checklist
- [ ] Every seller endpoint checks resource ownership
- [ ] BCrypt for passwords, no plaintext anywhere
- [ ] Validation on all request bodies
- [ ] Upload: check content type and size, random file names
- [ ] Rate limiting on login and public order endpoint
- [ ] CORS restricted to the frontend origin
- [ ] Secrets only in environment variables

## Later ideas
Online payment (Iranian gateway), email verification and password reset, custom domain, analytics dashboard, product categories, discount codes, automatic story/post images, theme templates.
