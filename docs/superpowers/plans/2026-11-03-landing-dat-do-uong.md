# Landing đặt đồ uống Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Website đồ uống: landing, menu có tùy chọn, giỏ, đặt đơn (lấy tại quán / giao tận nơi), thanh toán VNPay hoặc khi nhận, mã giảm giá, trang quản trị dùng chung một tài khoản có chuông, thông báo trình duyệt và email đơn mới.

**Architecture:** Một dự án Next.js App Router. Route Handlers làm backend trên Node.js runtime. Prisma truy cập PostgreSQL (Supabase). Logic tiền, phí, trạng thái, mã giảm giá là hàm thuần trong `lib/`, được kiểm thử riêng. Mọi giá và phí tính ở server.

**Tech Stack:** Next.js (App Router, TypeScript), Prisma, PostgreSQL, Vitest, Google Maps Distance Matrix API, VNPay (sandbox trước), Resend hoặc SMTP (nodemailer), bcryptjs cho mật khẩu.

**Spec:** `docs/superpowers/specs/2026-11-03-landing-dat-do-uong-design.md`

## Global Constraints

- Phí ship = `ceil(km / 3) × 5000` đồng; bậc 3 km; 5.000đ mỗi bậc (lấy từ Settings, mặc định 3 và 5000).
- Bán kính tối đa: trường tùy chọn, `null` = không giới hạn.
- Chỉ tiếng Việt; không có i18n.
- Trạng thái đơn: Mới, Đang pha chế, Sẵn sàng, Đang giao, Hoàn thành, Đã hủy.
- Trạng thái thanh toán: Chưa thanh toán, Đã thanh toán, Đã hoàn tiền.
- Đơn VNPay chưa thanh toán hết hiệu lực sau 30 phút.
- Polling quản trị: 30 giây.
- Không tin giá, phí, số tiền từ client.
- Không có Zalo trong phiên bản này.
- Tiền tệ: số nguyên đồng, không dùng float.

## Review Focus

1. Địa chỉ giao không tìm được (ZERO_RESULTS, chuỗi rỗng, địa chỉ viết tắt) → khách nhập lại, không tạo đơn phí 0. Test: Task 7.
2. Số lượng món 0, âm, số lẻ, hoặc rất lớn → từ chối. Test: Task 8.
3. IPN VNPay đến sau khi chủ quán đã hủy đơn, hoặc đến hai lần → không đổi sai trạng thái, không cộng lượt mã hai lần. Test: Task 10.
4. Giỏ lưu trên trình duyệt chứa món đã xóa hoặc đổi giá → báo cập nhật giỏ, không tạo đơn với giá cũ. Test: Task 8.
5. Polling `since` với nhiều tab quản trị, hoặc đơn đổi trạng thái sau khi đã báo → không kêu chuông lặp cho cùng một đơn. Test: Task 9.

## File Structure

- `app/(khach)/page.tsx`, `app/(khach)/menu/page.tsx`, `app/(khach)/gio-hang/page.tsx`, `app/(khach)/don/[ma]/page.tsx`: trang khách.
- `app/admin/...`: trang quản trị (đăng nhập, đơn, menu, mã giảm giá, cài đặt).
- `app/api/...`: Route Handlers.
- `lib/shipping.ts`, `lib/pricing.ts`, `lib/coupon.ts`, `lib/order-status.ts`: hàm thuần.
- `lib/db.ts`, `lib/auth.ts`, `lib/distance.ts`, `lib/vnpay.ts`, `lib/mailer.ts`: adapter có I/O.
- `lib/order-service.ts`: tạo đơn trong transaction (gọi pricing, coupon, db).
- `prisma/schema.prisma`, `prisma/seed.ts`.
- `middleware.ts`: bảo vệ `/admin/*` và `/api/admin/*`.
- `tests/unit/*.test.ts`, `tests/integration/*.test.ts`.
- `.env.example`.

---

