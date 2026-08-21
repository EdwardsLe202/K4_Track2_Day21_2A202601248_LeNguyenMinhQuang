# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Lê Nguyễn Minh Quang |
| MSSV | 2A202601248 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/EdwardsLe202/K4_Track2_Day21_2A202601248_LeNguyenMinhQuang |
| Ngày nộp | 21/08/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ siêu tham số ở lần chạy 3 đạt điểm `f1_score` cao nhất (0.7149) trên tập holdout, vượt qua ngưỡng chất lượng tối thiểu 0.65 của hệ thống CI/CD. Đáng chú ý, lần chạy 1 có `accuracy` cao nhất (0.8780) nhưng `f1_score` (0.7109) lại thấp hơn lần 3. Điều này phản ánh rõ hiện tượng accuracy bị chi phối bởi lớp đa số (thu nhập thấp), trong khi mục tiêu cốt lõi của bài toán là nhận diện chính xác lớp thiểu số (thu nhập cao >50K). Khi tăng số lượng cây `n_estimators` lên 200 kết hợp độ sâu `max_depth=5` và `learning_rate=0.1`, GradientBoosting có khả năng học sâu hơn các tương tác phi tuyến tính phức tạp giữa các đặc trưng nhân khẩu học.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult Census Income có sự mất cân bằng lớp nghiêm trọng khi lớp dương (thu nhập >50K USD/năm) chỉ chiếm 24.8% tổng số mẫu. Trong điều kiện này, một mô hình vô dụng chỉ cần liên tục dự đoán "thu nhập thấp" cho mọi trường hợp vẫn có thể đạt mức accuracy lên tới 75.2%, tạo cảm giác sai lệch rằng mô hình hoạt động hiệu quả. 

Chỉ số `f1_score` tính riêng trên lớp dương (target = 1) là trung bình điều hòa giữa Precision (độ chuẩn xác) và Recall (độ bao phủ), buộc mô hình phải vừa hạn chế dự đoán sai vừa không bỏ sót những người có thu nhập cao. Khi gọi hàm `f1_score(y_eval, preds)`, ta tuyệt đối không truyền `average="weighted"` hay `average="macro"` vì các tùy chọn tính trung bình này sẽ bị lớp đa số kéo điểm lên cao, làm mất đi khả năng kiểm soát chất lượng thật sự của Quality Gate trong pipeline CI/CD.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| MLflow UI trả về lỗi 403 / port bị chiếm khi mở `localhost:5000`. | macOS AirPlay/ControlCenter mặc định chiếm dụng cổng 5000. | Chuyển cổng chạy MLflow sang 5001 (`--port 5001 --host 127.0.0.1`). |
| Khởi tạo máy chủ EC2 ban đầu bị lỗi instance type. | Chọn nhầm AMI Ubuntu có cài sẵn Microsoft SQL Server yêu cầu cấu hình riêng. | Chọn lại AMI tiêu chuẩn Ubuntu Server 24.04 LTS (Free tier) và instance type t3.micro. |
| Model inference service trên EC2 báo lỗi phiên bản scikit-learn. | Sự lệch phiên bản module giữa môi trường CI runner và máy chủ EC2. | Cố định chính xác `scikit-learn==1.4.2` trong `requirements.txt` và cài đặt đồng bộ trên EC2. |

---

## 4. So Sánh Bước 2 và Bước 3

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1` - 22.361 mẫu) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2` - 44.722 mẫu) | 0.7354 | 0.8820 |

**Nhận xét:** Khi mở rộng tập dữ liệu huấn luyện từ 22.361 lên 44.722 mẫu, chỉ số `f1_score` tăng nhẹ từ 0.7149 lên 0.7354 (+0.0205) và `accuracy` tăng từ 0.8740 lên 0.8820. Do hai tập dữ liệu được trích xuất từ cùng phân phối, sự cải thiện phản ánh việc mô hình học được đường phân định tốt hơn ở vùng biên. Quan trọng nhất, toàn bộ chu trình Continuous Training đã vận hành tự động: commit dữ liệu mới kích hoạt GitHub Actions thực hiện kiểm thử, huấn luyện, vượt qua quality gate và deploy bản phát hành mới lên EC2 mà không cần can thiệp thủ công.
