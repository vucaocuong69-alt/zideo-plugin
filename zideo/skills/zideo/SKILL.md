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
   - `thư_viện_component`, `chế_độ_dựng`, `khung` (dọc 9:16 hay ngang 16:9), `theme`, `dải_graphic`,
     `hướng_dẫn_asset` — **luật dựng của kênh, theo sát**;
   - mảng `beat`: mỗi dòng có `số`, `beat` (id), `bắt_đầu`, `dài`; dòng đã có graphic thì kèm
     `clip_để_sửa` (id `a…` truyền vào write_motion_graphic) + `kind`/`zone`/`hình`;
   - `beat_chưa_có_component`: các beat còn TRỐNG ô vẽ — phải `add_clip` tạo ô trước (bước 3d).

   Beat nào chưa có graphic thì cần dựng. Chạy lại chỉ dựng beat còn thiếu. (KHÔNG có tool
   `get_project_status` — mọi trạng thái nằm trong `get_timeline`.)

2a. **KIỂU VIDEO — đọc `get_timeline.kiểu_video` TRƯỚC mọi thứ.** Người dùng chọn kiểu lúc tạo dự án (hoặc ở
   Inspector); đừng tự đổi — chỉ `set_video_format` khi họ yêu cầu. Ba kiểu:
   - **Giải thích bằng hình** (mặc định): mỗi ý một graphic, mọi luật chọn hình bên dưới áp đủ.
   - **Chữ nhấn**: KHÔNG motion graphic (`write_motion_graphic` bị từ chối). Video là ba lớp dựng sẵn: khung bám
     mặt phóng vừa, phụ đề chạy suốt video (server tự sinh từ lời), và vài cụm CHỮ NHẤN lớn trên đầu người nói.
     Việc của bạn chỉ là chọn chữ nhấn: `add_highlight` — 2–4 tiếng NGUYÊN VĂN một đoạn liền trong lời (server
     từ chối chữ không có trong lời, trùng giờ, vượt trần ~1 cụm / 15 giây), `key_words` = tiếng khoá đổi màu +
     gạch chân. Chọn câu trả lời thẳng, luận điểm, con số, câu chốt; rải đều, mỗi lúc một cụm; bỏ câu đưa đẩy.
   - **Trả lời câu hỏi**: y hệt Chữ nhấn, thêm THẺ CÂU HỎI ở đầu video — server tự dựng từ `câu_hỏi` (chữ sáng
     dần theo nhịp đọc rồi thu lại). Chưa có câu hỏi → báo người dùng điền (Inspector › Kiểu video) — KHÔNG tự
     nghĩ câu hỏi, không tên / avatar / tích xanh / khung bình luận giả. Chữ nhấn không được rơi vào lúc thẻ còn.
   Hai kiểu chữ chỉ có ở video DỌC 9:16. Bỏ qua 2b–4b (bảng hình, hợp đồng mg); duyệt bằng contact_sheet /
   capture_frame: chữ nhấn đọc được, không đè mặt, đúng lúc nói; `check_captions` soát chính tả phụ đề.

