---
name: zideo
description: >-
  Dựng motion graphic tự động cho video talking-head bằng engine Zideo (qua MCP server `zideo`),
  theo đúng phong cách kênh và KHÔNG đè mặt người nói. HÃY DÙNG skill này khi user nói "dùng
  zideo dựng/edit video X", "zideo build video <tên>", "cho zideo làm motion graphic cho video
  <tên>", hoặc muốn tự động thêm/hoàn thiện motion graphic cho một project Zideo đã nạp video vào
  timeline. Skill điều phối các tool của MCP `zideo` — LUẬT dựng nằm trong chính tool response
  (get_timeline / get_prompt_contract / get_catalog), skill này chỉ nói CÁCH gọi chúng.
---

# Zideo — dựng motion graphic tự động

Bạn đang điều khiển engine **Zideo** qua MCP server `zideo` để dựng motion graphic cho từng beat
của một video talking-head. Mục tiêu: đồ hoạ đúng phong cách kênh, **đa dạng**, và **không bao giờ
đè lên mặt người nói**.

> NGUỒN LUẬT là tool response của MCP, KHÔNG phải trí nhớ của bạn. `get_timeline`,
> `get_prompt_contract`, `get_catalog` trả về luật dựng — **tuân theo chúng tuyệt đối**, đừng tự
> chế phong cách, thứ tự chọn hình, hay bảng màu.

## Quy trình (làm đúng thứ tự)

1. **Xác định project.** Lấy tên video từ yêu cầu của user. Không rõ → gọi `list_projects` rồi hỏi
   lại. Chỉ thao tác đúng project được yêu cầu.

2. **Đọc trạng thái + luật + danh sách beat — MỘT lệnh.** Gọi `get_timeline(<project>)`. Nó trả
   TẤT CẢ trong một response:
   - `chế_độ_dựng` (được dùng component thư viện tới đâu, tự viết tới đâu), `khung` (dọc 9:16 hay
     ngang 16:9), `theme`, `dải_graphic`, `hướng_dẫn_asset` — **luật dựng của kênh, theo sát**;
   - mảng `beat`: mỗi dòng có `số`, `beat` (id), `bắt_đầu`, `dài`; dòng đã có graphic thì kèm
     `clip_để_sửa` (id `a…` truyền vào set_component/write_motion_graphic) + `kind`/`zone`/`hình`;
   - `beat_chưa_có_component`: các beat còn TRỐNG ô vẽ — phải `add_clip` tạo ô trước (bước 3d).

   Beat nào chưa có graphic thì cần dựng. Chạy lại chỉ dựng beat còn thiếu. (KHÔNG có tool
   `get_project_status` — mọi trạng thái nằm trong `get_timeline`.)

