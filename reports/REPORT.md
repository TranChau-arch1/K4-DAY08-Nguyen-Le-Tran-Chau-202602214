# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Lê Trân Châu

Công cụ gán nhãn đã dùng: CVAT
Mọi con số trong báo cáo được truy xuất chính xác từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json`, `outputs/round1_diff.md` và `outputs/compare_round*.jpg`.

---

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) trong bài toán này được trích xuất từ cùng một video giám sát giao thông ban đêm quay cố định. Cách phân chia tập dữ liệu được thiết kế theo **trục thời gian (temporal split) có vùng đệm (buffer zone)** vì các lý do sau:

1. **Bản chất tương quan chuỗi thời gian cực lớn (Temporal Correlation):**
   - Video được trích xuất từ camera tĩnh với tần suất cao (mỗi frame cách nhau chỉ 0.4 giây). Các frame liên tiếp nhau gần như giữ nguyên góc nhìn, điều kiện ánh sáng, mặt đường và vị trí của các phương tiện di chuyển chậm.
   - Nếu chia ngẫu nhiên (random split), hai frame kế tiếp nhau có thể một frame rơi vào tập huấn luyện và một frame rơi vào tập kiểm thử. Điều này dẫn tới hiện tượng **rò rỉ dữ liệu nghiêm trọng (data leakage)**: mô hình chỉ cần "học vẹt" bối cảnh và vị trí xe ở frame trước là có thể phát hiện chính xác ở frame sau, mà không học được khả năng tổng quát hóa thực sự.
2. **Vai trò của vùng đệm (Buffer Zone):**
   - Vùng đệm là các khoảng thời gian bị loại bỏ khỏi cả tập train và tập test (như các frame 34–43 nằm giữa pool và test). 
   - Vùng đệm tạo ra một khoảng cách thời gian đủ dài để các phương tiện đang lưu thông ở tập train kịp di chuyển ra khỏi khung hình camera, đảm bảo các xe xuất hiện trong tập kiểm thử là các tình huống giao thông mới độc lập.
3. **Ảnh hưởng nếu chia ngẫu nhiên:**
   - Nếu chia ngẫu nhiên, các số đo đánh giá trên tập kiểm thử (AP50, Precision, Recall) sẽ bị **lệch lạc theo hướng thổi phồng quá mức (artificially inflated / overly optimistic)**.
   - Điểm số cao lúc này chỉ phản ánh khả năng ghi nhớ (memorization) các frame kề cận trùng lặp, tạo ra "ảo tưởng" về chất lượng mô hình. Khi triển khai mô hình vào thực tế trên luồng video tương lai, hiệu năng sẽ sụt giảm nghiêm trọng.

---

## 2. Mô hình khởi đầu lạnh (cold start)

**Dòng vòng 0 trích xuất từ `reports/rounds_table.md`:**

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

*(Chi tiết từ `outputs/metrics_round0.json`: AP50 = 0.7714, TP = 197, FP = 16, FN = 206 trên 20 ảnh test với 403 box tham chiếu).*

**Phân tích dựa trên ảnh đối chiếu `outputs/compare_round0.jpg`:**
1. **Các loại xe mô hình không khớp nhãn tham chiếu:**
   - *Bỏ sót (False Negative - màu vàng):* Mô hình bỏ sót rất nhiều xe ở làn đường phía xa (chỉ thấy đốm sáng đèn xe li ti), các xe di chuyển sát mép dải phân cách hoặc lề đường chìm trong bóng tối, và các xe bị che khuất một phần (occlusion) bởi phương tiện đi trước.
   - *Dự đoán thừa (False Positive - màu đỏ):* Mô hình nhận nhầm một số vệt phản quang đèn pha trên mặt đường nhựa ướt và biển báo kim loại phản xạ ánh sáng thành xe.
2. **Ý nghĩa của độ phủ (Recall) theo kích thước xe:**
   - Xe lớn đạt R large = 0.561 (56.1%, 23/41 box) và xe vừa đạt R medium = 0.547 (54.7%, 162/296 box), cho thấy mô hình pretrained COCO nhận diện khá ổn các xe ở cự ly gần và trung bình.
   - Tuy nhiên, xe nhỏ chỉ đạt R small = 0.182 (18.2%, bắt được 12/66 box). Điều này chứng minh điểm nghẽn lớn nhất của mô hình khởi đầu lạnh là **khả năng phát hiện xe nhỏ ở xa trong điều kiện thiếu sáng**, do thiếu đặc trưng đường nét rõ ràng.
3. **Trường hợp cần rà soát lại nhãn tham chiếu:**
   - Quan sát tại góc phải mép đường trong `frame_0250` và `frame_0350`, có những vùng tối xuất hiện vệt đỏ/vàng nơi nhãn tham chiếu (cyan) đánh dấu box nhưng hình ảnh bị nhiễu hạt nặng, không thể khẳng định chắc chắn có xe hay không. Do nhãn tham chiếu được tạo tự động bởi mô hình khác mà chưa có chuyên viên rà soát thủ công 100%, những ca mơ hồ này cần người thẩm định trực tiếp hình ảnh gốc trước khi vội vã kết luận mô hình dự đoán sai.

---

## 3. Chiến lược chọn mẫu

**Giải thích công thức tính điểm ưu tiên và vai trò của `MIN_GAP_S`:**
- Công thức: `score = W_U·U + W_A·A + W_D·D` (trọng số mặc định: W_U = 0.5, W_A = 0.3, W_D = 0.2).
  - **`U` (Uncertainty):** Trung bình độ bất định của 5 box khó nhất trong ảnh, tính theo u(c) = 1 - |2c - 1|. Điểm đạt cực đại bằng 1.0 khi độ tin cậy c = 0.5 (mô hình phân vân cao nhất giữa có xe và không có xe).
  - **`A` (Ambiguity):** Tỷ lệ số box mập mờ (0.15 <= conf < 0.50), chuẩn hóa chia cho giá trị lớn nhất trong pool. Frame có nhiều box mập mờ thể hiện toàn cảnh đang gây rối loạn cho bộ phát hiện.
  - **`D` (Diversity):** Khoảng cách thời gian tới frame đã gán gần nhất (chặn ở 10.0 giây rồi chia cho 10.0). Ở vòng 1 chưa có ảnh nào gán nên D = 1.0; ở các vòng sau, D giúp trải đều các mẫu trên trục thời gian.
  - Nếu ảnh không có box nào (`empty = True`), thuật toán cộng thêm 0.5 (`EMPTY_BONUS`) để kích hoạt sự chú ý của annotator vì camera giao thông luôn có xe.
- **Vai trò của `MIN_GAP_S = 2.0s`:**
  - Vì camera góc cố định, hai frame cách nhau dưới 2 giây có bối cảnh và vị trí xe gần như trùng lặp hoàn toàn. `MIN_GAP_S` hoạt động như một bộ lọc triệt tiêu trùng lặp (temporal NMS), ép thuật toán bỏ qua các frame quá sát nhau để tiết kiệm chi phí nhân công và tăng độ bao phủ phân phối dữ liệu.

**Dẫn chứng từ `reports/SELECTION.md` và một frame so sánh:**
- Ba frame được chọn trong `reports/SELECTION.md`:
  - `frame_0182.jpg` (rank 1, t = 72.8s, score = 0.9591): Đạt A = 1.0 (18 box mơ hồ cao nhất pool), U = 0.9182. Đại diện cho đoạn giữa video có bối cảnh lóa đèn phức tạp nhất.
  - `frame_0369.jpg` (rank 2, t = 147.6s, score = 0.9324): Mật độ giao thông cực lớn (43 box), U = 0.9315, đại diện cho cụm xe ùn ứ ở cuối video.
  - `frame_0326.jpg` (rank 4, t = 130.4s, score = 0.9155): Đại diện duy nhất được chọn trong cụm 130s–133s. Ta chủ động loại bỏ frame kề cận `frame_0331.jpg` (rank 5, 132.4s) và `frame_0330.jpg` (rank 12, 132.0s) để tránh lãng phí ngân sách gán nhãn vào dữ liệu gần trùng.
- **Một frame khác để minh chứng: `frame_0372.jpg` (Rank 6, score = 0.9101, t = 148.8s):**
  - Mặc dù có điểm cao thứ 6 toàn pool (cao hơn rank 7, 8, 10, 11, 13, 14, 15), frame này **bị loại bỏ** (`selected = False`) vì nằm cách `frame_0369.jpg` (rank 2, t = 147.6s) chỉ 1.2 giây (< 2.0s). Việc loại bỏ frame này minh chứng rõ ràng cho việc đánh đổi thông minh: từ chối điểm cao cục bộ để tránh trùng lặp 42 box và tiết kiệm công sức gán nhãn.

**Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không? Vì sao?**
- **KHÔNG**. Điểm bất định cao chỉ phản ánh rằng mô hình hiện tại đang lúng túng trước frame đó (*known unknowns*). Nó không bảo đảm rằng sau khi fine-tune, mô hình sẽ học tốt hơn trên tập kiểm thử độc lập. Nếu ảnh bất định chứa quá nhiều nhiễu dị biệt (outliers, lóa sáng cực đoan, xe bị cắt vụn), việc huấn luyện có thể làm méo mó phân phối trọng số, gây suy giảm độ chính xác tổng thể hoặc làm mô hình trở nên quá thận trọng.

---

## 4. Các vòng học chủ động (active learning)

**Bảng so sánh tổng hợp từ `reports/rounds_table.md`:**

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 335 | 0.371 | -0.400 | 1.000 | 0.097 | 0.176 | 0.000 | 0.091 | 0.293 |

**Phân tích chi tiết Vòng 1:**
- **Mức độ sửa nhãn gợi ý (truy xuất từ `outputs/round1_diff.md`):**
  - Trên 12 ảnh, mô hình ban đầu đề xuất 169 box. Sau khi rà soát và hiệu chỉnh bằng CVAT, số box chuẩn đạt 335 box:
    - `accepted` (giữ nguyên): 123 box (tỷ lệ chấp nhận 73%).
    - `edited` (sửa lại khung): 30 box (điều chỉnh kích thước ôm sát thân xe hoặc tách box gộp).
    - `deleted` (xóa box giả do model đề xuất): 16 box (loại bỏ các vệt sáng mặt đường và bóng mờ).
    - `added` (thêm mới box do model bỏ sót): 182 box (chủ yếu là xe ở làn xa và xe bị bóng tối che khuất).
- **Biến động AP50:**
  - AP50 giảm mạnh từ 0.771 xuống 0.371, tức ΔAP50 = -0.400 so với khởi đầu lạnh.
- **Biến động số đo theo nhóm xe:**
  - *Mặt tích cực:* Precision tăng tuyệt đối lên 1.000 (từ 0.925), số lượng False Positive giảm triệt để về 0 (FP = 0 trên toàn bộ 20 ảnh test).
  - *Mặt tiêu cực:* Recall sụt giảm thảm hại từ 0.489 xuống còn 0.097 (FN = 364). Trong đó:
    - Xe lớn: Recall giảm từ 0.561 xuống 0.293.
    - Xe trung bình: Recall giảm từ 0.547 xuống 0.091.
    - Xe nhỏ: Recall rơi thẳng về 0.000 (mô hình không phát hiện được bất kỳ chiếc xe nhỏ nào ở ngưỡng conf >= 0.25).

**Chỉ ra ca kết quả thay đổi sau fine-tune từ `outputs/compare_round1.jpg`:**
- *Quan sát trên `frame_0050` và `frame_0150`:*
  - Ở vòng 0 (cold start), mô hình đạt 11 TP (xanh lá), 2 FP (đỏ) và 7 FN (vàng) trên `frame_0050`.
  - Sang vòng 1, toàn bộ 2 box đỏ FP đã biến mất hoàn toàn (FP = 0), nhưng số box xanh TP tụt xuống chỉ còn đúng 1 box (chiếc xe lớn nhất ngay tiền cảnh), trong khi số box vàng FN tăng vọt lên 17 box!
  - *Lý do có thể kiểm chứng:* Việc fine-tune trên tập dữ liệu nhỏ (12 ảnh) trong 50 epoch khiến mô hình bị lệch chuẩn độ tin cậy (confidence calibration shift) và trở nên quá khắt khe. Các dự đoán xe nhỏ ở xa vẫn tồn tại nhưng điểm confidence bị đẩy xuống dưới ngưỡng lọc 0.25, dẫn đến việc bị tính là False Negative trên tập test.

**Đối chiếu giữa quan sát độc lập, sửa nhãn và kết quả mô hình:**
- *Quan sát độc lập (`BLIND_SCAN.md` trên `frame_0326.jpg`):* Trước khi xem pre-label, người rà đếm được 34 xe và chỉ rõ 2 vị trí hiểm là xe sát lề phải phía xa và cụm xe nhỏ li ti dưới gầm cầu.
- *Lỗi pre-label đã sửa (`REVIEW_LOG.csv` và `round1_diff.md`):* Trên `frame_0326.jpg`, mô hình ban đầu chỉ gợi ý 15 box (bỏ sót hơn một nửa). Người rà đã giữ 13 box, xóa 2 box giả và bổ sung 20 box còn thiếu để đạt 33 box. Trên `frame_0099.jpg`, người rà đã sửa 1 box AI gộp chung 2 xe thành 2 box riêng biệt. Trên `frame_0331.jpg`, người rà xóa box nhận nhầm vệt phản quang trên mặt đường.
- *Kết quả mô hình sau train:* Dù con người đã nỗ lực bổ sung 182 box bị bỏ sót vào tập train, mô hình sau fine-tune lại học theo hướng triệt tiêu box giả cực đoan, dẫn đến việc mất hẳn độ nhạy với xe nhỏ trên tập test (R_small = 0.0).
- *Mô tả một ca khó theo guideline:* Ca xe bị che khuất một phần (occlusion) trên `frame_0369.jpg` (hoặc `frame_0182.jpg`): Hai xe con chạy song song ở làn giữa bị xe tải che mất nửa thân xe. Theo [GUIDELINE_LABEL.md](file:///d:/AI_Vinuni/Lab/Lab-d8/K4-DAY08-Nguyen-Le-Tran-Chau-202602214/GUIDELINE_LABEL.md), box chỉ được bao trọn phần thân xe nhìn thấy được, không phỏng đoán phần khuất và không được vẽ box trùm hai xe. Pre-label của AI bỏ sót hoàn toàn ca này do thiếu biên dạng chuẩn; người rà đã can thiệp thêm box độc lập chính xác.

---

## 5. Kết luận và giới hạn

**1. Đánh giá kết quả vòng này và quyết định:**
- Kết quả vòng 1 cho thấy mô hình đạt độ chính xác tuyệt đối (P = 1.000, không có dự đoán sai), nhưng đánh đổi bằng việc sụt giảm nghiêm trọng độ phủ (R = 0.097, AP50 giảm từ 0.771 xuống 0.371).
- **Quyết định:** Cần **TẠM DỪNG quy trình gán nhãn tiếp theo**. Việc tiếp tục gán nhãn vòng 2 theo cách cũ sẽ lãng phí công sức khi mô hình đang gặp vấn đề về siêu tham số huấn luyện (overfitting và co cụm confidence score). Cần tinh chỉnh lại cấu hình training trước khi gán thêm dữ liệu.

**2. Đề xuất hai ca còn yếu/bất định cho vòng tiếp theo:**
- *Ca 1 — Frame ở phân đoạn đầu video (`frame_0020.jpg` tại t = 8.0s):*
  - Điểm mạnh: Lấp khoảng trống dữ liệu ở đầu video, mật độ 24 box vừa phải giúp giảm áp lực công gán nhãn, cách xa các frame đã gán ở vòng 1 (> 30s) nên hoàn toàn không có nguy cơ gần trùng.
- *Ca 2 — Frame chứa phương tiện cỡ lớn ở làn giữa (`frame_0218.jpg` tại t = 87.2s):*
  - Giúp củng cố lại đặc trưng của các xe tải/xe khách cỡ lớn đang bị suy giảm Recall (R_large giảm còn 0.293). Khoảng cách thời gian cách frame gần nhất (`frame_0227.jpg` ở 90.8s) là 3.6s > MIN_GAP_S, đảm bảo an toàn về độ đa dạng bối cảnh.

**3. Tác động từ các giới hạn của bài toán:**
- *Tập kiểm thử chỉ 20 ảnh:* Kích thước mẫu quá nhỏ dẫn đến phương sai thống kê rất lớn; chỉ cần một vài thay đổi về confidence ở vài xe là số đo AP50 đã biến động mạnh.
- *Quy tắc bỏ qua xe cao dưới 16 px:* Trong khi người rà cố gắng dán nhãn đầy đủ cho các xe ở rất xa trong tập train, tập test lại loại bỏ 14 box dưới 16 px. Sự không nhất quán về ngưỡng kích thước giữa train và test có thể gây phạt oan điểm mô hình.
- *Nhãn tham chiếu do AI tạo:* Nhãn test chưa qua kiểm định thủ công toàn diện, có thể chứa lỗi False Negative và False Positive hệ thống. Do đó, việc AP50 giảm không đồng nghĩa 100% với việc mô hình thực tế kém đi, mà có thể do mô hình fine-tune không còn "bắt chước" các lỗi sai của mô hình tạo nhãn test ban đầu.

**4. Kế hoạch kiểm tra khi AP50 giảm trước khi train thêm:**
- **Kiểm tra phân phối Confidence Score:** Hạ ngưỡng đánh giá `conf_thr` từ 0.25 xuống 0.10 hoặc 0.05 để kiểm tra xem mô hình có thực sự bỏ sót xe hay chỉ vì điểm confidence bị hạ thấp sau fine-tune.
- **Điều chỉnh số Epoch và Learning Rate:** Giảm số epoch từ 50 xuống 15–20 epoch, hoặc đóng băng backbone (`freeze`) để tránh làm hỏng các đặc trưng trích xuất COCO mạnh mẽ vốn có của mô hình khởi đầu lạnh.
- **Rà soát tính đồng nhất của nhãn:** Đối chiếu lại nhãn đã sửa ở vòng 1 với tập nhãn test để đảm bảo quy chuẩn vẽ box (độ ôm sát viền xe, xử lý vùng khuất) có sự tương thích cao nhất.
