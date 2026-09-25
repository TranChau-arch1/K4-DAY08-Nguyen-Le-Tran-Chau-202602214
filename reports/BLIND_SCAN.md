# Quét độc lập trước khi xem pre-label

Frame: `to_label/round1/images/train/frame_0326.jpg`

Số xe nhìn thấy bằng mắt: 34

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe: 
- Xe sát lề phải phía xa: Nằm ở vùng tối bên phải màn hình, dễ bị lẫn vào nền tối hoặc bị bỏ qua do kích thước nhỏ.   
- Cụm xe xa ở dưới cầu: Chỉ hiển thị các đốm sáng nhỏ li ti, dễ bị AI nhầm lẫn với đèn đường hoặc bỏ sót do độ phân giải thấp. 
Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
