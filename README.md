# Zideo — Claude Code plugin marketplace

Cài plugin **zideo** để dựng motion graphic tự động cho video talking-head ngay trong Claude Code (**desktop** hoặc **terminal**) — đúng phong cách kênh, không đè mặt người nói. Engine chạy trên máy chủ Zideo; bạn chỉ cần một mã token.

## Cài (desktop hoặc terminal — chạy trong terminal)

Desktop app có **terminal tích hợp**; chạy 2 lệnh này (thay `<token>` bằng Zideo API Token của bạn):

```bash
claude plugin marketplace add vucaocuong69-alt/zideo-plugin
claude plugin install zideo@zideo --config api_token=<token>
```

Rồi **mở một phiên Claude Code mới** (plugin nạp lúc khởi động phiên). Kiểm tra:

```bash
claude mcp list        # thấy: plugin:zideo:zideo … ✔ Connected
```

> Nút ➕ → Plugins trên desktop chỉ *duyệt* marketplace có sẵn — **không** thêm được marketplace lạ; phải dùng lệnh `claude plugin marketplace add` như trên.

## Dùng

Trong phiên mới, gõ: **"dùng skill zideo dựng video &lt;tên project&gt;"**.

Hướng dẫn đầy đủ: [zideo/README.md](zideo/README.md).