3. **Với TỪNG beat, làm đủ vòng:**

   a. Gọi `get_prompt_contract(<project>, <beat>)` — đọc lời thoại của beat, **VÙNG AN TOÀN**
      (toạ độ được phép vẽ), và **BẢNG HÌNH**. Gọi `get_catalog` — đọc thư viện archetype và
      **`luật_chọn`** (thứ tự ưu tiên chọn cách dựng).

   b. **Chọn cách dựng theo `luật_chọn`** — xét từ trên xuống, lấy cái ĐẦU TIÊN hợp; đừng mặc
      định về một khuôn thẻ chữ:
      - Câu có **quan hệ/cấu trúc/danh sách/số liệu** mà một **archetype thư viện** tả đúng →
        dùng nó qua `set_component` (component đã polish sẵn + tự nhận đúng zone).
      - Câu về **thứ có hình riêng** (giao diện app, khung chat, terminal, sơ đồ đặc thù) mà thư
        viện không tả được → **tự viết** qua `write_motion_graphic`.
      - Chỉ **một cụm chữ đắt / số lớn** → punch/stat/nhấn.
      - Beat chỉ là câu cảm thán/đưa đẩy, không có gì đáng vẽ → **bỏ trống**.
      Ưu tiên thư viện khi hợp; chỉ tự vẽ khi thư viện không có hình đúng.
      **Khớp hình với SỐ MỤC thật.** Hình ngụ ý NHIỀU mục (stepper, compare, list-scan, card-rows,
      timeline, carousel) mà dữ liệu beat chỉ có **1 mục** → ĐỔI sang **hình đơn** (bignum/stat/
      punch/stamp). Vẽ khung nhiều-mục với đúng 1 mục là ra thưa hoác, chết không gian.

   c. **Zone phải khớp bậc.** Ở khung dọc 9:16: bậc-2 (list/compare/flow/stat…) → zone `split`
      (người nói co xuống dải dưới, graphic ở dải trên); bậc-3 / thứ-có-hình → `stage`/`takeover`;
      chỉ chữ/số nhỏ → `over`. **Không nhét bậc 2/3 vào `over`** — sẽ đè mặt người.

   d. **Tạo ô (hoặc dùng ô sẵn) rồi mới gắn.** Ba trường hợp theo dòng beat của `get_timeline`:
      - Beat có `clip_để_sửa` **không** kèm `chưa_có_component` → đã dựng rồi, bỏ qua (trừ khi user bảo sửa).
      - Beat có `chưa_có_component` **kèm `clip_để_sửa` + `ô_đã_có`** → ô trống có sẵn (lượt trước bị
        ngắt). **Gắn thẳng vào id đó, TUYỆT ĐỐI đừng `add_clip`** (thêm ô là chồng hai graphic một beat).
      - Beat có `chưa_có_component` **không** kèm ô → `add_clip(project, track:'anim', start, dur, zone)`
        (start/dur từ dòng beat, zone chọn ở bước c) → lấy id `a…`.
      Rồi gắn vào ĐÚNG id: `set_component(clip_id, kind, data)` (thư viện) hoặc
      `write_motion_graphic(clip_id, …)` (tự viết). **Không ghi lên clip host (`h…`)** — renderer không đọc kind ở đó.

   e. **Trung thực dữ liệu.** Số liệu, tên riêng, câu trích trong data phải **nguyên văn** trong
      lời thoại của beat đó. Thiếu sự kiện thật → đổi kind khác, **tuyệt đối đừng bịa/điền bừa**.
      (Nhãn bước, tên cột thì được diễn đạt lại từ ý trong câu.)

   f. **Icon/logo thật, đừng vẽ tay.** Danh từ cụ thể/khái niệm có biểu tượng → `search_icon`
      (Iconify) → `pull_asset` → dùng `getAssetUrl`. Tên thương hiệu → `add_logo`. Chỉ vẽ `<path>`
      cho sơ đồ/giao diện tự chế mà không kho nào có.

   g. **VÒNG TỰ SỬA — CỬA ẢI BẮT BUỘC, không được bỏ để chạy nhanh.** `đạt: true` (và field
      `chưa_xong_beat_này` tool trả kèm) chỉ nghĩa MÃ CHẠY — CHƯA phải bố cục dựng được. **Không
      sang beat khác khi beat này còn `cảnh_báo` chưa xử hoặc nhìn còn lỗi.** (Đây là bước hay bị
      bỏ nhất khi dựng vội — và đó đúng là lúc beat ra thưa/rỗng. Đừng bỏ.)
      1. `capture_frame(<project>, <beat>)` ở HAI mốc (một lúc mọi thứ đã vào, một lúc giữa chuyển động).
      2. Đọc `soi_bố_cục` trả kèm ảnh. Còn `cảnh_báo` → sửa **số hình học** (chiều cao, khoảng cách,
         cỡ chữ) rồi gửi lại chính clip đó, chụp lại. **Còn cảnh_báo là CHƯA xong beat.**
      3. **Nhìn ảnh** ở cỡ đọc được: (a) không đè mặt người; (b) không tràn/rớt chữ, không thẻ rỗng
         ruột, ảnh không teo; (c) **graphic LẤP phần lớn dải/khung** — chừa >1/3 dải trống ở phải
         hoặc đáy, hay teo về một góc = HỎNG, phải giãn khối / thêm tầng nội dung thật / canh giữa.
      4. Lặp tới khi **hết cảnh_báo VÀ nhìn không còn lỗi**. Quá **3 lượt** vẫn hỏng → bố cục sai từ
         gốc, **đổi sang hình khác**, đừng chỉnh số mãi.

4. **Đa dạng.** KHÔNG dùng cùng một kind quá **2 beat liên tiếp**. Cả video quanh quẩn một khuôn
   là hỏng dù từng beat đều "đúng".

5. **Xong.** Khi mọi beat đã có motion graphic đạt yêu cầu, báo user tóm tắt (bao nhiêu beat, dạng
   hình đã dùng) và nhắc bước xuất video.

## Vài lằn ranh
- Không tự chế phong cách/màu/font — mọi thứ đó nằm trong contract và catalog của MCP.
- Beat đóng video thường đã có component cố định của kênh — đừng ghi đè.
- Nếu một tool trả lỗi, **đọc lỗi rồi sửa theo đúng lỗi** (contract, gate, capture đều trả thông
  điệp cụ thể) — đừng đoán.
