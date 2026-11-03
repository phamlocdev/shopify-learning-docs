# Phụ lục A — Vòng đời một Shopify App từ đầu đến cuối

> Chương này gộp mọi kiến thức rời rạc thành **một câu chuyện liền mạch**.
> Đọc sau khi đã qua chương 1–5. Nếu hiểu hết phụ lục này, bạn đã nắm được phần nền tảng.

Kịch bản: app tên **StockGuard** — cảnh báo tồn kho thấp. Public app, embedded, có theme extension và billing.

---

## Giai đoạn 1 — Trước khi có merchant nào

```
Developer (bạn)
   │
   ├─► Tạo app trong Partner Dashboard
   │     → nhận client_id (API key) + client_secret (API secret)
   │
   ├─► Khai báo trong shopify.app.toml:
   │     application_url   = https://stockguard.fly.dev
   │     access_scopes     = "read_products,read_inventory,write_products"
   │     webhooks          = app/uninstalled, inventory_levels/update,
   │                         customers/data_request, customers/redact, shop/redact
   │
   ├─► shopify app deploy
   │     → Shopify lưu config này thành "app version 1"
   │     → extensions (theme app extension) được upload lên hạ tầng Shopify
   │
   └─► Deploy backend lên Fly.io + Postgres
         → https://stockguard.fly.dev đang chạy, DB trống
```

Lúc này Shopify **đã biết** app tồn tại, biết nó cần scope gì, cần webhook gì, UI ở đâu.
Nhưng chưa store nào cài.

---

## Giai đoạn 2 — Merchant cài app

```
┌──────────────┐                ┌───────────────────┐         ┌──────────────────┐
│  MERCHANT    │                │     SHOPIFY       │         │  STOCKGUARD      │
│  (browser)   │                │                   │         │  (your server)   │
└──────┬───────┘                └─────────┬─────────┘         └────────┬─────────┘
       │                                  │                            │
  (1)  │ Bấm "Add app" trên App Store     │                            │
       ├─────────────────────────────────►│                            │
       │                                  │                            │
  (2)  │ Shopify render màn hình consent  │                            │
       │ "StockGuard muốn: xem sản phẩm,  │                            │
       │  quản lý tồn kho"                │                            │
       │◄─────────────────────────────────┤                            │
       │  (nội dung lấy từ access_scopes  │                            │
       │   trong shopify.app.toml)        │                            │
       │                                  │                            │
  (3)  │ Bấm "Install"                    │                            │
       ├─────────────────────────────────►│                            │
       │                                  │ Shopify ghi nhận:          │
       │                                  │  - app installed on shop   │
       │                                  │  - scopes granted          │
       │                                  │  - đăng ký webhook theo    │
       │                                  │    app version hiện tại    │
       │                                  │                            │
  (4)  │ Redirect vào Admin, mở iframe:   │                            │
       │ GET /app?shop=demo.myshopify.com │                            │
       │     &host=<b64>&embedded=1       │                            │
       ├──────────────────────────────────┼───────────────────────────►│
       │                                  │                            │
       │                                  │        (5) Server CHƯA có  │
       │                                  │            token cho shop  │
       │                                  │            này → trả HTML  │
       │                                  │            có App Bridge   │
       │◄─────────────────────────────────┼────────────────────────────┤
       │                                  │                            │
  (6)  │ App Bridge load, sinh SESSION    │                            │
       │ TOKEN (JWT ký bằng API secret,   │                            │
       │ hạn ~60s), gọi lại server        │                            │
       │ Authorization: Bearer <jwt>      │                            │
       ├──────────────────────────────────┼───────────────────────────►│
       │                                  │                            │
       │                                  │        (7) Verify JWT:     │
       │                                  │            signature, aud, │
       │                                  │            dest, exp       │
       │                                  │                            │
       │                                  │◄───────────────────────────┤
       │                                  │ (8) TOKEN EXCHANGE          │
       │                                  │ POST /admin/oauth/access_token
       │                                  │ grant_type=…token-exchange │
       │                                  │ subject_token=<session jwt>│
       │                                  │                            │
       │                                  ├───────────────────────────►│
       │                                  │ { access_token:"shpat_…",  │
       │                                  │   scope:"read_products,…" }│
       │                                  │                            │
       │                                  │        (9) LƯU VÀO DB:     │
       │                                  │            Session{ shop,  │
       │                                  │            accessToken,    │
       │                                  │            scope }         │
       │                                  │                            │
       │                                  │◄───────────────────────────┤
       │                                  │ (10) Gọi Admin API GraphQL │
       │                                  │ X-Shopify-Access-Token:…   │
       │                                  ├───────────────────────────►│
       │                                  │                            │
       │◄─────────────────────────────────┼────────────────────────────┤
  (11) │ Thấy App Home: onboarding wizard │                            │
```

