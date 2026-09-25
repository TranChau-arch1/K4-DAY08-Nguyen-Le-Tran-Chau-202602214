# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét ảnh gần trùng hoặc trường hợp model không dự đoán được box:

Nếu chỉ có ngân sách hạn hẹp để rà và sửa nhãn cho đúng **5 ảnh**, chiến lược tối ưu là **tối đa hóa lượng thông tin gradient mới và độ đa dạng bối cảnh theo thời gian**, đồng thời **kiên quyết loại bỏ các frame gần trùng lặp** để không lãng phí chi phí nhân công. 

Bảng tổng hợp 5 frame được đề xuất ưu tiên:

| Thứ hạng (Rank) | Tên file | Thời điểm (`t_sec`) | Điểm tổng (`score`) | Độ bất định (`U`) | Tỷ lệ mập mờ (`A`) | Khoảng cách thời gian (`D`) | Số box (`n_boxes`) | Số box mơ hồ (`n_ambiguous`) |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **1** | `frame_0182.jpg` | 72.8s | 0.9591 | 0.9182 | 1.0000 | 1.0 | 28 | 18 |
| **2** | `frame_0369.jpg` | 147.6s | 0.9324 | 0.9315 | 0.8889 | 1.0 | 43 | 16 |
| **4** | `frame_0326.jpg` | 130.4s | 0.9155 | 0.9310 | 0.8333 | 1.0 | 39 | 15 |
| **8** | `frame_0099.jpg` | 39.6s | 0.9063 | 0.9460 | 0.7778 | 1.0 | 29 | 14 |
| **11** | `frame_0227.jpg` | 90.8s | 0.8915 | 0.9164 | 0.7778 | 1.0 | 37 | 14 |

**Lý do chi tiết cho từng lựa chọn:**
1. **`frame_0182.jpg` (Rank 1, t = 72.8s, score = 0.9591):**
   - Đạt điểm cao nhất toàn bộ tập pool 268 ảnh. Sở hữu số lượng box mập mờ tối đa (`n_ambiguous = 18`, đạt tỷ lệ chuẩn hóa `A = 1.0`) và độ bất định của 5 box khó nhất rất cao (`U = 0.9182`).
   - Nằm ở giữa video (72.8s). Đây là frame chứa nhiều trường hợp mô hình bối rối nhất do đèn xe phản chiếu trên nền đường ướt/tối, đem lại giá trị học tập lớn nhất cho mô hình.
2. **`frame_0369.jpg` (Rank 2, t = 147.6s, score = 0.9324):**
   - Đại diện cho phân cảnh giao thông mật độ cao ở cuối video (t = 147.6s) với 43 box phát hiện và 16 box mơ hồ, độ bất định `U = 0.9315`.
   - Các xe đi sát nhau tạo ra nhiều trường hợp che khuất (occlusion) và chùm đèn pha chiếu trực diện làm lóa biển số/thân xe, giúp mô hình học cách tách biệt các cụm xe đông đúc.
3. **`frame_0326.jpg` (Rank 4, t = 130.4s, score = 0.9155) — *Quyết định xử lý triệt để ảnh gần trùng*:**
   - Trong cụm t = 130s–133s, có nhiều ảnh điểm rất cao như `frame_0326.jpg` (rank 4, 130.4s, score 0.9155), `frame_0331.jpg` (rank 5, 132.4s, score 0.9154), `frame_0330.jpg` (rank 12, 132.0s, score 0.8899), `frame_0329.jpg` (rank 25, 131.6s) và `frame_0328.jpg` (rank 26, 131.2s).
   - Với ngân sách chỉ 5 ảnh, ta chỉ chọn một đại diện tốt nhất là `frame_0326.jpg` và **chủ động loại bỏ các ảnh gần trùng** (đặc biệt là rank 3 `frame_0380.jpg` ở 152.0s quá gần `frame_0369.jpg` ở 147.6s, và rank 5 `frame_0331.jpg` ở 132.4s quá gần `frame_0326.jpg` ở 130.4s). Do camera tĩnh, trạng thái giao thông di chuyển chậm giữa 2 giây gần như tương đồng; nếu chọn cả hai sẽ lãng phí 20%–40% ngân sách vào dữ liệu trùng lặp.
4. **`frame_0099.jpg` (Rank 8, t = 39.6s, score = 0.9063):**
   - Đảm bảo độ đa dạng thời gian (temporal diversity) khi đại diện cho đoạn đầu video (t = 39.6s), cách các frame khác từ 33s đến 108s.
   - Có độ bất định cực cao (`U = 0.9460` - cao nhất trong top 10), mật độ 29 box vừa phải giúp người gán nhãn thao tác nhanh chóng, hiệu quả. Khung cảnh có đèn xe chiếu rọi qua hàng rào mắt cáo tạo ra thách thức lớn về nhận dạng viền xe.
