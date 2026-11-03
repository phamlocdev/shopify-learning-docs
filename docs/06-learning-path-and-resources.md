# Chương 6 — Hands-on Learning Path & Learning Resources

---

## 6.1. Prerequisites — học trước khi chạm vào Shopify

Đừng học Shopify khi còn yếu những thứ này; bạn sẽ không phân biệt được "lỗi do mình" và "lỗi do platform".

| Kỹ năng | Mức cần đạt | Kiểm tra bằng cách |
|---|---|---|
| **HTTP** | Hiểu method, status code, header, CORS, redirect | Giải thích được 302 vs 307, preflight request |
| **JavaScript ES2020+** | async/await, Promise, module, destructuring | Viết được hàm retry với exponential backoff |
| **TypeScript** | Type, interface, generic cơ bản, `unknown` vs `any` | Đọc hiểu được type của một SDK |
| **React** | Component, hooks, state, effect | Viết form có validation |
| **Node.js** | Đọc env, chạy server, npm scripts | Dựng Express server trả JSON |
| **SQL + ORM** | SELECT/JOIN/INDEX, migration | Thiết kế schema 3 bảng có khoá ngoại |
| **GraphQL** | Query, mutation, variable, fragment | Viết query lồng 2 cấp |
| **Git** | Branch, PR, rebase cơ bản | |
| **Bảo mật cơ bản** | HMAC, JWT, CSRF, XSS, secret management | Giải thích được vì sao so sánh HMAC phải timing-safe |

**Nếu thiếu GraphQL** — đây là thứ nên bù ngay vì toàn bộ Admin API là GraphQL. 1–2 ngày là đủ:
https://graphql.org/learn/

---

## 6.2. Lộ trình 6 tuần (mỗi tuần ~10–15 giờ)

### Tuần 1 — Hiểu platform, chưa code

**Mục tiêu:** nói được app Shopify là gì mà không nhìn tài liệu.

- Đọc [Chương 1](01-ecosystem-and-accounts.md) và [Chương 2](02-app-types-and-architecture.md).
- Tạo Partner account + 1 development store, thêm 20 sản phẩm mẫu, bật Bogus Gateway, tạo vài đơn test.
- Cài 2–3 app miễn phí từ App Store vào dev store. **Quan sát như một developer:**
  - Màn hình xin quyền hiện scope gì?
  - App nằm trong iframe hay tab riêng?
  - Mở DevTools → Network: app gọi những endpoint nào? Header gì?
  - App có block trong Theme Editor không?
- Đọc 20 mục gần nhất ở https://shopify.dev/changelog để cảm nhận nhịp thay đổi.

✅ **Xong khi:** vẽ được bằng tay sơ đồ "merchant – Shopify – app server" và chỉ ra đâu là iframe,
đâu là API call, đâu là webhook.

---

### Tuần 2 — App chạy được + hiểu Auth

**Mục tiêu:** app của bạn hiện trong Admin và đọc được dữ liệu thật.

```bash
npm install -g @shopify/cli@latest
shopify app init --template=https://github.com/Shopify/shopify-app-template-react-router
cd my-app
npm install
shopify app dev
```

- Mở preview URL, cài app vào dev store.
- **Đọc code template, từng file**, đặc biệt:
  - `app/shopify.server.ts` — cấu hình
  - `app/routes/app.tsx` — layout embedded
  - `app/routes/app._index.tsx` — ví dụ gọi GraphQL
  - `app/entry.server.tsx` — CSP header
  - `shopify.app.toml`
- Đọc [Chương 3](03-auth-and-apis.md) song song.
- **Bài tập hiểu auth:** trong một loader, `console.log(session)`. Nhìn `accessToken`, `scope`, `isOnline`, `expires`.
  Rồi mở DevTools → Network → xem header `Authorization: Bearer ...`, decode JWT đó ở jwt.io
  và đối chiếu `aud`, `dest`, `exp`.

> 🧪 **Mini-project 1 — "Store Dashboard"**
> Trang hiển thị: tên shop, plan, số sản phẩm, 5 đơn gần nhất, tổng doanh thu 7 ngày.
> Yêu cầu: dùng 1 GraphQL query duy nhất cho phần shop info; UI bằng Polaris web components;
> có loading state và error state.
> *Học được:* authenticate.admin, GraphQL query, Polaris, SSR loader.

✅ **Xong khi:** giải thích được dòng `const { admin, session } = await authenticate.admin(request)` làm gì, từng bước.

