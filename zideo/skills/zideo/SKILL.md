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
   Response có `cập_nhật_plugin` → plugin của người dùng đã cũ: nhắc họ **một lần** trong cuộc trò
   chuyện (bản mới có gì + các bước trong `cách_cập_nhật`, chạy ở Terminal/PowerShell chứ không phải
   khung chat), rồi làm tiếp việc đang làm — đừng dừng chờ họ cập nhật.

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
     dùng muốn tắt tiếng động tự gắn thì `sfx_tu_dong: false`; không cho tốn tiền thì `tra_phi: "khong"`;
     nói «không caption / không phụ đề» thì `caption: "khong"` — server bỏ yêu cầu `<CaptionBand />` mà vài style
     (clay-proof) bắt buộc, và hợp đồng dặn không dùng dải phụ đề. Chỉ ghi `cam_dua_vao` thì KHÔNG đủ: cửa ải
     của style vẫn chặn mọi graphic thiếu CaptionBand.
     **Phụ đề thiết kế** (khung 9:16 + 16:9, kiểu «Giải thích bằng hình»): `get_timeline.phụ_đề_thiết_kế` BẬT thì
     server tự vẽ phụ đề theo lời, đổi kiểu theo bố cục người nói (dọc: nhãn trên đường chia / chữ to khi người nói
     vắng / dải karaoke; ngang: dải dưới). Graphic KHÔNG dùng <CaptionBand />, không tự làm dải chữ, và chừa làn
     phụ đề (toạ độ trong hợp đồng) — cửa ải trả `de-phu-de` khi chữ graphic rơi vào làn. Bật/tắt: phiếu `caption`.
     Video nhắc tên thương hiệu / sản phẩm / người → ghi `tu_rieng` (viết ĐÚNG như phải hiện) — hợp đồng
     nhắc lại cho mọi beat, và `check_captions` soát phụ đề theo danh sách đó.
   - `get_transcript` cả video, rồi `set_beat_direction` MỘT lượt cho mọi beat: mắt nhìn vào đâu (một
     thứ), hành động hình (cái gì đổi), chữ trên hình (thêm điều lời không nói), tư liệu, tiêu chí duyệt
     («đạt khi …»). Beat chỉ là ý kiến riêng, lời hứa, câu cảm thán → `trang_thai: "de-trong"` (người
     nói trọn khung). Nhìn cả bảng trước khi dựng: hai beat liền nhau đừng cùng hình VÀ cùng cách kể.
   - **Vật thể 3D — TỰ CHỌN, không chờ người dùng chỉ.** Lúc lập bảng, dò lời thoại tìm VẬT HỮU HÌNH có hình dạng quen
     thuộc mà xem khối của nó giúp người xem hiểu hơn: thiết bị, sản phẩm, linh kiện, máy móc, phương tiện, công trình,
     bộ phận cơ thể / máy — nhất là khi lời kể CẤU TẠO (bên trong có gì), so sánh bộ phận, tách lớp, hay xoay xem các
     mặt. Beat như vậy ghi «vật 3D: <vật>» vào hành động hình, và lúc dựng gọi `find_examples(ky_thuat: "vat-3d")`.
     TRẦN: tối đa 1 beat mỗi đoạn và 3 beat cả video (server cảnh báo `vat-3d-day` khi vượt); chọn chỗ đáng nhất.
     KHÔNG dùng cho khái niệm trừu tượng, giao diện app / web, logo, con người, chữ, biểu đồ, hay vật không dựng giống
     được bằng hình học (con vật, khuôn mặt, đồ ăn — dựng dở còn tệ hơn ảnh thật). Người dùng chỉ định thì làm theo
     họ, bỏ qua trần.
     **GIỐNG THẬT NHẤT CÓ THỂ** — chuẩn là robot hút bụi trong mẫu: nhựa bóng phủ clearcoat phản chiếu ánh sáng studio
     (`<MoiTruongStudio/>`), mép vát bằng mặt cắt xoay (latheGeometry) chứ không trụ / hộp cạnh sắc, đủ chi tiết nhỏ
     của vật thật (khe, ốc, đèn LED, gai lốp, lông chổi, chân linh kiện…), tỉ lệ đúng vật thật, bóng đổ mềm dưới vật,
     nhìn chéo từ trên ~20°, xoay chậm. Màu phẳng kiểu đồ chơi / khối thô = chưa đạt — `capture_frame` xem lại và
     làm tiếp tới khi trông như ảnh chụp sản phẩm.
   - Server tự chèn phiếu + dòng của beat vào `get_prompt_contract` — dựng đúng hướng đã chốt; đổi ý
     thì sửa bảng trước. Mã đạt thì dòng tự sang «chờ duyệt». Người dùng xin bảng → `export_direction_table` (CSV).
   - Dự án cũ đã dựng xong mà không có phiếu: không bắt buộc lập lại, trừ khi người dùng yêu cầu.
   - Người dùng đưa VIDEO MẪU («dựng giống video X») → `recipe_from_reference(<project>, du_an_mau: X,
     ap_dung: true)` TRƯỚC khi lập bảng: đo nhịp đổi cảnh, người nói đứng đâu bao nhiêu % thời lượng, hình
     đổi trước hay sau lời, nhạc / tiếng động. Dự án mẫu đã xuất thì đo bản xuất (video đã dựng). Mục nhãn
     «không» (hình kể gì) thì NHÌN ảnh bảng cảnh tool trả kèm. Hợp đồng mọi beat nhận khối LUẬT TỪ VIDEO MẪU.