**Điểm mấu chốt:**
- Merchant **không bao giờ** thấy token.
- Access token sinh ra ở bước 8–9, **chỉ tồn tại trong DB của bạn**.
- Từ lần mở app sau, bước 6–9 bị bỏ qua vì DB đã có token (trừ khi token bị revoke).

---

## Giai đoạn 3 — Onboarding (đừng bỏ qua)

```
App Home lần đầu:
  ┌───────────────────────────────────────────────────────┐
  │  Chào mừng đến StockGuard                             │
  │                                                       │
  │  [✓] 1. Kết nối store              (tự động, đã xong) │
  │  [ ] 2. Đặt ngưỡng cảnh báo mặc định  → [Thiết lập]  │
  │  [ ] 3. Bật widget trên theme         → [Mở Theme     │
  │         Editor]  ← deep link:                         │
  │         https://{shop}/admin/themes/current/editor    │
  │             ?context=apps&activateAppId=<uuid>/<handle>│
  │  [ ] 4. Chọn gói                      → [Xem gói]    │
  └───────────────────────────────────────────────────────┘
```

Song song, server tạo bản ghi:
```sql
INSERT INTO shop_config (shop, plan, default_threshold, active)
VALUES ('demo.myshopify.com', 'free', 5, true);
```

> Đây là bước nhiều app bỏ quên và bị App Review nhắc: merchant cài xong không biết làm gì tiếp.

---

## Giai đoạn 4 — App hoạt động hằng ngày

### 4a. Merchant mở app (read path)

```
Browser (iframe) ──session token──► StockGuard server
                                         │
                                         │ tra DB: Session WHERE shop = 'demo…'
                                         │ lấy accessToken
                                         ▼
                                  Admin API GraphQL
                                  POST /admin/api/2026-10/graphql.json
                                  query { productVariants(first: 50,
                                            query: "inventory_quantity:<5") { … } }
                                         │
                                         ▼
                                  render bảng "Sản phẩm sắp hết"
```

### 4b. Tồn kho thay đổi (event path)

```
Khách mua hàng trên storefront
        │
        ▼
Shopify trừ tồn kho
        │
        ▼
Webhook POST /webhooks/inventory_levels/update
   headers: X-Shopify-Hmac-Sha256, X-Shopify-Webhook-Id, X-Shopify-Shop-Domain
        │
        ▼
┌─────────────────────────────────────────────┐
│ StockGuard webhook handler                  │
│ 1. verify HMAC trên raw body                │
│ 2. dedupe theo X-Shopify-Webhook-Id         │
│ 3. enqueue job                              │
│ 4. return 200   ← trong < 5 giây            │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│ WORKER (bất đồng bộ)                        │
│ • đọc shop_config.threshold                 │
│ • nếu available < threshold:                │
│     - ghi alert vào DB                      │
│     - gửi email cho merchant                │
│     - metafieldsSet: product.stockguard.    │
│       low_stock = true  ← để theme đọc      │
└─────────────────────────────────────────────┘
```

### 4c. Buyer xem storefront (extension path)

```
Buyer mở trang sản phẩm
        │
        ▼
Theme render block của theme app extension:
   {% if product.metafields.stockguard.low_stock %}
     <span class="sg-badge">Chỉ còn vài sản phẩm!</span>
   {% endif %}
        │
        └─► KHÔNG gọi server của bạn.
            Dữ liệu đã nằm sẵn trong metafield → nhanh, không tốn hạ tầng.
```

> Đây là ví dụ điển hình của việc **chọn đúng chỗ lưu dữ liệu**: ghi metafield một lần ở worker,
> đọc hàng triệu lần ở storefront mà không tốn một request nào tới server bạn.

### 4d. Job định kỳ (reconciliation)

```
Cron 03:00 hằng ngày, cho từng shop active:
  • Bulk Operation query toàn bộ inventory levels
  • đối chiếu với DB
  • sửa sai lệch do webhook bị mất
  • log số bản ghi lệch (nếu > 0 → cảnh báo nội bộ)
```

Vì webhook là at-least-once và **có thể mất**, job này là thứ phân biệt app nghiệp dư và app production.

---

## Giai đoạn 5 — Merchant nâng cấp gói

```
Merchant bấm "Upgrade to Pro"
        │
        ▼
App gọi billing.request({ plan: "Pro" , returnUrl })
        │
        ▼
Shopify hiển thị trang xác nhận charge ($19/tháng)
  (App Bridge redirect lên top-level, không trong iframe)
        │
        ▼
Merchant approve
        │
        ▼
Shopify redirect về returnUrl
        │
        ▼
App gọi billing.check() → xác nhận subscription ACTIVE
        │
        ▼
Cập nhật shop_config.plan = 'pro' → mở khoá tính năng
        │
        ▼
Webhook app_subscriptions/update → đồng bộ khi merchant huỷ / downgrade / charge fail
```

---

## Giai đoạn 6 — Merchant gỡ app

