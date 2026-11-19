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

Nguồn: `qc/review_stats.md`. Có 24 ảnh batch; IoU ≥ 0.9 với box sau duyệt thì tính là accepted. Đã **loại 27 đối tượng quanh vùng bẫy** khỏi bảng này. Bẫy là lỗi do script cố ý cài vào, không phải lỗi của máy

- **Lớp máy tệ nhất là `three_wheeler`.** Máy ra 19 box cho lớp này (16 accepted + 2 edited + 1 deleted). Có 3/19 = 15.8% box phải sửa hoặc xoá, và mình vẽ thêm 3 box, trên tổng 21 đối tượng `three_wheeler` sau duyệt. Các lớp còn lại gần như không phải sửa: person 1/234, motorbike 1/89, car 0/49. Lý do: YOLO26 không có lớp này, chỉ YOLOE nhận ra, mà YOLOE cho điểm thấp trên tuk-tuk (ví dụ `img_012`: 0.19–0.30, xem mục 1), nên bị cắt ở `--conf 0.3`. Lớp `bus` cũng phải sửa 1/3 box
- **So với gold** (`qc/review_stats.md` mục 2, mẫu số là **61 đối tượng gold** trên 4 ảnh): precision của máy là **40/57 = 70.2%** (KTC 95%: 57–80%), recall là **40/61 = 65.6%** (KTC 95%: 53–76%). Tức là trên ảnh gold máy sót 21/61 vật và có 17 box thừa. Con số này còn là **lạc quan**, vì ngưỡng được chọn trên chính 4 ảnh này (mục 1).

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
| `img_003` (xe ba gác bán dưa hấu, có mui) | SAM 2.1 (`sam2.1_b.pt`) | 3 (2 dương: mui, thân thùng; 1 âm: nền bên phải) | **Hàng hoá: có.** Dưa hấu và rổ trong thùng nằm trong mask. **Người: một phần.** Mặt và vai em bé ngồi trên thùng bị tô vào mask; người lái phía sau gần như không bị tô | Mask dính vào người (em bé). Mui, thùng và 2 bánh lớn được tách khỏi nền khá sạch, bóng đổ trên đường không bị tô | `tuktuk_img003.jpg` |
| `img_004` (tuk-tuk hồng đỗ trong hàng tuk-tuk) | SAM 2.1 (`sam2.1_b.pt`) | 3 (2 dương, 1 âm) | **Người lái: không.** Người đứng sau xe không bị tô | 1 click dương rơi vào **tán cây / biển phía sau**, không phải xe. Mask vẫn ra đúng tuk-tuk hồng nhờ click ở bánh trước, còn click âm (trên kính tuk-tuk xanh) giúp không dính sang xe bên cạnh. Viền mui và hông xe chưa sát tuyệt đối | `tuktuk_img004.jpg` |
| `img_013` (tuk-tuk hồng chở khách, thành xe hở) | SAM 2.1 (`sam2.1_b.pt`) | 3 (2 dương: mui, thân xe; 1 âm: ngực người lái) | **Người lái: một phần.** Click âm chỉ bỏ được phần ngực, còn tay, chân, dép người lái vẫn bị tô | **Lỗ thủng lớn không bị bỏ.** Qua khung hở hai bên, SAM tô luôn cả nền phía sau: người mặc áo hồng, bàn, ghế, chai nước. Mask người lái lốm đốm | `tuktuk_img013.jpg` |

- **Cách đặt click:** cả 3 ảnh dùng **cùng một bộ toạ độ** lấy từ lệnh mẫu trong GUIDE.md: dương `(600,110)`, `(400,480)`, âm `(720,300)`. Với `img_003` và `img_013` các điểm này tình cờ rơi đúng mui và thân xe. Với `img_004` một click dương rơi ra nền. Kết quả vẫn dùng được, nhưng là **may mắn chứ không phải do chọn click**. Lần sau phải chọn click theo từng vật: dương trên mui, thân và bánh; âm trên người lái và nền nhìn qua khung hở.
- **Đối chiếu guideline mục 4.2:**
  - **Có ích:**
    - **4.2.3 (bóng đổ không thuộc vật):** dễ kiểm. `img_003` bóng trên đường không bị tô, đạt.
    - **4.2.4 (hai vật chạm nhau, vùng chạm thuộc vật phía trước):** giúp quyết định đặt click âm lên tuk-tuk xanh ở `img_004`, để mask không tràn sang xe bên cạnh.
  - **Bị vi phạm:**
    - **4.2.2 (lỗ lớn phải bỏ khỏi polygon):** `img_013` vi phạm rõ. Khoảng hở hai bên thành xe lớn hơn nhiều so với 5% diện tích xe, nhưng SAM tô kín cả nền phía sau. Muốn sửa phải thêm click âm vào từng khoảng hở, hoặc sửa polygon bằng tay.
  - **Còn thiếu:**
    - **Người lái / hành khách có thuộc mask xe không.** Mục 4.1.4 nói với **box** thì người và xe là hai box riêng, nhưng 4.2 không nói gì cho **mask**. Theo tinh thần 4.1.4 thì mask `three_wheeler` **không** được gồm người, nên `img_003` (em bé) và `img_013` (người lái) đều sai. Cần ghi rõ trong v1.1.
    - **Hàng hoá trong thùng (`img_003`).** Câu hỏi mở số 3 của v1.0 chưa chốt cho mask. Mình đề xuất: hàng **nằm trong** thùng thì tính vào mask xe, hàng **nhô ra** quá ~20% chiều dài xe thì không (giống 4.1.5 cho box). Ghi vào v1.1.