2c. **DỰNG THEO ĐOẠN LIỀN MẠCH — khi `get_timeline` có khối `đoạn_liền_mạch`** (khung ngang 16:9 HOẶC dọc
   9:16, kiểu «Giải thích bằng hình»). Video chia thành vài ĐOẠN dài (mỗi đoạn một đề mục, 25–120 giây). Trong một đoạn, mọi beat là CẢNH của cùng một
   THẾ GIỚI: vật cũ biến hình / đổi vai, máy quay đi xuyên thế giới, không cắt cảnh. Mỗi cảnh vẫn là một
   graphic riêng để người dùng sửa, nhưng mã do SERVER sinh — `write_motion_graphic` bị từ chối trên cảnh
   (`canh-cua-doan`). Bước 3 bên dưới thay bằng vòng này, cho TỪNG đoạn:
   1. Đọc lời CẢ đoạn (`get_transcript`, từ `bắt_đầu` tới hết đoạn) + tên/ý chính. AI chia đoạn sai đề mục
      → `set_doan` (doi_ten · gop · tach). Thẻ người nói: `phai` (thẻ dọc phải, mặc định) · `trai` ·
      `goc_tren` / `goc_duoi` (thẻ ngang NHỎ ở góc phải — khi cảnh cần gần trọn khung) · `tron` (beat
      chỉ là ý kiến riêng, không graphic). Không bao giờ đặt người nói giữa khung.
      KHUNG DỌC thay thẻ bằng BỐ CỤC: `chia` (B — người nói nửa dưới, sân khấu graphic y 96–979, mặc định) ·
      `vang` (A — người nói vắng, sân khấu giãn tới y 1400; cho cảnh cần cả sân khấu: sơ đồ cao, danh sách dài,
      màn hình điện thoại; server chặn khi A vượt 45% thời lượng đoạn) · `tron`.
   2. Gọi `get_prompt_contract(<project>, clip_id: <id beat một cảnh của đoạn, vd "h3">)` → HỢP ĐỒNG CỦA ĐOẠN:
      API thế giới, mọi cảnh với lời theo KHUNG (`từ@khung`), thẻ người nói / vùng trống, bảng màu, chỉ đạo.
      Luật nội dung thường (chữ không chép lời, không bịa số liệu) vẫn áp cho thế giới; được dựng lại giao diện sản phẩm thật.
   3. NGHĨ MỘT THẾ GIỚI cho cả đoạn trước khi viết: những vật nào sống suốt đoạn, mỗi ý của lời làm vật nào
      đổi thế nào (sáng lên, tách đôi, thu lại thành chi tiết của vật lớn hơn…), máy quay nhìn vào đâu ở
      từng cảnh. Thế giới lớn hơn khung (vd 6000×4000) để máy quay có đường đi. KHUNG DỌC: khung nhìn chỉ rộng
      972 — xếp thế giới THEO CHIỀU DỌC (vd 2400×5200), vật chính lấp ~70% bề ngang sân khấu ở zoom dự định.
      `write_doan_world`: mọi thứ đổi theo thời gian đi qua `s.<tham_số>` hoặc `pop(tên)`; chuyển động nền
      lặp dùng `frameGoc`, KHÔNG dùng `frame` (về 0 ở mỗi cảnh → giật ở chỗ nối).
      Vật 3D trong thế giới (beat đã đánh «vật 3D» ở bảng): `find_examples(ky_thuat: "vat-3d")` trả kèm mẫu
      THẾ GIỚI — mỗi bộ phận một `<ThreeCanvas orthographic>` riêng, ghép khít lúc lắp nguyên, bay theo `s` / `pop`.
   4. `write_doan_canh` cho từng cảnh THEO THỨ TỰ: chỉ khai cái đổi (`tr`, `pop`, `cam` — KHÔNG có ô
      tiêu đề / thanh HUD / bộ đếm ở cả hai khung, đừng khai tieuDe / dem), khung tính từ đầu cảnh. NEO THEO TỪ KHOÁ: thay số
      khung bằng chuỗi — `"deploy"`, `"hai nhánh"`, `"deploy#2"`, `"deploy+6"` — để vật đổi ĐÚNG lúc người nói gọi tên;
      không khai khung 0 (server tự nối từ cuối cảnh trước). Đọc `nay.trangThaiCuoi` — đó
      là đầu vào cảnh sau. Hành động chính xảy ra lúc máy quay ĐỨNG; máy lùi ra toàn cảnh thì cho nhãn nhỏ
      mờ đi (chữ trên màn phải ≥ 16px sau zoom — dọc ≥ 22px). `lop_canh` (lớp riêng một cảnh) phải mờ hẳn trước `dur`.
   5. Lỗi trả về (bảng `mã_lỗi` trong hợp đồng đoạn): `nhip-sai` (sửa đúng từng dòng, chưa có gì được lưu) ·
      `chu-nho-doan` (chữ < 16px trên màn, dọc < 22px) · `co-chu` (sàn cỡ chữ ở khung đã yên — tính TRÊN MÀN, tức cỡ trong thế
      giới × zoom) · `thoi-gian-doc` · `noiLech` (khung cuối cảnh trước ≠ khung đầu cảnh sau). Cảnh hỏng thì bảng nhịp
      đã lưu nhưng mã giữ bản cũ — danh sách cảnh đánh `hongCuaAi`. `canhBao` không chặn nhưng nên sửa (vd máy quay bị
      kẹp ở mép thế giới → vật lệch khỏi vùng trống: nới `the_gioi`, đặt vật cách mép ≥ 1920/zoom — khung dọc ≥ 486/zoom). Sửa tới khi `ok: true`.
      Màu chỉ dùng các khoá trong `bảng_màu.khoá_themeColor` của hợp đồng đoạn. Khai `archetype` / `co_che` cho mỗi cảnh.
   6. Contact sheet chụp mỗi cảnh: khung đầu · 1–2 khoảnh khắc HÀNH ĐỘNG CHÍNH (đọc từ bảng nhịp — lúc nhiều thay đổi
      xong nhất) · khung cuối; mỗi ô kèm `co_chu_tren_man` (cỡ chữ ĐO trên màn, đã nhân zoom, chỉ chữ trong khung).
      Ảnh thu nhỏ 1/3 — đừng ước cỡ chữ từ ảnh. Cần nhìn kỹ một lúc khác thì `capture_frame` (tối đa 6 mốc mỗi lượt).
   Cả hai khung: không có ô tiêu đề, thanh HUD hay bộ đếm nào trên khung — đừng tự vẽ chúng trong thế giới;
   phụ đề thiết kế (nếu bật) tự chạy ngoài sân khấu, không cần chừa chỗ trong thế giới.
   Câu mở đoạn (beat id đuôi `m`) là người nói trọn khung — không đặt graphic. Bảng đạo diễn (2b) vẫn
   lập theo beat = theo cảnh. Duyệt (4c) chấm thêm tiêu chí `lien_mach` cho các ô có `doan`.

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
      tự động) · `bam-ra-nhieu` (một cú bấm → quy mô lớn) · `thao-tac-app` (thao tác trong app thật — ảnh
      chụp hoặc giao diện dựng lại bằng mã) · `hanh-dong-he-qua`. Đừng lặp một cơ chế ba beat liền — `get_timeline.phân_bổ_cơ_chế`
      liệt kê cái chưa dùng. Dụng cụ có sẵn trong sandbox: `useCamera` (thế giới lớn + máy quay giữ → đi
      → giữ; chỉ takeover/glide/stage/split), `goChu` + `<ConTro/>` (terminal gõ chữ), `<ConTroChuot di={[{f,x,y}…]} nhan={F}/>` (con trỏ chuột macOS bấm giao diện), `<VetMarker/>`
      (vệt dạ quang sau chữ), `trangThai([f…])` (máy trạng thái theo lời — một giao diện đi qua 3–5 trạng thái thay vì
      cắt khung), `bayGiua(a, b, bat)` (FLIP — vật bay giữa hai bố cục), `dongHoTua` / `soDem` / `<SoLat/>` (đồng hồ tua,
      số đếm có nhoè, số lật), `<GoiTin/>` (gói tin chạy trên dây), `<WipeTruocSau/>`, `<Iris/>`. Nghiêng một TẤM PHẲNG (terminal,
      thẻ bay vào) chỉ bằng `perspective()` TRONG transform; VẬT CÓ KHỐI thì dựng 3D thật — xem «Vật thể 3D» bên dưới. Chữ chỉ làm giao diện trông thật (tên cột, đường dẫn, log) bọc `data-zd-ket-cau`:
      ra khỏi sàn chữ chính, nhưng sàn cứng 14px dọc / 12px ngang và ≤ 40% diện tích chữ — chữ mang ý không bao giờ gắn.
      Style tiết chế: quầng màu chỉ được trên VẬT (`<svg>`/`<img>` hoặc `data-zd-vat`), không trên chữ / thẻ.
      VÙNG «sau» (SAU LƯNG — chỉ khung dọc ĐÃ TÁCH NỀN): người nói đứng trọn khung phía TRƯỚC graphic — vật sau đầu / vai bị
      người che (logo «ném ra sau lưng», vật chui ra từ sau vai, vòng sáng sau đầu). Vẽ VẬT, không tô nền; chữ cần đọc đặt hai
      bên / trên đầu, không sau mặt. Dùng vài beat mỗi video. Chưa tách nền → server trả `sau-can-tach-nen`.
      MÁY QUAY NGƯỜI NÓI — `set_host_camera(beat, zoom, lia)` (cả hai khung): `day-cham` cho beat người nói trọn khung không
      graphic, `nhan` / `zoom-giat` cho câu chốt (zoom-giat 1–2 lần mỗi video), `vao-ra`, `cat-gan`; `lia: trai|phai` = khung
      vụt vào kèm nhoè ở chỗ đổi ý lớn (2–3 lần mỗi video). Không đặt cú máy mạnh trên beat graphic đang chuyển động nhiều.

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
      xám. Beat trước CÙNG vùng (trọn khung nối trọn khung) thì người nói đứng yên — vào ngay, đừng để
      trống trang; hợp đồng ghi rõ từng beat. **Pha RA:** máy quay lia cuối ≤ 40px, đẩy cuối ≤ 1,05 lần,
      mọi chữ cách mép HOP ≥ 60px — 16/39 clip bị cú lia cuối cắt chữ ở mép.
      **Beat dài (> 9 giây — dự án đặt mật độ beat thưa):** kể 2–3 NHỊP trên CÙNG một hình, mỗi nhịp mở
      đúng mốc lời trong khối timing (thêm tầng, đổi trạng thái, máy quay sang trạm kế); không đứng yên quá
      ~4 giây, không đổi hẳn hình giữa beat. Lập bảng đạo diễn thì ghi sẵn các nhịp vào `hanh_dong`, và xếp
      xen kẽ takeover / glide — beat dài mà hai takeover liền nhau là người nói vắng 25–40 giây.
      Sang nhịp mới thì cho lớp cũ rời khung hẳn hoặc để lớp mới đè lên — đừng THU NHỎ lớp cũ để nhường
      chỗ (duyệt gtkh-clay 27/9: chữ lớp cũ tụt còn 15–21px ở 3/24 clip). Lớp đè lên chữ phải có nền đục;
      màu surface của vài style hơi trong, chữ bên dưới lộ mờ. Hai beat liền nhau đừng dùng lại cùng một
      hình (cột bậc thang, cửa sổ ứng dụng) — beat dài thì người xem nhớ hình rõ hơn.

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
      (không bịa số, xếp hạng, lượt xem, giá). Được DỰNG LẠI giao diện của sản phẩm / thương hiệu thật được
      nhắc tới (logo qua add_logo) — nội dung bên trong lấy từ lời thoại hoặc để trung tính. (Nhãn bước, tên cột thì được diễn đạt
      lại từ ý trong câu.) Nhưng **đừng chép lại cả câu đang nói lên hình**: chữ trên hình trùng ≥ 60%
      câu thoại của beat thì cửa ải trả `chep-loi` — hình phải nói THÊM thứ lời không nói (số, cấu trúc,
      quan hệ, trước → sau); giữ tối đa một cụm từ khoá 2–5 tiếng làm tiêu đề.

   f. **Icon/logo thật, đừng vẽ tay.** Danh từ cụ thể/khái niệm có biểu tượng → `search_icon`
      (Iconify) → `pull_asset` → dùng `getAssetUrl`. Tên thương hiệu → `add_logo`. Chỉ vẽ `<path>`
      cho sơ đồ/giao diện tự chế mà không kho nào có.

   f1. **Người dùng dán ảnh vào chat để dùng trong video → tự tải lên, đừng nhờ họ.** Gọi
      `upload_link`, rồi chạy `curl -sS --data-binary "@<đường dẫn>" "<url>&ten=<tên gợi nhớ>"` cho
      từng ảnh — đường dẫn là dòng `[Image: source: …]` cạnh ảnh (Windows PowerShell: `curl.exe`).
      Mỗi lệnh trả `id` (`broll/up-…`); gọi `view_asset(id)` để nhìn lại ảnh nào vào ô nào, không
      bắt người dùng đổi tên tệp. Chỉ khi không có đường dẫn tệp (chat web) mới nhờ họ tải ở trang
      **Assets → Tải lên** của Zideo, xong gọi `list_assets`. Ảnh tải lên chỉ tài khoản đó thấy.

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
   - **Bộ NHÂN VẬT MÃ** (`get_mascot_catalog` có `loại_nhân_vật: MÃ`): nhân vật vẽ bằng code, tự thở / chớp / nhép
     môi — không có luật «tư thế trùng qua ranh giới beat». Thêm **cử chỉ tay** `cu_chi: {loai, chu?}` (khoá trong
     `cử_chỉ_có_sẵn`, ~40–60% số beat, khớp câu: «nhìn cái này» → chi-vao tự chỉ vào graphic, «thứ nhất» → mot-ngon,
     «chắc chắn» → chat) và `nhin: "graphic"` cho beat câu đang nói về graphic phía trên.

