# CAMPE – Affiliate ID Security Requirements

Tài liệu này là **yêu cầu bắt buộc** cho Codex khi triển khai module tạo link Shopee Affiliate cho CAMPE.

## 1. Nguồn chính thức phải đọc trước khi code

Shopee Help Center:

https://help.shopee.vn/portal/10/article/172955

Theo tài liệu Shopee, link Affiliate được xây từ:

```text
https://s.shopee.vn/an_redir
?origin_link=<SHOPEE_LANDING_PAGE_URL_ENCODED>
&affiliate_id=<CAMPE_AFFILIATE_ID>
&sub_id=<VALUE1>-<VALUE2>-<VALUE3>-<VALUE4>-<VALUE5>
```

Tài liệu Shopee xác nhận:

- `origin_link` là trang đích Shopee được URL encode.
- `affiliate_id` là Affiliate ID của tài khoản Affiliate.
- `sub_id` dùng để truyền tracking động.
- Link `an_redir` có thể được bọc bởi short-link của hệ thống riêng.

## 2. Nguyên tắc bảo mật quan trọng nhất

**Affiliate ID của CAMPE phải được quyết định hoàn toàn ở SERVER.**

Tuyệt đối không lấy `affiliate_id` từ:

- query string của trình duyệt;
- request body;
- form;
- cookie;
- localStorage/sessionStorage;
- header do client tự gửi;
- URL sản phẩm do client cung cấp;
- dữ liệu JavaScript phía frontend.

Frontend không được gửi `affiliate_id` lên backend.

Backend phải lấy Affiliate ID duy nhất từ cấu hình đáng tin cậy của server, ví dụ:

```env
SHOPEE_AFFILIATE_ID=YOUR_REAL_AFFILIATE_ID
```

Có thể dùng secret manager của hạ tầng thay cho `.env` production.

**Không commit Affiliate ID production hoặc file `.env` production vào GitHub.**

Affiliate ID có thể xuất hiện trong URL redirect của Shopee theo cơ chế Affiliate, nên không nên coi bản thân giá trị ID là một mật khẩu. Điều phải bảo vệ là **quyền quyết định Affiliate ID mà server dùng để tạo redirect**.

## 3. Client chỉ được gửi ID nội bộ của listing

API tạo click nên nhận tối thiểu:

```json
{
  "listing_id": "internal-listing-id"
}
```

Có thể nhận thêm metadata analytics đã whitelist như vị trí hiển thị hoặc source hợp lệ, nhưng KHÔNG nhận:

```json
{
  "affiliate_id": "...",
  "sub_id": "...",
  "origin_link": "...",
  "redirect_url": "..."
}
```

Nếu client cố gửi các field này, backend phải **ignore hoặc reject** theo schema strict.

Khuyến nghị schema API dùng `additionalProperties: false` / strict validation.

## 4. `origin_link` cũng phải do server quyết định

Không chỉ `affiliate_id`; `origin_link` cũng không được tin từ client.

Flow đúng:

```text
Client gửi listing_id
        ↓
Backend đọc listing trong database
        ↓
Backend lấy canonical Shopee URL đã lưu cho listing
        ↓
Backend validate URL
        ↓
Backend lấy SHOPEE_AFFILIATE_ID từ server config
        ↓
Backend tự build sub_id
        ↓
Backend tự build an_redir URL
        ↓
Redirect
```

Không làm:

```text
Client gửi destination_url + affiliate_id
        ↓
Backend ghép chuỗi rồi redirect
```

Cách sai trên tạo nguy cơ open redirect và cho phép attacker thay Affiliate ID.

## 5. Whitelist domain Shopee

Trước khi tạo redirect, backend phải parse URL bằng thư viện URL chuẩn và kiểm tra:

- scheme bắt buộc `https`;
- hostname nằm trong allowlist cấu hình;
- không kiểm tra bằng `contains("shopee.vn")`;
- không cho URL dạng `shopee.vn.evil.com`;
- không cho URL dạng `shopee.vn@evil.com`;
- không chấp nhận hostname sau decode khác hostname trước decode;
- không cho `javascript:`, `data:`, `file:` hoặc protocol khác.

Allowlist production phải cấu hình rõ ràng, ví dụ tùy dữ liệu thực tế:

```text
shopee.vn
www.shopee.vn
```

Nếu CAMPE cần thêm hostname Shopee khác, phải thêm có chủ đích sau khi kiểm chứng.

## 6. Canonicalize URL nguồn

Khi nhập một listing Shopee vào database:

- lưu URL canonical của sản phẩm;
- loại bỏ các tracking parameter không cần thiết;
- không lưu một affiliate link của người khác làm URL gốc;
- nếu có thể, derive URL chuẩn từ `shop_id/item_id` thay vì giữ URL tùy ý.

