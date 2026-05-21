# HBAAC2026
# 🏆 Giải pháp Dự báo Nhu cầu Phụ tùng Ô tô B2B - [Tên Team của bạn]

Tham gia cuộc thi: **HBAC Demand Forecasting Datathon 2026**

## 📖 1. Tổng quan dự án (Project Overview)
Dự án này giải quyết bài toán dự báo chuỗi thời gian cho danh mục bán lẻ B2B khổng lồ gồm **15,972 SKUs** của một nhà phân phối phụ tùng ô tô tại Việt Nam, dựa trên 5 năm lịch sử giao dịch (11/2020 - 09/2025). 

**Thách thức cốt lõi:**
1. **Phân phối "Đuôi dài" (Long-tail & Sparsity):** Phần lớn các mã phụ tùng bán cực kỳ thưa thớt (có mã cả tháng bán 1-2 lần), dẫn đến ma trận dữ liệu tràn ngập số 0 (Zero-Inflated).
2. **Hành vi Trả hàng (Returns):** Tỷ lệ khách hàng (Gara) trả lại phụ tùng do chẩn đoán sai bệnh rất cao, làm nhiễu tín hiệu lượng cầu thực tế.
3. **Hàm mục tiêu khắt khe:** Đánh giá bằng **WRMSSE** (Weighted Root Mean Squared Scaled Error), phạt cực kỳ nặng đối với các sai số trên nhóm sản phẩm mang lại lợi nhuận cao.

**Chiến lược của Team:** Áp dụng phương pháp tiếp cận **"Chia để trị" (Divide & Conquer)** kết hợp giữa Học máy (LightGBM với hàm loss Tweedie) cho nhóm sản phẩm chủ lực và Luật kinh doanh (Heuristic Rules) cho nhóm sản phẩm thưa thớt/lỗi thời.

---

## 📂 2. Kiến trúc Thư mục (Repository Structure)

Dự án được tổ chức theo chuẩn Modular Architecture để đảm bảo khả năng tái tạo 100%:

```text
├── .gitignore                   <- Chặn các file rác và data thô quá nặng.
├── README.md                    <- Tổng quan dự án và hướng dẫn chạy code.
├── data/                        
│   ├── raw/                     <- (Cần tự thêm) Chứa `train.csv` và `sample_submission.csv`.
│   └── processed/               <- Dữ liệu đã làm sạch (`full_time_series_ready_for_model.csv`, `sku_metadata.csv`).
├── notebooks/                   
│   ├── 01_data_cleaning.ipynb   <- Pipeline dọn rác, chuẩn hóa tiền tệ, xử lý hàng trả về và Time-Series Padding.
│   ├── 02_eda_insights.ipynb    <- Phân tích BCG, K-Means Sparsity, ACF/PACF, Macro Shocks.
│   └── 03_model_forecasting.ipynb <- Feature Engineering, Time-based Split, Train LightGBM & Hậu xử lý.
└── submissions/                 <- Chứa kết quả dự báo cuối cùng (VD: `submission_final.csv`).

--------------------------------------------------------------------------------
🚀 3. Hướng dẫn Cài đặt & Chạy Code (How to Run)
Để tái tạo lại toàn bộ kết quả của nhóm từ dữ liệu thô, vui lòng thực hiện theo thứ tự sau:
Chuẩn bị dữ liệu: Tải file train.csv và sample_submission.csv từ hệ thống Ban Tổ Chức, đặt vào thư mục data/raw/.
Bước 1 - Data Cleaning: Chạy file notebooks/01_data_cleaning.ipynb. File này sẽ ép kiểu dữ liệu an toàn, xử lý các anomalies, kẹp (clip) số lượng âm và lấp đầy lưới thời gian (padding) tạo ra ma trận full_time_series hơn 28 triệu dòng.
Bước 2 - Khai phá dữ liệu: Chạy file notebooks/02_eda_insights.ipynb. Hệ thống sẽ tự động quét và sinh ra bảng sku_metadata.csv chứa các cờ phân loại chiến lược (Is_EOL, Is_Cold_Start, BCG_Class).
Bước 3 - Huấn luyện & Dự báo: Chạy file notebooks/03_model_forecasting.ipynb. File này thực thi mô hình LightGBM theo phương pháp Lag-28 Direct Approach và đóng gói ra file nộp bài chuẩn tại submissions/submission_final.csv.

--------------------------------------------------------------------------------
🧠 4. Phương pháp luận & Insights Cốt lõi (Methodology & Key Insights)
Thay vì ném toàn bộ dữ liệu vào một mô hình Hộp đen (Black-box), team đã xây dựng các chiến lược dựa trên Data-driven Insights:
4.1. Phân loại chiến lược bằng Ma trận BCG & Profit Weights
Mô hình hóa đường cong Pareto cho thấy sự mất cân đối cực lớn về lợi nhuận. Nhóm sử dụng lợi nhuận ròng (LineProfit) làm Sample Weights trong quá trình huấn luyện, ép thuật toán LightGBM ưu tiên dồn tài nguyên để học chính xác nhóm Stars & Cash Cows.
4.2. Khống chế Rủi ro Trả hàng & Cú sốc Vĩ mô
Sử dụng IQR, team phát hiện ra 
 mã "Ngôi sao" đang chảy máu chi phí logistics do tỷ lệ trả hàng (Return Rate) vượt ngưỡng [53.46%].
Nhận diện cú sốc COVID-19 (Quý 3/2021) bẻ gãy đà tăng trưởng. Team đã tạo biến cờ Is_Macro_Shock để cô lập giai đoạn này, tránh làm mô hình học nhầm các đứt gãy trong quá khứ.
4.3. Định vị Lag Features bằng Lấy mẫu Phân tầng (Stratified Sampling)
Tránh rủi ro "thiên lệch chọn mẫu", team trích xuất "Hội đồng 5 mã đại diện" đa chiều (Top Profit, Top Volume, Sparse). Sau khi lấy sai phân (Differencing) để đảm bảo tính dừng (Stationarity), biểu đồ PACF xác nhận chu kỳ nhập hàng B2B theo tuần cực kỳ sắc nét.
=> Feature chốt: Lag_1, 2, 3 (Quán tính ngắn hạn) và Lag_7, 14, 21, 28 (Chu kỳ gom hàng).
4.4. Xử lý Dữ liệu Thưa thớt (Sparsity) bằng K-Means
Dùng thuật toán K-Means phát hiện ra ngay cả nhóm bán chạy cũng "trắng bảng" tới 81.24% thời gian.
Hành động: Sử dụng hàm mục tiêu Tweedie trong LightGBM để trị dứt điểm tình trạng over-forecasting cho dữ liệu Zero-Inflated.
4.5. Chiến lược Dự báo Đa tầng (Hybrid Forecasting)
Active SKUs: Dự báo bằng LightGBM.
End-of-Life (EOL) SKUs: Có tới 42.2% danh mục đã không bán được gì trong 274 ngày. Team dùng Rule-based (Heuristic) ép dự báo về 0 để tối ưu RAM và giảm sai số WRMSSE.
Cold Start SKUs: Với các tân binh chưa đủ lịch sử, team dùng chiến lược "vay mượn" trung vị của nhóm ngành hàng thay vì sử dụng Lags.

--------------------------------------------------------------------------------
🏆 5. Kết quả (Results)
Việc áp dụng chiến lược Hybrid và bảo vệ nghiêm ngặt chống Data Leakage (cắt tập Validation đúng 28 ngày mô phỏng đề thi) đã mang lại kết quả:
Public Leaderboard (Validation F1-F28): [Điền điểm số cao nhất của team vào đây, ví dụ: 0.452]
Private Leaderboard (Evaluation F29-F56): [Chờ kết quả cuối cùng từ BTC]

--------------------------------------------------------------------------------
👥 6. Đội ngũ (Team Members)
[Tên thành viên 1]: Data Cleaning, EDA & Business Insights.
[Tên thành viên 2]: Feature Engineering, Machine Learning Pipeline.
[Tên thành viên 3]: Rule-based Optimization & Model Tuning.
[Tên thành viên 4]: Validation Strategy & Post-processing.