4c. **DUYỆT ĐỘC LẬP — sau khi MỌI beat đã dựng xong, một lượt cho cả video.** Người dựng không tự chấm
   bài của mình. Mở MỘT agent phụ (tool Agent / subagent) không tham gia lúc dựng, giao nó đúng việc:
   gọi `contact_sheet(<project>)` — mỗi graphic BA khoảnh khắc **vào · giữa · ra**, kèm bảng ô (clip,
   archetype, cơ chế, lời thoại, mốc giây; cảnh của đoạn có thêm ô HÀNH ĐỘNG CHÍNH và mọi ô graphic kèm
   `co_chu_tren_man` — chấm `doc_duoc` theo số đo đó, không ước từ ảnh thu nhỏ) — **chỉ nhìn và chấm, không sửa gì**, rồi trả JSON mỗi clip:
   `{so, clip, diem: {doc_duoc, nhan_qua, bo_cuc, chat_lieu, cu_the, khong_chep_loi, dung_style,
   khong_bia, vao_ra, mot_tieu_diem}, ket_luan, loi, sua}` — thang 10, một câu lỗi và một câu cách sửa
   cho clip có tiêu chí dưới 8. Mười tiêu chí (cảnh của đoạn liền mạch thêm tiêu chí thứ mười một):
   - `doc_duoc` — chữ MANG Ý đọc được trên màn điện thoại ở cỡ thật; không chữ nhỏ, chìm nền, tràn thẻ (chữ kết cấu
     `data-zd-ket-cau` được nhỏ hơn — trừ điểm nếu nó đang mang thông tin chính).
   - `nhan_qua` — hình cho thấy một điều xảy ra / dẫn tới điều gì, không phải một trang chiếu tĩnh; cộng điểm khi
     cùng một giao diện ĐỔI TRẠNG THÁI đúng lúc từ khoá được nói thay vì cắt sang khung mới.
   - `bo_cuc` — lấp đúng vùng vẽ, không đè mặt người nói, không dồn một góc, không khối chồng nhau.
   - `chat_lieu` — ít vật hơn nhưng to, rõ ràng; không lưới thẻ rời rạc, không mẫu rẻ tiền (biên lai,
     mã vạch, giấy chứng nhận).
   - `cu_the` — nói đúng nội dung beat này, không phải một khuôn chung dán chữ khác vào.
   - `khong_chep_loi` — không viết lại câu đang nói lên hình.
   - `dung_style` — màu, font, cách tách lớp của style dự án (style tiết chế: không phát sáng chữ / giao diện — quầng chỉ trên vật thể).
   - `khong_bia` — không số liệu, lượt xem, xếp hạng bịa (giao diện dựng lại của sản phẩm thật thì được); tên, giá, bằng chứng, CTA khớp phiếu
     chỉ đạo; so sánh cùng thang đo; minh hoạ không có số thật thì có nhãn «minh hoạ».
   - `vao_ra` — ô VÀO không giật / không đè nhau lúc đang bay vào; ô RA đã kể xong (không còn chạy dở
     khi cắt sang ý sau).
   - `mot_tieu_diem` — hiểu được thông điệp mà không phải đọc nhiều thứ cùng lúc; chữ trên hình không
     tranh chỗ với mặt người nói hay phụ đề.
   - `lien_mach` (CHỈ ô có `doan` — cảnh của đoạn liền mạch; thêm vào `diem`) — ô RA của cảnh trước và ô
     VÀO của cảnh sau giống hệt; vật cũ biến hình / đổi vai thay vì biến mất rồi hiện cái mới; máy quay đi
     có chủ đích (tới đúng thứ lời đang nói), không lắc qua lại; thẻ người nói không đè nội dung chính. Đa dạng
     trong đoạn đo theo HÀNH ĐỘNG (co_che), không theo kiểu hình — hai cảnh liền cùng một hành động mới là lặp.
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

