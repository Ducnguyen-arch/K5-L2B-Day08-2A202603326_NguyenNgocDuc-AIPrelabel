# Báo cáo QC tự động (nháp)

> **Gợi ý** do script tính; bạn xác nhận. Script chỉ đếm theo phiếu kiểm bằng mắt người.

## Mẫu
- 5 ảnh (0 ca khó + 5 ngẫu nhiên), 93 đối tượng.
- taxonomy v1.0

## Số đo
| Chỉ số | Giá trị | Mẫu số |
|---|---|---|
| Lỗi **theo đối tượng** | 10/93 = 10.8% | 93 đối tượng trong mẫu |
| **Ảnh có lỗi** (dùng cho ngưỡng) | 5/5 = 100.0% | 5 ảnh trong mẫu |
| Máy: accepted / edited / deleted | 402/429 = 93.7% / 5/429 = 1.2% / 1/429 = 0.2% | 429 box máy |
| Máy: added | 3/410 = 0.7% | 410 đối tượng sau duyệt |
| Bắt lỗi bẫy | 3/12 = 25.0% | 12 bẫy |

## Top 3 loại lỗi
| Loại lỗi | Số lỗi | Ảnh ví dụ |
|---|---|---|
| bo_sot | 5 | img_003.jpg, img_006.jpg, img_035.jpg |
| thua | 2 | img_003.jpg, img_014.jpg |
| sai_vi_tri_kich_thuoc | 2 | img_031.jpg, img_035.jpg |

**Lỗi hệ thống**: 'bo_sot' lặp lại -> sửa quy tắc / recipe, không chỉ sửa từng ảnh.

## Giới hạn thống kê
- 5 ảnh lỗi trong 5 ảnh mẫu: cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là **100%**. Ngưỡng 2% không thể được **chứng minh** bằng mẫu nhỏ như vậy: hãy nói rõ giới hạn này trong báo cáo.

## Gợi ý quyết định
**REWORK**

- ảnh có lỗi 100.0% > 2%
- lỗi hệ thống: 'bo_sot' chiếm 5/10 lỗi
- bắt bẫy 25% < 80% -> lô duyệt chưa đáng tin

Cây quyết định: Hold (guideline chưa rõ / dữ liệu hỏng) -> Rework (> ngưỡng, lỗi lặp, bắt bẫy / nhất quán thấp) -> Release.
