# Chương 4 — Webhooks, App Extensions & Data Persistence

---

## 4.1. Webhooks & Event-driven Architecture

### What is it?

Webhook = **Shopify gọi HTTP POST đến server của bạn** khi có sự kiện trong store
(`orders/create`, `products/update`, `app/uninstalled`…).

Đây là chiều ngược lại của API: thay vì app hỏi Shopify, Shopify báo cho app.

### Why is it important?

Không có webhook, app bạn chỉ có hai lựa chọn tồi:
- **Polling** — gọi API mỗi phút để xem có gì mới → tốn rate limit, chậm, không scale.
- **Không biết gì** — merchant sửa sản phẩm trong Admin, app bạn không hay biết.

Thêm nữa, một số webhook là **bắt buộc theo chính sách**:

| Topic | Bắt buộc với | Vì sao |
|---|---|---|
| `app/uninstalled` | Mọi app | Phải xoá session/token khi merchant gỡ app |
| `app/scopes_update` | App dùng managed installation | Biết khi scopes thay đổi |
| `customers/data_request` | Public app | GDPR — merchant yêu cầu dữ liệu khách |
| `customers/redact` | Public app | GDPR — xoá dữ liệu khách |
| `shop/redact` | Public app | GDPR — xoá dữ liệu shop (48h sau uninstall) |

### How does it work?

```
 Merchant sửa 1 sản phẩm trong Admin
            │
            ▼
 ┌──────────────────────────────────────────┐
 │  Shopify event bus                       │
 │  tìm mọi subscription cho topic          │
 │  "products/update" của shop này          │
 └──────────────┬───────────────────────────┘
                │
                ▼
 POST https://your-app.com/webhooks/products/update
 Headers:
   X-Shopify-Topic:               products/update
   X-Shopify-Shop-Domain:         demo.myshopify.com
   X-Shopify-Hmac-Sha256:         <base64 HMAC của raw body>
   X-Shopify-Webhook-Id:          <uuid, dùng để chống xử lý trùng>
   X-Shopify-API-Version:         2026-10
   X-Shopify-Triggered-At:        <ISO timestamp>
 Body: { "id": 123, "title": "...", ... }
                │
                ▼
 ┌──────────────────────────────────────────┐
 │ YOUR SERVER                              │
 │ 1. Đọc RAW body (chưa parse JSON)        │
 │ 2. Verify HMAC bằng SHOPIFY_API_SECRET   │
 │ 3. Trả 200 NGAY (< 5 giây)               │
 │ 4. Đẩy job vào queue để xử lý thật       │
 └──────────────────────────────────────────┘
```

### Key concepts

| Concept | Giải thích |
|---|---|
| **Topic** | Tên sự kiện: `orders/create`, `products/delete`, `fulfillments/update`… |
| **Subscription** | Đăng ký nhận topic. Khai trong `shopify.app.toml` (app-level) hoặc tạo động qua API (shop-level) |
| **HMAC verification** | Shopify ký raw body bằng API secret. **Không verify = ai cũng gửi webhook giả được.** |
| **Timeout 5 giây** | Không trả 200 trong ~5s → Shopify coi là fail |
| **Retry** | Shopify retry ~19 lần trong 48 giờ với backoff. Sau đó tự xoá subscription nếu fail liên tục |
| **At-least-once delivery** | Webhook **có thể đến 2+ lần**, hoặc **đến sai thứ tự**, hoặc **mất** |
| **Idempotency** | Xử lý cùng webhook nhiều lần phải cho cùng kết quả. Dùng `X-Shopify-Webhook-Id` để dedupe |
| **Delivery methods** | HTTPS (mặc định), **Amazon EventBridge**, **Google Pub/Sub** (cho volume lớn) |

### Practical example

```ts
// app/routes/webhooks.products.update.tsx
import { authenticate } from "../shopify.server";
import { enqueue } from "../queue.server";

export const action = async ({ request }) => {
  // authenticate.webhook() verify HMAC + parse payload giúp bạn
  const { topic, shop, payload, webhookId, session } =
    await authenticate.webhook(request);

  // Dedupe: nếu đã thấy webhookId này rồi thì bỏ qua
  const seen = await db.webhookLog.findUnique({ where: { webhookId } });
  if (seen) return new Response(null, { status: 200 });
  await db.webhookLog.create({ data: { webhookId, topic, shop } });

  // KHÔNG xử lý nặng ở đây — đẩy vào queue
  await enqueue({ topic, shop, payload });

  return new Response(null, { status: 200 });
};
```

