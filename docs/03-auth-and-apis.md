# Chương 3 — Authentication, Authorization & APIs

> Chương quan trọng nhất. Nếu chỉ đọc một chương, đọc chương này.
> Mục tiêu: hiểu **ba loại token** khác nhau, luồng install đầy đủ, và cách dùng GraphQL Admin API đúng chuẩn.

---

## 3.1. Ba loại "token" — đừng nhầm lẫn

Đây là nguồn nhầm lẫn số 1 của newbie.

| | **Session token** (ID token) | **Access token** | **API secret key** |
|---|---|---|---|
| Là gì | JWT ngắn hạn do **App Bridge** sinh ra trong browser | Token dài hạn để gọi Admin API | Secret của app |
| Ai cấp | Shopify (qua App Bridge, trong iframe) | Shopify (sau OAuth/token exchange) | Partner Dashboard |
| Sống bao lâu | **~1 phút** | Offline: đến khi uninstall. Online: theo phiên staff (~24h) | Vĩnh viễn (cho tới khi rotate) |
| Dùng để | Chứng minh "request này đến từ app tôi, trong store X, bởi user Y" | Gọi `https://{shop}/admin/api/2026-10/graphql.json` | Verify HMAC webhook, exchange token |
| Nằm ở đâu | Header `Authorization: Bearer <jwt>` từ frontend → backend của bạn | **Database của bạn** | Biến môi trường server |
| Có được gửi cho Shopify API không | **Không** | **Có** | **Không bao giờ** |

```
   ┌──────────────┐  (1) session token (JWT, ~60s)   ┌──────────────┐
   │   Browser    │ ───────────────────────────────► │  Your Server │
   │ (App Bridge) │     Authorization: Bearer ...     │              │
   └──────────────┘                                   └──────┬───────┘
                                                             │ (2) tra DB
                                                             │     lấy access token
                                                             ▼
                                                   ┌──────────────────┐
                                                   │ Shopify Admin API│
                                                   │ X-Shopify-Access-│
                                                   │ Token: shpat_... │
                                                   └──────────────────┘
```

> **Nhớ:** Session token để *vào nhà bạn*. Access token để *bạn vào nhà Shopify*.

---

## 3.2. Access scopes

### What is it?

Scope là **quyền cụ thể** mà merchant cấp cho app: `read_products`, `write_orders`, `read_customers`…
Access token chỉ làm được đúng những gì scope cho phép.

### Key concepts

- `write_X` **bao hàm** `read_X`. Khai `write_products` là đủ, không cần thêm `read_products`.
- Scope khai trong `shopify.app.toml` → **Shopify managed installation** tự lo màn hình consent.
- Đổi scope → merchant phải **approve lại**. Shopify xử lý tự động khi họ mở app, nhưng đây là **friction** → đừng đổi scope vặt.
- Một số scope là **protected customer data** (`read_customers`, `read_orders`, `read_all_orders`…) — public app phải **khai báo mục đích và được Shopify duyệt**, kèm yêu cầu bảo mật dữ liệu.
- `read_all_orders` (đơn > 60 ngày) phải xin riêng.

### Practical example

```toml
[access_scopes]
scopes = "write_products,read_orders,write_inventory"
```

```ts
// Kiểm tra scope đã được cấp chưa (React Router template)
const { session } = await authenticate.admin(request);
if (!session.scope?.includes("write_inventory")) {
  // xử lý graceful, đừng crash
}
```

### Common mistakes

- ❌ Khai thừa scope "cho chắc" → App Review **sẽ reject**. Nguyên tắc **least privilege**.
- ❌ Khai cả `read_products` lẫn `write_products` → thừa.
- ❌ Giả định scope luôn có → app cũ cài từ trước khi bạn thêm scope sẽ **thiếu** scope đó. Phải code phòng thủ.
- ❌ Xin `read_customers` mà không thực sự cần → kéo theo cả quy trình protected data review.

---

## 3.3. Luồng cài đặt: Shopify managed installation + Token Exchange

### Cách làm **hiện tại** (dùng cho app mới)

Trước đây mọi app tự implement OAuth authorization code grant (`/auth` → `/auth/callback`).
Hiện nay, app embedded tạo bằng Shopify CLI dùng:

1. **Shopify managed installation** — Shopify lo toàn bộ màn hình xin quyền, dựa trên scopes trong `.toml`.
2. **Token exchange** — app đổi *session token* lấy *access token*, **không cần redirect**.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  MERCHANT CÀI APP (managed installation)                                 │
└──────────────────────────────────────────────────────────────────────────┘

 1. Merchant bấm "Add app" (App Store hoặc install link)
          │
          ▼
 2. SHOPIFY hiển thị màn hình permission
    "App X muốn: xem sản phẩm, quản lý đơn hàng…"   ← dựa trên shopify.app.toml
          │  merchant bấm Install
          ▼
 3. SHOPIFY ghi nhận app đã cài + scopes đã grant
    (app của bạn CHƯA có token lúc này)
          │
          ▼
 4. SHOPIFY redirect merchant vào Admin → iframe mở
    GET https://your-app.com/app?shop=demo.myshopify.com&host=<base64>&embedded=1
          │
          ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  TOKEN EXCHANGE (lần đầu, và mỗi khi token mất)                          │
└──────────────────────────────────────────────────────────────────────────┘

 5. App Bridge trong iframe sinh SESSION TOKEN (JWT, ký bằng API secret)
          │
          ▼
 6. Frontend gửi request kèm  Authorization: Bearer <session token>
          │
          ▼
 7. SERVER của bạn verify JWT:
       • signature  = HMAC-SHA256 với SHOPIFY_API_SECRET
       • aud        = API key của app
       • dest       = shop domain
       • exp / nbf  = còn hạn
          │
          ▼
 8. SERVER gọi Token Exchange:
       POST https://{shop}/admin/oauth/access_token
       {
         client_id, client_secret,
         grant_type: "urn:ietf:params:oauth:grant-type:token-exchange",
         subject_token: "<session token>",
         subject_token_type: "urn:ietf:params:oauth:token-type:id_token",
         requested_token_type:
           "urn:shopify:params:oauth:token-type:offline-access-token"
       }
          │
          ▼
 9. SHOPIFY trả { access_token: "shpat_...", scope: "..." }
          │
          ▼
10. SERVER lưu vào bảng Session (shop, accessToken, scope, isOnline)
          │
          ▼
11. SERVER gọi Admin API và render UI
```

### Khi nào vẫn phải dùng OAuth authorization code grant?

- App **không embedded** (chạy ở tab riêng, không có App Bridge → không có session token).
- Flow cài đặt bắt đầu từ ngoài Shopify.

Luồng đó là chuẩn OAuth 2.0: `/auth?shop=` → redirect tới Shopify → merchant approve → callback kèm `code`
→ server đổi `code` lấy `access_token`. **Luôn verify HMAC của query string** ở callback.

### Online vs Offline access token

| | **Offline token** (mặc định) | **Online token** |
|---|---|---|
| Gắn với | Store | Staff user cụ thể |
| Hết hạn | Không (tới khi uninstall) | Theo phiên đăng nhập (~24h) |
| Permission | Theo scopes của app | Giao của scopes app **và** permission của staff đó |
| Dùng cho | Webhook handler, cron job, background sync | Hành động cần attribution theo user, hoặc cần tôn trọng quyền staff |

> Hầu hết app dùng **offline** là đủ. Dùng online khi cần biết "ai đã bấm nút này" hoặc khi
> không muốn staff cấp thấp làm được việc mà app cho phép.

### Practical example — code thực tế với template

```ts
// app/shopify.server.ts
import "@shopify/shopify-app-react-router/adapters/node";
import {
  ApiVersion,
  AppDistribution,
  shopifyApp,
} from "@shopify/shopify-app-react-router/server";
import { PrismaSessionStorage } from "@shopify/shopify-app-session-storage-prisma";
import prisma from "./db.server";

const shopify = shopifyApp({
  apiKey: process.env.SHOPIFY_API_KEY!,
  apiSecretKey: process.env.SHOPIFY_API_SECRET!,
  apiVersion: ApiVersion.October26,          // 2026-10
  scopes: process.env.SCOPES?.split(","),
  appUrl: process.env.SHOPIFY_APP_URL!,
  authPathPrefix: "/auth",
  sessionStorage: new PrismaSessionStorage(prisma),
  distribution: AppDistribution.AppStore,
  future: { unstable_newEmbeddedAuthStrategy: true }, // token exchange
});

export default shopify;
export const authenticate = shopify.authenticate;
export const unauthenticated = shopify.unauthenticated;
```

```ts
// app/routes/app.products.tsx
import { authenticate } from "../shopify.server";

