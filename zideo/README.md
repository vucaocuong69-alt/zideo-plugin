# Zideo — plugin dựng motion graphic cho Claude Code

Dựng motion graphic tự động cho video talking-head, **đúng phong cách kênh và không đè mặt người nói**. Bạn nạp video vào Zideo, mở Claude Code, gõ *"dùng skill zideo dựng video X"* — agent tự dựng qua máy chủ Zideo.

> **Không cài gì nặng lên máy bạn.** Engine dựng nằm trên máy chủ Zideo; plugin này chỉ gồm phần điều phối (skill) + kết nối máy chủ. Bạn chỉ cần **một mã token**.

## Cần trước khi cài

1. Tài khoản Zideo + video đã nạp vào timeline (ở web app Zideo).
2. **Mã API token** của bạn — lấy trong trang tài khoản Zideo.
3. Claude Code (bản **desktop** hoặc **terminal** đều được — cùng một plugin).

## Cài (desktop lẫn terminal — chạy trong Terminal của máy)

> **Không dán lệnh vào khung chat của Claude.** Claude trong app desktop không chạy được lệnh `claude`. Mở **Terminal của máy** (macOS: app Terminal · Windows: PowerShell). App desktop dùng chung plugin với bản terminal: cài một lần ở Terminal là app desktop cũng có.

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

**③ Mở một phiên Claude Code MỚI** (plugin nạp lúc khởi động phiên; app desktop thì thoát hẳn rồi mở lại). Kiểm tra kết nối:

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