```ts
// app/routes/webhooks.app.uninstalled.tsx — BẮT BUỘC
export const action = async ({ request }) => {
  const { shop } = await authenticate.webhook(request);
  // Xoá session/token: token đã chết, giữ lại là rủi ro bảo mật
  await db.session.deleteMany({ where: { shop } });
  // Đánh dấu shop inactive (đừng xoá data ngay — merchant có thể cài lại)
  await db.shopConfig.update({ where: { shop }, data: { active: false } });
  return new Response(null, { status: 200 });
};
```

**Verify HMAC thủ công** (khi không dùng template — hiểu để biết nó làm gì):

```ts
import crypto from "node:crypto";

function verifyWebhook(rawBody: Buffer, hmacHeader: string) {
  const digest = crypto
    .createHmac("sha256", process.env.SHOPIFY_API_SECRET!)
    .update(rawBody)            // ⚠ RAW body, không phải JSON.stringify(parsed)
    .digest("base64");
  return crypto.timingSafeEqual(   // ⚠ so sánh an toàn, không dùng ===
    Buffer.from(digest),
    Buffer.from(hmacHeader)
  );
}
```

### Common mistakes

| Lỗi | Hậu quả |
|---|---|
| Parse JSON trước khi verify HMAC | HMAC **luôn sai** (body đã bị biến đổi) |
| So sánh HMAC bằng `===` | Timing attack |
| Xử lý nặng đồng bộ trong handler | Timeout 5s → Shopify retry → xử lý trùng → data hỏng |
| Không idempotent | Gửi email 3 lần, tạo 3 đơn trùng |
| Giả định webhook đến đúng thứ tự | `orders/updated` có thể đến **trước** `orders/create` |
| Giả định webhook không bao giờ mất | Phải có **reconciliation job** định kỳ đối chiếu với API |
| Trả 500 khi lỗi nghiệp vụ | Shopify retry 19 lần vô nghĩa. Trả 200 + log + dead-letter queue |
| Quên compliance webhooks | **Fail App Review chắc chắn** |
| Chỉ test webhook ở local với tunnel rồi quên đổi URL production | Webhook production không tới |

**Test webhook:** `shopify app webhook trigger --topic=orders/create --address=https://... `

> **Nguyên tắc vàng:** Webhook là **tín hiệu**, không phải **nguồn sự thật**.
> Khi nhận webhook `orders/create`, nếu cần dữ liệu đầy đủ và chính xác, hãy **gọi lại Admin API**
> để lấy state hiện tại thay vì tin tuyệt đối vào payload (payload có thể đã cũ).

---

## 4.2. App Extensions

### What is it?

Extension là **code của bạn chạy trên hạ tầng Shopify**, chèn vào các điểm mở rộng (extension points)
trong sản phẩm Shopify. Khác hoàn toàn với App Home — extension **không chạy trên server của bạn**.

### Why is it important?

Merchant đánh giá app qua trải nghiệm **tại nơi họ làm việc**: trang product, checkout, theme.
App chỉ có một trang dashboard riêng biệt luôn thua app tích hợp được vào workflow của họ.

Và quan trọng hơn: **có những việc chỉ extension làm được**. Ví dụ, bạn **không thể** thay đổi
logic tính discount ở checkout bằng Admin API — bắt buộc dùng Shopify Functions.

### Danh sách extension (nguồn: shopify.dev, list of app extensions)

| Nhóm | Extension | Xuất hiện ở | Mức độ |
|---|---|---|---|
| **Admin** | Admin action | Modal trên trang resource (product, order…) | Should |
| | Admin block | Card trong sidebar trang resource | Should |
| | Admin link | Link nhanh tới app từ bất kỳ trang admin nào | Should |
| | Navigation links | Mục điều hướng của app | Should |
| | Product configuration | Cấu hình bundle trên trang product | Advanced |
| | Discount function settings | UI cấu hình discount của app | Advanced |
| | Channel config / Subscription link / Purchase options | Admin | Advanced |
| | App Home UI extension | Landing page do Shopify host, không cần backend | Should |
| **Online store** | **Theme app extension** | Theme (app block / app embed) | **Should (học sớm)** |
| **Checkout** | Checkout UI extension | Trang checkout | Advanced |
| | Shopify Functions | Logic cart/checkout (discount, shipping, payment) | Advanced |
| | Post-purchase | Sau khi thanh toán (cần duyệt riêng) | Advanced |
| **Customer accounts** | Customer account UI extension | Trang tài khoản khách | Advanced |
| **Analytics** | Web pixel | Checkout, order status, online store | Advanced |
| **POS** | POS UI extension | Ứng dụng Shopify POS | Advanced |
| **Flow** | Trigger / Action / Template / Lifecycle events | Shopify Flow | Advanced |
| **Payments** | Payments extension | Checkout (cần duyệt riêng) | Advanced |