### Task 1: Khởi tạo dự án

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.ts`, `vitest.config.ts`, `.env.example`, `.gitignore`
- Test: `tests/unit/smoke.test.ts`

**Interfaces:**
- Consumes: không có
- Produces: lệnh `npm test` (Vitest), `npm run dev`, alias `@/` trỏ về thư mục gốc.

- [ ] **Step 1: Khởi tạo git và Next.js**

Chạy `git init` trong `E:\pun.caph` (chỉ local, không push). Chạy `npx create-next-app@latest . --ts --app --no-tailwind --eslint --src-dir=false --import-alias "@/*"`. Lý do: thư mục đang có `docs/`, nên dùng `--yes` nếu được hỏi về thư mục không rỗng.

- [ ] **Step 2: Cài Prisma, Vitest, bcryptjs**

Chạy `npm i @prisma/client bcryptjs nodemailer` và `npm i -D prisma vitest tsx @types/bcryptjs @types/nodemailer`.

- [ ] **Step 3: Viết smoke test**

`tests/unit/smoke.test.ts` với một `expect(1 + 1).toBe(2)`. Chạy `npx vitest run`. Kỳ vọng: PASS.

- [ ] **Step 4: Tạo `.env.example`**

Các biến: `DATABASE_URL`, `DIRECT_URL`, `ADMIN_PASSWORD_HASH`, `SESSION_SECRET`, `GOOGLE_MAPS_API_KEY`, `VNPAY_TMN_CODE`, `VNPAY_HASH_SECRET`, `VNPAY_URL`, `VNPAY_RETURN_URL`, `VNPAY_IPN_URL`, `MAIL_PROVIDER` (`resend` | `smtp`), `RESEND_API_KEY`, `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM`, `NOTIFY_EMAIL`. Mỗi biến có một dòng chú thích. Không ghi giá trị thật.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore: khởi tạo dự án Next.js và test"
```

Co-Authored-By: Claude Haiku 5.5 <noreply@anthropic.com>

---

### Task 2: Hàm thuần cho phí ship và trạng thái đơn

**Files:**
- Create: `lib/shipping.ts`, `lib/order-status.ts`
- Test: `tests/unit/shipping.test.ts`, `tests/unit/order-status.test.ts`

**Interfaces:**
- Consumes: không có
- Produces:
  - `shippingFee(km: number, opts?: { bandKm?: number; feePerBand?: number; maxKm?: number | null }): { fee: number; allowed: boolean }`
  - `canTransition(type: 'pickup' | 'delivery', from: OrderStatus, to: OrderStatus): boolean`
  - `OrderStatus = 'MOI' | 'DANG_PHA_CHE' | 'SAN_SANG' | 'DANG_GIAO' | 'HOAN_THANH' | 'DA_HUY'`

- [ ] **Step 1: Viết test phí ship**

Test theo đúng ví dụ spec: `shippingFee(0)` → fee 5000. `shippingFee(3)` → 5000. `shippingFee(3.2)` → 10000. `shippingFee(9)` → 15000. `shippingFee(9.1)` → 20000. Với `maxKm: 9`, `shippingFee(9.1)` → `allowed: false`. Với `maxKm: null`, `shippingFee(50)` → `allowed: true`.

- [ ] **Step 2: Chạy test, kỳ vọng FAIL** (`npx vitest run tests/unit/shipping.test.ts`)

- [ ] **Step 3: Implement `shippingFee`** trong `lib/shipping.ts`. Dùng `Math.ceil(km / bandKm) * feePerBand`. Với `km = 0`, trả về một bậc (5000). Kiểm tra `maxKm` để set `allowed`. Mặc định `bandKm=3`, `feePerBand=5000`.

- [ ] **Step 4: Viết test chuyển trạng thái** cho mọi cặp hợp lệ và không hợp lệ trong spec mục 5. Ví dụ: pickup `MOI→DANG_PHA_CHE` true; pickup `SAN_SANG→DANG_GIAO` false; delivery `SAN_SANG→HOAN_THANH` false; delivery `DANG_GIAO→HOAN_THANH` true; `MOI→DA_HUY` true; `HOAN_THANH→DA_HUY` false; `DA_HUY→MOI` false.

- [ ] **Step 5: Implement `canTransition`** bằng bảng map các cạnh hợp lệ theo `type`.