2b. **Phiếu chỉ đạo + bảng đạo diễn — TRƯỚC beat đầu tiên.** `get_timeline` trả `phiếu_chỉ_đạo` và
   `bảng_đạo_diễn`; dự án mới thường ghi «CHƯA CÓ» / «CHƯA LẬP».
   - `set_project_brief` — ghi những gì người dùng ĐÃ NÓI: khán giả/kênh, một câu thông điệp, CTA và
     ưu đãi thật, tư liệu (tên · vai trò · quyền dùng), cách đặt người nói, thứ phải giữ nguyên, thứ
     cấm. **Chưa nói thì để trống** — CTA, giá, ưu đãi, tên tư liệu mà đoán là bịa trên hình. Người
     dùng muốn tắt tiếng động tự gắn thì `sfx_tu_dong: false`; không cho tốn tiền thì `tra_phi: "khong"`.
     Video nhắc tên thương hiệu / sản phẩm / người → ghi `tu_rieng` (viết ĐÚNG như phải hiện) — hợp đồng
     nhắc lại cho mọi beat, và `check_captions` soát phụ đề theo danh sách đó.
   - `get_transcript` cả video, rồi `set_beat_direction` MỘT lượt cho mọi beat: mắt nhìn vào đâu (một
     thứ), hành động hình (cái gì đổi), chữ trên hình (thêm điều lời không nói), tư liệu, tiêu chí duyệt
     («đạt khi …»). Beat chỉ là ý kiến riêng, lời hứa, câu cảm thán → `trang_thai: "de-trong"` (người
     nói trọn khung). Nhìn cả bảng trước khi dựng: hai beat liền nhau đừng cùng hình VÀ cùng cách kể.
   - Server tự chèn phiếu + dòng của beat vào `get_prompt_contract` — dựng đúng hướng đã chốt; đổi ý
     thì sửa bảng trước. Mã đạt thì dòng tự sang «chờ duyệt». Người dùng xin bảng → `export_direction_table` (CSV).
   - Dự án cũ đã dựng xong mà không có phiếu: không bắt buộc lập lại, trừ khi người dùng yêu cầu.
   - Người dùng đưa VIDEO MẪU («dựng giống video X») → `recipe_from_reference(<project>, du_an_mau: X,
     ap_dung: true)` TRƯỚC khi lập bảng: đo nhịp đổi cảnh, người nói đứng đâu bao nhiêu % thời lượng, hình
     đổi trước hay sau lời, nhạc / tiếng động. Dự án mẫu đã xuất thì đo bản xuất (video đã dựng). Mục nhãn
     «không» (hình kể gì) thì NHÌN ảnh bảng cảnh tool trả kèm. Hợp đồng mọi beat nhận khối LUẬT TỪ VIDEO MẪU.

