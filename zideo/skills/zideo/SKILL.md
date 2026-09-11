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

   b'. **BẮT BUỘC gọi `find_examples` TRƯỚC KHI viết `write_motion_graphic`.** LLM tự sáng
      tác từ scratch có xu hướng bọc mọi archetype trong một khung/thẻ trắng "cho an toàn dễ đọc
      chữ" — đó là chỗ chất style biến mất. Đo trên kb27 clay-proof qua MCP: 13/17 MG bọc panel
      viền + boxShadow xung quanh chart/gauge/wf/proof/diagram, trong khi kb30-clay dev cùng
      style chỉ 6/16 (37%) dùng panel, còn lại vẽ TRỰC TIẾP trên trang giấy kem. Chênh do LLM
      không xem ví dụ thật của style.
      Với TỪNG beat sắp `write_motion_graphic`:
      - Gọi `find_examples(project, clip_id, archetype: '<đã chọn>', role: '<vai của beat>')`.
      - Đọc mã trả về — chú ý CẤU TRÚC OUTER: có `<AbsoluteFill style={{background: PAPER}}>`
        rồi vẽ trực tiếp, hay có bọc `<div style={{border, boxShadow, background: SURF}}>` bao
        nội dung? Style của bạn theo cấu trúc nào thì bám cấu trúc đó. Chép cấu trúc, KHÔNG
        chép chữ hay số của ví dụ.
      - Thư viện trống archetype đó → tool báo, cứ tự viết theo hợp đồng nhưng NÊN kiêng bọc
        panel ngoài nếu style yêu cầu "vẽ trực tiếp trên nền" (đọc kỹ tokens.notes).

   c. **Zone phải khớp bậc — và TÊN VÙNG ĐỔI THEO KHUNG.** Xem `get_timeline` để biết dự án
      dọc hay ngang trước khi chọn.

      | bậc | khung DỌC 9:16 | khung NGANG 16:9 |
      |---|---|---|
      | bậc-2 (list/compare/flow/stat…) | `split` — người nói co xuống dải dưới, graphic dải trên | `glide` — người nói dạt sang một bên, graphic vào nửa còn lại |
      | bậc-3 / thứ-có-hình | `stage` / `takeover` | `stage` / `takeover` |
      | chỉ chữ, số nhỏ | `over` | `over` |

      **`split` CHỈ có ở khung dọc.** Đặt nó cho dự án ngang thì server trả lỗi `zone-sai-khung`;
      và nếu lọt qua thì renderer bỏ luôn vùng vẽ, graphic phủ TOÀN KHUNG đè lên mặt người nói.
      Ngược lại `glide` ở khung dọc thì không tách được host (chỉ đè có scrim) — dọc dùng `split`.
      **Không nhét bậc 2/3 vào `over`** — sẽ đè mặt người.

      **Mode mascot** (`get_timeline` có `người_nói_là_nhân_vật`): TRỘN vùng, đừng để toàn
      `split`. Chọn trước **25–40% số beat** được trọn khung — sơ đồ quan hệ, so sánh trước/sau,
      số liệu lớn, danh sách ≥4 mục, ảnh nguồn thật — graphic zone `takeover` và **chừa trống góc
      dưới TRÁI** (nhân vật sẽ ló góc ở đó, cao ~560px). Beat bình luận, cảm thán, kể chuyện giữ
      `split`: ở đó nhân vật chính là nội dung. Beat `over` (nhân vật đứng lớn) thì chữ chỉ nằm
      TRÊN ĐẦU nhân vật. Số chính xác của cả hai luôn nằm trong VÙNG AN TOÀN của
      `get_prompt_contract` — lấy theo đó, đừng tự ước.

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

   f2. **Beat nhắc ĐÍCH DANH một thứ có thật trên web → `capture_reference`.** Một repo GitHub,
      một bài báo, một bài trên X, một video YouTube: `search_image` không bao giờ có, còn
      `generate_asset` thì **vẽ ra một thứ bịa** — sai đúng chỗ người xem kiểm được. Tool trả về
      dữ liệu thật (tiêu đề, tác giả, ngày, sao/fork) + ảnh của chính nguồn.
      · Có `anh` với `anh_tu` là `thẻ-github`/`og`/`thumbnail` → đặt nguyên khối, đừng cắt.
      · `anh_tu: chụp` → **phóng cho vừa bề ngang vùng vẽ**, thu nhỏ là chữ thành vệt xám; đọc
        `chụp_cảnh_báo` trước khi dùng.
      · `anh_tu: thẻ-<nền-tảng>-chính-thức` (X · Instagram · TikTok · Facebook) → thẻ bài do
        **chính nền tảng render**, đúng giao diện tới từng pixel và đã dịch tiếng Việt. **Đừng
        bao giờ tự vẽ lại giao diện của họ** — đây là thứ người xem thuộc lòng, sai một chút là
        thấy ngay. Đặt nguyên khối, phóng ~1,7–3× cho vừa bề ngang. `nen: 'toi'|'sang'` chọn nền.
      · YouTube → không có thẻ nhúng dùng được, nhưng có **thumbnail 1280×720** + tiêu đề + kênh:
        đặt thumbnail rồi tự phủ nút play và tiêu đề nếu muốn giống YouTube.
      · Reddit → **không có ảnh** (Reddit chặn trình duyệt tự động): vẽ thẻ từ tiêu đề +
        `u/tác-giả` + `r/chuyên-mục`.
      · **Không có ảnh → VẼ THẺ bằng mã từ các trường trả về.** Không phải thất bại: thẻ vẽ tay
        đọc rõ hơn ảnh chụp và đúng tông màu video. Giữ nguyên văn tiêu đề/tên tác giả — đây vẫn
        là luật (e) trung thực dữ liệu.
      · Chỉ có TÊN mà không có URL (lời thoại nói "repo Remotion") → dựng URL hiển nhiên
        (`https://github.com/remotion-dev/remotion`) rồi gọi; sai thì tool báo, không im lặng.

   g. **VÒNG TỰ SỬA — CỬA ẢI BẮT BUỘC, không được bỏ để chạy nhanh.** `đạt: true` (và field
      `chưa_xong_beat_này` tool trả kèm) chỉ nghĩa MÃ CHẠY — CHƯA phải bố cục dựng được. **Không
      sang beat khác khi beat này còn `cảnh_báo` chưa xử hoặc nhìn còn lỗi.** (Đây là bước hay bị
      bỏ nhất khi dựng vội — và đó đúng là lúc beat ra thưa/rỗng. Đừng bỏ.)
      1. `capture_frame(<project>, <beat>)` ở HAI mốc (một lúc mọi thứ đã vào, một lúc giữa chuyển động).
      2. Đọc `soi_bố_cục` trả kèm ảnh. Còn `cảnh_báo` → sửa **số hình học** (chiều cao, khoảng cách,
         cỡ chữ) rồi gửi lại chính clip đó, chụp lại. **Còn cảnh_báo là CHƯA xong beat.**
      3. **Nhìn ảnh** ở cỡ đọc được: (a) không đè mặt người; (b) không tràn/rớt chữ, không thẻ rỗng
         ruột, ảnh không teo; (c) **graphic LẤP phần lớn dải/khung** — chừa >1/3 dải trống ở phải
         hoặc đáy, hay teo về một góc = HỎNG, phải giãn khối / thêm tầng nội dung thật / canh giữa;
         (d) **chữ OVER phải đọc được trên nền host SÁNG** — chữ sáng đặt thẳng trên video (over/punch/
         stamp) rơi trúng tường/mic/ánh đèn là chìm; phải có nền pill/scrim tối hoặc shadow/stroke đậm.
      4. Lặp tới khi **hết cảnh_báo VÀ nhìn không còn lỗi**. Quá **3 lượt** vẫn hỏng → bố cục sai từ
         gốc, **đổi sang hình khác**, đừng chỉnh số mãi.

