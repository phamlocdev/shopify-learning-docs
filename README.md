# Fundamental Knowledge of Shopify App Development

> Tài liệu nền tảng cho developer mới bắt đầu với Shopify App Development.
> Biên soạn: 2026-11-03. Mốc kiểm chứng: **Admin API version 2026-10** (stable mới nhất).

---

## 0. Đọc tài liệu này như thế nào

Tài liệu chia thành 6 chương. **Đọc tuần tự lần đầu**, sau đó dùng như reference.

| # | Chương | Nội dung chính |
|---|--------|----------------|
| 1 | [Ecosystem, Accounts & Stores](docs/01-ecosystem-and-accounts.md) | Shopify là gì, Partner account, Development store, các "surface" của platform |
| 2 | [App Types & Architecture](docs/02-app-types-and-architecture.md) | Public/Custom app, Embedded app, App Bridge, Polaris, luồng request thực tế |
| 3 | [Authentication & APIs](docs/03-auth-and-apis.md) | OAuth, Token Exchange, Session token, Scopes, GraphQL Admin API, Storefront API |
| 4 | [Webhooks, Extensions & Data](docs/04-webhooks-extensions-data.md) | Event-driven, webhook reliability, App Extensions, Metafields, Database & Session storage |
| 5 | [Stack, Deploy, Billing, Security](docs/05-stack-deploy-billing-security.md) | React Router 7, Prisma, hosting, Billing, security, performance, App Review |
| 6 | [Learning Path & Resources](docs/06-learning-path-and-resources.md) | Lộ trình học, mini-projects, capstone project, link tài liệu chính thức |
| A | [Phụ lục A — End-to-end Lifecycle](docs/appendix-a-end-to-end-lifecycle.md) | Vòng đời đầy đủ của một app thật, từ lúc tạo app đến lúc merchant gỡ — đọc sau chương 5 |

---

## 1. Learning Roadmap & Knowledge Overview

### 1.1. Bản đồ các knowledge domains

Shopify App Development không phải "một công nghệ". Nó là **một web app bình thường** (Node.js + React + DB)
được cắm vào **một platform có luật chơi riêng** (auth, API, extension points, review process).
Phần khó với newbie gần như luôn là phần "luật chơi", không phải phần code.

```
                    ┌─────────────────────────────────────────┐
                    │   D0. WEB FUNDAMENTALS (prerequisite)   │
                    │   HTTP · REST · JSON · JS/TS · React    │
                    │   Node.js · SQL · Git · env vars        │
                    └────────────────────┬────────────────────┘
                                         │
                    ┌────────────────────▼────────────────────┐
                    │   D1. SHOPIFY ECOSYSTEM & ACCOUNTS      │
                    │   Store · Merchant · Partner · Dev store│
                    └────────────────────┬────────────────────┘
                                         │
                    ┌────────────────────▼────────────────────┐
                    │   D2. APP MODEL & ARCHITECTURE          │
                    │   Public vs Custom · Embedded · iframe  │
                    └──────┬──────────────────────────┬───────┘
                           │                          │
            ┌──────────────▼───────────┐   ┌──────────▼──────────────┐
            │  D3. AUTH & AUTHZ        │   │  D5. EXTENSIONS         │
            │  OAuth · Token Exchange  │   │  Theme · Checkout UI    │
            │  Session token · Scopes  │   │  Admin UI · Functions   │
            └──────────────┬───────────┘   └──────────┬──────────────┘
                           │                          │
            ┌──────────────▼───────────┐              │
            │  D4. DATA ACCESS         │              │
            │  GraphQL Admin API       │◄─────────────┘
            │  Storefront API · Bulk   │
            └──────────────┬───────────┘
                           │
            ┌──────────────▼───────────┐   ┌─────────────────────────┐
            │  D6. WEBHOOKS / EVENTS   │   │  D7. PERSISTENCE        │
            │  HMAC · idempotency      │◄─►│  Session storage · DB   │
            │  compliance webhooks     │   │  Metafields             │
            └──────────────┬───────────┘   └──────────┬──────────────┘
                           │                          │
            ┌──────────────▼──────────────────────────▼──────────────┐
            │  D8. TOOLING & STACK                                   │
            │  Shopify CLI · React Router 7 · Prisma · TypeScript    │
            └──────────────────────────┬─────────────────────────────┘
                                       │
            ┌──────────────────────────▼─────────────────────────────┐
            │  D9. PRODUCTION                                        │
            │  Hosting · Billing · Security · Perf · App Review      │
            └────────────────────────────────────────────────────────┘
```

