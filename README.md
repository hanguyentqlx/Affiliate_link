# CAMPE Affiliate Link

Spec kỹ thuật để Codex triển khai hệ thống tạo link Shopee Affiliate động cho **campe.vn**.

> Đây là repo đặc tả/POC cho module Affiliate Link. Khi tích hợp vào CAMPE chính, giữ nguyên các nguyên tắc nhận dạng user, click tracking và redirect được mô tả ở đây.

## 1. Source of Truth bắt buộc

Codex phải đọc tài liệu Shopee chính thức này trước khi sửa logic tạo link:

- Shopee Help Center: https://help.shopee.vn/portal/10/article/172955

Tài liệu trên là nguồn chính thức cho cấu trúc link `an_redir`, các tham số `origin_link`, `affiliate_id`, `sub_id` và cách dùng `sub_id` nhiều phần cho đối tác Product Feed.

Không tự suy đoán các giới hạn hoặc field Shopee không nêu trong tài liệu. Các thông tin liên quan đến báo cáo đơn hàng, attribution, cashback, thời gian cookie, giới hạn ký tự hoặc quyền Product Feed phải được kiểm chứng riêng từ tài khoản/tài liệu Shopee hiện hành trước khi triển khai production.

## 2. Mục tiêu hệ thống

Hệ thống phải:

- Tạo link Shopee Affiliate ngay khi người dùng bấm **Mua trên Shopee**.
- Không cần tạo trước một short-link riêng cho từng user hoặc từng click.
- Mỗi click có một `click_code` duy nhất.
- Người chưa đăng nhập vẫn có một `guest_id` ổn định lưu bằng cookie first-party của CAMPE.
- Người đã đăng nhập có một `affiliate_user_code` cố định gắn với tài khoản CAMPE.
- Khi user đăng nhập, `affiliate_user_code` của account luôn được ưu tiên thay cho guest ID.
- Không đưa email, số điện thoại, tên thật, UUID nội bộ hoặc dữ liệu cá nhân trực tiếp vào URL Shopee.
- CAMPE tự log click trước khi redirect sang Shopee.
- Redirect phải rất nhẹ, không gọi crawler, không gọi API lấy sản phẩm trong lúc click.
- Chỉ cho phép redirect tới domain Shopee hợp lệ đã whitelist.

## 3. Khái niệm chính

### 3.1 `guest_id`

Mã định danh ngẫu nhiên cho người chưa đăng nhập.

Ví dụ:

```text
G7K4Q2XZ
```

Yêu cầu:

- Sinh bằng CSPRNG, không dùng số tăng dần.
- 8-12 ký tự Base32/Base36 hoặc định dạng ngắn tương đương.
- Không chứa PII.
- Lưu bằng cookie first-party, ví dụ `campe_guest_id`.
- Cookie nên có `Secure`, `SameSite=Lax`, `Path=/`.
- Chỉ dùng `HttpOnly` nếu frontend không cần đọc cookie. Ưu tiên backend quản lý cookie.
- Guest ID không phải khóa chính database.

### 3.2 `affiliate_user_code`

Mã affiliate public cố định của user đã đăng nhập.

Ví dụ:

```text
U8F3X9AZ
```

Yêu cầu:

- Sinh một lần khi account được tạo hoặc lần đầu cần affiliate tracking.
- Không đổi khi user logout/login lại.
- Unique trong database.
- Không suy ra được `users.id`.
- Không chứa email/phone/name.

### 3.3 `click_code`

Mã duy nhất cho từng click outbound.

Ví dụ:

```text
C4KM92QH
```

Yêu cầu:

- Mỗi lần click hợp lệ tạo một mã mới.
- Sinh bằng CSPRNG/ULID/NanoID phù hợp.
- Unique có index trong database.
- Dùng làm khóa public trong URL CAMPE `/go/:clickCode`.

## 4. Quy tắc nhận dạng user

Pseudo logic:

```text
IF request có authenticated CAMPE user:
    đảm bảo account có affiliate_user_code
    tracking_user_code = affiliate_user_code
    identity_type = "account"
ELSE:
    đọc cookie campe_guest_id
    IF chưa có hoặc không hợp lệ:
        sinh guest_id mới
        set cookie campe_guest_id
    tracking_user_code = guest_id
    identity_type = "guest"
```

### Khi guest đăng nhập

Không đổi `guest_id` thành `affiliate_user_code` trong cookie.

Thay vào đó:

1. Giữ guest cookie để hỗ trợ continuity/analytics nếu cần.
2. Khi session đã authenticated, account code luôn thắng.
3. Có thể lưu mapping `guest_id -> user_id` tại thời điểm login nếu chính sách riêng tư của CAMPE cho phép và feature thực sự cần.
4. Tất cả click mới sau login phải dùng `affiliate_user_code` của account.

