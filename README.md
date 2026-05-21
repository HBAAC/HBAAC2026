# Model dự báo nhu cầu phụ tùng Ô tô B2B - [Team CMD]

Cuộc thi: **HBAC 2026**

## Project Overview
Dự án này tập trung giải quyết bài toán Time-series Forecasting cho danh mục B2B khổng lồ gồm **15,972 SKUs** của một nhà phân phối phụ tùng ô tô tại Việt Nam. Dữ liệu lịch sử giao dịch trải dài từ tháng 11/2020 đến tháng 09/2025 với hơn **700,000 bản ghi thô**

**Thách thức của bài toán:**
1. **Dữ liệu thưa thớt & Biến động lớn:** Phần lớn các mã phụ tùng có tần suất xuất hiện rất thấp, nhu cầu không liên tục dẫn đến tạo ra hiện tượng Zero-Inflated
2. **Hiện tượng giá trị âm do hàng Returns:** Hành vi gara trả lại phụ tùng do chẩn đoán sai tạo ra các giao dịch số lượng âm (`Quantity < 0`), làm gãy đổ các mô hình Time-series truyền thống
3. **Độ phức tạp của quy mô dữ liệu:** Khi thực hiện time padding để tạo lưới liên tục cho toàn bộ SKUs trong hơn 1,700 ngày, quy mô dữ liệu bùng nổ lên tới **hơn 28 triệu dòng**, dễ gây tràn RAM
4. **Hàm mục tiêu khắt khe:** Đánh giá bằng **WRMSSE** (Weighted Root Mean Squared Scaled Error), đặt trọng số phạt cực nặng vào các mặt hàng mang lại lợi nhuận cao (LineProfit)

**Chiến lược của Team:** Thay vì sử dụng các mô hình Machine Learning hộp đen (Black-box), team tiếp cận bằng phương pháp Data-driven: Tối ưu hạ tầng lưu trữ qua **SQL Server**, xử lý nhiễu bằng thuật toán tịnh tiến tích lũy (Rolling Compensation), và xây dựng mô hình **Rule-based (Heuristics & Moving Average)** tối ưu trực tiếp cho hàm mục tiêu WRMSSE

---

## 📂 2. Kiến trúc Thư mục (Repository Structure)

```text
├── .gitignore                   <- Chặn các file rác, file hệ thống và data thô quá nặng.
├── README.md                    <- Tổng quan dự án và hướng dẫn vận hành
├── data/                        
│   ├── raw/                     <- Chứa `train.csv` và `sample_submission.csv`
│   └── processed/               <- Dữ liệu đã làm sạch (train_cleaned.csv)
├── notebooks/                   
│   ├── 01_data_cleaning.ipynb   <- Pipeline dọn rác, xử lý số âm (hàng trả về) và Time-Series Padding
│   ├── 02_eda_insights.ipynb    <- Phân tích trực quan hóa, phân phối lợi nhuận (Pareto), đo lường Sparsity
│   └── 03_model.ipynb <- Feature Engineering, tính toán Moving Average và Hậu xử lý Rule-based
└── submissions/                 <- Chứa kết quả dự báo cuối cùng (`submission_final.csv`)
```

---

## 3. Hướng dẫn cài đặt và chạy code

Để tái tạo lại toàn bộ kết quả của nhóm từ dữ liệu thô, vui lòng thực hiện theo đúng thứ tự sau:

**Chuẩn bị dữ liệu:** Tải file `train.csv` và `sample_submission.csv` từ hệ thống Ban Tổ Chức, đặt vào thư mục `dataset`.

Bước 1 - Data Cleaning: Chạy file notebooks/01_data_cleaning.ipynb. File này sẽ ép kiểu dữ liệu an toàn, xử lý triệt để lượng hàng trả về (tịnh tiến số âm) và lấp đầy lưới thời gian (padding), tạo ra ma trận train_cleaned.csv hoàn chỉnh (Lưu ý: Khuyến nghị import data thô qua SQL Server trước để tránh tràn RAM như hướng dẫn trong báo cáo)
* **Bước 2 - Khai phá dữ liệu:** Chạy file `notebooks/02_eda_insights.ipynb`. Hệ thống sẽ tự động quét, vẽ biểu đồ phân phối và định vị các SKUs có trọng số lợi nhuận cao
* **Bước 3 - Tính toán Mô hình & Dự báo:** Chạy file `notebooks/03_rule_based_model_v6.ipynb`. Tập lệnh này sẽ áp dụng các công thức Moving Average, kích hoạt các Rule chặn sàn đối với các mã End-of-Life, và đóng gói ra file nộp bài chuẩn tại `submissions/submission_final.csv`