- [ ] **Step 6: Chạy toàn bộ unit test, kỳ vọng PASS. Commit.**

---

### Task 3: Hàm thuần cho giá món và mã giảm giá

**Files:**
- Create: `lib/pricing.ts`, `lib/coupon.ts`
- Test: `tests/unit/pricing.test.ts`, `tests/unit/coupon.test.ts`

**Interfaces:**
- Consumes: không có (nhận dữ liệu đã nạp từ DB, kiểu định nghĩa trong file)
- Produces:
  - `priceLine(input: { basePrice: number; quantity: number; choices: { delta: number }[]; groups: { min: number; max: number; chosen: number }[] }): { unitPrice: number; lineTotal: number; valid: boolean; error?: string }`
  - `applyCoupon(coupon: CouponRule, subtotal: number, usage: { totalUsed: number; phoneUsed: number }, now: Date): { ok: true; discount: number } | { ok: false; reason: string }`
  - `CouponRule` gồm `type: 'PERCENT' | 'FIXED'`, `value`, `maxDiscount: number | null`, `minOrder: number | null`, `totalLimit: number | null`, `perPhoneLimit: number | null`, `startsAt: Date | null`, `endsAt: Date | null`, `active: boolean`.

- [ ] **Step 1: Test giá món.** Món 35000, size M +5000, topping +7000, số lượng 2 → `unitPrice` 47000, `lineTotal` 94000. Chọn 2 size khi `max=1` → `valid: false`. Số lượng 0 hoặc âm hoặc số thập phân → `valid: false`.

- [ ] **Step 2: Test mã giảm giá.** PERCENT 10%, `maxDiscount` 20000, subtotal 300000 → discount 20000 (bị trần). FIXED 10000, `minOrder` 50000, subtotal 40000 → lỗi. Hết `totalLimit` → lỗi. `phoneUsed >= perPhoneLimit` → lỗi. Ngoài `startsAt`/`endsAt` → lỗi. `active: false` → lỗi. `discount` không vượt subtotal.

- [ ] **Step 3: Chạy test, kỳ vọng FAIL.**

- [ ] **Step 4: Implement `priceLine` và `applyCoupon`.** Số tiền làm tròn về số nguyên đồng bằng `Math.round` sau phép nhân phần trăm. Trả `reason` tiếng Việt cho từng lỗi.

- [ ] **Step 5: Chạy test, kỳ vọng PASS. Commit.**

---

### Task 4: Schema Prisma và seed

**Files:**
- Create: `prisma/schema.prisma`, `prisma/seed.ts`
- Test: `tests/integration/schema.test.ts`

**Interfaces:**
- Consumes: enum từ Task 2 (`OrderStatus`, phương thức thanh toán, trạng thái thanh toán dùng cùng tên chuỗi).
- Produces: model Prisma đúng spec mục 4: `Category`, `Product`, `OptionGroup`, `OptionChoice`, `Order`, `OrderItem`, `OrderItemOption`, `Payment`, `Coupon`, `CouponUsage`, `Settings` (một bản ghi, `id = 1`), `AdminAccount` (một bản ghi, có `failedLogins Int`, `lockedUntil DateTime?`).

- [ ] **Step 1: Viết schema** theo spec mục 4. Dùng enum Prisma: `OrderStatus`, `FulfillmentType` (PICKUP, DELIVERY), `PaymentMethod` (VNPAY, COD), `PaymentStatus` (UNPAID, PAID, REFUNDED), `CouponType` (PERCENT, FIXED), `PaymentTxStatus` (PENDING, SUCCESS, FAILED). Tiền dùng `Int`. Mã đơn có `unique`. `CouponUsage` có `@@index([couponId, phone])`.

- [ ] **Step 2: Chạy `npx prisma migrate dev --name init`** với database cục bộ (Postgres cục bộ hoặc Docker). Nếu chưa có database, dừng và báo chủ dự án, vì không chặn code.

- [ ] **Step 3: Test tích hợp schema:** tạo một `Settings` với `id=1` và đọc lại, kiểm `shipBandKm = 3`, `shipFeePerBand = 5000`, `maxDeliveryKm = null`.