```
Merchant bấm "Delete app"
        │
        ▼
Shopify:
  • revoke access token NGAY (token cũ thành vô hiệu)
  • huỷ subscription
  • gỡ theme app extension khỏi theme
  • gửi webhook app/uninstalled
        │
        ▼
POST /webhooks/app/uninstalled
        │
        ▼
StockGuard:
  • DELETE FROM session WHERE shop = …        ← bắt buộc
  • UPDATE shop_config SET active = false     ← giữ data phòng khi cài lại
  • dừng mọi cron job cho shop này
        │
        ▼
(48 giờ sau) Shopify gửi shop/redact
        │
        ▼
StockGuard: xoá vĩnh viễn mọi dữ liệu của shop này
```

> ⚠ Nếu không xoá session, app sẽ tiếp tục gọi API với token chết → 401 hàng loạt,
> và bạn đang giữ dữ liệu không được phép giữ.

---

## Giai đoạn 7 — Vận hành & tiến hoá

```
Hằng tuần   • đọc shopify.dev/changelog
            • xem error rate trên Sentry, webhook failure rate
            • xem queue depth

Hằng quý    • API version mới ra (1/1, 1/4, 1/7, 1/10)
            • đọc release notes → tìm breaking change
            • nâng apiVersion trong code + shopify.app.toml
            • test đầy đủ trên staging → deploy
            • `shopify app deploy` để cập nhật webhook api_version

Khi release • deploy backend TRƯỚC (tương thích ngược)
            • `shopify app deploy` SAU (extensions)
            • theo dõi 30 phút, sẵn sàng rollback app version

Khi cần     • thêm scope → sửa .toml → deploy → merchant re-approve khi mở app
              (cân nhắc kỹ: đây là friction, có thể làm merchant rời đi)
```

---

## Sơ đồ tổng hợp — tất cả trong một hình

```
                    ┌────────────────────────────────────────────┐
                    │              SHOPIFY CLOUD                 │
   ┌────────────┐   │                                            │
   │  MERCHANT  ├───┼─► ADMIN ──iframe──┐                        │
   └────────────┘   │                   │                        │
                    │  CORE DATA        │                        │
   ┌────────────┐   │  (products,       │                        │
   │   BUYER    ├───┼─► ONLINE STORE    │                        │
   └────────────┘   │   └─ theme ◄──────┼── theme app extension  │
                    │      └─ metafield │   (deployed to Shopify)│
                    │                   │                        │
                    │  EVENT BUS ───────┼──┐                     │
                    └───────────────────┼──┼─────────────────────┘
                                        │  │
                      ①session token    │  │ ③webhook (HMAC)
                      ②Admin API ◄──────┤  │
                                        ▼  ▼
                    ┌───────────────────────────────────────────┐
                    │          YOUR SERVER (Fly.io)             │
                    │  ┌─────────────┐    ┌──────────────────┐  │
                    │  │ React Router│    │ Webhook receiver │  │
                    │  │ App Home    │    │  → verify → 200  │  │
                    │  └──────┬──────┘    └────────┬─────────┘  │
                    │         │                    ▼            │
                    │         │           ┌──────────────────┐  │
                    │         │           │ Queue → Worker   │  │
                    │         │           └────────┬─────────┘  │
                    │         ▼                    ▼            │
                    │  ┌──────────────────────────────────────┐ │
                    │  │ POSTGRES                             │ │
                    │  │  Session(shop, accessToken, scope)   │ │
                    │  │  ShopConfig(shop, plan, settings)    │ │
                    │  │  Alert(shop, productId, …)           │ │
                    │  │  WebhookLog(webhookId)  ← dedupe     │ │
                    │  └──────────────────────────────────────┘ │
                    └───────────────────────────────────────────┘

   ① Browser → server bạn (chứng minh danh tính)
   ② Server bạn → Shopify (đọc/ghi dữ liệu)
   ③ Shopify → server bạn (thông báo sự kiện)
```

---

## Bài tập tự kiểm tra

Che tài liệu và trả lời:

1. Merchant mở app lần thứ 10. Những bước nào trong giai đoạn 2 **không** chạy lại? Vì sao?
2. Nếu DB của bạn mất toàn bộ bảng `Session`, chuyện gì xảy ra khi merchant mở app? App có hỏng vĩnh viễn không?
3. Webhook `inventory_levels/update` đến **hai lần** cho cùng một sự kiện. Nếu không dedupe, merchant nhận gì?
4. Vì sao badge "Chỉ còn vài sản phẩm" đọc từ metafield thay vì gọi API của bạn?
5. Merchant gỡ app rồi cài lại sau 1 tuần. Dữ liệu cũ của họ còn không? Nên còn hay không? Theo chính sách nào?
6. Bạn thêm scope `write_orders`. Merchant đang dùng app phải làm gì? Ai hiển thị màn hình đó?
7. Store của merchant có 80.000 sản phẩm. Vì sao vòng lặp `products(first: 250)` là lựa chọn sai?

Trả lời được cả 7 câu mà không mở tài liệu = bạn đã sẵn sàng bắt đầu dự án thật.
