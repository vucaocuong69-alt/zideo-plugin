# Zideo — plugin dựng motion graphic cho Claude Code

Dựng motion graphic tự động cho video talking-head, **đúng phong cách kênh và không đè mặt người nói**. Bạn nạp video vào Zideo, mở Claude Code, gõ *"dùng skill zideo dựng video X"* — agent tự dựng qua máy chủ Zideo.

> **Không cài gì nặng lên máy bạn.** Engine dựng nằm trên máy chủ Zideo; plugin này chỉ gồm phần điều phối (skill) + kết nối máy chủ. Bạn chỉ cần **một mã token**.

## Cần trước khi cài

1. Tài khoản Zideo + video đã nạp vào timeline (ở web app Zideo).
2. **Mã API token** của bạn — lấy trong trang tài khoản Zideo.
3. Claude Code (bản **desktop** hoặc **terminal** đều được — cùng một plugin).

## Cài trên Claude Code Desktop (đơn giản nhất)

1. Bấm nút **➕** cạnh ô chat → **Plugins**.
2. Bấm **Add plugin** → thêm marketplace: `vucaocuong69-alt/zideo-plugin` (hoặc dán link repo).
3. Chọn plugin **zideo** → **Install**.
4. Khi bật plugin, một **form hiện ra** — dán **Zideo API Token** của bạn vào ô *Token*. (Chỉ một ô duy nhất.)
5. Xong. Mở một phiên chat, gõ: **"dùng skill zideo dựng video &lt;tên video&gt;"**.

## Cài trên Claude Code Terminal

```
/plugin marketplace add vucaocuong69-alt/zideo-plugin
/plugin install zideo@zideo
```

Rồi nhập token khi được hỏi (hoặc bật plugin và nhập ở form cấu hình).

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