Ví dụ:

```text
Trước login:
G7K4Q2XZ -> click C111

Sau login user 18452:
U8F3X9AZ -> click C112
U8F3X9AZ -> click C113
```

## 5. Luồng tổng thể

```text
[Browser]
   |
   | user bấm "Mua trên Shopee"
   v
[CAMPE API]
   |
   | 1. resolve identity: account hoặc guest
   | 2. validate listing/product
   | 3. tạo click_code
   | 4. lưu affiliate_click
   | 5. tạo URL Shopee an_redir
   v
[CAMPE redirect URL]
https://campe.vn/go/C4KM92QH
   |
   | HTTP 302/303
   v
[Shopee an_redir]
https://s.shopee.vn/an_redir?...
   |
   | Shopee xử lý affiliate tracking
   v
[Trang sản phẩm Shopee]
```

## 6. Hai endpoint đề xuất

### 6.1 Tạo click

```http
POST /api/affiliate/click
```

Request:

```json
{
  "listingId": "listing_123",
  "source": "product_page",
  "campaign": "default"
}
```

Server tự lấy user từ session/cookie. Tuyệt đối không tin `user_id`, `guest_id`, `affiliate_user_code`, Shopee URL hoặc affiliate ID do client gửi lên.

Response:

```json
{
  "redirectUrl": "https://campe.vn/go/C4KM92QH"
}
```

Có thể tối ưu thành endpoint click trả redirect trực tiếp, nhưng tách hai bước giúp frontend analytics dễ hơn. Chọn một cách và giữ nhất quán.

### 6.2 Redirect

```http
GET /go/:clickCode
```

Server:

1. Tìm click bằng `click_code`.
2. Kiểm tra click tồn tại và còn hợp lệ.
3. Lấy `origin_link` từ listing đã lưu trong DB.
4. Xác nhận `origin_link` thuộc whitelist Shopee.
5. Build Shopee `an_redir` URL.
6. Trả HTTP 302 hoặc 303 đến Shopee.

Không cho phép dạng:

```text
/go?url=https://evil.example
```

vì tạo open redirect.

## 7. Cấu trúc link Shopee

Theo tài liệu Shopee chính thức:

```text
https://s.shopee.vn/an_redir
?origin_link=<URL_ENCODED_SHOPEE_URL>
&affiliate_id=<CAMPE_AFFILIATE_ID>
&sub_id=<PART1>-<PART2>-<PART3>-<PART4>-<PART5>
```

Codex phải dùng URL builder chuẩn của ngôn ngữ/framework, không nối query string thủ công bằng string concatenation nếu có nguy cơ encode sai.

### `origin_link`

- Lấy từ DB/server-side.
- URL encode đúng chuẩn.
- Chỉ cho domain Shopee hợp lệ.
- Không nhận arbitrary destination URL từ client.

### `affiliate_id`

- Lấy từ environment/config server-side.
- Không hard-code secret/config production vào source.
- Ví dụ env:

```text
SHOPEE_AFFILIATE_ID=...
```

### `sub_id`

Shopee mô tả `sub_id` nhiều phần trong tài liệu Product Feed. CAMPE dùng 5 slot theo cách có thể cấu hình.

Đề xuất mặc định:

```text
PART1 = tracking_user_code
PART2 = click_code
PART3 = source_code
PART4 = product_or_listing_code
PART5 = campaign_code
```

Ví dụ:

```text
U8F3X9AZ-C4KM92QH-WEB-P152-CAMPE
```

Guest:

```text
G7K4Q2XZ-C4KM92QH-WEB-P152-CAMPE
```

Lưu ý: đây là mapping nội bộ CAMPE dựa trên 5 vị trí tracking; nếu tài khoản/thoả thuận Product Feed của Shopee yêu cầu semantic cụ thể cho từng slot thì phải cấu hình lại cho đúng tài liệu/tài khoản, không cố định bằng code.

## 8. Nên giữ URL CAMPE ngắn

URL người dùng nhìn thấy:

```text
https://campe.vn/go/C4KM92QH
```

Không đưa toàn bộ `sub_id` vào URL CAMPE public.

`click_code` tra ngược trong DB để lấy:

```text
click_code
 -> identity_type
 -> tracking_user_code
 -> user_id nullable
 -> guest_id nullable
 -> canonical_product_id
 -> listing_id
 -> shop_id
 -> source
 -> campaign
 -> displayed_price
 -> origin_link
 -> clicked_at
```

## 9. Schema database đề xuất

### `users`

```text
id BIGINT/UUID PRIMARY KEY
affiliate_user_code VARCHAR UNIQUE NULL
...
```

### `affiliate_guest_identities` (optional)

Không bắt buộc nếu chỉ dùng cookie, nhưng hữu ích nếu cần chống abuse hoặc merge analytics.