4. **Đa dạng.** KHÔNG dùng cùng một kind quá **2 beat liên tiếp**. Cả video quanh quẩn một khuôn
   là hỏng dù từng beat đều "đúng".

4b. **MODE MASCOT — đặt tư thế nhân vật.** CHỈ khi `get_timeline` trả `người_nói_là_nhân_vật`.
   Ở mode này người nói không phải video người thật mà là một bộ ảnh tĩnh, nên **tư thế là nửa
   còn lại của nội dung** — bỏ qua bước này là nhân vật đứng nguyên một dáng suốt video, và ảnh
   tĩnh không đổi tư thế thì nhìn ra trong hai giây.

   Làm **sau khi graphic đã xong**, không phải trước: lúc đó mới biết beat nào graphic trọn
   khung (nhân vật `goc` hoặc `vang`), beat nào dải trên (`split`), beat nào không có gì (`over`).

   - `get_mascot_catalog(<project>)` — bộ này có ĐÚNG những tư thế nào (mỗi bộ một khác), kèm
     **câu nói** và **chữ đánh số** (`chỉ số:chữ@giây`) của từng beat, và vùng graphic đã dựng.
   - `set_mascot(<project>, beats: [...])` — gửi **cả video trong một lượt**. Mỗi mục là TOÀN BỘ
     kế hoạch tư thế của beat đó.
   - **Đổi tư thế GIỮA beat** khi câu đổi thái độ (nêu vấn đề → bất ngờ → chốt): thêm
     `doi: [{chu, cam_xuc}]`, `chu` là chỉ số chữ trong beat. **Beat không bị tách**, độ dài giữ
     nguyên. Tối đa 2 điểm đổi · beat < 3s giữ một tư thế · mỗi tư thế giữ ≥ 1s. Nhịp tốt là
     mỗi tư thế 3–4s: beat 6–10s thường có 1–2 điểm đổi.
   - **Không chắc thì chọn trung tính.** Nhân vật cười lúc câu đang nói chuyện buồn là hỏng cả
     đoạn; dùng lại một tư thế trung tính chỉ hơi nhàm. Hai cái sai đó không cùng hạng.
   - **Tư thế cuối beat trước ≠ tư thế đầu beat sau** — `set_mascot` chặn, có `force`.
   - Vùng: `split` (graphic dải trên) · `over` (nhân vật lớn, beat không graphic) · `goc`
     (graphic trọn khung, nhân vật ló góc dưới trái — **ưu tiên hơn `vang`**) · `vang` (nhân vật
     biến mất, chỉ khi graphic cần đúng từng góc khung). Beat đầu và beat cuối không được `vang`;
     không quá 8s liền vắng nhân vật. Toàn `split` (0 beat trọn khung) bị chặn.
   - Chuyển động **tự theo tư thế** (sốc giật mình, vui nảy, khóc run…) — không phải chọn. Muốn
     nhấn bằng tiếng động ở cú giật mình thì `add_sfx_clip` đúng mốc đổi tư thế, 2–3 lần một video.
   - **Nhép miệng cũng tự động** khi bộ đã có khẩu hình (`get_mascot_catalog` có mục
     `nhép_miệng`): im thì ngậm, đọc thì mỗi âm tiết một cử động — không phải đặt gì, đừng đi tìm
     tool. Tư thế chưa có khẩu hình thì miệng đứng yên: beat nói dài hoặc quan trọng ưu tiên tư thế
     đã có. Nhịp nhép (nhanh · vừa · chậm) là lựa chọn của người dùng trong editor — agent không đổi.
   - Bộ dưới 12 tư thế (`bộ_ít_tư_thế`) → cuối lượt nhắc người dùng vẽ thêm tư thế.

5. **Xong.** Khi mọi beat đã có motion graphic đạt yêu cầu, báo user tóm tắt (bao nhiêu beat, dạng
   hình đã dùng — mode mascot thì kèm cả chuỗi tư thế) và nhắc bước xuất video.

## Vài lằn ranh
- Không tự chế phong cách/màu/font — mọi thứ đó nằm trong contract và catalog của MCP.
- Beat đóng video thường đã có component cố định của kênh — đừng ghi đè.
- Nếu một tool trả lỗi, **đọc lỗi rồi sửa theo đúng lỗi** (contract, gate, capture đều trả thông
  điệp cụ thể) — đừng đoán.
