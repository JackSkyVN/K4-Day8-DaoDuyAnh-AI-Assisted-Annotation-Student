# Vì sao chọn lô này?

## Top 5 frame ưu tiên nếu chỉ có ngân sách 5 ảnh

Dựa trên 50 dòng đứng đầu của `outputs/selection_round1.csv` (chiến lược `uncertainty`, công thức
`score = W_U·U + W_A·A + W_D·D`):

| Ưu tiên | Frame | Score | U | n_boxes | n_ambiguous | Lý do chọn |
|---------|-------|-------|---|---------|------------|------------|
| 1 | frame_0182.jpg | 0.9591 | 0.9182 | 28 | 18 | Điểm bất định cao nhất, 18/28 box mơ hồ — AI chắc chắn về rất ít xe |
| 2 | frame_0099.jpg | 0.9063 | 0.9460 | 29 | 14 | U cao nhất trong lô; đã rà và thêm 164 box chứng minh AI bỏ sót nhiều |
| 3 | frame_0331.jpg | 0.9154 | 0.8308 | 47 | 18 | Nhiều xe nhất trong top 5 (47 box), cung cấp đa dạng mẫu học |
| 4 | frame_0326.jpg | 0.9155 | 0.9310 | 39 | 15 | Điểm đa dạng thời gian D=1.0, không trùng cảnh với ảnh đã chọn |
| 5 | frame_0312.jpg | 0.9100 | 0.8199 | 37 | 18 | A=1.0 — thời điểm 124.8s cách xa các frame khác, tăng đa dạng |

**Quyết định loại vì ảnh gần trùng:** `frame_0372.jpg` (rank 6, score=0.9101) có điểm cao nhưng
không được chọn vì `t_sec=148.8s` quá gần `frame_0369.jpg` (t=147.6s, khoảng cách chỉ 1.2 giây).
Bộ chọn mẫu đặt `A=0.833` phản ánh mức phạt cảnh gần trùng — hai ảnh cách nhau < 3 giây trên
video 25fps sẽ có nội dung gần như giống hệt nhau, gán nhãn thêm không mang thêm thông tin mới
cho mô hình.

---

## Ba frame thuộc lô 12 ảnh model đã chọn

**1. frame_0182.jpg** (rank 1, score=0.9591, t=72.8s)
- U=0.9182: AI dự đoán nhiều box với confidence thấp → đây là vùng mô hình không chắc nhất.
- 18 box mơ hồ trên 28 box tổng (64%) cho thấy cảnh phức tạp, nhiều xe chen nhau hoặc bị khuất.
- Rà nhãn ở frame này có giá trị học tập cao nhất theo chiến lược uncertainty sampling.

**2. frame_0099.jpg** (rank 8, score=0.9063, t=39.6s)
- U=0.9460 — độ bất định cao, được xác nhận khi rà thực tế: AI bỏ sót hàng loạt xe ở làn giữa
  (thêm 164 box so với 169 box AI đề xuất ban đầu, xem `outputs/round1_diff.md`).
- Cảnh khác thời điểm so với các frame khác (t=39.6s, trước frame_0312 ~85 giây), đảm bảo đa dạng.

**3. frame_0331.jpg** (rank 5, score=0.9154, t=132.4s)
- Nhiều xe nhất (47 box tổng, 18 mơ hồ) — frame này đại diện cho giờ cao điểm đông xe.
- A=1.0, D=1.0: không có frame nào trong lô được chọn gần về thời gian, tăng phủ đa dạng cảnh.

---

## Một frame điểm cao nhưng không chọn

**frame_0372.jpg** (rank 6, score=0.9101, t=148.8s) — điểm tốt nhưng bị loại:
- Cách `frame_0369.jpg` (rank 2, t=147.6s) chỉ **1.2 giây** (~30 khung hình ở 25fps).
- Nội dung hai frame gần như giống hệt nhau — gán nhãn cả hai chỉ tốn công gấp đôi mà không
  đóng góp thêm thông tin học tập mới (A=0.8333 thay vì 1.0).
- Đây là ví dụ điển hình cần cân bằng giữa **điểm bất định cao** và **chi phí gán nhãn / redundancy**.

---

## Điều phép chọn này chưa chứng minh

Chiến lược uncertainty sampling dựa trên độ bất định của mô hình **không bảo đảm** rằng những
ảnh được chọn sẽ cải thiện mô hình sau fine-tune. Cụ thể:

- AP50 giảm từ 0.771 xuống 0.570 sau vòng 1 — fine-tune trên 12 ảnh có thể gây **catastrophic
  forgetting**, mô hình "quên" các mẫu học từ COCO.
- Điểm bất định cao có thể phản ánh cảnh khó (đêm, xe xa) chứ không hẳn là thông tin mới hữu ích.
- Tập test chỉ 20 ảnh và nhãn tham chiếu do mô hình tạo (chưa được người rà), nên AP50 đo mức
  khớp với tập tham chiếu này, không phải chất lượng phát hiện thực địa.