## 6. Calibration (Part 6)

- **Số quyết định: 11** (`qc/calibration_decisions.csv`): 6 `cau_hoi_mo`, 3 `loi_may`, 1 `bay_lot`, 1 `can_review`. **9/11** ca được đưa vào v1.1. 2/11 ca (CAL-09, CAL-10) cố ý **không** đổi guideline, vì guideline đã rõ, lỗi là do máy hoặc do mình.
- **Quan trọng nhất là CAL-07** (`img_012`): tuk-tuk có mui bị YOLO26 gọi là `car` với điểm 0.68–0.81. `three_wheeler` là lớp tệ nhất trên batch (3/19 box máy phải sửa hoặc xoá, vẽ thêm 3/21), và vẫn còn lỗi ở QC (box xích lô lệch ở `img_035`, box thừa trên gánh hàng ở `img_014`). Đây là lớp duy nhất model COCO không biết, nên quy tắc và bước kiểm cho lớp này có ảnh hưởng lớn nhất.
- **Quy tắc mới đã vào `guideline_v1.1_calibrated.md`:**
  - Chốt câu hỏi mở 2 (`img_018`): xe cao ≥ 20 px gán từng chiếc; chỉ cụm ≥ 5 xe cao < 20 px mới dùng **một** `ignore_region`.
  - Chốt câu hỏi mở 3 (`img_003`): hàng nằm trong thùng thuộc box và mask xe, hàng nhô ra quá 20% chiều dài xe thì không.
  - Chốt câu hỏi mở 4 (`img_012`): gán `person` khi thấy rõ đầu hoặc thân trên.
  - Chốt câu hỏi mở 5 (`img_020`): kính thuộc xe; người chỉ thấy qua kính thì không gán.
  - Chốt câu hỏi mở 6 (`img_026`): ảnh đêm không gộp box tự động, mỗi xe một box.
  - Chốt câu hỏi mở 7 (`img_001`): không đặt mức `conf` để tự chấp nhận box máy.
  - Thêm: xe ba bánh có mui luôn là `three_wheeler` (CAL-07).
  - Thêm: mask xe không gồm người và bỏ khoảng hở lớn (CAL-08).
  - Làm rõ: `can_review` chỉ dùng khi không chắc quy tắc (CAL-11).
  - Câu hỏi mở 1 (xe đạp điện) **chưa chốt**, vì không có ảnh ví dụ.
- **Nguyên nhân gốc:**
  - **Guideline mơ hồ:** 6/11 ca, tức các câu hỏi mở. Hai trong số đó mình đã gặp ngay khi gán gold (ghi chú `time_log` của `img_012` và `img_020`).
  - **Máy:** 3/11 ca (tuk-tuk → `car`, nhầm car/truck/bus, mask dính người).
  - **Mình chưa nắm hoặc chưa làm đúng quy tắc:** 2/11 ca, gồm `can_review` dùng sai ở `img_036` và bẫy người bị gán `bicycle` lọt ở `img_004` / `img_031`.
  - Ngoài ra, nguyên nhân lớn nhất của chất lượng lô thấp **không phải guideline** mà là **cách duyệt**: bắt bẫy 3/12, QC có 5/10 lỗi là bỏ sót (mục 3 và mục 7).

## 7. QC và quyết định (Part 6)

