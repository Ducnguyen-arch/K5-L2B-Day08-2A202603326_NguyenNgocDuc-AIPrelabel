# Báo cáo Day 08 — 2A202603326 - Nguyễn Ngọc Đức

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26m (`r08`: `--split all --conf 0.05`) | `--conf 0.3` mặc định cho cả 6 lớp, vì `threshold_sweep` không chọn được ngưỡng riêng cho lớp nào | `r01` (conf 0.25: 215 box / 8 ảnh pilot) và `r02` (conf 0.5: 139 box) cho thấy ngưỡng đổi rất nhiều số box. Vì vậy chạy thô ở 0.05 (`r08`) rồi để `merge_prelabels` cắt ngưỡng |
| three_wheeler | YOLOE-26s, prompt: `three_wheeler` (`r09`: `--split all --conf 0.05`) | `--conf 0.3` mặc định (sweep không đạt recall 90%) | YOLO26 là model COCO nên không có lớp này (gọi tuk-tuk là car/truck); YOLOE gõ chữ được (`r04`) |

**Kết quả quét ngưỡng** (`qc/threshold_sweep.md`, mục tiêu recall 90%, trần ngưỡng 0.5, quét trên 4 ảnh gold). 

| Lớp | n_gold | Recall tối đa (ở ngưỡng) | Recall ở 0.3 (mặc định) | Kết luận của script |
|---|---:|---|---|---|
| person | 27 | 85% (0.10–0.15) | 78% (21/27) | không đạt 90% → mặc định |
| bicycle | 5 | 40% (0.10–0.30) | 40% (2/5) | không đạt 90% → mặc định |
| motorbike | 13 | 77% (0.10) | 69% (9/13) | không đạt 90% → mặc định |
| car | 8 | 88% (0.10–0.85) | 88% (7/8) | không đạt 90% (chỉ thiếu 1 vật) → mặc định |
| bus | 0 | — | — | 0 vật gold (< 5) → không chọn |
| truck | 1 | 0% | 0% (0/1) | 1 vật gold (< 5) → không chọn |
| three_wheeler | 7 | 86% (0.10–0.15) | 57% (4/7) | không đạt 90% → mặc định |

- Lệnh `merge_prelabels.py`:
  ```bash
  python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json \
      --thr "" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json
  ```
- **Phiên bản SAM bạn chọn:** **SAM 2.1** (`sam2.1_b.pt`, `r07`: YOLO26m ra box → SAM 2.1 ra mask, 16 mask / 2 ảnh, ~16.6 s/ảnh trên CPU). Lý do: mask tốt hơn SAM 1 (`mobile_sam`), mà vẫn chạy được trên CPU, tốn ít RAM và nhẹ hơn SAM 3. **Không dùng SAM 3** do chưa cấp quyền và nặng máy.
- Lỗi nào **không sửa được bằng đổi ngưỡng**? Sửa thế nào?
  - **Tuk-tuk bị gọi `car` với điểm cao (`img_012`).** YOLO26 gọi 3 tuk-tuk là `car` với điểm 0.68 / 0.71 / 0.81, box rất khớp (IoU 0.86–0.98 với gold). Đây là lỗi **sai lớp**, không phải lỗi điểm thấp. Trong `img_012`, YOLOE cho các tuk-tuk này điểm thấp (0.19 / 0.24 / 0.30). Ở `--conf 0.3`, ít nhất 2/3 box YOLOE bị cắt trước khi gộp, nên box `car` của YOLO26 vẫn được giữ. Vì vậy khi review phải **kiểm từng box `car` / `truck`** xem có phải xe 3 bánh không.
  - **Nhầm giữa các lớp xe 4 bánh.** Trong `img_001`, YOLO26 gọi chiếc xe gold `car` là `truck` (0.88, IoU 0.98). Trong `img_020`, nó gọi chiếc `truck` là `bus` (0.69, IoU 0.96). Box đúng chỗ nhưng sai lớp, nên chỉ sửa được bằng người review đổi lớp, kèm định nghĩa rõ car / truck / bus trong guideline.
  - **Không model nào thấy vật.** Có 1 tuk-tuk trong `img_012` (bị cắt ở mép trên, box gold `[290, 0, 173, 144]`) mà cả YOLO26 lẫn YOLOE đều không ra box nào ở điểm ≥ 0.1. Hạ ngưỡng cũng không cứu được, người review phải vẽ thêm (bước "bỏ sót").
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào):
  - **Điểm cao mà sai:**
    - `img_012`: 3 tuk-tuk bị gọi `car` (0.68–0.81) như ở trên.
    - `img_001`: `truck` 0.88 trên xe gold là `car`.
    - `img_020`: `bus` 0.69 trên xe gold là `truck`.
    - `img_036`: `car` 0.61 / 0.65 trên các vật rất nhỏ ở xa (~16×13 px), trùng vị trí người trong gold. Đây là nhận nhầm vật nhỏ, mờ.
    - `img_036`: `motorbike` 0.75 và 0.66 đúng lớp nhưng box lệch, IoU chỉ ~0.50 (vừa ở mức ngưỡng match), nên là lỗi box lỏng / sai kích thước.
  - **Điểm thấp mà đúng:**
    - `img_012`: YOLOE `three_wheeler` 0.19 và 0.24 đúng vật (IoU 0.79 / 0.74), nhưng sẽ bị cắt ở ngưỡng 0.3.
    - `img_036`: `motorbike` 0.12 (IoU 0.97), `person` 0.18 (IoU 0.92).
    - `img_001`: `motorbike` 0.12 / 0.13.
    - `img_020`: `truck` 0.07 đúng chiếc xe tải mà YOLO26 lại gọi là `bus` 0.69 với điểm cao hơn.

  - **Chọn và chấm trên cùng dữ liệu.** Ngưỡng được quét trên chính 4 ảnh gold (`img_001`, `img_012`, `img_020`, `img_036`), và ở Part 4 precision / recall của máy cũng được đo trên đúng 4 ảnh này. Nghĩa là ngưỡng đã được khớp vào dữ liệu dùng để chấm, nên con số trên gold là **lạc quan**.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 402 | 93.7% | 429 box máy |