3. **Với TỪNG beat, làm đủ vòng:**

   a. Gọi `get_prompt_contract(<project>, <beat>)` — đọc lời thoại của beat, **VÙNG AN TOÀN**
      (toạ độ được phép vẽ), và **BẢNG HÌNH**. Gọi `get_catalog` khi cần — đọc **`luật_chọn`** (thứ tự
      ưu tiên chọn hình) và xem thư viện component **để tham khảo**.

   b. **MỌI BEAT LÀ MOTION GRAPHIC BẠN TỰ VIẾT** (`write_motion_graphic`). Component thư viện **chỉ để
      tham khảo**: không đặt vào beat — `set_component` / `add_clip` kèm kind thư viện bị server từ chối
      (`thu-vien-chi-tham-khao`), trừ `outro-follow` có sẵn ở beat đóng video. Thấy một hình thư viện
      hợp với câu thì `see_components` xem ảnh / `get_catalog` xem data mẫu, mượn ý bố cục rồi **tự vẽ
      lại** theo lời thoại và hộp HOP của beat. Chọn **hình** (ghi vào `archetype`) theo `luật_chọn` —
      xét từ trên xuống, lấy cái ĐẦU TIÊN hợp; đừng mặc định về một khuôn thẻ chữ:
      - Câu về **thứ có hình riêng** (giao diện app, khung chat, terminal, sơ đồ đặc thù) → vẽ chính thứ đó.
      - Câu có **quan hệ/cấu trúc** (tăng giảm, phần của tổng, chuỗi bước, trước–sau, xếp hạng, phễu)
        → hình quan hệ (chart-*/proof-*/wf-*/diagram-node/map-tree/gauge).
      - **Danh sách phẳng / định nghĩa / hai vế** → list-scan/card-rows/compare/stepper.
      - Chỉ **một cụm chữ đắt / số lớn** → stamp/bignum.
      - Beat chỉ là câu cảm thán/đưa đẩy, không có gì đáng vẽ → **bỏ trống**.
      **Khớp hình với SỐ MỤC thật.** Hình ngụ ý NHIỀU mục (stepper, compare, list-scan, card-rows,
      timeline, carousel) mà dữ liệu beat chỉ có **1 mục** → ĐỔI sang **hình đơn** (bignum/stat/
      punch/stamp). Vẽ khung nhiều-mục với đúng 1 mục là ra thưa hoác, chết không gian.
      `stamp` (cụm chữ đóng dấu) **tối đa MỘT lần cả video** — server trả `stamp-qua-lieu`.

      **Kể bằng chuyển động, không bằng trang chiếu.** Chuyển động phải giải thích một điều — nhân quả,
      quy mô, thay thế, đổi trạng thái; mỗi beat MỘT hành động chính, xảy ra lúc hình đứng yên. Ngoài
      `archetype` (HÌNH), ghi thêm `co_che` (CÁCH hình kể) khi beat có một trong sáu cơ chế: `gom-mot-moi`
      (rời rạc → một mối) · `noi-tiep` (máy quay đi trạm này sang trạm kia) · `tay-sang-may` (làm tay →
      tự động) · `bam-ra-nhieu` (một cú bấm → quy mô lớn) · `thao-tac-app` (thao tác trong app thật, ảnh
      chụp thật) · `hanh-dong-he-qua`. Đừng lặp một cơ chế ba beat liền — `get_timeline.phân_bổ_cơ_chế`
      liệt kê cái chưa dùng. Dụng cụ có sẵn trong sandbox: `useCamera` (thế giới lớn + máy quay giữ → đi
      → giữ; chỉ takeover/glide/stage/split), `goChu` + `<ConTro/>` (terminal gõ chữ), `<VetMarker/>`
      (vệt dạ quang sau chữ). 3D chỉ bằng `perspective()` TRONG transform.

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
      - Beat định dùng máy quay / terminal / marker / 3D, hoặc một cơ chế kể → thêm `ky_thuat` hoặc
        `co_che` vào `find_examples`: tool trả kèm **mẫu viết sẵn** cho đúng dụng cụ đó. Mượn cấu trúc
        (thế giới + khoá máy quay, nhịp gõ, chỗ đặt vệt), nội dung lấy từ lời thoại.

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
      **Không nhét bậc 2/3 vào `over`** — sẽ đè mặt người. Ở khung NGANG dải `over` chỉ cao 400px:
      hình nhiều tầng (map-tree, wf-tree/gantt/swimlane/kanban…, chart-radar) → `glide`, hình giao
      diện (mockup-app, chat, terminal) → `stage`; server chặn nếu ép `over`. Cửa ải cũng trả về
      graphic khung ngang có **một nửa số chữ dưới 18px** — cỡ chữ đặt sàn px, đừng suy thuần từ H.
      Xem trên điện thoại thì nhãn nên ≥ 30px, «minh hoạ» ≥ 26px.
      **Khung NGANG — người nói chuyển chỗ ~1 giây đầu beat** (spring 26 khung, ~70% xong ở ~0,6s): beat
      `glide` thì khung người nói còn phủ sang nửa graphic, beat `takeover`/`stage` thì người nói còn mờ
      dần phía sau. Phần tử chính VÀO từ ~0,6s (hợp đồng ghi đúng số khung); trước đó để trống. Đo trên
      gtkh-peach 27/9: 31/39 graphic vào từ khung 0 → luồn dưới khung người nói hoặc nằm trên bóng người
      xám. **Pha RA:** máy quay lia cuối ≤ 40px, đẩy cuối ≤ 1,05 lần, mọi chữ cách mép HOP ≥ 60px — 16/39
      clip bị cú lia cuối cắt chữ ở mép.

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
      Rồi viết vào ĐÚNG id: `write_motion_graphic(clip_id, …)`. **Không ghi lên clip host (`h…`)** —
      renderer không đọc kind ở đó.
      **Làm lại / sửa một beat = ghi đè ĐÚNG `clip_id` cũ** (`write_motion_graphic` cùng id). Mỗi khoảng thời
      gian chỉ MỘT ô graphic: tạo ô mới chồng khít ô đã có bị server từ chối (`o-trung-cho`, kèm `clip_da_co`
      là id phải dùng). Muốn bỏ hẳn bản cũ thì `edit_clip(clip_id, delete: true)` — **đừng** viết mã trong suốt
      để «tắt» nó: bản thừa vẫn nằm trên timeline và dễ vẽ chồng lên bản thật.

   e. **Trung thực dữ liệu.** Số liệu, tên riêng, câu trích trong graphic phải **nguyên văn** trong
      lời thoại của beat đó. Thiếu sự kiện thật → đổi hình khác, **tuyệt đối đừng bịa/điền bừa**
      (không bịa số, xếp hạng, lượt xem, giá, logo, giao diện). (Nhãn bước, tên cột thì được diễn đạt
      lại từ ý trong câu.) Nhưng **đừng chép lại cả câu đang nói lên hình**: chữ trên hình trùng ≥ 60%
      câu thoại của beat thì cửa ải trả `chep-loi` — hình phải nói THÊM thứ lời không nói (số, cấu trúc,
      quan hệ, trước → sau); giữ tối đa một cụm từ khoá 2–5 tiếng làm tiêu đề.

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

   h. **NHẤN ZOOM — chỉ khi người nói THẬT SỰ chỉ vào một thứ cụ thể** trong graphic («dòng lệnh này», «con số 42%
      ở đây», «tin nhắn thứ hai»). Mã phải khai vùng trước bằng `vungNhan("ten", {x, y, w, h}, hienTu)` —
      `write_motion_graphic` trả `vùng_nhấn_đã_khai`; mẫu ở `find_examples(ky_thuat: 'nhan-zoom')`. Rồi gọi
      `nhan_zoom(project, clip_id, lan: [{vung, chu: '<từ khoá trong lời thoại>'}])`:
      - vùng NHỎ cần đọc to (một dòng, một con số) → `zoom` (mặc định); vùng lớn hoặc cần giữ ngữ cảnh xung quanh →
        `zoom: false, lam_toi: true`; khoảnh khắc chốt quan trọng nhất → cả hai.
      - Trần (máy chủ chặn): 1 cú mỗi beat (2 nếu beat ≥ 8 giây) · ≤ 25% số beat cả video · không hai graphic liền
        nhau · không ở dải `over` khung ngang · không dùng với graphic có `useCamera` (máy quay đã tự nhấn).
        `get_timeline.nhấn_zoom` cho biết còn bao nhiêu lượt.
      - Đặt xong thì `capture_frame` ở các mốc `chụp_kiểm` tool trả về: vùng phải nằm giữa khung, chữ đọc được,
        không cắt mất phần đang được nói tới.

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
   - **Beat không có motion graphic tự được vá một BONG BÓNG THOẠI** kiểu truyện tranh (chữ là
     chính lời đang nói, cắt thành cụm theo nhịp). Nên chụp khung một beat chưa dựng sẽ thấy nó —
     đó không phải phần tử ai đó đặt vào và **không phải chừa chỗ** cho nó: dựng graphic cho beat
     đó là bong bóng tự mất. Đừng vẽ lại một cái bong bóng trong mã của mình.
   - Bộ dưới 12 tư thế (`bộ_ít_tư_thế`) → cuối lượt nhắc người dùng vẽ thêm tư thế.