Mục tiêu là bảo đảm `origin_link` chỉ là trang Shopee đích, còn Affiliate ID của CAMPE chỉ được gắn tại bước redirect server-side.

## 7. Cấu trúc nhận dạng user

### Guest chưa đăng nhập

Tạo `guest_id` ngẫu nhiên đủ entropy và lưu first-party cookie.

Cookie khuyến nghị:

```text
HttpOnly
Secure
SameSite=Lax
Path=/
```

Không dùng email, số điện thoại, tên hoặc dữ liệu cá nhân làm guest ID.

### User đã đăng nhập

Mỗi tài khoản CAMPE có một `affiliate_user_code` ngẫu nhiên cố định.

Sau khi đăng nhập:

```text
account affiliate_user_code > guest_id
```

Tức là backend luôn ưu tiên ID gắn với account.

Không cho client tự chọn hoặc sửa `affiliate_user_code`.

## 8. Mỗi click phải có `click_code` riêng

Mỗi lần tạo outbound click hợp lệ:

```text
user_code = guest_id hoặc account affiliate_user_code
click_code = random unique code mới
```

Database lưu ánh xạ:

```text
affiliate_clicks
- id
- click_code UNIQUE
- user_type
- user_code
- account_id nullable
- listing_id
- product_id
- shop_id
- source
- displayed_price nullable
- created_at
- redirected_at nullable
```

`click_code` phải đủ khó đoán. Không dùng số tăng dần công khai như `1`, `2`, `3`.

## 9. `sub_id` do backend tự build

Codex không được nhận `sub_id` hoàn chỉnh từ frontend.

Ví dụ logical format:

```text
<USER_CODE>-<CLICK_CODE>-<SOURCE>-<PRODUCT_CODE>-<CAMPAIGN>
```

Backend phải sanitize/encode từng thành phần theo giới hạn Shopee hiện hành.

Nếu chưa xác minh giới hạn ký tự/độ dài của Shopee, để cấu hình/TODO và không tự bịa giới hạn.

## 10. Endpoint đề xuất

### `POST /api/affiliate/click`

Input:

```json
{
  "listing_id": "abc123"
}
```

Server thực hiện:

1. Resolve user từ authenticated session hoặc guest cookie.
2. Query listing từ database.
3. Kiểm tra listing active.
4. Lấy canonical Shopee URL từ DB.
5. Validate hostname/scheme.
6. Generate `click_code` cryptographically secure.
7. Build `sub_id` server-side.
8. Lưu click vào DB.
9. Trả URL nội bộ:

```json
{
  "redirect_url": "/go/C4KM92QH"
}
```

Không trả quyền cấu hình Affiliate ID cho client.

### `GET /go/:clickCode`

Server thực hiện:

1. Tìm click theo `click_code`.
2. Load listing từ DB.
3. Validate lại canonical Shopee URL.
4. Load `SHOPEE_AFFILIATE_ID` từ server configuration.
5. Rebuild/verify `sub_id` từ dữ liệu server-side.
6. Build Shopee `an_redir` URL.
7. Set `Location` tới URL đó.
8. Trả HTTP redirect 302/303 phù hợp.

**Không đọc `affiliate_id`, `sub_id`, `origin_link` từ query của `/go`**.

URL public của CAMPE chỉ nên như:

```text
https://campe.vn/go/C4KM92QH
```

## 11. Hàm build Affiliate URL phải có interface an toàn

Nên thiết kế theo kiểu:

```text
buildShopeeAffiliateUrl(canonicalShopeeUrl, serverConfig, trackingData)
```

Không thiết kế:

```text
buildShopeeAffiliateUrl(originLink, affiliateId, subIdFromClient)
```

Affiliate ID nên nằm trong object config inject từ server startup để developer khó vô tình nhận ID từ request.

Pseudo-code:

```text
function buildShopeeAffiliateUrl(canonicalUrl, tracking):
    affiliateId = config.SHOPEE_AFFILIATE_ID
    assert affiliateId is configured

    destination = validateShopeeUrl(canonicalUrl)
    subId = buildSubId(tracking)

    url = new URL("https://s.shopee.vn/an_redir")
    url.searchParams.set("origin_link", destination.toString())
    url.searchParams.set("affiliate_id", affiliateId)
    url.searchParams.set("sub_id", subId)

    return url.toString()
```

Dùng URL builder chuẩn; không nối chuỗi thủ công nếu framework có API URL/query encoder.

## 12. Test bảo mật bắt buộc

Codex phải viết automated tests cho ít nhất các case sau.

### Test 1 — Client cố thay Affiliate ID

Request:

```json
{
  "listing_id": "abc123",
  "affiliate_id": "ATTACKER_ID"
}
```

Expected:

- request bị reject vì field thừa, hoặc field bị ignore;
- redirect cuối **không bao giờ chứa `ATTACKER_ID`**;
- redirect phải chứa đúng `SHOPEE_AFFILIATE_ID` server config.

### Test 2 — Query injection vào `/go`

```text
/go/C4KM92QH?affiliate_id=ATTACKER_ID
```

Expected:

- query `affiliate_id` bị bỏ qua;
- Location vẫn chứa Affiliate ID của CAMPE.

### Test 3 — Client cố thay destination

```text
/go/C4KM92QH?origin_link=https://evil.example
```

Expected:

- ignored;
- destination lấy từ DB.

### Test 4 — Open redirect

Listing URL là:

```text
https://shopee.vn.evil.example/product/1/2
```

Expected: reject.

### Test 5 — Userinfo URL trick

```text
https://shopee.vn@evil.example/product/1/2
```

Expected: reject.

### Test 6 — Protocol injection

```text
javascript:alert(1)
```

Expected: reject.

### Test 7 — Missing server Affiliate ID

Nếu `SHOPEE_AFFILIATE_ID` chưa cấu hình:

- app fail fast lúc startup hoặc affiliate module không start;
- tuyệt đối không fallback về ID do client gửi.

### Test 8 — Affiliate ID invariant

Generate 1.000 redirect với:

- nhiều guest;
- nhiều account;
- nhiều listing;
- nhiều source;

Expected:

```text
affiliate_id luôn == config.SHOPEE_AFFILIATE_ID
```

### Test 9 — Guest chuyển sang logged-in

- guest có cookie guest ID;
- user login;
- click mới phải dùng `affiliate_user_code` của account;
- không cho cookie giả override account user code.

### Test 10 — Tamper cookie

User tự sửa guest cookie.

Expected:

- format không hợp lệ => regenerate;
- nếu dùng signed cookie thì signature sai => regenerate;
- không ảnh hưởng Affiliate ID của CAMPE.

## 13. Defense in depth

Nên thêm:

- strict input validation;
- rate limiting `/api/affiliate/click` và `/go/:clickCode`;
- logging các request có field `affiliate_id`, `origin_link` hoặc `sub_id` bất thường;
- metrics số redirect và số validation failure;
- dependency scanning;
- secret scanning;
- `.env` trong `.gitignore`;
- `.env.example` chỉ chứa placeholder;
- production config read-only đối với app process nếu hạ tầng hỗ trợ;
- code review bắt buộc nếu sửa builder hoặc redirect logic.

## 14. Không nhúng Affiliate ID vào frontend làm nguồn quyết định

Frontend có thể cuối cùng nhìn thấy Affiliate ID trong URL Shopee sau redirect; điều này không tránh được và không phải vấn đề chính.

Nhưng frontend **không bao giờ là nơi quyết định giá trị ID**.

Không làm:

```js
const affiliateId = window.CONFIG.affiliateId;
fetch('/api/link', { affiliateId });
```

Phải làm:

```text
client -> listing_id -> server
server -> configured Affiliate ID -> Shopee redirect
```

## 15. Acceptance criteria cho Codex

Chỉ coi module hoàn thành khi:

- [ ] Affiliate ID chỉ có một nguồn production-side đáng tin cậy.
- [ ] Không endpoint nào nhận Affiliate ID từ client để build link.
- [ ] `origin_link` lấy từ DB, không lấy trực tiếp từ client.
- [ ] `sub_id` do server build.
- [ ] `/go/:clickCode` chỉ cần click code.
- [ ] URL Shopee được validate bằng parser + allowlist.
- [ ] Có test client cố đổi Affiliate ID và test pass.
- [ ] Có test open redirect và test pass.
- [ ] Guest cookie và account user code hoạt động đúng priority.
- [ ] `.env` production không nằm trong Git.
- [ ] README/source code trỏ về tài liệu Shopee chính thức.

## 16. Rule cho Codex

Nếu trong code review Codex phát hiện bất kỳ đoạn nào kiểu:

```text
affiliate_id = req.query.affiliate_id
affiliate_id = req.body.affiliate_id
affiliate_id = req.cookies.affiliate_id
```

phải coi đó là **security bug mức cao** và sửa trước khi merge.

Tương tự với việc lấy raw `origin_link` từ client rồi redirect.

Mục tiêu cuối cùng:

> Người dùng có thể sửa mọi thứ ở browser của họ, nhưng không thể làm CAMPE sinh ra link Affiliate sử dụng Affiliate ID của họ thay cho Affiliate ID cấu hình của CAMPE.