| edited | 5 | 1.2% | 429 box máy |
| deleted | 1 | 0.2% | 429 box máy |
| added | 3 | 0.7% | 410 đối tượng sau duyệt |

Nguồn: `qc/review_stats.md`. Có 24 ảnh batch; IoU ≥ 0.9 với box sau duyệt thì tính là accepted. Đã **loại 27 đối tượng quanh vùng bẫy** khỏi bảng này. Bẫy là lỗi do script cố ý cài vào, không phải lỗi của máy, nên nếu tính vào thì profile sẽ đổ lỗi cho máy về những gì máy không làm. Không có trường hợp đổi lớp nào (`class_changed_info = 0`), nên số vật máy sót thật ≈ added = 3.

- **Lớp máy tệ nhất là `three_wheeler`.** Máy ra 19 box cho lớp này (16 accepted + 2 edited + 1 deleted). Có 3/19 = 15.8% box phải sửa hoặc xoá, và mình vẽ thêm 3 box, trên tổng 21 đối tượng `three_wheeler` sau duyệt (14.3%). Các lớp còn lại gần như không phải sửa: person 1/234, motorbike 1/89, car 0/49. Lý do: YOLO26 không có lớp này, chỉ YOLOE nhận ra, mà YOLOE cho điểm thấp trên tuk-tuk (ví dụ `img_012`: 0.19–0.30, xem mục 1), nên bị cắt ở `--conf 0.3`. Lớp `bus` cũng phải sửa 1/3 box, nhưng chỉ có 3 box nên không kết luận được.
- **So với gold** (`qc/review_stats.md` mục 2, mẫu số là **61 đối tượng gold** trên 4 ảnh): precision của máy là **40/57 = 70.2%** (KTC 95%: 57–80%), recall là **40/61 = 65.6%** (KTC 95%: 53–76%). Tức là trên ảnh gold máy sót 21/61 vật và có 17 box thừa. Con số này còn là **lạc quan**, vì ngưỡng được chọn trên chính 4 ảnh này (mục 1).
- **Mâu thuẫn quan trọng:** trên gold máy sót khoảng 1/3 số vật, nhưng trên batch mình chỉ vẽ thêm 3/410 (0.7%) và chỉ xoá 1/429 box. Không có lý do gì máy lại tốt hơn hẳn trên batch, vì batch còn có nhiều ảnh dày xe máy,  accepted 93.7% **không** có nghĩa là máy tốt 93.7%.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- **Bắt bẫy: 3/12 = 25%** (KTC 95%: 9–53%; ngưỡng đạt là 80%). Kể cả cận trên 53% cũng còn xa 80%, nên dù mẫu chỉ có 12 bẫy, kết luận "chưa đạt" vẫn chắc chắn.

  | Loại bẫy | Bắt được | Ảnh lọt |
  |---|---|---|
  | extra (box thừa) | 0/3 | `img_025`, `img_026`, `img_034` |
  | wrong_class (sai lớp) | 1/3 | `img_004`, `img_031` |
  | loose (box lỏng) | 1/3 | `img_013`, `img_033` |
  | missing (bỏ sót) | 1/3 | `img_005`, `img_024` |

- **Tôi bỏ bước nào của checklist:**
  - **Không kiểm từng box có thật sự chứa vật không (bước "thừa").** Lọt cả 3/3 bẫy extra. Mình chỉ nhìn qua thấy "có box ở chỗ có xe" là bấm qua, không xem box đó có trùng hay đang nằm trên nền trống.
  - **Không kiểm lớp và độ ôm của box (bước "sai nhãn" và "lỏng").** Lọt 2/3 bẫy wrong_class và 2/3 bẫy loose. Khi box nằm đúng chỗ, mắt mình mặc định nhãn và kích thước cũng đúng.
- **Accepted cao mà bắt bẫy thấp nói gì:** accepted 93.7% (402/429) đi cùng bắt bẫy 25% (3/12) là dấu hiệu rõ của **thiên kiến tự động hóa**. 

