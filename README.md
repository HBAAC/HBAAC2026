# Model dự báo nhu cầu phụ tùng Ô tô B2B - [Team CMD]

Cuộc thi: **HBAC 2026**

## 1. Project Overview
Dự án này tập trung giải quyết bài toán Time-series Forecasting cho danh mục B2B khổng lồ gồm **15,972 SKUs** của một nhà phân phối phụ tùng ô tô tại Việt Nam. Dữ liệu lịch sử giao dịch trải dài từ tháng 11/2020 đến tháng 09/2025 với hơn **700,000 bản ghi thô**

**Thách thức của bài toán:**
1. **Dữ liệu thưa thớt & Biến động lớn:** Phần lớn các mã phụ tùng có tần suất xuất hiện rất thấp, nhu cầu không liên tục dẫn đến tạo ra hiện tượng Zero-Inflated.
2. **Hiện tượng giá trị âm do hàng Returns:** Hành vi gara trả lại phụ tùng do chẩn đoán sai tạo ra các giao dịch số lượng âm (`Quantity < 0`), làm gãy đổ các mô hình Time-series truyền thống
3. **Độ phức tạp của quy mô dữ liệu:** Khi thực hiện time padding để tạo lưới liên tục cho toàn bộ SKUs trong hơn 1,700 ngày, quy mô dữ liệu bùng nổ lên tới **hơn 28 triệu dòng**, dễ gây tràn RAM
4. **Hàm mục tiêu khắt khe:** Đánh giá bằng **WRMSSE** (Weighted Root Mean Squared Scaled Error), đặt trọng số phạt cực nặng vào các mặt hàng mang lại lợi nhuận cao

**Hướng tiếp cận của Team:** Thay vì sử dụng các mô hình Machine Learning hộp đen (Black-box), team tiếp cận bằng phương pháp Data-driven: Tối ưu hạ tầng lưu trữ qua **SQL Server**, xử lý nhiễu bằng thuật toán tịnh tiến tích lũy (Rolling Compensation), và xây dựng mô hình **Rule-based (Heuristics & Moving Average)** tối ưu trực tiếp cho hàm mục tiêu WRMSSE

---

## 2. Kiến trúc Thư mục (Repository Structure)

```text
├── .gitignore                   <- Chặn các file rác, file hệ thống và data thô quá nặng.
├── README.md                    <- Tổng quan dự án và hướng dẫn vận hành.
├── data/                        
│   ├── raw/                     <- Chứa `train.csv` và `sample_submission.csv` (Không đưa lên Git).
│   └── processed/               <- Dữ liệu đã làm sạch (`train_cleaned.csv`).
├── notebooks/                   
│   ├── 01_data_cleaning_and_eda.ipynb <- Pipeline dọn rác, xử lý số âm, Time-Series Padding kết hợp Phân tích trực quan hóa (Pareto, Sparsity).
│   └── 02_rule_based_model_v6.ipynb   <- Feature Engineering, tính toán Moving Average và Hậu xử lý Rule-based.
└── submissions/                 <- Chứa kết quả dự báo cuối cùng (`submission.csv`).
```

---

## 3. Hướng dẫn cài đặt và chạy code

Để tái tạo lại toàn bộ kết quả của nhóm từ dữ liệu thô, vui lòng thực hiện theo đúng thứ tự sau:

**Chuẩn bị dữ liệu:** Tải file `train.csv` và `sample_submission.csv` từ hệ thống Ban Tổ Chức, đặt vào thư mục `dataset` trong máy

* **Bước 1 - Data Cleaning & EDA (Làm sạch & Khai phá dữ liệu):** Chạy file `notebooks/01_data_cleaning_and_eda.ipynb`. Tập lệnh này sẽ thực hiện song song 2 nhiệm vụ: 
  * Ép kiểu dữ liệu an toàn, xử lý triệt để lượng hàng trả về (tịnh tiến số âm), lấp đầy lưới thời gian (padding) để xuất ra ma trận `train_cleaned.csv` hoàn chỉnh
  * Tự động quét, vẽ biểu đồ phân phối lợi nhuận và định vị đặc tính thưa thớt (sparsity) của các mã hàng.
* **Bước 2 - Tính toán Mô hình & Dự báo:** Chạy file `notebooks/02_model.ipynb`. Tập lệnh này sẽ lấy file data sạch ở Bước 1, áp dụng các công thức Moving Average, kích hoạt các Rule chặn sàn đối với các mã End-of-Life, và đóng gói ra file nộp bài chuẩn tại `submissions/submission.csv`