- [ ] **Step 4: Viết `prisma/seed.ts`** tạo Settings mặc định và một `AdminAccount` từ `ADMIN_PASSWORD_HASH`. Chạy `npx prisma db seed`. Kỳ vọng: không lỗi, đọc được bản ghi.

- [ ] **Step 5: Commit.**

---

### Task 5: Đăng nhập quản trị và middleware

**Files:**
- Create: `lib/auth.ts`, `middleware.ts`, `app/api/admin/login/route.ts`, `app/api/admin/logout/route.ts`, `app/admin/dang-nhap/page.tsx`
- Test: `tests/integration/auth.test.ts`

**Interfaces:**
- Consumes: `AdminAccount` từ Task 4, `lib/db.ts` (Prisma client, tạo ở task này).
- Produces:
  - `createSession(): Promise<void>` (set cookie `httpOnly`, `secure` khi production, `sameSite: 'lax'`)
  - `requireAdmin(req: Request): Promise<boolean>`
  - `verifyPassword(input: string): Promise<{ ok: true } | { ok: false; reason: 'WRONG' | 'LOCKED' }>`
- Ngưỡng khóa: 5 lần sai liên tiếp, khóa 15 phút.

- [ ] **Step 1: Test đăng nhập:** sai mật khẩu → `{ok:false, reason:'WRONG'}`; đúng → ok; sau 5 lần sai → `LOCKED` ngay cả khi nhập đúng; sau khi hết `lockedUntil` → đăng nhập được. Thông báo lỗi trả về không tiết lộ tài khoản hay mật khẩu sai.

- [ ] **Step 2: Chạy test, kỳ vọng FAIL. Implement** `verifyPassword` dùng bcryptjs, cập nhật `failedLogins` và `lockedUntil` trong DB.

- [ ] **Step 3: Test middleware:** request tới `/admin` và `/api/admin/orders` không có cookie → chuyển hướng đăng nhập (trang) hoặc 401 (API). Có cookie hợp lệ → đi tiếp.

- [ ] **Step 4: Implement `middleware.ts`** với `matcher: ['/admin/:path*', '/api/admin/:path*']`, trừ `/admin/dang-nhap` và `/api/admin/login`.

- [ ] **Step 5: Implement route login/logout và trang đăng nhập.** Chạy test, kỳ vọng PASS. Commit.

---

### Task 6: API menu và quản trị menu

**Files:**
- Create: `app/api/menu/route.ts`, `app/api/admin/categories/route.ts`, `app/api/admin/products/route.ts`, `app/api/admin/products/[id]/route.ts`, `app/api/admin/option-groups/route.ts`, `app/admin/menu/page.tsx`
- Test: `tests/integration/menu.test.ts`

**Interfaces:**
- Consumes: `lib/db.ts`, `requireAdmin` từ Task 5.
- Produces: `GET /api/menu` trả `{ categories: [{ id, name, products: [{ id, name, description, imageUrl, price, optionGroups: [{ id, name, min, max, choices: [{ id, name, delta }] }] }] }] }`, chỉ món đang bán và nhóm đang bật.

- [ ] **Step 1: Test:** `GET /api/menu` không trả món `active=false`; không trả nhóm tùy chọn ẩn. Admin `POST /api/admin/products` không có cookie → 401.

- [ ] **Step 2: Chạy test, kỳ vọng FAIL. Implement** các route CRUD, kiểm tra `price >= 0` và số nguyên ở server.

- [ ] **Step 3: Implement trang `/admin/menu`:** form nhóm, món, tùy chọn (min, max, lựa chọn có mức cộng thêm). Ảnh: nhập URL ảnh (không upload trong phiên bản này, ghi vào Global Constraints nếu chủ dự án muốn upload sau).

- [ ] **Step 4: Chạy test, PASS. Commit.**

---

### Task 7: Cài đặt quán và tính khoảng cách

**Files:**
- Create: `lib/distance.ts`, `app/api/admin/settings/route.ts`, `app/admin/cai-dat/page.tsx`, `app/api/shipping/quote/route.ts`
- Test: `tests/unit/distance.test.ts`, `tests/integration/quote.test.ts`