---

### Tuần 3 — GraphQL sâu + Database

**Mục tiêu:** ghi dữ liệu vào Shopify và lưu dữ liệu riêng của app.

- Dùng **GraphiQL Explorer** (CLI in ra link khi chạy `shopify app dev`) để thử query trước khi viết code.
- Học: mutation + `userErrors`, cursor pagination, `extensions.cost`, metafields.
- Bật `npm run graphql-codegen`.
- Sửa Prisma schema: thêm bảng nghiệp vụ của bạn, **luôn có cột `shop` + index**.

> 🧪 **Mini-project 2 — "Bulk Product Tagger"**
> Chọn nhiều sản phẩm → thêm/xoá tag hàng loạt → lưu lịch sử thao tác vào DB → có nút undo.
> Yêu cầu: pagination đầy đủ, xử lý `userErrors`, xử lý throttling với retry, resource picker của App Bridge.
> *Học được:* mutation, pagination, rate limit, Prisma, multi-tenancy.

✅ **Xong khi:** xử lý được store có 500+ sản phẩm mà không bị throttled.

---

### Tuần 4 — Webhooks + Extension đầu tiên

- Đọc [Chương 4](04-webhooks-extensions-data.md).
- Thêm webhook `products/update`, `orders/create`, `app/uninstalled` + 3 compliance webhooks.
- Test bằng `shopify app webhook trigger`, và bằng thao tác thật trong Admin.
- Cố tình trả 500 một lần để quan sát retry.

> 🧪 **Mini-project 3 — "Low Stock Alert"**
> Nghe `inventory_levels/update` → nếu tồn kho dưới ngưỡng merchant cấu hình → ghi alert vào DB
> + gửi email. Có trang cấu hình ngưỡng theo từng sản phẩm.
> Yêu cầu: HMAC verify, idempotency bằng `X-Shopify-Webhook-Id`, xử lý bất đồng bộ (queue hoặc ít nhất là
> trả 200 trước rồi `await` sau), và một reconciliation job chạy hằng ngày đối chiếu bằng API.
> *Học được:* event-driven, idempotency, background job, độ tin cậy.

> 🧪 **Mini-project 4 — "Product Badge" (theme app extension)**
> App block hiển thị badge ("Bán chạy", "Sắp hết hàng") trên product page, cấu hình trong Theme Editor,
> dữ liệu đọc từ metafield do app ghi.
> *Học được:* Liquid, theme app extension, metafields, deploy lên Shopify.

✅ **Xong khi:** webhook của bạn chịu được việc nhận trùng 3 lần mà dữ liệu vẫn đúng.

---

### Tuần 5 — Production

- Đọc [Chương 5](05-stack-deploy-billing-security.md).
- Đổi SQLite → Postgres (Neon/Supabase/Fly Postgres). Chạy migration.
- Deploy backend lên Fly.io (hoặc Railway). Cập nhật `application_url`, `redirect_urls`, webhook URI.
- `shopify app deploy`.
- Thêm Sentry + structured logging có `shop`.
- Thêm billing (test mode) với 1 plan free + 1 plan paid.

> 🧪 **Mini-project 5 — "Đưa mini-project 3 lên production"**
> Gỡ app khỏi dev store, cài lại từ đầu như một merchant mới. Mọi thứ phải chạy ngay, không cần can thiệp tay.
> *Học được:* deployment, env config, onboarding flow, cold install.

✅ **Xong khi:** uninstall → reinstall → app hoạt động bình thường, và `app/uninstalled` đã dọn session.

---

### Tuần 6+ — Capstone Project

> 🎯 **Capstone — "Smart Product Reviews & Social Proof"**
>
> Một app thương mại thực thụ, chạm vào gần như mọi domain đã học.

**Chức năng:**

1. **Thu thập review**
   - Webhook `orders/fulfilled` → sau N ngày, gửi email mời review (background job có lịch).
   - Trang submit review công khai (không cần login Shopify).
2. **Hiển thị trên storefront**
   - Theme app extension: block review list + block star rating tóm tắt.
   - Ghi `average_rating` và `review_count` vào **metafield** để theme/SEO đọc được.
3. **Quản lý trong Admin**
   - App Home: danh sách review, duyệt/từ chối, trả lời, lọc & tìm kiếm, phân trang.
   - **Admin block** trên trang product: hiện review của đúng sản phẩm đó.
   - **Admin action**: "Mời khách review" từ trang order.
