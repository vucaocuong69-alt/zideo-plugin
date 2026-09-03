# Zideo — plugin dựng motion graphic cho Claude Code

Dựng motion graphic tự động cho video talking-head, **đúng phong cách kênh và không đè mặt người nói**. Bạn nạp video vào Zideo, mở Claude Code, gõ *"dùng skill zideo dựng video X"* — agent tự dựng qua máy chủ Zideo.

> **Không cài gì nặng lên máy bạn.** Engine dựng nằm trên máy chủ Zideo; plugin này chỉ gồm phần điều phối (skill) + kết nối máy chủ. Bạn chỉ cần **một mã token**.

## Cần trước khi cài

1. Tài khoản Zideo + video đã nạp vào timeline (ở web app Zideo).
2. **Mã API token** của bạn — lấy trong trang tài khoản Zideo.
3. Claude Code (bản **desktop** hoặc **terminal** đều được — cùng một plugin).

## Cài (desktop lẫn terminal — chạy trong terminal)

Desktop app có **terminal tích hợp**. Chạy 2 lệnh (thay `<token>` bằng Zideo API Token của bạn):

```bash
claude plugin marketplace add vucaocuong69-alt/zideo-plugin
claude plugin install zideo@zideo --config api_token=<token>
```

Rồi **mở một phiên Claude Code MỚI** (plugin nạp lúc khởi động phiên). Kiểm tra kết nối:

```bash
claude mcp list        # phải thấy: plugin:zideo:zideo … ✔ Connected
```

> **Lưu ý:** nút ➕ → Plugins trên desktop chỉ *duyệt* các marketplace có sẵn (official/community/đã-thêm) — **không** thêm được marketplace lạ. Phải dùng lệnh `claude plugin marketplace add` như trên. Đặt token phải kèm `--config api_token=…` lúc install; nếu cài rồi mới đặt thì `claude plugin uninstall zideo@zideo` rồi cài lại kèm cờ đó.

## Dùng

Sau khi cài, trong bất kỳ phiên chat nào:

> dùng skill zideo dựng video &lt;tên video&gt;

Agent sẽ: đọc timeline → từng beat chọn cách dựng → viết/gắn motion graphic → tự soi ảnh sửa cho tới khi đạt → báo bạn xuất video.

## Cần biết

- **Phải có mạng.** Mọi thao tác đi qua máy chủ Zideo; máy chủ tạm ngừng thì tool không chạy.
- **Token là của riêng bạn** — đừng chia sẻ. Nó gắn với hạn mức video theo gói của bạn.
- Render bản cuối chạy **trên máy bạn** (nút "Xuất trên máy bạn" trong web app) — nhanh và không tốn tài nguyên máy chủ.

---
Cường Mê AI · https://cuongmeai.com