export const loader = async ({ request }) => {
  // Một dòng này làm TẤT CẢ:
  //   verify session token → token exchange nếu cần → nạp session từ DB
  //   → trả về admin client đã gắn access token
  const { admin, session } = await authenticate.admin(request);

  const response = await admin.graphql(`
    query ProductList($first: Int!) {
      products(first: $first, sortKey: UPDATED_AT, reverse: true) {
        edges {
          cursor
          node { id title handle status totalInventory }
        }
        pageInfo { hasNextPage endCursor }
      }
    }`,
    { variables: { first: 20 } }
  );

  const { data } = await response.json();
  return { products: data.products.edges.map((e) => e.node) };
};
```

### Common mistakes

| Lỗi | Hậu quả |
|---|---|
| Tự viết lại OAuth khi template đã lo | Bug bảo mật, mất thời gian, fail review |
| Không verify session token signature | Bất kỳ ai cũng giả mạo request được → **lỗ hổng nghiêm trọng** |
| Lưu access token vào localStorage / cookie client | Token lộ hoàn toàn |
| Lưu access token không mã hóa trong DB | Fail App Review / data breach |
| Không xoá session khi nhận `app/uninstalled` | Dùng token chết → lỗi 401 hàng loạt; vi phạm chính sách data |
| Dùng một access token cho nhiều shop | Token **gắn chặt với 1 shop**, không bao giờ dùng chung |
| Quên rằng token có thể bị revoke bất cứ lúc nào | Phải bắt lỗi 401 và trigger re-auth |

---

## 3.4. GraphQL Admin API

### What is it?

API chính để đọc/ghi dữ liệu store: products, orders, customers, inventory, fulfillment, discounts, metafields…
Một endpoint duy nhất:

```
POST https://{shop}.myshopify.com/admin/api/2026-10/graphql.json
X-Shopify-Access-Token: shpat_xxx
Content-Type: application/json
```

### Why is it important?

- **REST Admin API là legacy từ 2024-10-01.** Từ **2025-04-01**, mọi **public app mới bắt buộc dùng GraphQL**.
- Nhiều tính năng mới (bundles, một số phần của B2B, nhiều object mới) **chỉ có ở GraphQL**.
- Rate limit của GraphQL tính theo **cost**, linh hoạt hơn REST.

> Khi nào còn gặp REST? App cũ đang maintain, hoặc một vài endpoint legacy. Biết đọc là đủ, không học sâu.

### Key concepts

| Concept | Giải thích |
|---|---|
| **GID** | ID dạng `gid://shopify/Product/1234567890`. Đây là ID chuẩn của GraphQL, **không phải** số nguyên như REST. |
| **Connection / edges / node** | Pattern phân trang chuẩn Relay: `products(first: 50) { edges { cursor node {...} } pageInfo { hasNextPage endCursor } }` |
| **Cursor pagination** | Không có `page=2`. Dùng `after: <endCursor>`. |
| **`userErrors`** | Mutation **không** ném lỗi HTTP khi nghiệp vụ sai. Lỗi nằm trong `userErrors`. **Luôn phải check.** |
| **Query cost** | Mỗi query có `cost` tính theo số field/object. Trả về trong `extensions.cost`. |
| **Throttling** | Leaky bucket: mặc định **2000 điểm**, hồi **100 điểm/giây** (Plus ×10). Vượt → lỗi `THROTTLED`. |
| **Bulk Operations** | Query/mutation bất đồng bộ cho dataset lớn (hàng chục nghìn bản ghi). Shopify tạo file JSONL để bạn tải. |
| **API version** | Bắt buộc có trong URL. Ra mỗi quý, hỗ trợ **tối thiểu 12 tháng**. |

### Practical examples

**Query có pagination:**

```graphql
query GetOrders($first: Int!, $after: String) {
  orders(first: $first, after: $after, query: "financial_status:paid") {
    edges {
      node {
        id
        name
        createdAt
        totalPriceSet { shopMoney { amount currencyCode } }
        customer { id email }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

```ts
// Vòng lặp phân trang đúng cách
async function fetchAllOrders(admin) {
  const all = [];
  let after = null;
  do {
    const res = await admin.graphql(QUERY, { variables: { first: 100, after } });
    const { data } = await res.json();
    all.push(...data.orders.edges.map((e) => e.node));
    after = data.orders.pageInfo.hasNextPage
      ? data.orders.pageInfo.endCursor
      : null;
  } while (after);
  return all;
}
```

**Mutation — luôn xử lý `userErrors`:**

```graphql
mutation UpdateProductTitle($input: ProductInput!) {
  productUpdate(input: $input) {
    product { id title updatedAt }
    userErrors { field message }
  }
}
```

```ts
const res = await admin.graphql(MUTATION, {
  variables: {
    input: { id: "gid://shopify/Product/123", title: "Áo thun mới" },
  },
});
const { data } = await res.json();