4. **Billing**
   - Free: 50 review/tháng. Pro ($19): không giới hạn + tuỳ biến giao diện.
   - Chặn tính năng Pro khi ở plan Free, nhưng không chặn trang upgrade.
5. **Vận hành**
   - Compliance webhooks đầy đủ (xoá review của customer khi `customers/redact`).
   - Rate-limit cho endpoint submit review công khai.
   - Import/export CSV bằng **Bulk Operations**.
   - Onboarding: sau khi cài, hướng dẫn merchant bật app block trong Theme Editor (có deep link).

**Tiêu chí hoàn thành:**

- [ ] Cài từ đầu trên một dev store sạch, không lỗi
- [ ] Mọi bảng DB có `shop`, mọi query filter theo `shop`
- [ ] Webhook idempotent, verify HMAC, xử lý qua queue
- [ ] Không có `any` trong TypeScript ở code chính
- [ ] GraphQL codegen bật, không query thừa field
- [ ] Chịu được store có 10.000 đơn hàng (dùng Bulk Operations)
- [ ] Deploy production + monitoring
- [ ] Tự review theo checklist App Review ở chương 5 và liệt kê được điểm còn thiếu

---

## 6.3. Learning Resources

> Ưu tiên tuyệt đối: **shopify.dev**. Blog bên thứ ba chỉ dùng để tham khảo, luôn kiểm ngày đăng.

### Bắt đầu

| Tài liệu | Link | Đọc để hiểu gì |
|---|---|---|
| Build an app overview | https://shopify.dev/docs/apps/build | Bức tranh tổng thể — đọc hết, không skip |
| Getting started / Build your first app | https://shopify.dev/docs/apps/getting-started | Làm theo từ đầu đến cuối **một lần**, đừng copy-paste |
| App architecture overview | https://shopify.dev/docs/apps/build/architecture | Chốt lại mental model ở chương 2 |
| Shopify CLI for apps | https://shopify.dev/docs/api/shopify-cli/app | Tra lệnh |

### Authentication & Authorization

| Tài liệu | Link | Đọc phần nào |
|---|---|---|
| Authentication & authorization overview | https://shopify.dev/docs/apps/build/authentication-authorization | Toàn bộ — chương quan trọng nhất của docs |
| Token exchange | https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/token-exchange | Cơ chế hiện đại thay cho OAuth redirect |
| Session tokens | https://shopify.dev/docs/apps/build/authentication-authorization/session-tokens | Cấu trúc JWT và cách verify |
| Access scopes | https://shopify.dev/docs/api/usage/access-scopes | Danh sách scope + scope nào cần duyệt |
| Protected customer data | https://shopify.dev/docs/apps/launch/protected-customer-data | Bắt buộc nếu đụng dữ liệu khách hàng |

### APIs

| Tài liệu | Link | Đọc phần nào |
|---|---|---|
| GraphQL Admin API reference | https://shopify.dev/docs/api/admin-graphql | Dùng như từ điển; bookmark các object Product, Order, Customer, InventoryLevel |
| API versioning | https://shopify.dev/docs/api/usage/versioning | Chu kỳ release, cách pin version |
| Rate limits | https://shopify.dev/docs/api/usage/rate-limits | Cost-based throttling, cách đọc `throttleStatus` |
| Pagination | https://shopify.dev/docs/api/usage/pagination-graphql | Cursor pagination |
| Bulk operations | https://shopify.dev/docs/api/usage/bulk-operations/queries | Khi dataset lớn |
| GraphiQL Explorer app | cài vào dev store từ shopify.dev | Thử query trực tiếp trên dữ liệu thật |
| Storefront API | https://shopify.dev/docs/api/storefront | Khi làm headless / theme |
| REST Admin API (legacy) | https://shopify.dev/docs/api/admin-rest | Chỉ để đọc hiểu code cũ |

### Webhooks

| Tài liệu | Link | Đọc phần nào |
|---|---|---|
| Webhooks overview | https://shopify.dev/docs/apps/build/webhooks | Cơ chế, delivery method |
| Configure webhooks in TOML | https://shopify.dev/docs/apps/build/webhooks/subscribe/app-specific | Cách khai báo |
| Webhook best practices | https://shopify.dev/docs/apps/build/webhooks/best-practices | Idempotency, reliability — **đọc kỹ** |
| Privacy law compliance | https://shopify.dev/docs/apps/build/privacy-law-compliance | 3 compliance webhooks bắt buộc |

