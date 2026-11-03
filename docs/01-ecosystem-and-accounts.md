# Chương 1 — Shopify Ecosystem, Accounts & Stores

> Mục tiêu chương: hiểu bạn đang đứng ở đâu trong hệ sinh thái, và chuẩn bị đủ tài khoản để code.

---

## 1.1. Shopify Platform là gì?

### What is it?

Shopify là một **commerce platform dạng SaaS multi-tenant**. Mỗi merchant thuê một "tenant" gọi là **store**
(còn gọi là *shop*), được định danh duy nhất bằng **shop domain**: `my-store.myshopify.com`.

Điểm quan trọng với developer: Shopify **không bán cho bạn một database**. Shopify bán cho merchant một hệ thống
đóng (sản phẩm, đơn hàng, khách hàng, thanh toán, theme), và mở ra cho developer **ba loại cửa**:

1. **APIs** — đọc/ghi dữ liệu store từ server của bạn.
2. **Extension points** — chèn UI/logic của bạn vào bên trong sản phẩm Shopify (admin, storefront, checkout, POS).
3. **Webhooks** — Shopify chủ động gọi server của bạn khi có sự kiện.

> **Mental model quan trọng nhất:** App của bạn là **một web app riêng biệt, chạy trên server riêng của bạn**.
> Shopify không host code backend của bạn. Shopify chỉ *nhúng* (embed) giao diện app của bạn vào Admin
> và cho phép bạn gọi API. Nếu server của bạn chết, app của bạn chết — store vẫn sống bình thường.

### Why is it important?

Newbie thường tưởng "làm Shopify app" giống "làm plugin WordPress" — tức code chạy cùng chỗ với store.
Hoàn toàn không. Hiểu sai điều này dẫn đến hiểu sai về hosting, latency, bảo mật và chi phí vận hành.

### How does it work? — các "surface" của platform

```
                         ┌──────────────────────────────┐
                         │      SHOPIFY (cloud)         │
                         │                              │
   Merchant ───────────► │  ADMIN            ◄──────────┼── App UI (iframe, embedded)
   (staff)               │  (backoffice)                │
                         │                              │
   Customer ───────────► │  ONLINE STORE     ◄──────────┼── Theme App Extension
   (buyer)               │  (storefront/theme)          │
                         │                              │
                         │  CHECKOUT         ◄──────────┼── Checkout UI Extension / Functions
                         │                              │
                         │  CUSTOMER ACCOUNTS◄──────────┼── Customer Account UI Extension
                         │                              │
   Staff in-store ─────► │  POS              ◄──────────┼── POS UI Extension
                         │                              │
                         │  CORE DATA (products,        │
                         │  orders, customers, …)       │
                         └───────┬──────────────▲───────┘
                                 │              │
                     Webhooks    │              │   Admin API (GraphQL)
                     (push)      ▼              │   (pull/push)
                         ┌──────────────────────┴───────┐
                         │   YOUR APP SERVER            │
                         │   Node.js + React Router     │
                         │   + your database            │
                         └──────────────────────────────┘
```

### Key concepts

| Term | Nghĩa |
|---|---|
| **Store / Shop** | Một tenant của Shopify. Định danh bởi `*.myshopify.com` (shop domain) — khác với custom domain khách hàng thấy. |
| **Merchant** | Người/doanh nghiệp sở hữu store. Khách hàng của bạn. |
| **Staff account** | Nhân viên của merchant, có permissions riêng trong Admin. |
| **Customer** | Người mua hàng ở storefront. **Không phải** user của app bạn. |
| **Shopify Admin** | Backoffice tại `admin.shopify.com/store/<store-handle>`. Nơi app embedded hiển thị. |
| **Online Store** | Sales channel mặc định — storefront chạy theme (Liquid). |
| **Theme** | Template storefront, viết bằng **Liquid** + JSON templates (Online Store 2.0). |
| **Sales channel** | Nơi bán hàng: Online Store, POS, Shop app, hoặc app của bạn. |
| **Shopify Plus** | Gói enterprise. Mở thêm tính năng (checkout extensibility nâng cao, B2B, multiple stores, 10x API rate limit). |
| **App Store** | Marketplace public app: https://apps.shopify.com |

### Common mistakes

- ❌ Nhầm **shop domain** (`abc.myshopify.com`) với **custom domain** (`abc.com`). API và auth **luôn dùng shop domain**. Luôn lưu `shop` dưới dạng `*.myshopify.com` trong DB.
- ❌ Nghĩ "customer" là user đăng nhập app → Không. User dùng app là **staff của merchant**.
- ❌ Giả định mọi store đều có đủ tính năng → tính năng phụ thuộc **plan** (Basic/Grow/Advanced/Plus). App phải xử lý khi tính năng không có.

---

## 1.2. Các loại tài khoản bạn sẽ cần