5. **`frame_0227.jpg` (Rank 11, t = 90.8s, score = 0.8915):**
   - Lấp vào khoảng trống thời gian lớn giữa 72.8s và 130.4s (t = 90.8s, cách frame gần nhất 18 giây).
   - Điểm `U = 0.9164`, 37 box và 14 box mơ hồ; trên ảnh có sự xuất hiện của các phương tiện kích thước lớn (xe tải, xe thùng) ở làn giữa, bổ sung mẫu xe đa dạng ngoài dòng xe con thông thường.

*(Xét trường hợp model không dự đoán được box - `empty`): Trong 50 dòng đầu (và toàn bộ 268 ảnh pool), cột `empty` đều là `False` (mô hình luôn phát hiện được từ 24 đến 53 box). Nếu xuất hiện frame có `empty = True`, thuật toán `al_select.py` sẽ cộng thêm `EMPTY_BONUS = 0.5`. Khi đó annotator phải lập tức ưu tiên kiểm tra vì camera giao thông luôn có xe; việc mô hình không tìm được box nào biểu thị lỗi bỏ sót toàn bộ (false negative 100%). Tuy nhiên thực tế mô hình khởi đầu lạnh chưa bao giờ bị trống hoàn toàn trên tập dữ liệu này.*

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

1. **`frame_0182.jpg` (Ảnh ghép hàng 1, ô 3 — `frame_0182.jpg t=72.8s s=0.96`):**
   - **Bằng chứng từ CSV:** Rank 1, `score = 0.9591` (cao nhất toàn bộ tập pool), `U = 0.9182`, `A = 1.0000` (`n_ambiguous = 18` — số box mơ hồ cao nhất), `n_boxes = 28`, `D = 1.0`, `selected = True`.
   - **Bằng chứng từ ảnh ghép (`selection_round1.jpg`):** Tại ô hàng 1, cột 3, luồng giao thông ở làn đường phía xa có mật độ đèn pha và đèn hậu đỏ đan xen dày đặc. Ánh sáng phản chiếu chói lòa trên mặt đường nhựa làm mờ ranh giới gầm xe, khiến mô hình phân vân mạnh (confidence dao động quanh mức 0.5, sinh ra 18 box mập mờ trong dải `[0.15, 0.50]`). Đây là ca khó điển hình về độ bất định ngữ cảnh ban đêm.
2. **`frame_0331.jpg` (Ảnh ghép hàng 2, ô 3 — `frame_0331.jpg t=132.4s s=0.92`):**
   - **Bằng chứng từ CSV:** Rank 5, `score = 0.9154`, `U = 0.8308`, `A = 1.0000` (`n_ambiguous = 18`), tổng số box phát hiện lên tới `n_boxes = 47` (thuộc nhóm nhiều xe nhất toàn bộ lô), `selected = True`.
   - **Bằng chứng từ ảnh ghép (`selection_round1.jpg`):** Ô hàng 2, cột 3 thể hiện rõ tình trạng dồn ứ giao thông nghiêm trọng. Xe ở làn ngoài cùng bên trái chạy sát mép camera (box rất lớn) trong khi các làn giữa và xa có hàng chục xe nối đuôi sát rạt (box nhỏ). Các xe che khuất nhau liên tục dẫn tới 18 box mơ hồ. Mặc dù ở t = 132.4s (rất gần `frame_0326.jpg` ở 130.4s), model vẫn chọn frame này vì khoảng cách `132.4 - 130.4 = 2.0s`, vừa đúng chạm ngưỡng `min_gap_s >= 2.0s`.
3. **`frame_0099.jpg` (Ảnh ghép hàng 1, ô 1 — `frame_0099.jpg t=39.6s s=0.91`):**
   - **Bằng chứng từ CSV:** Rank 8, `score = 0.9063`, độ bất định `U = 0.9460` (thuộc top cao nhất lô), `A = 0.7778` (`n_ambiguous = 14`), `n_boxes = 29`, `selected = True`.
   - **Bằng chứng từ ảnh ghép (`selection_round1.jpg`):** Ô hàng 1, cột 1 là frame sớm nhất trong toàn bộ 12 ảnh chọn (t = 39.6s). Ảnh cho thấy luồng đèn pha rọi thẳng về phía máy quay, tạo quầng sáng lớn làm lóa các phương tiện đi sau. Đồng thời, khung hình bị các thanh lưới thép của hàng rào tiền cảnh cắt xẻ, gây nhiễu cấu trúc nhận dạng của mạng nơ-ron (khiến U đạt tới 0.9460). Việc chọn frame này bổ sung mẫu bối cảnh đầu video rất giá trị cho lô huấn luyện.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