**Interfaces:**
- Consumes: `shippingFee` từ Task 2, Settings từ Task 4, `priceLine` và `applyCoupon` từ Task 3.
- Produces:
  - `distanceKm(origin: string, destination: string): Promise<{ ok: true; km: number } | { ok: false; reason: 'NOT_FOUND' | 'API_ERROR' }>`. Gọi Distance Matrix với `mode=driving`, `language=vi`, `region=vn`. Đổi mét sang km.
  - Quote API trả `{ subtotal, shippingFee, discount, total, km }` hoặc lỗi `{ code, message }`.

- [ ] **Step 1: Test `distanceKm`** với `fetch` được giả lập: `status: OK` và element `OK` → km đúng; element `ZERO_RESULTS` → `NOT_FOUND`; HTTP 500 hoặc `REQUEST_DENIED` → `API_ERROR`. Không gọi API thật trong test.

- [ ] **Step 2: Test quote:** giao tận nơi nhưng `Settings.address` rỗng → lỗi `NO_ORIGIN`, chỉ cho lấy tại quán. Địa chỉ không tìm được → lỗi `ADDRESS_NOT_FOUND`, không trả phí 0. `maxDeliveryKm` đặt 9 và khoảng cách 9.1 → lỗi `OUT_OF_RANGE`.

- [ ] **Step 3: Chạy test, FAIL. Implement** `distanceKm`, quote route (gọi `distanceKm` với địa chỉ quán làm gốc, `shippingFee`, `applyCoupon` nếu có mã). Tọa độ quán được lưu sau khi chủ quán nhập địa chỉ (geocode một lần qua Distance Matrix hoặc để chủ nhập). Ghi rõ lựa chọn này trong Commit message.

- [ ] **Step 4: Implement trang `/admin/cai-dat`:** địa chỉ quán, giờ mở cửa (không bắt buộc), bán kính tối đa (không bắt buộc), email nhận thông báo, bậc km và phí mỗi bậc.

- [ ] **Step 5: Chạy test, PASS. Commit.**

---

### Task 8: Tạo đơn (khách) với mã giảm giá và thanh toán khi nhận

**Files:**
- Create: `lib/order-service.ts`, `app/api/orders/route.ts`, `app/api/orders/[ma]/route.ts`, `lib/order-code.ts`
- Test: `tests/integration/create-order.test.ts`

**Interfaces:**
- Consumes: `priceLine`, `applyCoupon` (Task 3); `shippingFee` (Task 2); quote logic (Task 7); `distanceKm` (Task 7).
- Produces:
  - `createOrder(input: CreateOrderInput): Promise<{ ok: true; code: string; status: OrderStatus; paymentStatus: PaymentStatus } | { ok: false; code: string; message: string }>`
  - `CreateOrderInput`: `{ items: { productId; quantity; options: { choiceId }[]; note? }[]; fulfillment: 'PICKUP' | 'DELIVERY'; address?: string; customerName: string; phone: string; couponCode?: string; method: 'COD' | 'VNPAY' }`
  - Mã đơn: `PC-` + 4 chữ số, đếm theo ngày, reset mỗi ngày.

- [ ] **Step 1: Test tạo đơn COD:** đơn vào `MOI`, `paymentStatus=UNPAID`, tổng = tạm tính + ship − giảm giá, tên món được sao chép. Sửa giá món sau đó, đơn cũ không đổi.

- [ ] **Step 2: Test từ chối:** số lượng 0, số lẻ, món ngừng bán, `choiceId` không thuộc món, giỏ có món đã xóa → lỗi `CART_STALE` với thông báo "Giỏ hàng đã thay đổi, vui lòng cập nhật".

- [ ] **Step 3: Test mã giảm giá:** đơn COD dùng mã → `CouponUsage` được tạo, `Coupon.usedCount` tăng 1, trong cùng transaction. Hai đơn đồng thời cùng mã hết lượt (`totalLimit=1`) → chỉ một đơn thành công.

- [ ] **Step 4: Chạy test, FAIL. Implement** `createOrder` trong transaction Prisma với điều kiện cập nhật `usedCount` (`where usedCount < totalLimit`). Implement mã đơn tăng theo ngày bằng bộ đếm trong transaction.