// ⚠ HTTP 200 KHÔNG có nghĩa là thành công
if (data.productUpdate.userErrors.length > 0) {
  throw new Error(
    data.productUpdate.userErrors.map((e) => `${e.field}: ${e.message}`).join("; ")
  );
}
```

**Xử lý throttling:**

```ts
async function graphqlWithRetry(admin, query, variables, attempt = 0) {
  const res = await admin.graphql(query, { variables });
  const body = await res.json();

  const throttled = body.errors?.some(
    (e) => e.extensions?.code === "THROTTLED"
  );
  if (throttled && attempt < 5) {
    const wait = Math.min(2 ** attempt * 500, 10_000);
    await new Promise((r) => setTimeout(r, wait));
    return graphqlWithRetry(admin, query, variables, attempt + 1);
  }
  return body;
}

// Chủ động: đọc ngân sách còn lại
// body.extensions.cost.throttleStatus = { maximumAvailable, currentlyAvailable, restoreRate }
```

**Bulk operation (khi cần kéo 100k sản phẩm):**

```graphql
mutation {
  bulkOperationRunQuery(
    query: """
    { products { edges { node { id title vendor } } } }
    """
  ) {
    bulkOperation { id status }
    userErrors { field message }
  }
}
```
Sau đó poll `currentBulkOperation` cho tới `status: COMPLETED`, rồi tải file JSONL từ `url`.
Hoặc tốt hơn: đăng ký webhook `bulk_operations/finish`.

### Common mistakes

| Lỗi | Vì sao sai |
|---|---|
| Không check `userErrors` | "Save thành công" nhưng dữ liệu không đổi — bug kinh điển |
| Over-fetching (`first: 250` + mọi field) | Query cost tăng vọt → throttled |
| Không pin API version, dùng `unstable` | App vỡ không báo trước |
| Lặp `for (product of products) { await getVariants(product) }` | N+1 query → throttled. Hãy query lồng hoặc bulk |
| Nhầm GID với numeric ID | `gid://shopify/Product/123` ≠ `123`. Parse cẩn thận khi tích hợp hệ thống khác |
| Dùng `products(first: 250)` thay vì bulk cho dataset lớn | Timeout, throttle. >~10k bản ghi → Bulk Operations |
| Bỏ qua `extensions.cost` | Không biết mình sắp bị throttle |
| Gọi API trực tiếp từ browser | Lộ access token. **Luôn gọi qua server của bạn.** |

---

## 3.5. Các API khác cần biết

| API | Dùng cho | Auth | Mức ưu tiên học |
|---|---|---|---|
| **Admin API (GraphQL)** | Đọc/ghi dữ liệu store từ app | Access token | **Must** |
| **Storefront API** | Xây storefront tùy biến / headless; đọc catalog công khai, tạo cart, checkout | Storefront access token (public hoặc private) | Should |
| **Customer Account API** | Dữ liệu tài khoản khách hàng (orders của chính họ) | Customer access token (OAuth của buyer) | Advanced |
| **Partner API** | Lấy dữ liệu về app của bạn (install, doanh thu) | Partner API token | Advanced |
| **App Events API** | Gửi usage event cho Shopify App Pricing | Access token | Should (nếu usage billing) |
| **Payments Apps API** | App cung cấp phương thức thanh toán | Access token | Advanced |
| **Ajax API** (storefront, không versioned) | JS trên theme thao tác cart | Không | Should (khi làm theme extension) |
| **Function API / Shopify Functions** | Can thiệp logic discount, shipping, cart | N/A (chạy trên Shopify) | Advanced |

### Storefront API — điểm cần phân biệt

```
Admin API                          Storefront API
─────────────────────────────      ──────────────────────────────
Dùng bởi SERVER của app            Dùng bởi STOREFRONT/client
Token: shpat_… (bí mật)            Token: public storefront token
                                   (được phép lộ, scope rất hẹp)
Thấy mọi dữ liệu theo scope        Chỉ thấy dữ liệu published
Rate limit theo cost               Rate limit theo IP
Endpoint: /admin/api/…             Endpoint: /api/2026-10/graphql.json
```

Khi nào app bạn cần Storefront API? Khi bạn build headless storefront (Hydrogen), hoặc khi
theme app extension cần đọc catalog mà không muốn đi qua server của bạn.

---

## 3.6. Tóm tắt chương

- Ba loại token: **session token** (browser → server bạn), **access token** (server bạn → Shopify), **API secret** (chỉ ở server).
- Scopes khai trong `shopify.app.toml`; **least privilege**; `write_` bao hàm `read_`.
- App embedded hiện đại dùng **managed installation + token exchange**, không tự viết OAuth redirect.
- **Offline token** cho background work; **online token** khi cần quyền theo staff.
- **GraphQL Admin API** là bắt buộc cho public app mới; REST là legacy.
- Luôn: check `userErrors`, dùng cursor pagination, pin API version `2026-10`, xử lý throttling, bulk cho dataset lớn.

**Tiếp theo:** [Chương 4 — Webhooks, Extensions & Data](04-webhooks-extensions-data.md)