### Extensions & UI

| Tài liệu | Link | Đọc phần nào |
|---|---|---|
| List of app extensions | https://shopify.dev/docs/apps/build/app-extensions/list-of-app-extensions | Bảng tra: cần gì thì dùng extension nào |
| Theme app extensions | https://shopify.dev/docs/apps/build/online-store/theme-app-extensions | Học đầu tiên trong nhóm extension |
| App Bridge | https://shopify.dev/docs/api/app-bridge-library | API của `shopify` global, các web component |
| Polaris | https://shopify.dev/docs/api/app-home | Component reference cho App Home |
| Admin UI extensions | https://shopify.dev/docs/api/admin-extensions | Khi muốn nhúng vào trang resource |
| Shopify Functions | https://shopify.dev/docs/apps/build/functions | Advanced — discount, shipping, cart logic |
| Checkout UI extensions | https://shopify.dev/docs/api/checkout-ui-extensions | Advanced |

### Billing, Launch & Review

| Tài liệu | Link | Đọc phần nào |
|---|---|---|
| Billing | https://shopify.dev/docs/apps/launch/billing | So sánh Shopify App Pricing vs Manual Pricing |
| App Store requirements | https://shopify.dev/docs/apps/launch/app-requirements-checklist | **Đọc trước khi code**, không phải trước khi submit |
| App review process | https://shopify.dev/docs/apps/launch/app-review | Quy trình, thời gian, cách chuẩn bị |
| Built for Shopify | https://shopify.dev/docs/apps/launch/built-for-shopify | Khi nhắm badge |

### Code & Tooling

| Nguồn | Link | Dùng để |
|---|---|---|
| React Router app template | https://github.com/Shopify/shopify-app-template-react-router | **Template chính thức cho app mới** |
| shopify-app-js (monorepo) | https://github.com/Shopify/shopify-app-js | Đọc source của `authenticate.admin` để hiểu bên trong |
| Shopify Dev MCP | https://github.com/Shopify/dev-mcp | Cho AI coding assistant truy cập docs & schema Shopify mới nhất |
| Shopify Dev Assistant / docs search | https://shopify.dev | Tra nhanh |

### Cộng đồng & cập nhật

| Nguồn | Link | Tần suất |
|---|---|---|
| **Developer changelog** | https://shopify.dev/changelog | **Đọc hằng tuần** — nguồn số 1 để không bị lỗi thời |
| Shopify Dev Community | https://community.shopify.dev | Hỏi đáp kỹ thuật |
| Shopify Partners blog | https://www.shopify.com/partners/blog | Tin tức, best practice |
| Shopify Developers (YouTube) | https://www.youtube.com/@ShopifyDevs | Video chính thức, Shopify Editions |
| Shopify Editions | https://www.shopify.com/editions | 2 lần/năm — tổng hợp tính năng mới lớn |

---

## 6.4. Thói quen giúp bạn không bị lỗi thời

1. **Thứ Hai hằng tuần:** lướt https://shopify.dev/changelog, đọc mục có tag `Breaking`, `Deprecation`.
2. **Mỗi quý (1/1, 1/4, 1/7, 1/10):** API version mới ra → đọc release notes → lên lịch nâng `apiVersion`.
   Đừng để tới tháng cuối trước deadline 12 tháng.
3. **Mỗi khi đọc blog:** kiểm ngày đăng + đối chiếu với shopify.dev trước khi copy code.
4. **Khi gặp lỗi lạ:** tìm trên community.shopify.dev trước Stack Overflow — nội dung Shopify trên SO
   thường đã cũ 2–3 năm.

---

## 6.5. Lời khuyên cuối

- **Đừng học hết rồi mới làm.** Làm mini-project song song với đọc. Kiến thức Shopify rất khó nhớ nếu không gõ tay.
- **Đọc code của template cho kỹ.** Nó là bản mẫu best practice do chính Shopify viết. Hiểu nó = hiểu 60% nghề.
- **Mô phỏng tình huống xấu:** webhook trùng, token hết hạn, merchant uninstall rồi reinstall, store 50k sản phẩm,
  API throttled. App thật sống chết ở những tình huống này, không phải ở happy path.
- **Phần khó nhất không phải code** — là hiểu đúng ràng buộc của platform. Khi bí, 90% câu trả lời nằm trên shopify.dev.

Xem thêm: [Phụ lục A — Vòng đời app từ đầu đến cuối](appendix-a-end-to-end-lifecycle.md)