- [ ] **Step 5: Implement route `POST /api/orders` và `GET /api/orders/[ma]`** (GET chỉ trả thông tin đơn và trạng thái thanh toán, không trả dữ liệu nhạy cảm). Chạy test, PASS. Commit.

---

### Task 9: Quản trị đơn: danh sách, polling, đổi trạng thái, chuông

**Files:**
- Create: `app/api/admin/orders/route.ts`, `app/api/admin/orders/[id]/status/route.ts`, `app/api/admin/orders/[id]/payment-status/route.ts`, `app/admin/page.tsx`, `app/admin/OrdersLive.tsx`, `lib/notify-client.ts`
- Test: `tests/integration/admin-orders.test.ts`

**Interfaces:**
- Consumes: `canTransition` (Task 2), `requireAdmin` (Task 5).
- Produces:
  - `GET /api/admin/orders?since=<ISO>&status=<csv>` trả `{ orders, serverTime }`. Có cờ `isNew` cho đơn có `enteredMoiAt > since`.
  - `PATCH .../status` body `{ to: OrderStatus }`; lỗi 409 nếu không hợp lệ.
  - `PATCH .../payment-status` body `{ to: PaymentStatus }`. Chỉ cho `UNPAID→PAID` khi đơn COD, và `PAID→REFUNDED` khi đơn VNPay. Đơn VNPay không được chuyển sang `PAID` từ quản trị.
  - Đơn có trường `enteredMoiAt`.

- [ ] **Step 1: Test:** chuyển trạng thái hợp lệ → 200; không hợp lệ → 409 và trạng thái giữ nguyên. Đổi `PAID` cho đơn VNPay từ quản trị → 409. Polling `since` trả đơn mới, không trả lại đơn cũ.

- [ ] **Step 2: Chạy test, FAIL. Implement** các route.

- [ ] **Step 3: Implement `OrdersLive.tsx`:** polling 30 giây. Chuông (file âm thanh trong `public/`), thông báo trình duyệt chỉ khi đã cấp quyền. Một lần bấm "Bật âm thanh" để mở khóa âm thanh. Lưu tập `notifiedIds` trong bộ nhớ để không báo lặp. Khi polling lỗi: hiện cảnh báo và thử lại.

- [ ] **Step 4: Implement `app/admin/page.tsx`:** lọc theo trạng thái, nút đổi trạng thái (chỉ hiện bước hợp lệ tiếp theo, theo `canTransition`), nút đổi trạng thái thanh toán.

- [ ] **Step 5: Chạy test, PASS. Kiểm thủ công trên HTTPS** (có thể qua Vercel preview hoặc `next dev` với tunnel): chuông, thông báo, và cảnh báo khi mất mạng. Commit.

---

### Task 10: Thanh toán VNPay, IPN và hết hạn đơn chờ

**Files:**
- Create: `lib/vnpay.ts`, `app/api/payments/vnpay/ipn/route.ts`, `app/api/payments/vnpay/return/route.ts`, `app/api/cron/expire-pending/route.ts`, `vercel.json`
- Test: `tests/unit/vnpay.test.ts`, `tests/integration/vnpay-flow.test.ts`

**Interfaces:**
- Consumes: `createOrder` (Task 8), `Payment` model (Task 4), `canTransition`.
- Produces:
  - `buildPaymentUrl(params: { orderCode: string; amount: number; ipAddr: string; returnUrl: string; createdAt: Date }): string` (ký HMAC-SHA512 theo tài liệu VNPay; `vnp_TxnRef` = mã đơn).
  - `verifyIpn(query: Record<string, string>): { valid: boolean }`.
  - IPN xử lý: nếu chữ ký sai → ghi log, trả `RspCode 97`. Nếu số tiền khác tổng đơn → `RspCode 04`. Nếu đơn đã `PAID` (trùng) → `RspCode 02`, không đổi gì. Nếu thành công → đơn `Chưa thanh toán → Đã thanh toán`, vào `MOI`, ghi `CouponUsage` và tăng `usedCount` trong transaction.
  - Đơn VNPay không có IPN thành công sau 30 phút → cron chuyển sang `DA_HUY` (trạng thái thanh toán giữ `UNPAID`).