## Cảm giác chuyển động

`get_timeline` có mục `cảm_giác_chuyển_động` khi người dùng đã tick ở modal tạo video (ease bậc 4 · nhoè theo tốc
độ · vào mờ → nét, hoặc mục họ tự lưu). Đã chọn thì đó là LUẬT của dự án: đọc mục CẢM GIÁC CHUYỂN ĐỘNG trong
`get_prompt_contract`, dùng dụng cụ tương ứng (`chuyenMuot` · `nhoeTocDo` · `vaoMoNet`; video theo đoạn nhận chúng
qua props của VeTheGioi), và ghi mã xong đọc `cảnh_báo_cảm_giác` trong phản hồi — sửa trước khi sang beat khác (máy
chủ chỉ cảnh báo, không chặn). Chế độ «chỉ» = không thêm kiểu chuyển động nào khác (không spring nảy). Không chọn =
bạn tự sáng tạo. Người dùng đổi ý trong chat → `set_motion_feel`.

Người dùng thấy một chuyển động ưng ý và bảo **«lưu cảm giác này / lưu cách chuyển động ở beat N để lần sau dùng»**:
đọc mã của beat đó (beat N = clip host thứ N theo thời gian; video theo đoạn thì mã thế giới nằm trong hợp đồng của
đoạn), RÚT RA cảm giác — đường cong, nhịp so với lời, nhoè, cách vật hiện / đi — chứ không chép bố cục hay nội dung
riêng của video đó, viết luật cho một agent khác đọc là làm lại được (không số khung tuyệt đối). Đưa người dùng xem
bản tóm tắt (tên ô tick, một dòng mô tả, luật, dụng cụ nếu có) và CHỜ họ đồng ý, rồi mới `save_motion_feel`. Lần tạo
video sau nó hiện thành ô tick «của bạn». Xem / sửa / xoá: `list_motion_feels`, `edit_motion_feel`. Lỗi «thiếu quyền»
= tài khoản chưa được admin giao «Lưu cảm giác chuyển động» — báo người dùng, đừng tìm đường vòng.

