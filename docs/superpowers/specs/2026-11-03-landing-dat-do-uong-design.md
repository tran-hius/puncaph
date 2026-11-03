# Landing page và đặt đồ uống: thiết kế

Ngày: 2026-11-03
Trạng thái: chờ chủ dự án duyệt

## 1. Mục tiêu

Một website giới thiệu thương hiệu đồ uống, cho khách xem menu, đặt trước online (lấy tại quán hoặc giao tận nơi), thanh toán bằng VNPay hoặc trả khi nhận. Chủ quán và nhân viên quản lý đơn và menu qua trang quản trị. Chỉ tiếng Việt.

Thành công: khách đặt được đơn trọn vẹn trên điện thoại; chủ quán không bỏ sót đơn; giá và phí ship luôn do server tính.

## 2. Ràng buộc và quyết định đã chốt

| Hạng mục | Quyết định |
|---|---|
| Ứng dụng | Next.js (App Router) + Node.js, một dự án |
| Database | PostgreSQL trên Supabase, truy cập qua Prisma |
| Triển khai | Vercel (ứng dụng) + Supabase (database) |
| Ngôn ngữ | Chỉ tiếng Việt |
| Quản trị | Một tài khoản dùng chung, session cookie |
| Nhận đơn | Lấy tại quán hoặc giao tận nơi |
| Phí ship | Tính theo bậc 3 km, mỗi bậc 5.000đ: phí = ceil(km / 3) × 5.000đ |
| Khoảng cách | Google Maps Distance Matrix, gọi từ server |
| Bán kính tối đa | Tùy chọn, để trống (chưa giới hạn) |
| Thanh toán | VNPay (tiền về tài khoản MBBank) hoặc trả khi nhận |
| Hoàn tiền | Thủ công: chủ quán hoàn trên VNPay, rồi đổi trạng thái thanh toán |
| Mã giảm giá | Phần trăm (có mức tối đa) hoặc số tiền cố định (có đơn tối thiểu); giới hạn tổng lượt, mỗi SĐT một lần, thời hạn hiệu lực |
| Thông báo đơn mới | Chuông và thông báo trình duyệt trên trang quản trị + email (Resend hoặc SMTP) |
| Zalo | Chưa làm ở giai đoạn đầu |
| Địa chỉ quán, giờ mở cửa | Tùy chọn, chủ quán nhập sau trong trang quản trị |

Khi chưa có địa chỉ quán: không tính được phí ship, chỉ cho khách lấy tại quán.

## 3. Kiến trúc

- **Một dự án Next.js App Router.** Trang khách và trang quản trị trong cùng dự án. Backend là Route Handlers (Node.js runtime).
- **Database:** Supabase PostgreSQL, truy cập bằng Prisma.
- **Phiên quản trị:** session cookie (httpOnly, secure), middleware bảo vệ `/admin/*`.
- **Thanh toán VNPay:** server tạo URL thanh toán; VNPay chuyển khách về trang kết quả và gửi IPN (callback server-to-server). Chỉ IPN đã xác minh chữ ký mới đổi trạng thái thanh toán. Trang kết quả chỉ hiển thị, không tin để cập nhật đơn.
- **Phát hiện đơn mới trên quản trị:** polling `GET /api/admin/orders?since=...` mỗi 30 giây. Khi có đơn mới: phát chuông, hiện thông báo trình duyệt.
- **Email:** khi đơn mới vào trạng thái **Mới** (đơn trả khi nhận, hoặc đơn VNPay đã thanh toán), server gửi email cho chủ quán qua Resend hoặc SMTP.
- **Tính giá và phí:** luôn ở server. Trình duyệt chỉ hiển thị.

## 4. Mô hình dữ liệu