- [ ] **Step 1: Test `buildPaymentUrl` và `verifyIpn`** bằng bộ dữ liệu mẫu có chữ ký đã tính sẵn từ tài liệu VNPay. Chữ ký bị sửa một ký tự → `valid: false`.

- [ ] **Step 2: Test IPN:** thành công lần đầu → `MOI`. Gọi lại cùng `vnp_TransactionNo` → không đổi, không cộng lượt mã giảm giá thêm. Đơn đã `DA_HUY` mà IPN thành công đến → không chuyển sang `MOI`, ghi log để chủ quán xử lý hoàn tiền thủ công, đánh dấu `REFUNDED` theo quy trình thủ công (Payment ghi "cần hoàn").
- Test: số tiền IPN khác → không đổi đơn.

- [ ] **Step 3: Test cron:** đơn VNPay `UNPAID` quá 30 phút → `DA_HUY`; đơn còn trong 30 phút không bị đụng.

- [ ] **Step 4: Chạy test, FAIL. Implement** `vnpay.ts`, IPN, return (chỉ đọc kết quả để hiển thị, không cập nhật đơn), cron. Cấu hình `vercel.json` gọi cron mỗi 5 phút. Kiểm tra `CRON_SECRET` header.

- [ ] **Step 5: Chạy test, PASS. Kiểm thử thủ công trên VNPay sandbox** (cần khóa sandbox của chủ dự án, ghi là bước chờ). Commit.

---

### Task 11: Email đơn mới và cảnh báo lỗi gửi

**Files:**
- Create: `lib/mailer.ts`, `app/api/admin/alerts/route.ts`
- Modify: `lib/order-service.ts` (gọi mailer khi đơn vào `MOI`), `lib/order-service.ts` và IPN (cùng điểm chuyển sang `MOI`)
- Test: `tests/unit/mailer.test.ts`, `tests/integration/notify.test.ts`

**Interfaces:**
- Consumes: Settings.notifyEmail (Task 7), `createOrder` (Task 8), IPN (Task 10).
- Produces:
  - `sendNewOrderEmail(order: OrderSummary): Promise<{ ok: true } | { ok: false; error: string }>`. Chọn Resend hoặc SMTP theo `MAIL_PROVIDER`.
  - Bảng `MailFailure` (id, orderId, error, createdAt, resolved) hoặc cột `emailFailed` trên `Order`. Chọn một: đề xuất cột `emailFailed Boolean` và `emailError String?` trên Order, sửa spec nếu cần.
  - `GET /api/admin/alerts` trả đơn có `emailFailed=true`.

- [ ] **Step 1: Test:** gửi thành công → không ghi lỗi; gửi lỗi (giả lập transport) → đơn vẫn tạo, `emailFailed=true`, endpoint alerts trả đơn đó. Đơn VNPay chưa thanh toán không gọi mailer.

- [ ] **Step 2: Chạy test, FAIL. Implement** `mailer.ts` và cờ lỗi. Gọi mailer sau transaction (không trong transaction) để lỗi email không làm rollback đơn.

- [ ] **Step 3: Hiển thị cảnh báo** trên `/admin` (banner đỏ kèm mã đơn). Chạy test, PASS. Commit.

---

### Task 12: Quản trị mã giảm giá

**Files:**
- Create: `app/api/admin/coupons/route.ts`, `app/api/admin/coupons/[id]/route.ts`, `app/admin/ma-giam-gia/page.tsx`
- Test: `tests/integration/coupons-admin.test.ts`

**Interfaces:**
- Consumes: `Coupon` (Task 4), `requireAdmin` (Task 5).
- Produces: CRUD với các trường `code`, `type`, `value`, `maxDiscount`, `minOrder`, `totalLimit`, `perPhoneLimit`, `startsAt`, `endsAt`, `active`. Mã được chuyển thành chữ hoa khi lưu, duy nhất.

