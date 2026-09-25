# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Đào Duy Anh

Công cụ gán nhãn đã dùng: CVAT Docker local (v2.76.0) tại `http://localhost:8080`

---

## 1. Dữ liệu và cách chia tập

Pool (268 ảnh) và test set (20 ảnh) được **chia theo trục thời gian với vùng đệm ở giữa** thay
vì chia ngẫu nhiên, vì hai lý do kỹ thuật chính:

**Tránh data leakage qua thời gian (temporal leakage):** Video đường cao tốc có tính liên tục —
hai frame cách nhau vài giây thường có cùng xe, cùng góc chụp, cùng điều kiện ánh sáng. Nếu
chia ngẫu nhiên, rất dễ xảy ra trường hợp một frame từ giây thứ 45.0 vào pool và frame 45.2 vào
test — mô hình fine-tune trên pool sẽ "nhìn thấy" gần như y hệt cảnh trong test, làm AP50 bị
thổi phồng lên cao giả tạo.

**Vùng đệm giữa pool và test:** Ngay cả khi chia theo thời gian, hai vùng vẫn cần khoảng trống
(buffer) để loại bỏ những cặp frame rất gần nhau nhưng rơi vào hai phía của đường chia. Xem
`data/DATA.md` để biết cụ thể khoảng cách thời gian tối thiểu.

Nếu chia ngẫu nhiên, số đo AP50 trên test set sẽ bị lệch **cao hơn** giá trị thực, vì mô hình
được evaluate trên những cảnh nó đã "gần như thấy" trong quá trình học, không phản ánh khả năng
tổng quát hóa thực sự của mô hình.

---

## 2. Mô hình khởi đầu lạnh (cold start)

Từ `outputs/metrics_round0.json` và `reports/rounds_table.md`:

| Vòng | Model | Ảnh train | AP50 | P@0.25 | R@0.25 | F1 | R nhỏ | R vừa | R lớn |
|------|-------|-----------|------|--------|--------|----|-------|-------|-------|
| 0 | yolov8n cold start (COCO) | 0 | **0.771** | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

**Phân tích từ `outputs/compare_round0.jpg`:**

- **Xe nhỏ / xa (R_small = 0.182):** Đây là điểm yếu lớn nhất của cold start. Model bỏ sót ~82%
  xe nhỏ — thường là xe đang ở xa, chỉ còn 2 chấm đèn, hoặc xe bị khuất sau xe khác. Về đêm,
  thiếu contrast làm model không tự tin đủ để vượt threshold.

- **Xe vừa và lớn (R_medium=0.547, R_large=0.561):** Model hoạt động khá tốt với xe nhìn rõ thân,
  nhưng vẫn bỏ sót gần một nửa. Một số nguyên nhân: xe nhoè do chuyển động (motion blur), xe
  bị cắt mép ảnh, hoặc hai xe đứng sát nhau bị model gộp thành một box.

- **Precision = 0.925:** Rất cao — hầu hết box model vẽ đều là xe thực. Tuy nhiên, con số này
  cũng phụ thuộc vào chất lượng nhãn tham chiếu. Nhãn test do mô hình khác tạo và **chưa được
  người rà từng box**, vì vậy một số false positive của mô hình thực ra có thể là xe bị bỏ sót
  trong nhãn tham chiếu — cần người rà lại trước khi kết luận model sai.

---

## 3. Chiến lược chọn mẫu

**Công thức:** `score = W_U·U + W_A·A + W_D·D`

- **U (Uncertainty):** Đo độ bất định trung bình của các dự đoán trong frame. Model có confidence
  thấp với xe tối, xe xa, xe bị che khuất → U cao đồng nghĩa frame đó khó với model hiện tại.
- **A (Anti-redundancy):** Phạt các frame có cảnh quá giống nhau (gần nhau về thời gian). Công
  thức tính A dựa trên khoảng cách tối thiểu `MIN_GAP_S` giữa frame được chọn và các frame đã
  trong lô — giảm gán nhãn lặp không cần thiết.
- **D (Density/Diversity):** Ưu tiên frame có nhiều xe đa dạng, đảm bảo lô không chỉ toàn cảnh
  thưa xe hay toàn cảnh một loại xe.

**Ba frame từ `reports/SELECTION.md`:**

1. **frame_0182.jpg** (score=0.9591, U=0.9182): Độ bất định cao nhất — 18/28 box là mơ hồ. Model
   không chắc với đa số xe trong frame này, đây là nơi nhãn người rà tạo ra nhiều thông tin học
   tập nhất.

2. **frame_0099.jpg** (score=0.9063, U=0.9460): U cao, xác nhận thực tế — khi rà thực tế phát
   hiện AI bỏ sót nhiều xe (thêm 164 box, xem `outputs/round1_diff.md`).

3. **frame_0331.jpg** (score=0.9154, 47 xe): Frame đông xe nhất, cung cấp số lượng mẫu học lớn
   trong một lần gán nhãn.

**Frame loại vì gần trùng:** `frame_0372.jpg` (rank 6) bị loại vì cách `frame_0369.jpg` chỉ
1.2 giây (A=0.833 < 1.0). Điểm bất định chưa chứng minh frame đó cải thiện model — nếu cảnh
gần trùng với frame khác đã được chọn, model học cùng một pattern, tốn công gán nhãn gấp đôi
mà không thêm thông tin mới.

---

## 4. Các vòng học chủ động (active learning)

Từ `reports/rounds_table.md`:

| Vòng | Model | Ảnh train | AP50 | ΔAP50 | P@0.25 | R@0.25 | F1 | R nhỏ | R vừa | R lớn |
|------|-------|-----------|------|-------|--------|--------|----|-------|-------|-------|
| 0 | yolov8n cold start | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune | 12 | **0.570** | **-0.201** | 1.000 | 0.171 | 0.292 | 0.000 | 0.132 | 0.732 |