## 4. Máy có đáng dùng không?

- **Thời gian** (từ `time_log.csv`): gán tay 4 ảnh gold mất 15 + 5 + 8 + 10 = 38 phút, tức **9.5 phút/ảnh**.
  - **Thiên lệch khi so sánh:**
    - Mình gán gold trước nên khi duyệt đã quen kiểu cảnh và quen guideline, làm thời gian duyệt có vẻ nhanh hơn thực tế.
    - Ngược lại, gold chỉ có 4 ảnh. 6 ảnh batch dày xe máy (005, 018, 021, 024, 028, 029) nặng hơn ảnh gold trung bình.
- **Kết luận: có, nhưng chỉ dùng máy để đề xuất box, không dùng để chấp nhận thẳng.** Cụ thể:
  - Với **person / motorbike / car**, máy đáng dùng: trên gold, phần lớn box đúng (precision 70.2%, 40/57), và trên batch rất ít box phải sửa vị trí (person 1/234, motorbike 1/89, car 0/49). Vẽ lại từ đầu sẽ chậm hơn.
  - Với **`three_wheeler`**, máy kém: 3/19 box phải sửa hoặc xoá, mình vẽ thêm 3/21, và còn bị gọi nhầm thành `car`. Lớp này phải kiểm kỹ từng xe.
  - Nhưng recall trên gold chỉ 65.6% (40/61), tức máy sót khoảng 1/3 số vật. 
- **Ba quy tắc giảm thiên kiến của mình** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. **Đếm trước, xem nhãn máy sau.** Với mỗi ảnh, ẩn nhãn máy, đếm số vật theo lớp và ghi vào cột `note`. Sau đó mới bật nhãn máy, và nếu số box máy khác số mình đếm thì phải tìm ra chỗ lệch.
     - Kiểm tra bằng: 24/24 ảnh có số đếm trong `note`.
     - Mục tiêu: bắt được ≥ 2/3 bẫy missing khi duyệt lại.
  2. **Mỗi box phải trả lời được 3 câu: có vật thật không, đúng lớp không, có ôm sát không.** Phóng to mọi box nhỏ hơn 30 px và mọi box `car` / `truck` / `bus` để xem có phải xe 3 bánh hoặc nhầm loại xe không.
     - Kiểm tra bằng: bắt bẫy extra từ 0/3 và wrong_class từ 1/3 lên ≥ 2/3.
     - Bằng chứng phụ: số box `deleted` và `edited` không còn gần 0 (hiện là 1 và 5 trên 429).
  3. **Dùng tỉ lệ accepted làm báo động, không phải thành tích.** Nếu một lô có accepted > 90% mà added < 2% trong khi recall gold < 80%, thì phải duyệt lại từ đầu 5 ảnh chọn ngẫu nhiên (seed 42) với nhãn máy ẩn, so số lỗi tìm thêm được. Đồng thời mỗi ảnh tối thiểu 2 phút và nghỉ sau mỗi 10 ảnh, ghi trong `time_log.csv`.
     - Kiểm tra bằng: có 5 ảnh duyệt lại kèm số lỗi tìm thêm, và trong `time_log.csv` không ảnh nào dưới 2 phút.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| <<img_0xx>> | <<>> | <<>> | <<>> | <<>> | <<>> |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu? <<>>
- *(Tuỳ chọn)* Interactor SAM trong CVAT: làm được / không (lỗi gì, đã thử gì); khác gì offline: <<>>

## 6. Calibration (Part 6)

- Số quyết định: <<>> (`calibration_decisions.csv`); quan trọng nhất (dẫn `CAL-xx`): <<>>
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: <<>>
- Nguyên nhân gốc: <<guideline mơ hồ / chưa nắm quy tắc / máy>>

## 7. QC và quyết định (Part 6)

- Mẫu: <<n>> ảnh (<<m>> ảnh `can_review` + <<k>> ngẫu nhiên, seed 42); người kiểm: bạn (**hạn chế**: bạn vừa là người duyệt).
- Lỗi theo **đối tượng**: <<.../... = ...%>> · theo **ảnh**: <<.../... = ...%>> (ngưỡng Release ≤ 2%)
- Top lỗi (trong: bỏ sót, thừa, sai lớp, sai vị trí/kích thước, sai thuộc tính, sai mask) + một ảnh ví dụ mỗi loại: <<>>
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): <<>>
- Script kiểm được / không kiểm được (ví dụ ôm sát, cùng bỏ sót với máy, theo checklist thật): <<>>

**Quyết định: <<Release / Hold / Rework>>**, ngày <<YYYY-MM-DD>>. Lý do (đi theo Hold → Rework → Release): <<>>
Nếu Rework / Hold, việc cần làm tiếp: <<>>

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào; "không dùng" nếu không): <<>>
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)? <<không / có, như sau: ...>>
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: <<>>
- Dùng cache `data/cache/` thay vì tự chạy? <<không / có, vì ...>>
- Lỗi / bug script đã gặp (nếu có): <<>>
- Một điều bạn làm khác lần sau: <<>>
