# Chương 5 — Stack, CLI, Deployment, Billing, Security & App Review

---

## 5.1. Shopify CLI & Development Workflow

### What is it?

Shopify CLI là công cụ dòng lệnh chính thức: scaffold project, chạy dev server kèm tunnel HTTPS,
tạo extension, deploy, quản lý app config.

### Các lệnh phải thuộc

```bash
npm install -g @shopify/cli@latest
shopify version

# Tạo app mới từ template chính thức (React Router)
shopify app init --template=https://github.com/Shopify/shopify-app-template-react-router

# Chạy dev: tạo tunnel HTTPS, tự cập nhật application_url, hot reload,
# in ra link cài app vào dev store
shopify app dev

# Tạo extension mới (CLI hỏi loại extension)
shopify app generate extension

# Deploy extensions + config lên Shopify → tạo 1 app version
shopify app deploy

# Quản lý nhiều môi trường
shopify app config link        # liên kết thư mục với 1 app trong Partner Dashboard
shopify app config use staging # chuyển sang shopify.app.staging.toml

# Debug
shopify app webhook trigger --topic=orders/create
shopify app env show
shopify app function run        # chạy thử Shopify Function cục bộ
```

### Development workflow thực tế

```
1. shopify app dev
   ├─ CLI tạo tunnel: https://random-abc.trycloudflare.com
   ├─ CLI cập nhật application_url + redirect_urls trong app config
   ├─ CLI build & watch frontend + backend
   └─ CLI in ra: "Preview URL: https://admin.shopify.com/store/.../apps/..."

2. Mở Preview URL → app cài vào dev store → thấy App Home trong iframe

3. Sửa code → hot reload → refresh iframe

4. Thêm extension → shopify app generate extension → CLI thêm vào extensions/
   → shopify app dev tự phục vụ extension ở chế độ dev preview

5. Hài lòng → shopify app deploy (extensions) + deploy backend lên hosting
```

### Common mistakes

- ❌ Chạy `shopify app dev` rồi đóng terminal → tunnel chết → app báo lỗi, tưởng code sai.
- ❌ Dev trên app production → `automatically_update_urls_on_dev` ghi đè `application_url` production. **Luôn dùng app riêng cho dev.**
- ❌ Quên `shopify app deploy` → extension/config mới không lên production.
- ❌ Commit `.env` chứa `SHOPIFY_API_SECRET` lên git.

---

## 5.2. Technology Stack

### Stack chuẩn hiện nay

| Lớp | Công nghệ | Ghi chú |
|---|---|---|
| Runtime | **Node.js LTS (≥ 20)** | |
| Framework | **React Router 7** (full-stack mode) | Remix đã merge vào React Router. Template mới: `shopify-app-template-react-router`. App cũ vẫn chạy Remix v2 — biết cả hai |
| UI | **React 18+** + **Polaris web components** + **App Bridge** | Polaris unified (stable từ 10/2025), load từ CDN, dạng web component |
| Ngôn ngữ | **TypeScript** | Gần như bắt buộc trong production |
| Shopify SDK | `@shopify/shopify-app-react-router`, `@shopify/shopify-api` | Lo auth, session, webhook, API client |
| ORM | **Prisma** | Template dùng sẵn |
| DB | **PostgreSQL** (prod), SQLite (chỉ local) | |
| Queue | BullMQ + Redis / Inngest / QStash / SQS | Cho webhook & job nặng |
| Function | **Rust** (hoặc JS) → WebAssembly | Chỉ khi dùng Shopify Functions |
| GraphQL tooling | `@shopify/api-codegen-preset` | Sinh TypeScript types từ schema |

### Vì sao Shopify chọn React Router/Remix?

- **Server-side rendering** → app load nhanh trong iframe (merchant rất nhạy với tốc độ).
- **Loader/action** → tách rõ server code và client code, access token **không bao giờ** rò xuống browser.
- **Nested routing + progressive enhancement** → hợp với UI dạng dashboard.

Nhưng **không bắt buộc**. Có thể dùng Next.js, Nuxt, Laravel, Rails, Django, .NET… Shopify cung cấp
library cho Node, Ruby, PHP, Python. Chỉ là bạn sẽ phải tự làm nhiều việc mà template đã lo.

### Cấu trúc project từ template