---

## 4. Phương pháp luận và Insights 

Thay vì áp dụng các mô hình học máy dễ bị bẫy sai số phân số phá hủy điểm số WRMSSE trên tập dữ liệu thưa thớt, team tập trung thiết lập một hệ thống **Stale-Aware Conservative Rule-Based Forecasting**. Các kỹ thuật được triển khai trực tiếp từ cấu trúc mã nguồn hệ thống bao gồm:

### 4.1. Khấu trừ Lượng hàng trả (Negative Returns)
* **Thuật toán tịnh tiến lũy kế (Rolling Compensation):** Khắc phục nhiễu do garage trả hàng (`Quantity < 0`). Lượng hàng trả được bù trừ ngược logic về các ngày mua hàng trước đó của chính SKU đó, bảo toàn tổng lượng cầu thực tế mà không làm biến dạng chuỗi thời gian phân tích

### 4.2. Cô lập phân đoạn Thưa thớt (Sparse vs. Active Segments)
Mô hình toán học của chỉ số WRMSSE phạt cực nặng lỗi dự báo thừa (Over-forecasting) trên nhóm sản phẩm bán chậm do mẫu số phương sai lịch sử của nhóm này rất nhỏ. 
* **Hành động toán học:** Team thiết lập ngưỡng `sparse_threshold` để chia tách danh mục SKU làm 2 phân đoạn rõ rệt: Nhóm Hoạt động (Active) và Nhóm Thưa thớt (Sparse)
* Nhóm Sparse được áp dụng một mức dự báo nền cực thấp (Conservative Baseline) để triệt tiêu hoàn toàn rủi ro tích lũy sai số WRMSSE trên diện rộng

### 4.3. Cơ chế Stale-Aware & Kiểm soát Hoạt động Gần nhất (Low Recent Activity)
Đây là cải tiến cốt lõi giúp nâng cấp mô hình từ các phiên bản trước lên cấu trúc model cuối cùng:
* **Stale Rules - Hard/Soft):** Trích xuất từ phân tích vòng đời sản phẩm, hệ thống đo lường khoảng cách từ lần bán cuối cùng (`Days since last sale`). Nếu một SKU vượt ngưỡng đóng băng (`stale_days`), kết quả dự báo lập tức bị ép (clip) về `0` hoặc mức sàn bảo thủ để tiết kiệm tài nguyên và tối ưu metric
* **Under-forecast Control (Low Recent Activity):** Bản model cuối cùng bổ sung cơ chế kiểm soát động cho các mã hàng có hoạt động cực kỳ yếu trong vòng 14 ngày gần nhất (`low_recent_activity`). Bộ lọc này hoạt động như một lớp "phanh hãm" ngăn chặn mô hình đưa ra các dự báo cầu đột biến thiếu căn cứ khi chuỗi đang có dấu hiệu đi xuống

### 4.4. Tinh chỉnh Lịch trình kinh doanh và Tự động Fallback
* **Calendar Multipliers:** Mô hình tích hợp bộ điều chỉnh trọng số ngày đặc biệt, thực hiện giảm mạnh lượng cầu vào các ngày có hoạt động giao dịch thấp theo hành vi ngành (ví dụ: ngày Chủ Nhật với `sunday_multiplier`)
* **Grid Search & Fallback Framework:** Thay vì cố định tham số, version model cuối cùng vận hành một không gian tìm kiếm hẹp (Focused Search Space) quanh vùng cấu hình tối ưu của tập Validation nội bộ. Điểm đặc sắc nhất là cơ chế an toàn: Nếu cấu hình mới không vượt qua được điểm kiểm định chéo của phiên bản cũ, hệ thống sẽ tự động kích hoạt chế độ Fallback bảo thủ để đảm bảo file nộp bài cuối cùng luôn giữ độ ổn định cao nhất
* **Hậu xử lý (Post-processing):** Toàn bộ giá trị dự báo sau tính toán nếu xuất hiện số âm đều được cắt gọt nghiêm ngặt về ngưỡng sàn kho bãi thực tế bằng phương pháp `clip(0)`

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
* **[Huỳnh Phúc Nguyên]**: Data Cleaning, Negative Value Handling (Rolling Compensation) & Time Padding, Exploratory Data Analysis (EDA), Profit Weights Analysis, Rule-based Model Development
* **[Đỗ Hoàng Quân]**: Project manager, Strategic planning, Insight reader