```text
id
 guest_code UNIQUE
first_seen_at
last_seen_at
linked_user_id NULL
```

### `affiliate_clicks`

```text
id BIGINT/UUID PRIMARY KEY
click_code VARCHAR UNIQUE NOT NULL
identity_type ENUM('guest','account') NOT NULL
tracking_user_code VARCHAR NOT NULL
user_id NULL
guest_code NULL
canonical_product_id NULL
listing_id NOT NULL
shop_id NULL
source_code NULL
campaign_code NULL
displayed_price NULL
origin_link_snapshot TEXT NULL
sub_id TEXT NOT NULL
created_at TIMESTAMP NOT NULL
redirected_at TIMESTAMP NULL
```

Index tối thiểu:

```text
UNIQUE(click_code)
INDEX(user_id, created_at)
INDEX(guest_code, created_at)
INDEX(listing_id, created_at)
INDEX(created_at)
```

Không cần lưu raw IP lâu dài để feature hoạt động. Nếu cần rate limiting/anti-abuse, ưu tiên hash/HMAC ngắn hạn hoặc Redis key có TTL.

## 10. Thuật toán tạo mã

Không dùng:

```text
user1
user2
user3
```

Không dùng trực tiếp database PK:

```text
18452
```

Có thể dùng NanoID alphabet an toàn URL, Base32 Crockford hoặc random bytes encode Base36/Base62.

Ví dụ pseudo:

```text
user code:  prefix U + random 8 chars
 guest code: prefix G + random 8 chars
click code: prefix C + random 10 chars
```

Database UNIQUE constraint là lớp bảo vệ cuối cùng; nếu collision thì generate lại.

## 11. Cookie guest

Tên đề xuất:

```text
campe_guest_id
```

Value:

```text
G7K4Q2XZ
```

Thuộc tính đề xuất:

```text
Path=/
Secure
SameSite=Lax
Max-Age=<configurable>
HttpOnly=true nếu chỉ backend cần đọc
```

Không lưu trong cookie:

- email
- phone
- Shopee affiliate_id
- click history
- product history
- full sub_id list

Cookie chỉ là identifier ngẫu nhiên. Dữ liệu analytics nằm server-side.

## 12. Source code mapping

Không đưa chuỗi tùy ý từ client trực tiếp vào `sub_id`.

Dùng allowlist:

```text
WEB = search nội bộ CAMPE
SEO = organic search
FB = Facebook
TT = TikTok
DIRECT = direct/unknown
```

Campaign cũng dùng slug đã sanitize/configured.

## 13. Security requirements

Codex phải implement/test:

1. **No open redirect**: destination lấy từ listing server-side + whitelist Shopee.
2. **No PII in URL**: không email/phone/name/address.
3. **CSPRNG IDs**: guest/user/click public codes không đoán được.
4. **Rate limit** endpoint tạo click và redirect nếu cần.
5. **Dedup analytics**: click spam lặp rất nhanh có thể đánh dấu/dedup cho báo cáo nội bộ, nhưng không làm hỏng redirect hợp lệ.
6. **Validate click code format** trước DB lookup.
7. **Environment config** cho affiliate ID và Shopee redirect base URL.
8. **Logging** không log cookie/session token hoặc PII không cần thiết.
9. **CSRF**: nếu POST endpoint dựa trên cookie auth, dùng cơ chế CSRF phù hợp framework hoặc SameSite + token theo kiến trúc ứng dụng.
10. **Referer/source is untrusted**: chỉ dùng như analytics, không dùng cho authorization.

## 14. Config đề xuất

```text
SHOPEE_AFFILIATE_REDIRECT_BASE=https://s.shopee.vn/an_redir
SHOPEE_AFFILIATE_ID=...
CAMPE_GUEST_COOKIE_NAME=campe_guest_id
CAMPE_GUEST_COOKIE_MAX_AGE_DAYS=...
AFFILIATE_CLICK_CODE_LENGTH=...
```

Không commit `.env` thật.

## 15. Pseudo code build link

```text
function buildShopeeAffiliateUrl(click):
    listing = loadListing(click.listing_id)
    assert listing exists
    assert isAllowedShopeeUrl(listing.origin_url)

    subId = buildSubId(
        click.tracking_user_code,
        click.click_code,
        click.source_code,
        compactProductCode(click.canonical_product_id),
        click.campaign_code
    )

    url = new URL(config.SHOPEE_AFFILIATE_REDIRECT_BASE)
    url.searchParams.set("origin_link", listing.origin_url)
    url.searchParams.set("affiliate_id", config.SHOPEE_AFFILIATE_ID)
    url.searchParams.set("sub_id", subId)

    return url.toString()
```

Lưu ý: `URL.searchParams` tự encode value. Không encode `origin_link` hai lần.