### Theme App Extension — học cái này trước

Đây là extension dễ nhất và hữu dụng nhất với newbie. Nó cho phép merchant **kéo thả** block của app
vào theme qua Theme Editor, **không cần sửa file theme** (quan trọng: sửa file theme trực tiếp là
anti-pattern và bị App Review phạt).

```
extensions/my-theme-extension/
├── shopify.extension.toml
├── blocks/
│   ├── review-widget.liquid     # app block — merchant kéo vào section
│   └── app-embed.liquid         # app embed — chạy toàn site (script, popup)
├── snippets/
├── assets/
│   ├── widget.js
│   └── widget.css
└── locales/
    └── en.default.json
```

```liquid
{% comment %} blocks/review-widget.liquid {% endcomment %}
<div class="my-reviews" data-product-id="{{ product.id }}">
  <h3>{{ block.settings.heading }}</h3>
  <div id="reviews-root"></div>
</div>

<script src="{{ 'widget.js' | asset_url }}" defer></script>

{% schema %}
{
  "name": "Product Reviews",
  "target": "section",
  "settings": [
    { "type": "text", "id": "heading", "label": "Heading", "default": "Reviews" },
    { "type": "range", "id": "limit", "label": "Số review", "min": 1, "max": 20, "default": 5 }
  ]
}
{% endschema %}
```

```js
// assets/widget.js — gọi API công khai của app bạn
const el = document.querySelector(".my-reviews");
fetch(`https://your-app.com/api/reviews?shop=${Shopify.shop}&product=${el.dataset.productId}`)
  .then(r => r.json())
  .then(render);
```

> ⚠ Endpoint `/api/reviews` là **public** (ai cũng gọi được). Phải: rate limit, validate `shop`,
> chỉ trả dữ liệu công khai, set CORS đúng. Đừng dùng access token ở đây.

### Deploy extensions

```bash
shopify app deploy      # đẩy TẤT CẢ extensions + config lên Shopify, tạo 1 app version
```

Mỗi lần deploy tạo một **app version**. Có thể rollback version trong Partner Dashboard.
Lưu ý: backend của bạn deploy riêng → phải đảm bảo **tương thích ngược** giữa 2 lần deploy.

### Common mistakes

- ❌ Sửa trực tiếp file `theme.liquid` của merchant qua Asset API → **bị cấm**, dùng theme app extension.
- ❌ Quên theme app extension cần merchant **bật thủ công** trong Theme Editor → phải có hướng dẫn onboarding (có deep link `https://{shop}/admin/themes/current/editor?context=apps`).
- ❌ Load JS nặng trong app embed → tụt điểm Core Web Vitals → fail Built for Shopify.
- ❌ Tưởng extension gọi được Admin API với access token → không; extension chạy ở client/sandbox, phải đi qua server của bạn hoặc Storefront API.
- ❌ Deploy extension mà quên deploy backend tương ứng → extension gọi endpoint chưa tồn tại.

---

## 4.3. Data Persistence — lưu dữ liệu ở đâu?