```
my-app/
├── app/
│   ├── routes/
│   │   ├── app.tsx                    # layout embedded (App Bridge, nav menu)
│   │   ├── app._index.tsx             # trang chủ app
│   │   ├── app.settings.tsx
│   │   ├── auth.$.tsx                 # auth routes
│   │   ├── webhooks.app.uninstalled.tsx
│   │   └── webhooks.app.scopes_update.tsx
│   ├── shopify.server.ts              # cấu hình shopifyApp()
│   ├── db.server.ts                   # Prisma client
│   ├── root.tsx
│   └── entry.server.tsx               # set CSP frame-ancestors
├── extensions/                        # deploy lên Shopify
│   └── my-theme-extension/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── shopify.app.toml                   # config app
├── shopify.web.toml
├── package.json
├── vite.config.ts
└── Dockerfile
```

### TypeScript codegen cho GraphQL

```bash
npm run graphql-codegen   # sinh types từ Admin API schema
```
```ts
// Nhờ codegen, query sai field sẽ báo lỗi lúc compile thay vì lúc chạy
const response = await admin.graphql(
  `#graphql
   query getProduct($id: ID!) {
     product(id: $id) { id title }
   }`,
  { variables: { id } }
);
```

> Đây là một trong những cải thiện chất lượng lớn nhất cho app thật. Bật từ đầu.

---

## 5.3. Deployment, Hosting & Environment

### Hai đích deploy (nhắc lại vì quan trọng)

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│ Backend + App Home          │        │ Extensions + app config      │
│ → hosting của BẠN           │        │ → SHOPIFY                    │
│   Fly.io / Railway / Render │        │   `shopify app deploy`       │
│   Vercel / AWS / GCP        │        │   tạo một "app version"      │
└─────────────────────────────┘        └──────────────────────────────┘
        Deploy độc lập — phải tương thích ngược giữa hai bên!
```

### Chọn hosting

| Hosting | Phù hợp | Lưu ý |
|---|---|---|
| **Fly.io** | Lựa chọn phổ biến cho Shopify app | Có Postgres, chạy long-running process, deploy nhiều region |
| **Railway / Render** | Dễ bắt đầu | Có managed Postgres |
| **Vercel** | Frontend-heavy | Serverless → cần DB bên ngoài (Neon/Supabase), chú ý cold start & timeout cho webhook |
| **AWS / GCP** | Enterprise, scale lớn | Cấu hình nhiều hơn |
| **Heroku** | Legacy | Vẫn dùng được |

### Environment variables tối thiểu

```bash
SHOPIFY_API_KEY=                 # = client_id trong shopify.app.toml
SHOPIFY_API_SECRET=              # ⚠ TUYỆT ĐỐI không để lộ
SHOPIFY_APP_URL=https://my-app.fly.dev
SCOPES=write_products,read_orders
DATABASE_URL=postgresql://...
SESSION_SECRET=                  # random, để mã hoá
NODE_ENV=production
```

### Checklist go-live

- [ ] Đổi SQLite → **PostgreSQL**, chạy `prisma migrate deploy`
- [ ] `application_url` trỏ domain production, không phải tunnel
- [ ] Mọi `redirect_urls` dùng HTTPS domain production
- [ ] Webhook URI trỏ production
- [ ] `shopify app deploy` đã chạy
- [ ] Health check endpoint (`/healthz`)
- [ ] Error monitoring (Sentry) + structured logging (gắn `shop` vào mọi log)
- [ ] Database backup tự động
- [ ] Graceful shutdown (đóng connection, xử lý nốt job)
- [ ] Rate-limit handling + retry với backoff
- [ ] Rollback plan (app version của Shopify + previous deploy của hosting)

### Common mistakes

- ❌ Deploy backend trước extension (hoặc ngược lại) mà không nghĩ tới tương thích → extension gọi API chưa có.
- ❌ Quên chạy migration → app 500 ngay sau deploy.
- ❌ Serverless với cold start > 5s cho webhook endpoint → Shopify timeout.
- ❌ Không có staging app riêng → test trên production app của khách.

---

## 5.4. Billing & Subscription

### Hai lựa chọn hiện nay

| | **Shopify App Pricing** (managed pricing) | **Manual Pricing** (Billing API) |
|---|---|---|
| Ai dùng | **Mặc định cho public app mới** | App có model Shopify chưa hỗ trợ, hoặc one-time charge |
| Thiết lập ở đâu | Partner Dashboard / form submit app | Code trong app |
| Shopify lo giúp | Trang chọn plan, trial, proration, upgrade/downgrade | Không — bạn tự build |
| Hỗ trợ | Free, monthly, annual, một số kiểu usage-based | Recurring + **one-time charge** |
| Usage charge | Qua **App Events API** | Qua Billing API |
| One-time charge | **Không hỗ trợ** | Có |

> Shopify khuyến nghị **Shopify App Pricing** cho public app mới. Tích hợp Billing API cũ vẫn chạy,
> không bắt buộc migrate.

### Key concepts

- **Recurring application charge** — subscription theo tháng/năm, Shopify thu hộ qua hoá đơn của merchant.
- **Usage charge** — tính theo lượng dùng, có `cappedAmount`.
- **Trial** — số ngày dùng thử.
- **Test charge** — trên development store, charge luôn ở chế độ test (không mất tiền thật).
- **Custom app không dùng được Shopify Billing** — phải tự thu tiền ngoài.

### Practical example — Manual Pricing (hiểu cơ chế)

```ts
// app/routes/app.pricing.tsx
const { billing } = await authenticate.admin(request);