**Mức độ sửa nhãn vòng 1** (từ `outputs/round1_diff.md`):
- 12 ảnh, 321 box cuối
- Model đề xuất: 169 box → Accepted: 144 | Edited: 13 | Deleted: 12 | **Added: 164**
- Tỷ lệ box AI bị giữ nguyên: 144/169 = 85% — phần lớn box AI đúng vị trí
- Added nhiều hơn cả số box AI đề xuất ban đầu: 164 box được thêm, chứng tỏ AI bỏ sót rất nhiều
  xe trong cảnh đêm (đặc biệt xe tối, xe xa, xe bị che)

**AP50 giảm từ 0.771 → 0.570 (ΔAP50 = -0.201):**

Đây là kết quả bất ngờ nhưng có giải thích rõ ràng. Nhìn vào chi tiết:
- **P tăng lên 1.000:** Mọi box model dự đoán đều là xe thực — model học được "chắc chắn hơn
  trước khi dự đoán".
- **R giảm xuống 0.171:** Model chỉ phát hiện 17% số xe trong tập test — bỏ sót ~83%.
- **R_large tăng từ 0.561 → 0.732:** Model học tốt hơn với xe lớn/gần (đã được rà kỹ trong lô).
- **R_small giảm từ 0.182 → 0.000:** Model hoàn toàn mất khả năng phát hiện xe nhỏ/xa.

**Nguyên nhân:** Fine-tune 50 epochs trên chỉ 12 ảnh gây **catastrophic forgetting** — model
"quên" các mẫu đa dạng từ COCO (xe nhỏ, xe xa, nhiều điều kiện ánh sáng), chỉ nhớ đặc điểm
của 12 ảnh trong lô. Kết quả là P=1.0 nhưng R rất thấp — model chỉ dự đoán khi chắc 100%.

**Ca thay đổi sau fine-tune** (quan sát từ `outputs/compare_round1.jpg`):
- Xe lớn ở tiền cảnh (R_large 0.561→0.732): Cải thiện rõ — box bao chính xác hơn thân xe lớn.
- Xe xa, nhỏ: Mất hoàn toàn (R_small 0.182→0.000) — đây là điểm thụt lùi nghiêm trọng cần
  xử lý ở vòng sau.

**So sánh với BLIND_SCAN.md và REVIEW_LOG.csv:**
- Quan sát độc lập trước khi xem pre-label (frame_0270.jpg): Đếm được 27 xe, xác định 2 vùng
  dễ bỏ sót — góc phải tối và xe bị lấp bởi xe lớn. Những vùng này đúng là nơi AI bỏ sót
  (xác nhận bởi REVIEW_LOG: ca `added` tại frame_0099.jpg, frame_0107.jpg).
- Ca khó theo guideline: Xe chỉ thấy đèn hậu mà thân xe tối hoàn toàn → vẽ box ôm phần thân
  xe đoán được quanh cụm đèn, không chỉ khoanh hai chấm đèn (theo GUIDELINE_LABEL.md mục 2).

---

## 5. Kết luận và giới hạn

**Kết quả vòng 1 so với cold start:** AP50 giảm -0.201, từ 0.771 xuống 0.570.

**Quyết định dừng tại vòng 1:** Dừng vì đã đủ yêu cầu bắt buộc (1 vòng hoàn chỉnh) và cần
hiểu rõ nguyên nhân thụt lùi trước khi tiếp tục.

**Hai ca còn yếu đề xuất cho vòng sau:**
1. **Xe nhỏ/xa (R_small=0.000):** Cần thêm ảnh có nhiều xe nhỏ trong pool — chọn frame theo
   tiêu chí n_ambiguous cao + ưu tiên frame có xe xa. Chi phí rà: mỗi frame có thể có 20-30
   xe nhỏ khó nhìn, tốn thời gian; nguy cơ inconsistency cao với xe dưới 16px.
2. **Overfitting (R thấp dù P=1.0):** Cần tăng số ảnh train (chọn lô 20-24 ảnh thay vì 12),
   và giảm epochs hoặc dùng learning rate thấp hơn để bảo toàn kiến thức từ COCO pre-train.
   Nguy cơ: lô lớn hơn tăng công gán nhãn và thời gian rà CVAT.

**Giới hạn của kết quả đo:**

- **Tập test chỉ 20 ảnh:** Với 20 ảnh, AP50 có thể dao động mạnh do may rủi — thêm hoặc bỏ
  1-2 ảnh có thể thay đổi AP50 đáng kể. Không thể kết luận chắc chắn mô hình tốt hay xấu hơn
  chỉ từ 20 ảnh.
- **Nhãn tham chiếu do mô hình tạo, chưa được người rà:** AP50 đo mức khớp với tập tham chiếu
  này. Nếu tham chiếu bỏ sót xe thực, box "đúng" của mô hình sẽ bị tính là false positive —
  P thực tế có thể thấp hơn P=1.000 được báo cáo.
- **Luật bỏ qua xe tiny (height < 16px):** 14 box bị bỏ qua trong tập test, ảnh hưởng đến
  con số R_small nhưng không phản ánh hiệu năng thực tế với xe xa.
- **Nếu AP50 giảm thêm ở vòng sau:** Kiểm tra (1) phân phối lô mới có thiên lệch cảnh nào không,
  (2) số epochs và learning rate, (3) so sánh diff labels để xác nhận nhãn thêm/sửa đúng định
  dạng, (4) thử fine-tune từ checkpoint round1 thay vì từ yolov8n.pt gốc.