- **Category**: id, tên, thứ tự, đang bật.
- **Product**: id, category, tên, mô tả, ảnh, giá gốc, thứ tự, đang bán.
- **OptionGroup**: id, product, tên (Size, Mức đường, Mức đá, Topping), số lựa chọn tối thiểu, tối đa, thứ tự.
- **OptionChoice**: id, group, tên, mức cộng thêm (đ), đang bán.
- **Order**: id, mã đơn (ví dụ `PC-0153`, tăng theo ngày), trạng thái đơn, hình thức nhận (lấy tại quán / giao tận nơi), phương thức thanh toán (VNPay / trả khi nhận), trạng thái thanh toán, tên khách, SĐT, địa chỉ giao, khoảng cách (km), phí ship, mã giảm giá đã áp dụng, số tiền giảm, tạm tính, tổng tiền, ghi chú, thời điểm tạo.
- **OrderItem**: id, đơn, tên món tại thời điểm đặt, giá gốc, số lượng, ghi chú.
- **OrderItemOption**: id, order item, tên nhóm, tên lựa chọn, mức cộng thêm tại thời điểm đặt.
- **Payment**: id, đơn, mã giao dịch VNPay, số tiền, trạng thái (chờ / thành công / thất bại), dữ liệu IPN thô, thời điểm.
- **Coupon**: id, mã, loại (phần trăm / số tiền), giá trị, mức giảm tối đa (cho phần trăm), đơn tối thiểu (cho số tiền), tổng lượt tối đa, số lượt đã dùng, số lần tối đa mỗi SĐT, ngày bắt đầu, ngày kết thúc, đang bật.
- **CouponUsage**: id, coupon, SĐT, đơn. Dùng để đếm lượt theo SĐT.
- **Settings** (một bản ghi): tên quán, địa chỉ, tọa độ, giờ mở cửa (tùy chọn), bán kính tối đa (tùy chọn), bậc km (3), phí mỗi bậc (5.000), email nhận thông báo.
- **AdminAccount**: một bản ghi, mật khẩu băm (bcrypt/argon2), số lần đăng nhập sai và thời điểm khóa tạm.

Giá, tên món và tùy chọn được sao chép vào đơn lúc đặt. Sửa menu sau này không ảnh hưởng lịch sử đơn.

## 5. Trạng thái

**Trạng thái đơn:**
- Lấy tại quán: Mới → Đang pha chế → Sẵn sàng → Hoàn thành
- Giao tận nơi: Mới → Đang pha chế → Đang giao → Hoàn thành
- Mới hoặc Đang pha chế → Đã hủy

**Trạng thái thanh toán** (tách riêng):
- Trả khi nhận: luôn ở "Chưa thanh toán" cho đến khi nhận hàng; chủ quán đánh dấu "Đã thanh toán" khi thu tiền.
- VNPay: Chưa thanh toán → Đã thanh toán (qua IPN xác minh) → Đã hoàn tiền (chủ quán đánh dấu sau khi hoàn thủ công trên VNPay).

Đơn VNPay chưa thanh toán không vào trạng thái **Mới** và không gửi email/thông báo. Nếu khách bỏ thanh toán, đơn ở "Chưa thanh toán" và tự hết hiệu lực sau 30 phút (không xuất hiện trong danh sách quản trị đang xử lý).

Hủy đơn đã thanh toán VNPay: được phép, trạng thái đơn thành Đã hủy; chủ quán hoàn tiền thủ công và đánh dấu "Đã hoàn tiền".

## 6. Luồng đặt hàng

1. Khách chọn món, tùy chọn, ghi chú; giỏ lưu trên trình duyệt (localStorage).
2. Khách chọn lấy tại quán hoặc giao tận nơi (nhập địa chỉ), nhập tên, SĐT, và có thể nhập mã giảm giá.
3. `POST /api/shipping/quote`: server kiểm tra món còn bán, tính lại giá, gọi Distance Matrix (nếu giao), tính phí, kiểm tra mã giảm giá, trả về tạm tính, phí, giảm giá, tổng.
4. Khách bấm đặt, chọn phương thức thanh toán. `POST /api/orders` server tính lại toàn bộ (không tin số liệu từ client), tạo đơn.
   - Trả khi nhận: đơn vào **Mới** ngay, gửi email, trả mã đơn.
   - VNPay: đơn ở "Chưa thanh toán", server trả URL VNPay. Sau khi thanh toán thành công, IPN xác minh chữ ký và đổi đơn sang **Mới**, gửi email. Trang `/don/[ma]` hiển thị kết quả.
5. Mã giảm giá được ghi nhận (tăng lượt, ghi CouponUsage) khi đơn vào **Mới**, trong cùng transaction.

**Lỗi:**
- Distance Matrix lỗi hoặc không tìm được địa chỉ: yêu cầu nhập lại hoặc chọn lấy tại quán. Không tạo đơn với phí 0.
- Món/tùy chọn ngừng bán: từ chối, yêu cầu cập nhật giỏ.
- Số lượng tùy chọn sai min/max: từ chối.
- Mã giảm giá không hợp lệ (hết lượt, hết hạn, đã dùng bởi SĐT này, chưa đạt đơn tối thiểu): báo lý do cụ thể cho khách.
- Chữ ký IPN sai hoặc số tiền không khớp: bỏ qua, ghi log.
- IPN đến nhiều lần: xử lý idempotent theo mã giao dịch.
- Email lỗi: đơn vẫn tạo; ghi log và hiện cảnh báo trên quản trị để chủ quán biết.
- Đăng nhập sai: thông báo chung; khóa tạm sau số lần sai nhất định.
- Mất kết nối khi polling: quản trị hiện cảnh báo và thử lại.

