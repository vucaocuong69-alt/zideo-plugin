# Zideo — Claude Code plugin marketplace

Cài plugin **zideo** để dựng motion graphic tự động cho video talking-head ngay trong Claude Code (**desktop** hoặc **terminal**) — đúng phong cách kênh, không đè mặt người nói. Engine chạy trên máy chủ Zideo; bạn chỉ cần một mã token.

## Cài

> **Chạy trong Terminal của máy** (macOS: app Terminal · Windows: PowerShell) — **không dán vào khung chat của Claude**: Claude trong app desktop không chạy được lệnh `claude`. App desktop dùng chung plugin với bản terminal, nên cài một lần ở Terminal là app desktop cũng có.

**① Chỉ làm một lần** — khi `claude --version` báo không có lệnh, cài Claude Code CLI:

```bash
# macOS / Linux
curl -fsSL https://claude.ai/install.sh | bash
```

```powershell
# Windows (PowerShell)
irm https://claude.ai/install.ps1 | iex
```

Cài xong **đóng Terminal và mở cửa sổ MỚI** — cửa sổ vừa cài vẫn giữ PATH cũ nên chưa thấy lệnh `claude`. Trên Windows mở PowerShell bình thường, **không chọn «Run as administrator»** — tiêu đề cửa sổ không có chữ «Administrator» là đúng; dấu nhắc hiện `C:\Windows\system32` hay `C:\Users\…` đều không sao, lệnh cài không phụ thuộc thư mục đang đứng.

Windows: vẫn báo `claude` is not recognized → dán cả khối dưới. Hai dòng đầu thêm thư mục cài vào PATH cho các cửa sổ **sau**; dòng thứ ba cho **cửa sổ đang mở** dùng được ngay (đặt PATH kiểu hai dòng đầu không áp cho cửa sổ hiện tại):

```powershell
$p = [Environment]::GetEnvironmentVariable('PATH', 'User')
[Environment]::SetEnvironmentVariable('PATH', "$p;$env:USERPROFILE\.local\bin", 'User')
$env:Path += ";$env:USERPROFILE\.local\bin"
claude --version
```

Vẫn không có: `Test-Path "$env:USERPROFILE\.local\bin\claude.exe"` ra `False` nghĩa là tài khoản Windows này chưa có Claude Code — chạy lại lệnh cài ở bước ① trong cửa sổ PowerShell bình thường này.

> Gõ `claude` mà lại mở ra app Claude desktop: cập nhật app desktop lên bản mới nhất (bản cũ chiếm tên lệnh `claude`).

**② Cài plugin** (thay `<token>` bằng Zideo API Token của bạn; dán nguyên từng dòng, **không thêm dấu `\` ở cuối**):

> Windows: plugin được tải bằng Git. Gõ `git --version` trước; báo không có lệnh thì cài [Git for Windows](https://git-scm.com/downloads/win) (chọn «Add to PATH») rồi mở cửa sổ mới.

```bash
claude plugin marketplace add vucaocuong69-alt/zideo-plugin
claude plugin install zideo@zideo --config api_token=<token>
```

**③ Mở phiên Claude Code mới** — plugin chỉ nạp lúc phiên bắt đầu; app desktop thì thoát hẳn rồi mở lại. Kiểm tra trong Terminal:

```bash
claude mcp list        # thấy: plugin:zideo:zideo … ✔ Connected
```

> Nút ➕ → Plugins trên desktop chỉ *duyệt* marketplace có sẵn — **không** thêm được marketplace lạ; phải dùng lệnh `claude plugin marketplace add` như trên.

> **Token là chìa khoá tài khoản** — đừng chụp màn hình hay gửi lệnh có token cho ai. Lỡ lộ: phát token mới trong trang tài khoản Zideo (token cũ hết hiệu lực ngay), chạy `claude plugin uninstall zideo@zideo` rồi cài lại bằng lệnh ở bước ②.

## Dùng

Trong phiên mới, gõ: **"dùng skill zideo dựng video &lt;tên project&gt;"**.

Hướng dẫn đầy đủ: [zideo/README.md](zideo/README.md).
