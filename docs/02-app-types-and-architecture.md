# Chương 2 — App Types & App Architecture

> Mục tiêu chương: biết chọn đúng loại app, và hiểu **chính xác** một request đi từ browser của merchant
> đến server của bạn như thế nào khi app chạy trong iframe.

---

## 2.1. Shopify App là gì?

### What is it?

Một Shopify app là **một web application độc lập** mà merchant *cấp quyền* truy cập vào store của họ.
Về mặt kỹ thuật, nó gồm tối đa 3 phần, không phần nào bắt buộc phải có đủ:

| Thành phần | Chạy ở đâu | Dùng để |
|---|---|---|
| **App backend + App Home (frontend)** | Server của bạn | UI chính của app, business logic, gọi Admin API, nhận webhook |
| **App Extensions** | Hạ tầng Shopify (deploy lên Shopify) | Chèn UI/logic vào theme, checkout, admin, POS, Flow |
| **Shopify Functions** | Runtime Wasm của Shopify | Thay đổi logic nghiệp vụ của Shopify (discount, shipping, cart) |

Một app có thể là:
- chỉ backend (ví dụ: app đồng bộ đơn hàng sang ERP),
- chỉ extension (ví dụ: app hiển thị badge trên product page, không cần server),
- hoặc cả hai (phổ biến nhất).

### Why is it important?

Chọn sai kiến trúc ở đầu dự án = viết lại. Ví dụ: nếu tính năng chính là "thay đổi giá ở cart",
bạn **không thể** làm bằng Admin API + webhook — bắt buộc dùng **Shopify Functions**. Biết điều này sớm
tiết kiệm hàng tuần.

---

## 2.2. App distribution types — Public vs Custom

Thuật ngữ chính xác hiện nay là **distribution** (cách phân phối), chọn trong Partner Dashboard.

```
┌─────────────────────────────────────────────────────────────────────────┐
│ PUBLIC APP (Public distribution)                                        │
├─────────────────────────────────────────────────────────────────────────┤
│ • Cài được lên NHIỀU store                                              │
│ • Có thể "listed" (lên App Store, phải qua App Review)                  │
│   hoặc "unlisted" (chia sẻ bằng install link, vẫn qua review)           │
│ • OAuth đầy đủ; dùng được Shopify Billing                               │
│ • BẮT BUỘC: GraphQL Admin API (với app mới), compliance webhooks,       │
│   đạt mọi App Store requirement                                         │
│ → Chọn khi: làm sản phẩm thương mại                                     │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ CUSTOM APP (Custom distribution)                                        │
├─────────────────────────────────────────────────────────────────────────┤
│ • Cài được lên ĐÚNG MỘT store (hoặc 1 Plus organization)                │
│ • Không lên App Store, không qua App Review                             │
│ • Cài bằng install link do Partner Dashboard sinh ra                    │
│ • KHÔNG dùng được Shopify Billing API                                   │
│ → Chọn khi: làm riêng cho 1 merchant theo hợp đồng                      │
│ ⚠ Một khi đã chọn Custom, KHÔNG đổi sang Public được → phải tạo app mới │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ ADMIN-CREATED APP (tạo trong Admin: Settings → Apps → Develop apps)     │
├─────────────────────────────────────────────────────────────────────────┤
│ • Không cần Partner account. Merchant tự tạo trong store của họ.        │
│ • Sinh ra Admin API access token TRỰC TIẾP — không có OAuth.            │
│ • Không có UI embedded, không có extension (trừ hạn chế).               │
│ → Chọn khi: viết script/integration nội bộ, backend-only, 1 store duy   │
│   nhất. Ví dụ: cron job kéo đơn hàng về Google Sheet.                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Bảng quyết định nhanh

| Câu hỏi | Trả lời → loại app |
|---|---|
| Bán cho nhiều merchant? | **Public** |
| Chỉ 1 khách hàng, có UI, cần extension? | **Custom** |
| Chỉ cần script gọi API cho 1 store, không UI? | **Admin-created app** |
| Cần Shopify xử lý thanh toán subscription? | **Public** (bắt buộc) |

### Common mistakes

- ❌ Bắt đầu bằng Admin-created app vì "dễ lấy token", rồi phát hiện không thể bán cho khách khác → phải làm lại từ đầu với OAuth.
- ❌ Chọn Custom distribution cho sản phẩm thương mại → **không thể đổi**, mất app ID, mất data mapping.
- ❌ Tưởng "unlisted public app" không cần App Review → vẫn cần.

> **Lời khuyên cho người học:** luôn tạo **Public app** khi học, kể cả chỉ để test.
> Bạn sẽ học được đúng luồng OAuth, billing, review — là thứ dùng trong công việc thật.

---

## 2.3. Embedded App & App Bridge

### What is it?

**Embedded app** = App Home (giao diện chính của app) được render **bên trong một iframe** đặt trong
Shopify Admin. Merchant không rời khỏi `admin.shopify.com`; họ thấy app của bạn như một trang của Shopify.

**App Bridge** là thư viện JavaScript cầu nối giữa iframe (app của bạn) và trang cha (Shopify Admin).
Nó giải quyết những việc iframe không tự làm được:

- Điều hướng trang cha (đổi URL thật trên thanh địa chỉ).
- Render UI của Shopify *bên ngoài* iframe: title bar, nav menu, toast, modal, resource picker.
- **Cấp session token (ID token)** để app xác thực request — xem chương 3.

### How does it work?

```
Browser của merchant
┌────────────────────────────────────────────────────────────────────┐
│  admin.shopify.com/store/demo/apps/my-app                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Shopify Admin chrome (sidebar, search, nav)                  │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │ <iframe src="https://your-app.com/app?shop=...&host=..."│  │  │
│  │  │                                                        │  │  │
│  │  │   ← App Home của bạn (React Router app) render ở đây   │  │  │
│  │  │                                                        │  │  │
│  │  │   App Bridge script chạy trong iframe này:             │  │  │
│  │  │   • postMessage ↔ parent frame                         │  │  │
│  │  │   • shopify.idToken()  → JWT ngắn hạn                  │  │  │
│  │  │   • <ui-title-bar>, <ui-nav-menu>, shopify.toast       │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
                                 │  HTTPS requests kèm
                                 │  Authorization: Bearer <session token>
                                 ▼
                      ┌──────────────────────────┐
                      │   your-app.com (server)  │
                      └──────────────────────────┘