## 7. Trang và API

**Trang khách:** `/` landing; `/menu`; `/gio-hang` (giỏ, hình thức nhận, địa chỉ, mã giảm giá, phí ship); `/don/[ma]` (xác nhận và trạng thái thanh toán).

**Trang quản trị:** `/admin/dang-nhap`; `/admin` (danh sách đơn, lọc, đổi trạng thái, đổi trạng thái thanh toán, chuông và thông báo); `/admin/menu` (nhóm, món, tùy chọn); `/admin/ma-giam-gia`; `/admin/cai-dat` (quán, giờ, ship, email nhận thông báo).

**API khách:**
- `GET /api/menu`
- `POST /api/shipping/quote`
- `POST /api/orders`
- `GET /api/orders/[ma]`
- `POST /api/payments/vnpay/ipn` (callback VNPay, xác minh chữ ký)
- `GET /api/payments/vnpay/return` (chỉ chuyển hướng hiển thị kết quả)

**API quản trị (cần session):**
- `POST /api/admin/login`, `POST /api/admin/logout`
- `GET /api/admin/orders?since=...`
- `PATCH /api/admin/orders/:id/status`
- `PATCH /api/admin/orders/:id/payment-status`
- `GET/POST/PATCH/DELETE /api/admin/products`, `/categories`, `/option-groups`, `/coupons`
- `PATCH /api/admin/settings`

## 8. Cấu trúc thư mục

```
app/
  (khach)/        landing, menu, giỏ, /don/[ma]
  admin/          đăng nhập, đơn, menu, mã giảm giá, cài đặt
  api/            Route Handlers
lib/
  db.ts           Prisma client
  auth.ts         session, kiểm tra đăng nhập, khóa thử sai
  shipping.ts     tính phí theo bậc km (hàm thuần)
  pricing.ts      giá món + tùy chọn (hàm thuần)
  coupon.ts       tính giảm giá và kiểm tra điều kiện (hàm thuần)
  order-status.ts bảng chuyển trạng thái (hàm thuần)
  distance.ts     gọi Google Distance Matrix
  vnpay.ts        tạo URL, xác minh chữ ký IPN
  mailer.ts       gửi email (Resend/SMTP)
prisma/
  schema.prisma
```

## 9. Kiểm thử

- **Unit (hàm thuần):** `shipping` (0 km, 3 km, 3,2 km, 9 km), `pricing`, `coupon` (phần trăm có trần, số tiền có đơn tối thiểu, hết lượt, hết hạn, quá lượt mỗi SĐT), `order-status` (mọi chuyển đổi hợp lệ và không hợp lệ), `vnpay` (chữ ký đúng/sai).
- **Integration (database test):** tạo đơn trả khi nhận, tạo đơn VNPay, IPN lặp lại (idempotent), IPN sai chữ ký, chuyển trạng thái, cộng lượt mã giảm giá trong transaction.
- **Giả lập:** Google Distance Matrix, VNPay sandbox cho luồng thanh toán, Resend/SMTP cho email.
- **Thủ công trên trình duyệt:** luồng đặt hàng trên điện thoại, chuông và thông báo trình duyệt (cần HTTPS, cần một lần bấm để mở âm thanh), nhận email thật.

## 10. Ngoài phạm vi (phiên bản này)

- Hoàn tiền tự động qua API VNPay.
- Tài khoản khách, tra cứu lịch sử đơn.
- Nhiều tài khoản nhân viên và phân quyền.
- Đa ngôn ngữ.
- Giới hạn bán kính giao hàng (chỉ có trường cấu hình trống).
- Zalo (OA hoặc ZNS). Sẽ xem xét sau khi có quyết định về OA và kiểm chứng quy định hiện hành.
- Ứng dụng di động riêng.
- Thông báo SMS.

## 11. Việc cần chủ dự án chuẩn bị ngoài code

- Đăng ký merchant VNPay, xác nhận tài khoản MBBank nhận tiền, lấy TMN code và hash secret (sandbox trước).
- Google Cloud: bật Distance Matrix API, lấy API key, đặt giới hạn chi phí.
- Resend hoặc SMTP: lấy khóa và địa chỉ gửi.
- Supabase: tạo project, lấy chuỗi kết nối.
- Vercel: nối repo, đặt biến môi trường.
- Địa chỉ quán (để tính ship), giờ mở cửa, email nhận thông báo: nhập sau trong trang quản trị.

## 12. Tự kiểm tra spec

- Không còn TBD trong các mục quyết định.
- Thanh toán, hoàn tiền, mã giảm giá khớp với mục 5 và 6.
- Phí ship khớp với công thức ceil(km / 3) × 5.000đ.
- Phạm vi không mâu thuẫn với các mục đã chốt.