// Chặn truy cập nếu chưa trả tiền; chưa có thì redirect tới trang confirm của Shopify
await billing.require({
  plans: ["Pro Plan"],
  isTest: process.env.NODE_ENV !== "production",
  onFailure: async () =>
    billing.request({
      plan: "Pro Plan",
      isTest: process.env.NODE_ENV !== "production",
      returnUrl: `${process.env.SHOPIFY_APP_URL}/app`,
    }),
});
```

```ts
// shopify.server.ts
const shopify = shopifyApp({
  // ...
  billing: {
    "Pro Plan": {
      amount: 29,
      currencyCode: "USD",
      interval: BillingInterval.Every30Days,
      trialDays: 14,
    },
  },
});
```

### Common mistakes

- ❌ Tự thu tiền bằng Stripe cho public app → **vi phạm chính sách App Store** (trừ ngoại lệ được duyệt).
- ❌ Không xử lý khi merchant **huỷ** subscription → app vẫn cho dùng free mãi. Lắng nghe `app_subscriptions/update`.
- ❌ Không set `isTest: true` khi dev → tạo charge thật.
- ❌ Chặn toàn bộ app sau trial, kể cả trang billing → merchant không trả tiền được.
- ❌ Quên hiển thị giá rõ ràng trong listing → fail App Review.

---

## 5.5. Security

### Checklist bảo mật bắt buộc

| Hạng mục | Yêu cầu |
|---|---|
| **Session token** | Verify JWT: signature (HMAC-SHA256 bằng API secret), `aud` = API key, `dest` = shop, `exp`/`nbf` |
| **Webhook** | Verify HMAC trên **raw body**, so sánh **timing-safe** |
| **OAuth callback** | Verify HMAC của query params + verify `state` (chống CSRF) |
| **Access token** | Mã hoá at-rest, không bao giờ gửi xuống client, không log |
| **Secrets** | Chỉ trong env var / secret manager. Không trong git, không trong client bundle |
| **Scopes** | Least privilege |
| **Multi-tenancy** | Mọi query filter theo `shop`. Không bao giờ tin `shop` từ request body |
| **CSP** | `frame-ancestors https://{shop} https://admin.shopify.com` |
| **HTTPS** | Bắt buộc mọi nơi. TLS ≥ 1.2 |
| **Input validation** | Validate & sanitize mọi input (dùng zod) |
| **Public endpoints** (theme extension gọi) | Rate limit, CORS hẹp, không trả dữ liệu nhạy cảm |
| **Dependencies** | `npm audit`, Dependabot |
| **Protected customer data** | Nếu dùng `read_customers`/`read_orders`: có privacy policy, mã hoá, data retention, chỉ thu thập cái cần |
| **Logging** | Không log token, PII, payload đầy đủ của customer |

### Lỗ hổng phổ biến nhất của Shopify app