### 1.2. Mối liên hệ giữa các domain (đọc như một câu chuyện)

Một merchant có **Store** (D1). Bạn — developer — có **Partner account** (D1) để tạo app.
App của bạn là **Public hoặc Custom** (D2), và hầu như luôn là **Embedded** — chạy trong iframe của Shopify Admin (D2).
Merchant cài app → Shopify chạy **OAuth / Token Exchange** để cấp **access token** bị giới hạn bởi **scopes** (D3).
Có token, app gọi **GraphQL Admin API** để đọc/ghi dữ liệu store (D4).
App không thể "hỏi liên tục" xem có gì thay đổi → đăng ký **Webhooks** để Shopify chủ động báo (D6).
Muốn xuất hiện trên storefront / checkout / admin page → dùng **App Extensions** (D5).
Token, session, dữ liệu riêng của app phải **lưu ở DB của bạn** (D7).
Tất cả được dựng bằng **Shopify CLI + React Router template** (D8), rồi **deploy, tính tiền, bảo mật, submit review** (D9).

### 1.3. Phân loại kiến thức

#### Must-have — không hiểu thì không viết nổi app

| Chủ đề | Lý do bắt buộc |
|---|---|
| Partner account + Development store | Không có thì không test được gì |
| Public vs Custom app | Quyết định cả cách deploy lẫn cách phân phối |
| Embedded app + App Bridge | ~95% app thực tế là embedded; iframe gây rất nhiều bug lạ |
| OAuth / Token Exchange / access token | Mọi API call đều cần token |
| Access scopes | Sai scope = API trả lỗi, merchant phải re-approve |
| Session token (JWT) verification | Đây là lớp bảo mật chính của embedded app |
| GraphQL Admin API | API chính thức duy nhất cho public app mới |
| API versioning (2026-10) | Pin sai version = app chết sau ~12 tháng |
| Webhooks + HMAC verification | Bắt buộc có `app/uninstalled` + compliance webhooks |
| Shopify CLI workflow | Tooling chuẩn: scaffold, dev tunnel, deploy |
| Session storage (Prisma/DB) | Mất token = app hỏng |
| `shopify.app.toml` | File cấu hình trung tâm của app |

#### Should-have — cần cho app chạy thật

Rate limiting & cost-based throttling · Bulk Operations · Cursor pagination · Metafields & Metaobjects ·
Theme App Extensions · Admin UI Extensions · Billing (Shopify App Pricing) · Deployment & hosting ·
Background jobs / queue · Error monitoring & logging · App Review requirements · Polaris web components ·
Compliance (GDPR) webhooks · Online vs Offline tokens.

#### Advanced — học sau, khi có nhu cầu thật

Shopify Functions (Rust/Wasm: discount, cart transform, delivery/payment customization) ·
Checkout UI Extensions & Checkout Extensibility · Customer Account UI Extensions · Web Pixels ·
POS UI Extensions · Shopify Flow triggers/actions · Payments Apps · Sales Channel apps ·
Hydrogen/Oxygen (headless storefront) · Multi-region scaling · B2B & Markets ·
Built for Shopify (BFS) certification.

### 1.4. Thứ tự học đề xuất (tổng quan)