## 16. Pseudo code resolve identity

```text
function resolveAffiliateIdentity(request, response):
    user = getAuthenticatedUser(request)

    if user != null:
        code = ensureAffiliateUserCode(user)
        return {
            type: "account",
            trackingUserCode: code,
            userId: user.id,
            guestCode: readValidGuestCookie(request) // optional analytics only
        }

    guestCode = readValidGuestCookie(request)

    if guestCode == null:
        guestCode = generateGuestCode()
        setGuestCookie(response, guestCode)

    return {
        type: "guest",
        trackingUserCode: guestCode,
        userId: null,
        guestCode: guestCode
    }
```

## 17. Test cases bắt buộc

### Guest

- Guest lần đầu chưa có cookie -> tạo `G...`, set cookie, click thành công.
- Guest quay lại -> dùng lại đúng guest code cũ.
- Cookie format giả -> bỏ cookie, tạo mã mới.
- Hai click khác nhau -> cùng guest code nhưng hai click code khác nhau.

### Logged-in

- Account chưa có `affiliate_user_code` -> tạo một mã cố định.
- Account đã có code -> không generate lại.
- User có guest cookie rồi login -> click mới dùng `U...`, không dùng `G...`.
- Logout -> nếu guest cookie vẫn tồn tại, guest flow có thể tiếp tục bằng `G...`.

### Redirect

- `origin_link` hợp lệ -> redirect sang `s.shopee.vn/an_redir`.
- `origin_link` không phải Shopee -> reject.
- `affiliate_id` lấy từ config.
- `sub_id` có đúng 5 phần theo mapping configured.
- Query params encode đúng, không double encode.
- Click code không tồn tại -> 404/410, không arbitrary redirect.

### Security

- Client cố truyền affiliate ID khác -> bỏ qua.
- Client cố truyền guest/user code khác -> bỏ qua.
- Client cố truyền URL `evil.com` -> không redirect.
- Source/campaign có ký tự lạ -> map/sanitize hoặc reject.
- Không có email/phone trong generated URL/log fixture.

## 18. Observability / analytics

Tối thiểu cần dashboard/query được:

- clicks theo ngày
- unique guest/account tracking codes
- clicks theo product/listing/shop
- CTR theo vị trí listing
- source/campaign
- guest vs logged-in
- duplicate/suspicious click rate

Không dùng số click nội bộ để khẳng định Shopee đã ghi nhận hoa hồng. Attribution cuối cùng phải đối chiếu báo cáo/nguồn chính thức Shopee.

## 19. Phần chưa được phép tự suy luận

Codex không được tự viết business logic production dựa trên giả định về:

- thời gian attribution/cookie của Shopee
- đơn nào được tính hoa hồng
- format order report
- Shopee có trả `sub_id` ở field nào trong báo cáo
- cách xử lý cashback
- giới hạn ký tự chính xác của từng sub_id nếu tài liệu hiện hành không nêu
- Product Feed có sẵn cho mọi tài khoản hay không

Các phần này phải để interface/config/TODO rõ ràng cho đến khi có tài liệu hoặc sample thật.

## 20. Definition of Done cho Codex

Module được coi là hoàn thành khi:

- Guest không login vẫn click affiliate được và có ID ổn định qua cookie.
- User login có affiliate user code cố định.
- Mỗi outbound click có click code riêng.
- `/go/:clickCode` build đúng Shopee `an_redir` URL và redirect.
- Không có open redirect.
- Không expose PII.
- Có unit/integration tests cho các flow ở trên.
- Có `.env.example` nhưng không có secret thật.
- README/source comment trỏ về tài liệu Shopee chính thức: https://help.shopee.vn/portal/10/article/172955

## 21. Luồng mẫu hoàn chỉnh

### Guest

```text
Browser chưa login
  -> GET campe.vn/product/esp32
  -> server thấy chưa có campe_guest_id
  -> set cookie campe_guest_id=G7K4Q2XZ
  -> user bấm listing Shop A
  -> POST /api/affiliate/click { listingId }
  -> tạo click C4KM92QH
  -> sub_id = G7K4Q2XZ-C4KM92QH-WEB-P152-CAMPE
  -> response redirectUrl=/go/C4KM92QH
  -> GET /go/C4KM92QH
  -> 302 Shopee an_redir
  -> Shopee product
```

### Logged-in user

```text
User login CAMPE
  -> account có affiliate_user_code=U8F3X9AZ
  -> user bấm Shop A
  -> tạo click C7AB91ZX
  -> sub_id = U8F3X9AZ-C7AB91ZX-WEB-P152-CAMPE
  -> /go/C7AB91ZX
  -> Shopee an_redir
  -> Shopee product
```

---

**Shopee reference:** https://help.shopee.vn/portal/10/article/172955