```

### Key concepts

| Term | Giải thích |
|---|---|
| `host` query param | Base64 của `admin.shopify.com/store/<handle>`. App Bridge cần nó để biết parent là ai. **Phải forward qua mọi navigation.** |
| `shop` query param | Shop domain `*.myshopify.com`. |
| `embedded=1` | Shopify báo app đang chạy nhúng. |
| **App Bridge script** | `<script src="https://cdn.shopify.com/shopifycloud/app-bridge.js" data-api-key="...">` — phải load **trong `<head>`, trước mọi script khác**, và **không** tự bundle qua npm. |
| `shopify` global | Object toàn cục do App Bridge tạo: `shopify.idToken()`, `shopify.toast.show()`, `shopify.resourcePicker()`, `shopify.loading()`. |
| **Web components** | UI của App Bridge/Polaris là custom element (`<s-page>`, `<s-button>`, `<ui-nav-menu>`, `<ui-title-bar>`) — framework-agnostic. |
| **Non-embedded app** | App mở ở tab riêng. Hiếm, và App Store yêu cầu embedded trừ trường hợp đặc biệt. |

### Practical example — App Home tối thiểu

```html
<!-- Trong <head>, LUÔN đứng trước các script khác -->
<script
  src="https://cdn.shopify.com/shopifycloud/app-bridge.js"
  data-api-key="%SHOPIFY_API_KEY%"></script>
```

```tsx
// app/routes/app._index.tsx  (React Router 7 / Shopify template)
export default function Index() {
  return (
    <s-page heading="Dashboard">
      <ui-title-bar title="Dashboard">
        <button variant="primary" onClick={() => shopify.toast.show("Đã lưu")}>
          Save
        </button>
      </ui-title-bar>

      <s-section heading="Tổng quan">
        <s-paragraph>Xin chào từ app của bạn.</s-paragraph>
      </s-section>
    </s-page>
  );
}
```

```tsx
// Lấy session token thủ công (khi gọi API từ client)
const token = await shopify.idToken();
await fetch("/api/sync", {
  method: "POST",
  headers: { Authorization: `Bearer ${token}` },
});
```

> Trong template chính thức, bạn hiếm khi phải tự gọi `shopify.idToken()`:
> hàm `authenticate.admin(request)` ở phía server đã làm hết. Nhưng **phải hiểu** nó tồn tại.

### Common mistakes — iframe gây đau đầu

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| App hiện trang trắng / "refused to connect" | Thiếu header `Content-Security-Policy: frame-ancestors https://<shop> https://admin.shopify.com` | Template đã set sẵn; đừng ghi đè bằng middleware helmet mặc định |
| Mất session khi reload / Safari chặn | Dùng **cookie** để lưu session trong iframe → third-party cookie bị chặn | Dùng **session token (JWT)**, đừng dựa vào cookie |
| Bấm link → app thoát khỏi Admin | Dùng `<a href>` thường hoặc `window.location` | Dùng `<Link>` của React Router (template đã patch) hoặc `target="_top"` đúng cách |
| Mất `host` param → App Bridge lỗi | Navigation tự viết làm rớt query string | Luôn giữ `shop` & `host` |
| Redirect bên trong iframe bị chặn | Redirect 302 sang domain khác trong iframe | Dùng App Bridge redirect / `authenticate.admin()` xử lý |
| Dev tunnel URL đổi mỗi lần chạy | ngrok/cloudflare tunnel ephemeral | Dùng `shopify app dev` (tự cập nhật app URL), hoặc tunnel cố định |

---

## 2.4. Kiến trúc tổng thể của một production app

