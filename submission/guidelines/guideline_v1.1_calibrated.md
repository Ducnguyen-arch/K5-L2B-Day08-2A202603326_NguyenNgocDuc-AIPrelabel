# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603326 - Nguyễn Ngọc Đức

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Mỗi ca khó có ảnh ví dụ (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.
> Nguồn ca: câu hỏi mở mục 6 của v1.0, ghi chú khi gán gold (`time_log.csv`), lỗi lặp của máy (`qc/review_stats.md`, đối chiếu gold ở REPORT mục 1), bẫy lọt (`qc/review_stats.md` mục 3), box `can_review`, mask SAM (`masks/`).

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Xe đạp điện / xe máy điện nhỏ: `bicycle` hay `motorbike`? | **Chưa chốt.** Trong 28 ảnh gold + batch mình không gặp ca nào đủ rõ để làm ví dụ, chốt mà không có ảnh thì chỉ là đoán. Tạm thời: chọn lớp tin nhất và tick `can_review` (theo 4.3 v1.0). | — | — |
| 2 | Khối xe máy đỗ dày **gần camera**: từng xe hay `ignore_region`? | Xe cao **≥ 20 px** thì **gán từng chiếc**, kể cả khi đỗ sát nhau. Chỉ cụm **≥ 5 xe cao < 20 px** mới dùng **một** `ignore_region` bao cả cụm. **Không** vẽ `ignore_region` cho từng xe riêng lẻ. | `img_018` | CAL-04 |
| 3 | Hàng nhô ra: box / mask xe có gồm hàng không? | Hàng **nằm trong** thùng hoặc trên khung xe (dưa hấu, rổ…) **thuộc** box và mask xe. Hàng **nhô ra** khỏi xe quá ~20% chiều dài xe thì **không** tính vào box và mask. Áp dụng như nhau cho box (4.1.5) và mask (4.2). | `img_003` | CAL-01 |
| 4 | Người ngồi trong xe ba bánh có mui (thấy thấp thoáng): ngưỡng 20% có hợp lý? | Thay ước lượng "≥ 20% diện tích" (hai người đánh giá dễ lệch nhau) bằng tiêu chí dễ kiểm hơn: gán `person` khi **thấy rõ đầu hoặc thân trên**. Chỉ thấy tay, chân hay bóng mờ sau mui thì **không** gán. | `img_012` | CAL-02 |
| 5 | Kính chắn gió / cửa sổ: thuộc mask xe không? | Kính **thuộc** box và mask xe (không khoét lỗ). Người **chỉ thấy qua kính**, không rõ đường bao, thì **không** gán `person`. | `img_020` | CAL-03 |
| 6 | Box máy chồng nhau nhiều trong ảnh đêm: gộp hay giữ? | **Không gộp tự động.** Mỗi xe nhìn thấy được giữ **đúng một** box. Box chồng lên cùng một xe, hoặc nằm trên nền tối không có vật, thì **xoá**. Ảnh đêm phải phóng to để kiểm từng box. | `img_026` | CAL-05 |
| 7 | Có đặt mức `conf` tối thiểu để chấp nhận box máy không cần sửa? | **Không đặt.** Box sai lớp có điểm rất cao (`truck` 0.88 trên xe thật là `car` ở `img_001`, `car` 0.81 trên tuk-tuk ở `img_012`). `conf` chỉ dùng để sắp thứ tự duyệt, **mọi box đều phải xem**. | `img_001`, `img_012` | CAL-06 |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_012` | 3 tuk-tuk có mui, cabin kín, nhìn giống ô tô nhỏ | YOLO26m gọi là `car` (0.68 / 0.71 / 0.81). YOLOE-26s nhận đúng `three_wheeler` nhưng điểm thấp (0.19–0.30) nên bị cắt ở `--conf 0.3` | Cả 3 là `three_wheeler` (CAL-07) | **Mục 2, lớp `three_wheeler`:** xe 3 bánh có mui hay cabin kín **vẫn luôn** là `three_wheeler`; đếm số bánh nhìn thấy, không xét theo hình dáng thân xe. Khi duyệt ảnh có xe ba bánh, phải kiểm **mọi** box `car` / `truck`. |
| `img_013` | Tuk-tuk có người lái, thành xe hở hai bên | Mask SAM 2.1 (3 click) tô luôn tay, chân người lái và nền nhìn qua khung hở (người, bàn ghế phía sau). Tương tự ở `img_003`: dính cả em bé ngồi trên thùng | Mask chỉ gồm xe; người và nền phía sau phải bỏ ra (CAL-08) | **Mục 4.2, thêm 4.2.5:** mask xe **không** gồm người lái / hành khách (người là vật riêng, giống box ở 4.1.4). Khoảng hở lớn nhìn thấy nền phải bỏ khỏi mask (áp dụng 4.2.2). Khi dùng SAM, đặt **≥ 1 click âm trên mỗi người** ngồi trên xe và trên mỗi khoảng hở lớn. |
| `img_036` | 2 `person` ở xa, mờ, cao 42 px; lúc gán gold mình đã tick `can_review` | — (ca gán tay) | Gán bình thường, **bỏ tick** `can_review` (CAL-11) | **Mục 3, thuộc tính `can_review`** (làm rõ, không đổi nghĩa): vật ≥ 20 px vẫn gán dù mờ, vẽ box quanh phần nhìn thấy. `can_review` **chỉ** dùng khi không chắc **quy tắc**, không dùng khi vật khó vẽ. |

**Các ca đã ghi trong `calibration_decisions.csv` nhưng KHÔNG đổi guideline** (do mình hoặc máy làm sai, không phải do guideline mơ hồ):

| Ca | Ảnh | Vì sao không đổi guideline |
|---|---|---|
| CAL-09 (`loi_may`) | `img_001`, `img_020` | YOLO26 nhầm `car` ↔ `truck` ↔ `bus`. Định nghĩa ba lớp này ở mục 2 v1.0 đã đủ rõ, nên đây là lỗi máy; người duyệt chỉ cần đổi lớp. |
| CAL-10 (`bay_lot`) | `img_004`, `img_031` | Lọt bẫy `person` bị đổi thành `bicycle`. Quy tắc đã rõ; lỗi là do mình bỏ bước kiểm nhãn khi duyệt, nên sửa ở checklist chứ không sửa guideline. |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt 6/7 câu hỏi mở (câu 2, 3, 4, 5, 6, 7); câu 1 (xe đạp điện) để mở vì chưa có ảnh ví dụ. Bổ sung quy tắc: tuk-tuk có mui luôn là `three_wheeler`; mask xe không gồm người và bỏ khoảng hở lớn; làm rõ cách dùng `can_review`. | CAL-01 → CAL-08, CAL-11 (CAL-09, CAL-10 không đổi guideline) |