---

## 4. Phương pháp luận & Insights

Thay vì chạy theo các thuật toán phức tạp nhưng khó kiểm soát, team tập trung giải quyết triệt để 3 vấn đề cốt lõi của Dữ liệu:

### 4.1. Khống chế Rủi ro trả hàng (Negative Returns)
Dữ liệu thô chứa rất nhiều bản ghi có số lượng âm. Nếu xóa bỏ sẽ làm mất thông tin, nếu giữ nguyên sẽ làm sai lệch dự báo. Team đã phát triển **Thuật toán Tịnh tiến Lũy kế (Rolling Compensation)**: Lượng hàng trả lại được trừ ngược về các giao dịch mua hàng trước đó của chính SKU đó. Điều này giúp trả lại bản chất tiêu thụ thực tế mà không làm thay đổi tổng lượng cầu (Total Demand).

### 4.2. Xử lý big data
Với bài toán yêu cầu kẹp đủ lưới ngày (Time Padding) cho 15,972 SKUs trong 5 năm, dung lượng nội suy vọt lên hơn 28 triệu dòng. Team đã chuyển dịch khâu tổng hợp (Aggregation) từ Pandas sang **RDBMS (SQL Server)**. Việc gộp nhóm theo mã vật tư và ngày ngay từ Database giúp giảm thiểu rủi ro tràn bộ nhớ và tăng tốc độ Pipeline đáng kể.

### 4.3. Mô hình Rule-based tối ưu hóa WRMSSE
Sau quá trình làm sạch, team áp dụng mô hình Rule-based được tinh chỉnh chuyên biệt:
* **Weighted Moving Average:** Sử dụng các cửa sổ thời gian (Window Size) chiến lược `[Điền các mốc ngày vào đây, VD: 7, 14, 28, 56 ngày]` để bắt được đà quán tính ngắn hạn của thị trường B2B.
* **Heuristics & End-of-Life Filtering:** Nhận diện các SKUs đã chết (không phát sinh giao dịch trong `[Điền số ngày, VD: 90]` ngày gần nhất) hoặc có xu hướng giảm sâu, team áp dụng luật ép kết quả dự báo về `0`. Do hàm mục tiêu WRMSSE phạt nặng các lỗi over-forecasting trên tập thưa thớt, hành động "dự báo bằng 0" mang lại sự ổn định và tối ưu điểm số hiệu quả hơn hẳn so với việc để mô hình tự đoán
* **Post-processing Clip:** Bất kỳ giá trị dự báo toán học nào trả ra số âm đều được tự động làm tròn về sàn `0` để khớp với logic vận hành kho bãi thực tế

---

## 5. Kết quả

Việc áp dụng chiến lược xử lý nhiễu tinh gọn và bảo vệ nghiêm ngặt chống Data Leakage đã mang lại kết quả:
* **Tập Validation nội bộ:** `[Điền điểm số WRMSSE thử nghiệm của team vào đây]`
* **Public Leaderboard:** `[Điền điểm số trên hệ thống cuộc thi vào đây]`
* **Private Leaderboard:** Đang chờ kết quả cuối cùng từ Ban Tổ Chức

---

## 6. Đội ngũ thành viên

* **[Phan Vũ Đức Trung]**: Data Cleaning, Negative Value Handling (Rolling Compensation) & Time Padding, Exploratory Data Analysis (EDA), Profit Weights Analysis
* **[Đặng Biên Phúc Lâm]**: Rule-based Model Development (Moving Average), WRMSSE Tuning & Post-processing
* **[Huỳnh Phúc Nguyên]**: Data Cleaning, Negative Value Handling (Rolling Compensation) & Time Padding, Exploratory Data Analysis (EDA), Profit Weights Analysis
* **[Đỗ Hoàng Quân]**: Project manager, Strategic planning, Insight reader