```
                               INTERNET
                                   │
     ┌─────────────────────────────┼─────────────────────────────┐
     │                             │                             │
     ▼                             ▼                             ▼
Merchant (Admin)           Shopify (webhooks)            Buyer (storefront)
     │                             │                             │
     │ iframe + session token      │ POST + HMAC header          │ load theme
     ▼                             ▼                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        YOUR INFRASTRUCTURE                           │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  WEB TIER — React Router 7 app (Node.js)                   │      │
│  │  • /app/*          App Home routes (embedded UI)           │      │
│  │  • /auth/*         OAuth / token exchange callbacks        │      │
│  │  • /webhooks/*     Webhook receivers (verify HMAC)         │      │
│  │  • /api/*          Public endpoints cho theme extension    │      │
│  └───────────┬──────────────────────────────┬─────────────────┘      │
│              │                              │                        │
│              ▼                              ▼                        │
│  ┌────────────────────────┐    ┌──────────────────────────────┐      │
│  │  DATABASE (Postgres)   │    │  QUEUE / WORKER              │      │
│  │  • Session (tokens)    │    │  • xử lý webhook nặng        │      │
│  │  • App domain data     │    │  • bulk sync, retry          │      │
│  │  • Shop settings       │    │  • scheduled jobs            │      │
│  └────────────────────────┘    └──────────────────────────────┘      │
│                                                                      │
│  ┌────────────────────────┐    ┌──────────────────────────────┐      │
│  │  CACHE (Redis)         │    │  OBSERVABILITY               │      │
│  │  • rate-limit buckets  │    │  • logs, traces, Sentry      │      │
│  └────────────────────────┘    └──────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼  Admin API (GraphQL, 2026-10)
                              Shopify cloud

        ┌──────────────────────────────────────────────────┐
        │  DEPLOY LÊN SHOPIFY (không phải server của bạn): │
        │  • Theme app extension                           │
        │  • Admin/Checkout/Customer-account UI extensions │
        │  • Shopify Functions (Wasm)                      │
        │  → qua `shopify app deploy`                      │
        └──────────────────────────────────────────────────┘
```

**Điểm then chốt cần nhớ:** có **hai nơi deploy khác nhau**.
Backend + App Home deploy lên hosting của bạn (Fly.io, Vercel, Railway, AWS…).
Extensions & Functions deploy lên **Shopify** bằng `shopify app deploy`.
Hai thứ này **phiên bản độc lập** — đây là nguồn bug phổ biến khi release.

---

## 2.5. `shopify.app.toml` — file cấu hình trung tâm

Đây là "nguồn sự thật" về app của bạn. `shopify app deploy` đẩy file này lên Shopify.

```toml
# shopify.app.toml
client_id = "a1b2c3d4e5f6..."           # = API key của app
name = "my-inventory-app"
application_url = "https://my-app.fly.dev"
embedded = true

[access_scopes]
# Shopify managed installation: khai scopes ở đây,
# Shopify tự lo màn hình xin quyền và việc merchant re-approve khi scope đổi.
scopes = "read_products,write_products,read_orders"

[auth]
redirect_urls = [
  "https://my-app.fly.dev/auth/callback",
  "https://my-app.fly.dev/auth/shopify/callback"
]

[webhooks]
api_version = "2026-10"

  [[webhooks.subscriptions]]
  topics = ["app/uninstalled"]
  uri = "/webhooks/app/uninstalled"

  [[webhooks.subscriptions]]
  topics = ["products/update"]
  uri = "/webhooks/products/update"

  # Compliance webhooks — BẮT BUỘC với public app
  [[webhooks.subscriptions]]
  topics = ["customers/data_request", "customers/redact", "shop/redact"]
  uri = "/webhooks/compliance"

[build]
automatically_update_urls_on_dev = true
include_config_on_deploy = true

[pos]
embedded = false
```

### Common mistakes

- ❌ Sửa scopes trong code mà quên sửa `shopify.app.toml` → scope không bao giờ được cấp.
- ❌ Quên `shopify app deploy` sau khi đổi `.toml` → config cũ vẫn chạy trên production.
- ❌ Hard-code `application_url` của tunnel dev vào production config.
- ❌ Không set `include_config_on_deploy = true` → config không được đồng bộ.
- ❌ Dùng nhiều môi trường mà chỉ có 1 file → nên dùng `shopify.app.staging.toml`, `shopify.app.production.toml` và `shopify app config use <name>`.

---

## 2.6. Tóm tắt chương

- App = backend của bạn + (tùy chọn) extensions chạy trên hạ tầng Shopify.
- **Public** (nhiều store, App Store, có billing) vs **Custom** (1 store) vs **Admin-created** (script nội bộ). Quyết định này khó đảo ngược.
- Embedded app sống trong **iframe** → **không dùng cookie session**, dùng **session token**.
- **App Bridge** là cầu nối iframe ↔ Admin; load từ CDN, trong `<head>`, trước mọi script.
- Hai đích deploy: hosting của bạn (backend) và Shopify (extensions).
- `shopify.app.toml` là nguồn sự thật — scopes, webhooks, URLs.

**Tiếp theo:** [Chương 3 — Authentication & APIs](03-auth-and-apis.md)