## Logic SFX

`get_timeline` có mục `logic_sfx`: «MẶC ĐỊNH» (graphic lẻ tự gắn âm theo hình; video theo đoạn KHÔNG có âm tự
động), «TẮT», hoặc một BỘ người dùng đã lưu. Có bộ thì server TỰ ĐẶT âm theo sự kiện (graphic / mốc moc() ở MG lẻ;
cảnh · vật bật · máy lia · vật trượt ở video theo đoạn) — đừng đặt tay chồng lên; ở MG lẻ khai moc() đúng mốc nhấn.
Âm còn lại làm theo luật tay trong mục LOGIC SFX của `get_prompt_contract`; chế độ «chỉ» = không dùng tệp ngoài bộ.
Đổi bộ: `set_sfx_logic`.

Người dùng ưng tiếng động của một video và bảo **«lưu logic sfx này (tên X)»**: `learn_sfx_logic(project)` → đọc
bảng sẽ học + `am_tay` (âm đặt tay không khớp sự kiện, kèm lời và graphic quanh đó) → viết `luat_tay` (khi nào đặt
âm gì, không giây tuyệt đối) → đưa người dùng xem tóm tắt và CHỜ họ đồng ý → `save_sfx_logic(project, ten, mo_ta,
luat_tay)`. Xem / sửa / xoá: `list_sfx_logic`, `edit_sfx_logic`. Lỗi «thiếu quyền» = tài khoản chưa được admin giao
«Lưu logic SFX» — báo người dùng.

## Dựng nhân vật mã từ ảnh mẫu

Người dùng gửi ảnh một nhân vật (linh vật thương hiệu, chân dung hoạt hình) và muốn nó làm người nói trong mode
Mascot: `get_mascot_code_contract` (hợp đồng + mã mẫu Zi) → viết mã theo hợp đồng → `write_mascot_code(bo, ten, code)`.
Cửa kiểm trượt thì sửa đúng lỗi nó nêu; qua thì app trả **ảnh tổng tư thế** — NHÌN và so với ảnh mẫu (màu, tỉ lệ, nét
nhận diện, tay không vắt ngang mặt), sửa rồi gửi lại cùng `bo` tới khi giống. Xong báo người dùng id bộ để chọn ở panel
Người nói. Sửa một bộ có sẵn: `get_mascot_code_contract(bo)` trả kèm mã hiện tại.

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
