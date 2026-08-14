# CAMPE Affiliate Link

Tài liệu thiết kế hệ thống tạo link Shopee Affiliate động cho CAMPE.

## Mục tiêu

Hệ thống phải:

- Tạo link Shopee Affiliate ngay khi người dùng bấm `Mua trên Shopee`.
- Không cần tạo sẵn/rút gọn từng link trong dashboard Shopee.
- Theo dõi được từng lượt click bằng `click_code` riêng.
- Người chưa đăng nhập vẫn có một mã guest ổn định lưu trong cookie.
- Người đã đăng nhập dùng một mã affiliate user cố định gắn với tài khoản CAMPE.
- Không đưa email