- **Mẫu: 5 ảnh** (0 ảnh `can_review` + 5 ngẫu nhiên, seed 42; `qc_sample.py --n 5`): `img_003`, `img_006`, `img_014`, `img_031`, `img_035`, tổng **93 đối tượng**. Batch không có box `can_review` nào nên cả mẫu là ngẫu nhiên. Người kiểm là mình (**hạn chế**: mình vừa là người duyệt lô, nên dễ bỏ sót đúng những lỗi đã bỏ sót lúc duyệt).
- **Lỗi theo đối tượng: 10/93 = 10.8%** · **theo ảnh: 5/5 = 100%** (ngưỡng Release ≤ 2%). Không ảnh nào trong mẫu sạch.
- **Top lỗi** (`qc/qc_review.csv`):

  | Loại lỗi | Số | Tỉ lệ trong tổng lỗi | Ảnh ví dụ |
  |---|---|---|---|
  | bỏ sót (`bo_sot`) | 5 | 5/10 | `img_006`: 2 xe máy đỗ phía sau #67, cao > 20 px, không có box |
  | thừa (`thua`) | 2 | 2/10 | `img_014` #127: box `three_wheeler` nằm trên gánh hàng, không phải xe |
  | sai vị trí / kích thước | 2 | 2/10 | `img_031` #383: box xe đạp thiếu bánh trước; `img_035` #444: box xích lô chỉ bao phần mui |
  | sai lớp (`sai_lop`) | 1 | 1/10 | `img_031` #382: người dắt xe bị gán `bicycle` (chính là bẫy wrong_class đã lọt) |
  | sai thuộc tính / sai mask | 0 | 0/10 | — (QC chỉ kiểm box) |

  Bỏ sót chiếm 5/10 lỗi, và `qc_report_auto.md` đánh dấu đây là **lỗi hệ thống**. 
- **Giới hạn mẫu nhỏ:** 5/5 ảnh lỗi, cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là **100%**. Chỉ 5 ảnh thì **không thể chứng minh** lô đạt ngưỡng 2%
- **Script kiểm được / không kiểm được:**
  - **Kiểm được:** đếm và tính tỉ lệ từ phiếu `qc_review.csv`, tính cận trên thống kê, nhận ra loại lỗi lặp lại, và áp ngưỡng của `taxonomy.json` vào cây quyết định.
  - **Không kiểm được:**
    - Box có ôm sát thật không, và vật bị **cả máy lẫn mình cùng bỏ sót**: script chỉ đếm những gì người kiểm ghi vào phiếu.
    - Mình có thật sự làm theo checklist khi duyệt không.
    - Lỗi trong 19/24 ảnh **không** nằm trong mẫu. Ví dụ, `img_018` vẫn còn 16 `ignore_region` nhỏ trái với CAL-04 mà không bị đếm.

**Quyết định: Rework**, ngày 2026-11-19. Lý do: không phải Hold (guideline đã chốt 6/7 câu hỏi mở trong v1.1, dữ liệu không hỏng), nhưng lỗi ảnh 5/5 = 100% vượt xa ngưỡng 2%, bỏ sót lặp lại (5/10 lỗi) và bắt bẫy chỉ 3/12, nên phải Rework, chưa được Release.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài:** có, dùng **Claude Code** (trợ lý AI trong terminal) **sau khi** đã gán gold và review batch xong. Việc AI làm:
  - Đọc repo và chạy `check_submission.py` để liệt kê phần còn thiếu.
  - Viết script phụ đối chiếu dự đoán YOLO26 / YOLOE với gold (số liệu điểm cao mà sai / điểm thấp mà đúng ở mục 1).
  - Soạn nháp REPORT mục 1–7, `calibration_decisions.csv` và `guideline_v1.1_calibrated.md`.

  Cách  kiểm tra:
  - Mở lại từng ảnh trong `work/qc/review/` và `masks/` để xác nhận lỗi.
  - Chạy lại `qc_report.py` và `check_submission.py`.

  AI **không** gán nhãn gold và không review batch thay mình.
- **Gold có bị nhiễm không:** không. `gold_manual.json` được lưu lúc 2026-11-17 18:00, **trước** lần chạy model đầu tiên (`t_y26`, 2026-11-18 01:04)
- **Máy:** Windows 10 Pro, chạy **CPU** (không có GPU CUDA). CVAT v2.81.0 chạy bằng Docker. Không có GPU nên chạy chậm: Grounding DINO mất ~116 s/ảnh (`r06`), SAM 2.1, không dùng SAM 3.
- **Dùng cache `data/cache/` thay vì tự chạy?** Không. Tự chạy `y26_all` (`r08`) và `yoloe_all` (`r09`) trên CPU, xem `work/runs.csv`.
- **Lỗi / quy trình đã gặp:**

  - `time_log.csv` bị lưu bằng mã hoá Windows (không phải UTF-8) nên phần ghi chú tiếng Việt bị vỡ chữ; nội dung ghi chú gốc được dẫn lại ở mục 6.
- **Một điều mình sẽ làm khác lần sau:** **quét ảnh và đếm vật trước khi bật nhãn máy**, cho từng ảnh. Accepted 402/429 đi cùng bắt bẫy 3/12 và 5/10 lỗi QC là bỏ sót.