Đây là phần newbie hay rối nhất. Có **ba** thứ khác nhau:

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. PARTNER ACCOUNT          partners.shopify.com                   │
│    - Miễn phí. Dành cho developer/agency.                          │
│    - Nơi TẠO APP, lấy API key/secret, tạo development store,       │
│      submit app review, xem doanh thu.                             │
│    - Một Partner account chứa nhiều app và nhiều dev store.        │
└────────────────────────────────────────────────────────────────────┘
                     │ tạo ra
                     ▼
┌────────────────────────────────────────────────────────────────────┐
│ 2. DEVELOPMENT STORE        <name>.myshopify.com                   │
│    - Store miễn phí để test app.                                   │
│    - Có đầy đủ tính năng, nhưng KHÔNG nhận thanh toán thật         │
│      (dùng Bogus Gateway để test order).                           │
│    - Có thể tạo nhiều store cho nhiều kịch bản test.               │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ 3. MERCHANT / PRODUCTION STORE                                     │
│    - Store trả phí của khách hàng thật.                            │
│    - App của bạn được cài vào đây qua App Store hoặc install link. │
└────────────────────────────────────────────────────────────────────┘
```

Ngoài ra còn **Shopify ID** (tài khoản đăng nhập cá nhân) — một Shopify ID có thể được mời vào nhiều
Partner account và nhiều store. Đừng nhầm "tài khoản đăng nhập" với "Partner organization".

### Development store: các điểm cần biết

| Điểm | Chi tiết |
|---|---|
| Tạo ở đâu | Partner Dashboard → Stores → Add store → Development store |
| Mục đích | `Test an app` (dùng cho bạn) hoặc `Create a store for a client` |
| Giới hạn | Không checkout thật; một số app bên thứ ba bị hạn chế; password-protected storefront mặc định |
| Test order | Bật **Bogus Gateway** trong Settings → Payments để tạo order giả |
| Dữ liệu mẫu | Dùng `shopify app dev` + tự tạo, hoặc cài app "Simple Sample Data" |
| Plus features | Có thể tạo dev store với **Shopify Plus** plan để test B2B/checkout extensibility |

### Practical: checklist khởi động (làm ngay hôm nay)

1. Tạo Partner account: https://partners.shopify.com/signup
2. Partner Dashboard → **Stores** → tạo 1 development store (chọn "Test an app").
3. Tạo 10–20 sản phẩm mẫu trong store đó.
4. Settings → Payments → bật **Bogus Gateway**.
5. Cài Node.js LTS (≥ 20) và Shopify CLI: `npm install -g @shopify/cli@latest`
6. Kiểm tra: `shopify version`

### Common mistakes

- ❌ Tạo app trong **Admin của store** (Settings → Apps → Develop apps) rồi tưởng đó là cách làm public app. Đó là **custom app cấp store**, không submit App Store được. Xem chương 2.
- ❌ Dùng production store của khách để dev → rủi ro làm hỏng dữ liệu thật.
- ❌ Quên rằng dev store **hết hạn/bị đóng băng** nếu không dùng lâu, hoặc bị chuyển sang trial khi transfer cho client.
- ❌ Tưởng app cài trên dev store sẽ "tự có" trên store thật — mỗi store phải cài riêng.

---

## 1.3. Bức tranh business: bạn kiếm tiền thế nào

Hiểu phần này giúp bạn chọn đúng app type ở chương 2.

| Mô hình | Mô tả | Tiền |
|---|---|---|
| **Public app trên App Store** | Ai cũng cài được, listing công khai | Thu phí qua Shopify Billing; Shopify thu revenue share (0% cho phần doanh thu đầu tiên theo chính sách hiện hành, sau đó theo tỉ lệ công bố) |
| **Public app, unlisted** | Public app nhưng không hiện trên App Store, chia sẻ bằng install link | Như trên |
| **Custom app cho 1 merchant** | Làm riêng theo hợp đồng | Merchant trả bạn trực tiếp (không qua Shopify Billing) |
| **Agency / project work** | Build + maintain app cho client | Hợp đồng dịch vụ |

> Revenue share và ngưỡng miễn phí thay đổi theo thời gian — luôn kiểm tra
> https://shopify.dev/docs/apps/launch/billing và trang Partner Program agreement trước khi làm business plan.

---

## 1.4. Tóm tắt chương

- Shopify = SaaS multi-tenant; mỗi merchant là một **store** với **shop domain** `*.myshopify.com`.
- App của bạn **chạy trên server của bạn**, chỉ được *nhúng* vào Shopify.
- Ba loại cửa vào platform: **API**, **Extensions**, **Webhooks**.
- Ba loại tài khoản: **Partner account** (tạo app) → **Development store** (test) → **Production store** (khách thật).
- Luôn lưu và dùng **shop domain**, không dùng custom domain.

**Tiếp theo:** [Chương 2 — App Types & Architecture](02-app-types-and-architecture.md)