- [ ] **Step 1: Test:** tạo mã trùng → 409; `type=PERCENT` không có `value` hoặc `value > 100` → 400; `type=FIXED` không có `minOrder` → cho phép (null = không yêu cầu); `startsAt > endsAt` → 400.

- [ ] **Step 2: Chạy test, FAIL. Implement** routes và trang. Chạy test, PASS. Commit.

---

### Task 13: Trang khách: landing, menu, giỏ, xác nhận đơn

**Files:**
- Create: `app/(khach)/layout.tsx`, `app/(khach)/page.tsx`, `app/(khach)/menu/page.tsx`, `app/(khach)/gio-hang/page.tsx`, `app/(khach)/don/[ma]/page.tsx`, `lib/cart.ts`, `components/OptionPicker.tsx`
- Test: `tests/unit/cart.test.ts`

**Interfaces:**
- Consumes: `GET /api/menu` (Task 6), quote và orders API (Tasks 7, 8, 10).
- Produces: `lib/cart.ts` với `addItem`, `removeItem`, `setQuantity`, `loadCart`, `saveCart` (localStorage, khóa `cart.v1`). Giỏ lưu `{ productId, quantity, options: choiceId[], note }`, không lưu giá.

- [ ] **Step 1: Test `cart.ts`:** thêm món trùng cùng tùy chọn gộp số lượng; số lượng không vượt 99; `loadCart` với JSON hỏng → giỏ rỗng, không crash.

- [ ] **Step 2: Chạy test, FAIL. Implement** `cart.ts`, PASS.

- [ ] **Step 3: Implement trang.** Landing: tên thương hiệu, giới thiệu, nút xem menu. Menu: nhóm, món, popup chọn tùy chọn (`OptionPicker` kiểm min/max phía client để gợi ý, server vẫn kiểm). Giỏ: chọn hình thức nhận; nếu chưa có địa chỉ quán thì chỉ hiện lấy tại quán; nhập địa chỉ khi giao, gọi quote để hiện phí; nhập mã giảm giá; chọn VNPay hoặc khi nhận. Xác nhận: hiện mã đơn và trạng thái thanh toán, polling `/api/orders/[ma]` cho đến khi đổi trạng thái.

- [ ] **Step 4: Kiểm thủ công trên điện thoại** (checklist trong Task 14). Commit.

---

### Task 14: Kiểm thử đầu-cuối và chuẩn bị triển khai

**Files:**
- Create: `docs/deploy.md`, `tests/e2e/README.md` (checklist thủ công)
- Modify: `.env.example` (nếu có biến mới)

**Interfaces:**
- Consumes: mọi task trước.
- Produces: checklist kiểm thử thủ công và hướng dẫn deploy.

- [ ] **Step 1: Chạy toàn bộ test:** `npx vitest run`. Kỳ vọng: toàn bộ PASS.

- [ ] **Step 2: Chạy `npm run build`.** Kỳ vọng: không lỗi TypeScript.

- [ ] **Step 3: Viết `docs/deploy.md`:** thứ tự tạo tài khoản (Supabase, Google Cloud, VNPay sandbox, Resend, Vercel), biến môi trường, chạy `prisma migrate deploy` và `prisma db seed`, cấu hình IPN URL, cấu hình cron.

- [ ] **Step 4: Viết checklist thủ công** trong `tests/e2e/README.md`: đặt đơn COD lấy tại quán; giao tận nơi với địa chỉ thật; địa chỉ sai; VNPay sandbox thành công và bỏ dở; mã giảm giá hết lượt; chuông và thông báo; email thật; đăng nhập sai 5 lần.

- [ ] **Step 5: Commit.**

---

## Ghi chú cho chủ dự án

- Dự án chưa là git repo. Task 1 khởi tạo git local. Commit sẽ có dòng Co-Authored-By theo yêu cầu.
- Các bước cần tài khoản thật (Supabase, Google Maps, VNPay sandbox, Resend, Vercel) chỉ chặn Task 4 (migrate thật), Task 7 (Distance Matrix thật), Task 10 (VNPay sandbox) và Task 14 (deploy). Phần còn lại dùng giả lập.
- Task 4 cần Postgres cục bộ hoặc project Supabase để chạy migrate.