4c. **DUYỆT ĐỘC LẬP — sau khi MỌI beat đã dựng xong, một lượt cho cả video.** Người dựng không tự chấm
   bài của mình. Mở MỘT agent phụ (tool Agent / subagent) không tham gia lúc dựng, giao nó đúng việc:
   gọi `contact_sheet(<project>)` — mỗi graphic BA khoảnh khắc **vào · giữa · ra**, kèm bảng ô (clip,
   archetype, cơ chế, lời thoại, mốc giây) — **chỉ nhìn và chấm, không sửa gì**, rồi trả JSON mỗi clip:
   `{so, clip, diem: {doc_duoc, nhan_qua, bo_cuc, chat_lieu, cu_the, khong_chep_loi, dung_style,
   khong_bia, vao_ra, mot_tieu_diem}, ket_luan, loi, sua}` — thang 10, một câu lỗi và một câu cách sửa
   cho clip có tiêu chí dưới 8. Mười tiêu chí:
   - `doc_duoc` — chữ đọc được trên màn điện thoại ở cỡ thật; không chữ nhỏ, chìm nền, tràn thẻ.
   - `nhan_qua` — hình cho thấy một điều xảy ra / dẫn tới điều gì, không phải một trang chiếu tĩnh.
   - `bo_cuc` — lấp đúng vùng vẽ, không đè mặt người nói, không dồn một góc, không khối chồng nhau.
   - `chat_lieu` — ít vật hơn nhưng to, rõ ràng; không lưới thẻ rời rạc, không mẫu rẻ tiền (biên lai,
     mã vạch, giấy chứng nhận).
   - `cu_the` — nói đúng nội dung beat này, không phải một khuôn chung dán chữ khác vào.
   - `khong_chep_loi` — không viết lại câu đang nói lên hình.
   - `dung_style` — màu, font, cách tách lớp của style dự án (style tiết chế thì không phát sáng).
   - `khong_bia` — không số liệu, giao diện, logo, lượt xem bịa; tên, giá, bằng chứng, CTA khớp phiếu
     chỉ đạo; so sánh cùng thang đo; minh hoạ không có số thật thì có nhãn «minh hoạ».
   - `vao_ra` — ô VÀO không giật / không đè nhau lúc đang bay vào; ô RA đã kể xong (không còn chạy dở
     khi cắt sang ý sau).
   - `mot_tieu_diem` — hiểu được thông điệp mà không phải đọc nhiều thứ cùng lúc; chữ trên hình không
     tranh chỗ với mặt người nói hay phụ đề.
   **Kết luận mỗi clip — một trong ba:** `dat` (mọi tiêu chí ≥ 8) · `sua-cuc-bo` (hỏng ở số hình học,
   cỡ chữ, màu, nhịp — sửa đúng clip đó, giữ nguyên hướng) · `xem-lai-huong` (hình kể sai điều lời nói,
   sai quan hệ — đổi hình hoặc cách kể). Ghi vào bảng: `set_beat_direction` với `dat` → `da-duyet`,
   hai mức còn lại → `can-sua` kèm `ghi_chu` là lời chấm. Clip cần sửa → dựng lại đúng clip đó (bước 3,
   vẫn qua vòng tự sửa g), **chỉ sửa khoảng đó, giữ nguyên phần đã đạt**, rồi gọi agent phụ chấm lại
   bằng `contact_sheet(<project>, clips: [các clip vừa sửa])` thêm MỘT lượt. Tối đa hai lượt chấm cho
   một video — lượt thứ hai vẫn còn clip dưới 8 thì báo người dùng tên các clip đó cùng lời chấm, đừng lặp.
   Cuối cùng ghi kết luận CẢ VIDEO vào phiếu: `set_project_brief(ket_luan_duyet, ghi_chu_duyet)`.
   Không mở được agent phụ (môi trường không có tool đó) → bỏ phần chấm và **nói rõ với người dùng** là
   video chưa qua duyệt độc lập — không tự chấm thay.
   `contact_sheet` trả «đang dựng» ở lần gọi đầu mỗi phiên (máy chủ dựng bundle): gọi lại sau ~30 giây.
   Video dài trả nhiều trang: mỗi lượt tối đa 8 trang, gọi tiếp với `trang_tu`.