```
Tuần 1   D0 ôn lại  +  D1 ecosystem  +  D2 app model
Tuần 2   D8 CLI → scaffold app chạy được  +  D3 auth
Tuần 3   D4 GraphQL Admin API  +  D7 persistence
Tuần 4   D6 webhooks  +  D5 extension đầu tiên (theme app extension)
Tuần 5   D9 billing + deploy production
Tuần 6+  Capstone project + advanced domains
```

Chi tiết từng tuần, mini-project và tiêu chí "đã học xong" nằm ở
[chương 6](docs/06-learning-path-and-resources.md).

---

## 2. Cảnh báo về tài liệu lỗi thời

Shopify thay đổi nhanh. Khi đọc blog/tutorial trên mạng, **luôn kiểm tra ngày đăng**.
Dấu hiệu tutorial đã lỗi thời:

| Dấu hiệu trong tutorial | Trạng thái hiện tại (11/2026) |
|---|---|
| Dùng `shopify.rest.resources` / REST Admin API | REST là **legacy từ 2024-10-01**. Từ **2025-04-01**, mọi **public app mới bắt buộc dùng GraphQL Admin API**. |
| Tự implement OAuth redirect `/auth` + `/auth/callback` cho embedded app | Đã thay bằng **Shopify managed installation + Token Exchange**. Scopes khai trong `shopify.app.toml`. |
| App Bridge v2 với `createApp()` / `Provider` từ `@shopify/app-bridge-react` v2 | App Bridge hiện tại load qua `<script src="https://cdn.shopify.com/shopifycloud/app-bridge.js">` và dùng web components. |
| `@shopify/polaris` React components cho app mới | Polaris đã **unified sang web components** (stable từ 2025-10), load từ CDN. Polaris React vẫn chạy nhưng không còn là hướng đi mới. |
| Template `shopify-app-template-remix` | Remix đã merge vào React Router. Template chính thức cho app mới: **`shopify-app-template-react-router`**. |
| `shopify.app.toml` không có `access_scopes` | Managed installation yêu cầu khai scopes trong file này. |
| Tự viết pricing page + `appSubscriptionCreate` | Public app mới nên dùng **Shopify App Pricing (managed pricing)**; Billing API nay gọi là "Manual Pricing", chỉ cần cho one-time charge & model đặc biệt. |
| API version `2023-xx`, `2024-xx`, `2025-04/07/10` | Đã hết hỗ trợ. Dùng `2026-10`. |

> Quy tắc: **shopify.dev luôn thắng blog**. Nếu mâu thuẫn, tin shopify.dev và changelog.

---

## 3. Expected outcome checklist

Sau khi đọc hết tài liệu + làm hết mini-project, bạn phải tự trả lời được:

- [ ] Vẽ được sơ đồ: merchant bấm "Install" → điều gì xảy ra, ai gọi ai, token sinh ra ở đâu, lưu ở đâu.
- [ ] Giải thích sự khác nhau giữa **session token (ID token)** và **access token**.
- [ ] Giải thích vì sao embedded app **không nên** dựa vào cookie session như web app thường.
- [ ] Viết được một GraphQL mutation có xử lý `userErrors` và một query có cursor pagination.
- [ ] Nói được vì sao phải verify HMAC của webhook và vì sao handler phải **idempotent**.
- [ ] Phân biệt khi nào lưu data vào **metafield** và khi nào lưu vào **DB của app**.
- [ ] Biết app của bạn sẽ fail App Review ở điểm nào nếu submit hôm nay.

---

## Nguồn chính thức (bookmark ngay)

- Shopify Dev Docs — https://shopify.dev/docs
- Build an app — https://shopify.dev/docs/apps/build
- GraphQL Admin API reference — https://shopify.dev/docs/api/admin-graphql
- API versioning — https://shopify.dev/docs/api/usage/versioning
- Developer changelog — https://shopify.dev/changelog
- Partner Dashboard — https://partners.shopify.com
- Shopify Dev Community — https://community.shopify.dev