### Ba nơi có thể lưu dữ liệu

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. DATABASE CỦA BẠN                                                  │
│    Postgres / MySQL / MongoDB, thường qua Prisma                     │
│    → Session/token, dữ liệu nghiệp vụ riêng của app, logs, settings, │
│      analytics, bảng trung gian                                      │
│    ✓ Toàn quyền, query linh hoạt, không giới hạn                     │
│    ✗ Bạn chịu trách nhiệm backup, scale, GDPR                        │
│    ✗ Merchant gỡ app → dữ liệu mất (phải xoá theo chính sách)        │
├──────────────────────────────────────────────────────────────────────┤
│ 2. METAFIELDS / METAOBJECTS (trên Shopify)                           │
│    Dữ liệu gắn vào resource Shopify (product, order, customer, shop) │
│    → Cấu hình hiển thị, thuộc tính mở rộng của sản phẩm              │
│    ✓ Theme đọc được trực tiếp bằng Liquid (không cần gọi server bạn) │
│    ✓ Sống cùng store, merchant thấy được trong Admin                 │
│    ✗ Không query phức tạp, có giới hạn kích thước                    │
├──────────────────────────────────────────────────────────────────────┤
│ 3. SHOPIFY RESOURCES GỐC                                             │
│    tags, product type, collections, customer segments                │
│    → Dữ liệu merchant cần nhìn thấy/lọc trong Admin                  │
└──────────────────────────────────────────────────────────────────────┘
```

### Quy tắc chọn

| Dữ liệu | Lưu ở đâu | Vì sao |
|---|---|---|
| Access token, session | **DB của bạn** (mã hoá) | Bí mật |
| Cấu hình app theo shop (plan, feature flags) | **DB của bạn** | Nhiều, hay đổi |
| "Sản phẩm này có badge Eco không" (theme cần đọc) | **Metafield** | Theme đọc bằng Liquid, nhanh, không tốn request tới server bạn |
| Lịch sử gửi email marketing | **DB của bạn** | Shopify không có chỗ chứa |
| Review của khách hàng | **DB của bạn** + metafield tổng hợp (rating trung bình) | Chi tiết ở DB, số tổng hợp ở metafield cho theme/SEO |
| Nội dung cấu trúc do merchant tạo (FAQ, size chart) | **Metaobject** | Merchant quản lý được trong Admin |

### Metafields — practical

```graphql
mutation SetProductMetafield($metafields: [MetafieldsSetInput!]!) {
  metafieldsSet(metafields: $metafields) {
    metafields { id namespace key value type }
    userErrors { field message }
  }
}
```
```json
{
  "metafields": [{
    "ownerId": "gid://shopify/Product/123",
    "namespace": "$app:reviews",
    "key": "average_rating",
    "type": "number_decimal",
    "value": "4.7"
  }]
}
```

Đọc trong theme:
```liquid
{{ product.metafields.reviews.average_rating }}
```

> `$app:` là **reserved namespace** của app bạn — tránh đụng tên với app khác.
> Muốn theme/Storefront API đọc được, metafield definition phải được cấp quyền truy cập phù hợp.

### Session storage

Session (chứa access token) phải được lưu **persistent**. Template dùng Prisma:

```prisma
// prisma/schema.prisma
model Session {
  id            String    @id
  shop          String
  state         String
  isOnline      Boolean   @default(false)
  scope         String?
  expires       DateTime?
  accessToken   String
  userId        BigInt?
  firstName     String?
  lastName      String?
  email         String?
  accountOwner  Boolean   @default(false)
  locale        String?
  collaborator  Boolean?  @default(false)
  emailVerified Boolean?  @default(false)

  @@index([shop])
}
```

### Common mistakes

| Lỗi | Hậu quả |
|---|---|
| Dùng SQLite trên production (template mặc định) | Filesystem ephemeral trên Fly/Railway/Vercel → **mất hết session khi redeploy** → merchant phải cài lại app |
| Lưu accessToken plaintext | Rủi ro breach, fail review |
| Không index theo `shop` | Query chậm khi có nhiều shop |
| Không có cột `shop` trong mọi bảng nghiệp vụ | **Rò rỉ dữ liệu giữa các merchant** — lỗi nghiêm trọng nhất của multi-tenant app |
| Copy dữ liệu toàn bộ store về DB mình "cho nhanh" | Tốn kém, lệch dữ liệu, vi phạm data minimization. Chỉ lưu cái thực sự cần |
| Không xoá dữ liệu sau `shop/redact` | Vi phạm GDPR |

> **Quy tắc multi-tenancy:** mọi bảng nghiệp vụ **phải** có cột `shop`, và **mọi query phải filter theo `shop`**.
> Viết một repository layer bắt buộc truyền `shop` để không thể quên.

---

## 4.4. Tóm tắt chương

- Webhook là cách Shopify báo sự kiện. **Verify HMAC trên raw body**, trả 200 trong 5s, xử lý bất đồng bộ, **idempotent**.
- Webhook là **at-least-once**, có thể trùng/mất/sai thứ tự → cần dedupe + reconciliation job.
- `app/uninstalled` + 3 compliance webhooks là **bắt buộc**.
- Extension chạy **trên Shopify**, deploy bằng `shopify app deploy`. Theme app extension là cái nên học đầu tiên.
- Ba nơi lưu data: DB của bạn (bí mật, nghiệp vụ), metafield/metaobject (theme đọc được), Shopify resource gốc.
- **Không bao giờ dùng SQLite trên production.** Mọi bảng phải có `shop` và mọi query phải filter theo nó.

**Tiếp theo:** [Chương 5 — Stack, Deploy, Billing, Security](05-stack-deploy-billing-security.md)
