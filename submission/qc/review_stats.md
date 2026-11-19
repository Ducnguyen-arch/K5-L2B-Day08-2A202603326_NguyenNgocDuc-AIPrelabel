# Số đo review (det)

- Ảnh tính: 24 (batch,gold); box máy: 429; đối tượng sau duyệt: 410; ngưỡng accepted: IoU >= 0.9
- Đã **loại 27 đối tượng nằm quanh các bẫy** khỏi bảng này: lỗi nhân tạo không phản ánh chất lượng máy.

## 1. Bốn trạng thái (AI error profile)
| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted (giữ nguyên) | 402 | 402/429 = 93.7% | 429 box máy |
| edited (sửa) | 5 | 5/429 = 1.2% | 429 box máy |
| deleted (xoá) | 1 | 1/429 = 0.2% | 429 box máy |
| added (vẽ thêm) | 3 | 3/410 = 0.7% | 410 đối tượng sau duyệt |

Trong số deleted + added, **0 trường hợp là đổi lớp** (máy gán sai lớp, box đúng chỗ): đó không phải máy bỏ sót. Số vật máy **sót thật** ≈ added − đổi lớp = 3.

### Theo lớp
| Lớp | accepted | edited | deleted | added |
|---|---|---|---|---|
| person | 233 | 1 | 0 | 0 |
| bicycle | 7 | 0 | 0 | 0 |
| motorbike | 88 | 1 | 0 | 0 |
| car | 49 | 0 | 0 | 0 |
| bus | 2 | 1 | 0 | 0 |
| truck | 7 | 0 | 0 | 0 |
| three_wheeler | 16 | 2 | 1 | 3 |

## 2. So với gold (nhãn gán tay từ đầu)
Ảnh gold: 4; đối tượng gold: 61 (mẫu số của recall)
| | TP | FP | FN | Precision | Recall |
|---|---|---|---|---|---|
| Máy (pre-label) | 40 | 17 | 21 | 70.2% (KTC 95%: 57–80%) | 65.6% (KTC 95%: 53–76%) |

*Lạc quan: ngưỡng được chọn trên chính các ảnh gold này (in-sample), và chỉ có 4 ảnh. Đọc khoảng tin cậy, đừng đọc con số đơn.*


(Ảnh gold không nằm trong nhãn sau duyệt: không tính residual / new error. Đó là chủ ý: duyệt lại ảnh bạn vừa tự gán tay sẽ đo trí nhớ, không đo thiên kiến.)

## 3. Ảnh bẫy
- Bắt được 3/12 = 25.0% (KTC 95%: 9–53%) (ngưỡng đạt: 80%). Chỉ 12 bẫy nên khoảng tin cậy rất rộng: đọc cùng danh sách bẫy lọt, đừng kết luận chỉ từ %.
  - extra: 0/3
  - wrong_class: 1/3
  - loose: 1/3
  - missing: 1/3
  - lọt: img_025.jpg (extra), img_026.jpg (extra), img_004.jpg (wrong_class), img_013.jpg (loose), img_034.jpg (extra), img_024.jpg (missing), img_005.jpg (missing), img_031.jpg (wrong_class), img_033.jpg (loose)

## Gợi ý đọc số
- `deleted` cao → ngưỡng thấp hoặc ánh xạ nhãn sai; `added` cao → máy bỏ sót (xem lớp nào);
- `accepted` thấp (< 50%) → sửa mỗi box lâu hơn vẽ mới: đổi model/ngưỡng hoặc vẽ tay;
- bắt bẫy < 80% → duyệt lại lô, siết checklist (quét ảnh TRƯỚC khi nhìn nhãn máy).