```ts
// ❌ SAI NGHIÊM TRỌNG — tin shop từ client
export const action = async ({ request }) => {
  const { shop, productId } = await request.json();
  // ai cũng có thể gửi shop = "competitor.myshopify.com"
  const session = await db.session.findFirst({ where: { shop } });
  // → đọc/ghi dữ liệu store của người khác
};

// ✅ ĐÚNG — shop luôn lấy từ session đã xác thực
export const action = async ({ request }) => {
  const { session, admin } = await authenticate.admin(request);
  const shop = session.shop;  // Shopify đảm bảo
  const { productId } = await request.json();
};
```

---

## 5.6. Performance & Scalability

| Vấn đề | Giải pháp |
|---|---|
| App load chậm trong iframe | SSR, code splitting, giảm bundle, dùng Polaris từ CDN |
| Query Admin API chậm/throttled | Chỉ query field cần; tránh N+1; dùng cursor pagination; cache dữ liệu ít đổi |
| Webhook dồn dập (flash sale) | Nhận → trả 200 ngay → queue; worker scale ngang |
| Sync dataset lớn | **Bulk Operations** thay vì vòng lặp pagination |
| Nhiều shop cùng lúc | Rate limit là **per shop** → tách bucket theo shop; đừng để 1 shop làm nghẽn shop khác |
| DB chậm | Index theo `shop` + cột lọc chính; connection pooling (PgBouncer) |
| Cold start serverless | Dùng container long-running cho webhook endpoint |

**Built for Shopify (BFS)** còn đặt ngưỡng performance cụ thể (LCP của app admin, ảnh hưởng của theme extension
tới Core Web Vitals của storefront). Nếu nhắm BFS badge, đo từ đầu.

---

## 5.7. App Review & Platform Requirements

### Quy trình

```
1. Hoàn thiện app trên development store
2. Partner Dashboard → App → Distribution → chọn Public
3. Điền App listing: tên, mô tả, screenshot, demo video, pricing, privacy policy
4. Cung cấp test credentials + hướng dẫn cho reviewer (CỰC KỲ quan trọng)
5. Submit → Automated checks → Manual review (vài ngày đến vài tuần)
6. Nhận feedback → sửa → resubmit (reject lần đầu là bình thường)
7. Approved → app lên App Store
```

### Những điểm bị reject nhiều nhất

| Lý do reject | Cách tránh |
|---|---|
| Thiếu/không hoạt động compliance webhooks | Implement đủ 3 GDPR webhooks + test |
| Xin scope thừa | Least privilege, giải thích từng scope |
| App không embedded hoặc không dùng App Bridge/Polaris | Theo template chính thức |
| UX không nhất quán với Admin | Dùng Polaris, không tự chế design system |
| Không có onboarding rõ ràng | Thêm bước setup hướng dẫn sau khi cài |
| Lỗi khi cài / cài không được | Test luồng install trên dev store **sạch** |
| Billing không khớp với listing | Đồng bộ giá trong app và listing |
| Chèn script trực tiếp vào theme | Dùng theme app extension |
| Không xoá dữ liệu khi uninstall | Implement `app/uninstalled` + `shop/redact` |
| Screenshot/mô tả gây hiểu nhầm | Trung thực |
| Thiếu hướng dẫn test cho reviewer | Viết step-by-step + tài khoản test |

### Built for Shopify (BFS)

Badge cao hơn mức "approved". Yêu cầu thêm về performance, embedded UX, adoption, review score, support.
**Không cần quan tâm khi mới học**, nhưng nếu làm sản phẩm thương mại thì nên đọc tiêu chí sớm để không phải refactor.

---

## 5.8. Tóm tắt chương

- CLI lo scaffold/dev/deploy. `shopify app dev` cho tunnel; `shopify app deploy` đẩy extension + config.
- Stack chuẩn: Node + **React Router 7** + TypeScript + Prisma + Postgres + Polaris web components + App Bridge.
- **Hai đích deploy độc lập**: backend (hosting của bạn) và extensions (Shopify).
- Billing: **Shopify App Pricing** cho app mới; Billing API ("Manual Pricing") cho one-time & model đặc biệt.
- Security: verify mọi thứ (session token, webhook HMAC, OAuth HMAC+state), least privilege, **shop luôn lấy từ session**.
- App Review: compliance webhooks, scope tối thiểu, embedded UX, hướng dẫn test cho reviewer.

**Tiếp theo:** [Chương 6 — Learning Path & Resources](06-learning-path-and-resources.md)
