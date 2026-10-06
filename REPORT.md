# BÁO CÁO NHIỆM VỤ LYNX-07

**Kỹ sư:** Nguyễn Văn Giáp  
**Mã sinh viên:** `2A202602903`  
**Nguồn kết quả:** [Notebook Kalman Fusion](Lab/kalman_fusion_lab_STUDENT.ipynb), Phần 9.

Nhiệm vụ kéo dài 90 giây, gồm 450 phép đo GPS (5 Hz) và 900 phép đo UWB (10 Hz). Các số liệu dưới đây được lấy từ kết quả chạy với mã sinh viên trên.

## 1. Bằng chứng

| Cảm biến | Mean NIS | Median NIS | Residual trung bình [x, y] (m) |
|:--|--:|--:|:--|
| GPS | 33.57 | 22.54 | [-0.0599, -0.0073] |
| UWB | 3.48 | 2.45 | [0.0045, -0.0221] |

Với phép đo vị trí hai chiều, NIS kỳ vọng xấp xỉ 2 khi mô hình nhiễu phù hợp. GPS có cả mean và median NIS cao hơn nhiều mức này, trong khi residual trung bình gần [0, 0]. UWB có NIS thấp hơn rõ rệt và median gần mức kỳ vọng.

**Kết luận:** GPS có nhiễu bị đánh giá thấp (`underrated_noise`). Dấu hiệu chính là NIS lớn trên phần lớn phép đo, không phải residual lệch cố định theo một hướng.

## 2. Cách sửa

Chọn `FIX_SENSOR = "GPS"` và `FIX_METHOD = "inflate_R"` để tăng phương sai phép đo GPS, giảm trọng số của GPS khi hợp nhất với UWB.

Theo hàm `apply_mission_fix`, hệ số tăng R là:

```text
gps_r_mult = max(2, mean(NIS_GPS) / 2)
           = 16.7869 ≈ 16.79
```

GPS khai báo độ lệch chuẩn 0.5 m nên R ban đầu là `0.25 × I₂` m². Sau sửa, R GPS là khoảng `4.1967 × I₂` m², tương ứng độ lệch chuẩn phép đo khoảng 2.05 m trên mỗi trục. R của UWB được giữ nguyên.

Hai phương pháp còn lại không phù hợp với dấu hiệu quan sát được:

- **`bias`:** residual trung bình của GPS gần 0, không cho thấy offset cố định đáng kể; trừ offset không giải quyết độ phân tán lớn.
- **`gate`:** median NIS GPS cũng rất cao, nên lỗi không mang dấu hiệu vài outlier đột biến xen giữa các phép đo bình thường. Gating với R ban đầu có thể loại quá nhiều phép đo GPS.

Sau sửa, bộ lọc chấp nhận đủ **1350 phép đo**, với **pooled mean NIS = 2.16** và **median NIS = 1.52**. Mean NIS gần mức kỳ vọng 2 và đạt điều kiện kiểm tra của bài lab (`mean NIS < 8`). Đây là bằng chứng về tính nhất quán của bộ lọc, chưa phải phép đo trực tiếp sai số vị trí thật.

## 3. Độ bất định vị trí cuối cùng

Khối hiệp phương sai vị trí ở lần cập nhật cuối là:

```text
P_position ≈ [[0.157276, 0.000000],
              [0.000000, 0.157276]] m²
```

Độ lệch chuẩn trên từng trục:

```text
σx = sqrt(P[0, 0]) ≈ 0.397 m
σy = sqrt(P[1, 1]) ≈ 0.397 m
```

Theo chỉ số độ bất định tổng hợp trên dashboard:

```text
σ_position = sqrt(Pxx + Pyy) ≈ 0.561 m
```

Giá trị 0.561 m tổng hợp hai trục, khác với 1σ của riêng trục x hoặc y. Với giả định sai số Gaussian hai chiều và khối P ở trên, vùng tin cậy 95% là đường tròn bán kính khoảng **0.971 m**, tính bằng `sqrt(χ²(2, 0.95) × 0.157276)`. Quy tắc gần đúng `2 × σ_position` cho bán kính **1.122 m**, bảo thủ hơn vùng 95% này.

## 4. Hạn chế

Hệ số tăng R cố định được suy ra từ toàn bộ log 90 giây. Nếu GPS thay đổi mức nhiễu khi xe đi vào khu vực bị che khuất, hoặc phát sinh bias hay outlier mới, hệ số 16.79 có thể không còn phù hợp. Tăng R cũng không trực tiếp loại bỏ một sai lệch có hướng.

Trong hệ thống chạy trực tuyến, không thể dùng dữ liệu tương lai để hiệu chỉnh từ đầu nhiệm vụ. Cần ước lượng tham số từ dữ liệu quá khứ, theo dõi NIS và residual trên cửa sổ thời gian, rồi điều chỉnh R hoặc chuyển sang xử lý bias/gating khi có bằng chứng phù hợp. Độ tin cậy suy ra từ P cũng phụ thuộc vào giả định mô hình chuyển động và nhiễu cảm biến độc lập; nếu cả GPS và UWB cùng chịu một sai số chung, P có thể đánh giá thấp độ bất định thực tế.