4d. **CHỮ — hai cửa ải mới ở `write_motion_graphic`, không phải việc tự nhớ.** Cửa ải quét chữ THEO THỜI
   GIAN: đoạn chữ ≥ 2 tiếng hiện chưa tới max(0,5s, 40% nhu cầu đọc ≈ 0,3s + 0,25s/tiếng) → trả
   `thoi-gian-doc` (sửa: cho vào sớm hơn, giữ tới hết beat, hoặc rút chữ). Sàn cỡ chữ áp CẢ HAI khung
   (`co-chu`): khung dọc — chữ phụ ≥ 22px, nửa số chữ ≥ 26px; khung ngang — chữ phụ ≥ 20px, nửa ≥ 22px.
   Đạt mà response có `chữ_cần_xem` (chữ hiện hơi ngắn, hoặc chữ gõ nằm trong khung canh giữa nên trôi)
   thì sửa luôn trong vòng tự sửa. Cuối video: `check_captions` — từ máy nghe không chắc + tên riêng lệch.

4e. **SOÁT KHÁCH QUAN — `review_video(<project>)`, cùng lúc với 4c.** Thứ ảnh tĩnh không thấy: tiếng
   động đè lên lời đang nói hoặc to giật mình ở khoảng lặng, tiếng động dồn, nhạc lấn giọng, và những
   khoảng hai lớp hình cùng đòi mắt (graphic chồng media, chữ chồng graphic). Có mốc giây + id clip —
   agent dựng tự sửa (hạ `vol` của clip sfx bằng `edit_clip`, dời/rút ngắn một clip), không cần agent phụ.
   Video có hiệu ứng chớp / glitch / đổi nền liên tục → gọi thêm với `nhay_sang: true` (dựng bản nháp,
   vài phút) để dò nháy sáng > 3 lần/giây. Phiếu chỉ đạo tắt tiếng động tự gắn thì phần âm chỉ còn nhạc.

4f. **NGƯỜI NÓI — bám mặt và tách nền (khi cần).** Dự án mới nạp video đã tự đo đường đi của mặt và tự
   bật bám mặt; dự án cũ thì không đổi gì. Người dùng than «mặt bị cắt / lệch trong dải chia nửa / ô glide»
   → `track_face(<project>)` đọc số trôi, rồi `ap_dung: true` và chụp lại vài beat. Dự án đã TÁCH NỀN →
   `check_matting` (người trên nền trắng + nền đen ở 6 mốc): thấy vệt ma, viền sáng, ngón tay mất thì báo
   người dùng MỐC GIÂY đó — đừng tự tách lại.