- **Frame có điểm cao nhưng KHÔNG được chọn:** `frame_0372.jpg`
  - **Dữ liệu đối chiếu từ CSV:** Rank 6, `score = 0.9101`, `U = 0.9202`, `A = 0.8333` (`n_ambiguous = 15`), `n_boxes = 42`, `t_sec = 148.8s`, `selected = False`.
  - **So sánh điểm:** `score = 0.9101` của `frame_0372.jpg` cao hơn hầu hết các frame được chọn trong lô như `frame_0312.jpg` (rank 7, score 0.9100), `frame_0099.jpg` (rank 8, score 0.9063), `frame_0187.jpg` (rank 10, score 0.8995), `frame_0227.jpg` (rank 11, score 0.8915), `frame_0270.jpg` (rank 13), `frame_0107.jpg` (rank 14) và `frame_0392.jpg` (rank 15).
  - **Lý do không được chọn:** Thuật toán lựa chọn tham lam (`_greedy` trong `tools/al_select.py`) áp dụng quy tắc khoảng cách thời gian tối thiểu `min_gap_s = 2.0s`. Trước đó, thuật toán đã chọn `frame_0369.jpg` (rank 2) tại thời điểm `t = 147.6s`. Khoảng cách thời gian giữa hai frame là `|148.8 - 147.6| = 1.2s < 2.0s`.
  - **Ý nghĩa thực tế:** Vì góc camera giao thông là cố định và các phương tiện lưu thông trong vòng 1.2 giây gần như giữ nguyên vị trí và hình thái, nội dung của `frame_0372.jpg` và `frame_0369.jpg` gần như trùng lặp hoàn toàn. Việc thuật toán bỏ qua `frame_0372.jpg` là hoàn toàn đúng đắn, giúp tránh lãng phí công sức rà soát 42 box trùng lặp và ngăn chặn mô hình bị overfit vào một phân cảnh cục bộ tại giây thứ 148.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:

1. **Độ bất định (Uncertainty) không đồng nghĩa với độ chính xác hay khả năng tổng quát hóa:**
   - Điểm bất định được tính dựa trên độ phân vân của chính mô hình hiện tại (`u = 1 - |2·conf - 1|`). Cách tính này chỉ nhận diện được những lỗi mà mô hình "tự biết mình đang do dự" (*known unknowns*).
   - Nó hoàn toàn "mù" trước những sai lầm mà mô hình tự tin thái quá (*overconfident errors / unknown unknowns*). Ví dụ: mô hình nhận diện nhầm các vệt phản quang trên mặt đường hoặc đèn biển hiệu thành xe với `conf = 0.95`, hoặc bỏ sót hoàn toàn các xe đi trong vùng tối (`conf < 0.05` nên không sinh ra box). Ở các trường hợp này, `U` đều xấp xỉ 0 và frame bị xếp hạng rất thấp, khiến lỗi sai hệ thống bị bỏ qua.
2. **Không bảo đảm việc fine-tune trên các mẫu này sẽ cải thiện hiệu năng trên tập test:**
   - Việc gom các frame có điểm bất định cao chỉ chứng minh ta đang thu thập các mẫu gây bối rối nhất cho mô hình trên tập pool. Với kích thước lô nhỏ (12 ảnh), việc tập trung quá nhiều vào các ca dị biệt (xe bị che khuất nặng, lóa đèn phức tạp) có thể làm dịch chuyển phân phối dữ liệu (distribution shift), dẫn tới hiện tượng suy giảm Precision (sinh thêm nhiều False Positive) hoặc giảm khả năng phát hiện xe nhỏ (small objects) trên tập test chuẩn.
3. **Quy tắc khoảng cách thời gian (`min_gap_s = 2.0s`) chỉ là xấp xỉ thô:**
   - Khoảng cách thời gian không phản ánh chính xác độ đa dạng thị giác (semantic diversity). Khi xảy ra ùn tắc giao thông, các xe dừng yên thì hai frame cách nhau 10 giây vẫn có thể trùng lặp hoàn toàn; ngược lại khi dòng xe chạy với tốc độ cao (80 km/h) thì chỉ cần 1.5 giây đã là một trạng thái giao thông hoàn toàn khác. Bộ chọn hiện tại chưa đo đạc khoảng cách không gian đặc trưng (embedding feature distance) giữa các ảnh.
4. **Không đo lường được tính đại diện và chất lượng nhãn tham chiếu:**
   - Phép chọn hoàn toàn là quá trình suy luận nội bộ của mô hình trên pool dữ liệu không nhãn, không liên quan tới tập test. Nó không chứng minh được liệu mô hình có tổng quát hóa tốt hơn hay chỉ đang học vẹt để khớp với nhãn tham chiếu (vốn cũng được tạo tự động bởi mô hình khác và chưa được con người rà soát toàn diện).
