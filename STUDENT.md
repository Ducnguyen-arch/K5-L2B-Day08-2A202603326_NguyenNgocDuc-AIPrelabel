# Thông tin học viên

- Họ và tên: Nguyễn Ngọc Đức
- MSHV: 2A202603326
- Lớp (L2 / L3 / khác): L2B
- Tài khoản GitHub: Ducnguyen-arch
- Repo bài nộp (URL): https://github.com/Ducnguyen-arch/K5-L2B-Day08-2A202603326_NguyenNgocDuc-AIPrelabel
- Commit chốt bài (hash): <<>>
- Máy: Windows 10 Pro, không có GPU (chạy CPU)
- CVAT (Docker): v2.81.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 402/429 = 93.7% / 5/429 = 1.2% / 1/429 = 0.2% (mẫu số: 429 box máy); added: 3/410 = 0.7% (mẫu số: 410 đối tượng sau duyệt)
- So với gold: precision / recall của **máy** 40/57 = 70.2% / 40/61 = 65.6% (61 đối tượng gold, 4 ảnh; lạc quan vì ngưỡng chọn trên chính gold)
- Bắt bẫy: 3/12 = 25%
- Thời gian: gán tay 38/4 = 9.5 phút/ảnh; duyệt có pre-label <<chưa ghi batch_review trong time_log.csv>> phút/ảnh
- Lỗi QC: 100% theo ảnh (5/5), 10.8% theo đối tượng (10/93)
- **Quyết định:** Rework — lý do một dòng: lỗi ảnh 5/5 = 100% > ngưỡng 2%, bỏ sót lặp lại (5/10 lỗi) và bắt bẫy chỉ 3/12, nên lô duyệt chưa đáng tin.