4g. **BIẾN THỂ — chỉ khi người dùng xin «làm thêm vài bản để thử».** Bản gốc phải ĐÃ DUYỆT. `make_variants`
   (tối đa 6): mỗi biến thể là một dự án riêng, đổi ĐÚNG MỘT thứ (chu-mo-dau · hinh-beat · style · khung ·
   nhac · nhip · mo-dau-noi) kèm một câu giả thuyết. Dựng đúng thay đổi đó trên dự án biến thể, KHÔNG sửa gì
   khác và không bao giờ sửa bản gốc. `variants_status` so từng biến thể với bản gốc («đúng một thay đổi» /
   «đổi ngoài phạm vi» / «chưa đổi gì») và đổi trạng thái (cho-duyet → da-duyet chỉ khi người dùng đã xem).
   Câu nói mở đầu mới mà chưa có bản ghi thật → «chờ tư liệu», báo người dùng cần ghi gì — không sinh giọng,
   không cắt ghép câu khác. Xuất: `export_variants` dựng lần lượt các bản đã duyệt và kiểm từng tệp (thời
   lượng, khung, tiếng). Bảng có CSV để người dùng mở bằng bảng tính.

5. **Xong.** Khi mọi beat đã có motion graphic đạt yêu cầu, báo user tóm tắt (bao nhiêu beat, dạng
   hình đã dùng — mode mascot thì kèm cả chuỗi tư thế — và kết quả duyệt độc lập) và nhắc bước xuất video.
   User báo **file xuất ra khác bản xem trước** (mất mũi tên, hình méo, thiếu nền, sai font) → việc
   ĐẦU TIÊN là bảo họ **tải lại trang editor (F5) rồi xuất lại**: tab mở từ trước lần máy chủ cập
   nhật vẫn xuất bằng bộ vẽ cũ. Vẫn khác sau khi tải lại thì mới báo lỗi, kèm tên video và giây bị
   lệch — đừng sửa mã graphic để «né» chỗ file xuất sai khi bản xem trước đang đúng.

## Vài lằn ranh
- Không tự chế phong cách/màu/font — mọi thứ đó nằm trong contract và catalog của MCP.
- **Font chỉ lấy qua `themeFont()`** — không gõ tên font trong `fontFamily` (máy người xem không có
  font đó thì chữ đổi hẳn mặt). Hướng thẩm mỹ có dòng **VAI CHỮ RIÊNG** (thường là font thương hiệu
  người dùng tự tải lên) → chữ đúng vai đó gọi `themeFont("<tên vai>")`; không có thì dùng khoá
  chuẩn trong contract. Đừng tự bịa tên vai hay khoá `tl-…`. Người dùng muốn video dùng font riêng
  của họ: agent **không tải font lên được** — hướng dẫn họ vào Cài đặt → Style thư viện → Chỉnh
  style → **FONT TẢI LÊN**, rồi «+ thêm vai chữ» trỏ vào font đó và Lưu; lượt dựng sau sẽ thấy vai
  trong hướng thẩm mỹ.
- Beat đóng video thường đã có component cố định của kênh — đừng ghi đè.
- Nếu một tool trả lỗi, **đọc lỗi rồi sửa theo đúng lỗi** (contract, gate, capture đều trả thông
  điệp cụ thể) — đừng đoán.
- **Lỗi THIẾU QUYỀN không phải lỗi để sửa.** «tài khoản của bạn không được dựng…», «gói của bạn
  không có…» (một style, một tính năng), «Hết hạn mức…», «chưa được mở…» là quyền quản trị viên giao
  cho tài khoản này. Dừng đúng việc đó, báo người dùng nguyên câu lỗi và gợi ý liên hệ quản trị viên
  (hết hạn mức thì chờ sang tháng hoặc nâng gói). **Đừng lách:** không tự đổi sang theme/style hay
  kiểu dựng khác thứ người dùng chọn, không tạo project mới để né hạn mức, không gọi lại với tham số
  khác cho lọt. Tool bị khoá (vd sinh ảnh AI) thì beat vẫn dựng bằng cách khác mà `luật_chọn` cho
  phép được — nhưng nói rõ với người dùng là đã bỏ qua tool đó.
