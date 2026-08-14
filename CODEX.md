# Codex Instructions — CAMPE Shopee Affiliate Link

Codex: trước khi viết hoặc sửa bất kỳ code nào liên quan đến Affiliate Link, bắt buộc đọc theo thứ tự:

1. `README.md`
2. `docs/AFFILIATE_ID_SECURITY.md`
3. Nguồn Shopee chính thức: https://help.shopee.vn/portal/10/article/172955

## Quy tắc không được vi phạm

- `SHOPEE_AFFILIATE_ID` chỉ được lấy từ config/secret phía server.
- Không bao giờ nhận `affiliate_id` từ browser, query string, request body, cookie, localStorage, header hoặc dữ liệu frontend.
- Client chỉ gửi `listing_id` nội bộ; server tự lấy URL Shopee canonical từ database.
- `origin_link` phải do server lấy từ DB và validate bằng URL parser + hostname allowlist.
- `sub_id` phải do server tự build từ dữ liệu server-side.
- `/go/:clickCode` không được nhận `affiliate_id`, `origin_link` hay `sub_id` làm tham số có hiệu lực.
- Nếu user cố thêm `?affiliate_id=ATTACKER_ID`, server phải bỏ qua và vẫn redirect bằng Affiliate ID của CAMPE.
- Nếu user sửa guest cookie, tuyệt đối không được ảnh hưởng tới Affiliate ID.
- Khi user đã đăng nhập, `affiliate_user_code` gắn với account luôn ưu tiên hơn guest cookie.
- Viết automated security tests bắt buộc theo `docs/AFFILIATE_ID_SECURITY.md`.

## Acceptance gate

Không được coi implementation hoàn thành nếu chưa có test chứng minh:

```text
affiliate_id trong mọi Shopee redirect == SHOPEE_AFFILIATE_ID từ server config
```

kể cả khi client cố tình gửi Affiliate ID khác.

Nếu phát hiện code kiểu:

```text
affiliate_id = req.query.affiliate_id
affiliate_id = req.body.affiliate_id
affiliate_id = req.cookies.affiliate_id
```

hãy coi đó là **security bug nghiêm trọng** và sửa trước khi tiếp tục.
