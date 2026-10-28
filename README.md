# Phân tích hành vi và nguy cơ rời bỏ người chơi Flood-It! bằng Survival Analysis

**Loại dự án:** Khóa luận tốt nghiệp / Data Science Portfolio  
**Đề tài:** Phân tích hành vi, thời gian duy trì và nguy cơ rời bỏ người chơi game di động bằng Survival Analysis  
**Nguồn dữ liệu:** Google Analytics 4 (GA4) – BigQuery public sample của game *Flood-It!*  
**Công cụ:** SQL, Python, Jupyter Notebook, Power BI

## 1. Tổng quan dự án

Dự án phân tích hành vi và thời gian duy trì của người chơi *Flood-It!* bằng dữ liệu sự kiện GA4. Quy trình gồm chuẩn bị dữ liệu, phân tích khám phá, ước lượng thời gian sống sót, xây dựng và đánh giá mô hình, sau đó tổng hợp kết quả thành các bảng phục vụ trực quan hóa trên Power BI.

Khác với bài toán phân loại churn nhị phân, Survival Analysis cho phép phân tích **thời điểm xảy ra sự kiện** và xử lý những trường hợp chưa quan sát được sự kiện khi kết thúc cửa sổ dữ liệu (*right-censoring*). Dashboard được dùng để hỗ trợ phân tích và định hướng điều tra.

## 2. Câu hỏi nghiên cứu

- **RQ1 – Thời gian duy trì:** Xác suất người chơi tiếp tục hoạt động thay đổi như thế nào theo thời gian?
- **RQ2 – Đặc điểm hành vi:** Những đặc điểm hành vi nào có liên quan đến nguy cơ rời bỏ?
- **RQ3 – Phân khúc tương tác:** Các nhóm người chơi có mức tương tác ban đầu khác nhau có mô hình sống sót khác nhau như thế nào?
- **RQ4 – So sánh mô hình:** Cox Proportional Hazards và Random Survival Forest (RSF) khác nhau như thế nào về khả năng phân biệt và sai số dự đoán?
- **RQ5 – Thời điểm rời bỏ:** Những giai đoạn nào có hazard rời bỏ cao hơn và nên được ưu tiên điều tra?

## 3. Dữ liệu

Dữ liệu bao phủ **114 ngày, từ 12/06/2018 đến 03/10/2018**. Nghiên cứu chọn mẫu ở cấp người dùng, xấp xỉ 15% số người dùng duy nhất, và giữ trọn chuỗi sự kiện của mỗi người dùng được chọn.

### Các file đang có trong thư mục `Datasets/`

Thư mục hiện tại có **15 file CSV**. Chúng được phân theo vai trò như sau.

**A. Năm bảng đầu vào dùng để chuẩn bị dữ liệu**

| File | Vai trò |
|---|---|
| `dim_user.csv` | Thông tin/định danh người dùng |
| `fact_event_daily.csv` | Số lượng sự kiện theo người dùng, ngày và loại sự kiện |
| `fact_level.csv` | Dữ liệu sự kiện liên quan đến level |
| `fact_screen.csv` | Dữ liệu sự kiện liên quan đến screen |
| `fact_economy.csv` | Dữ liệu sự kiện liên quan đến economy |

**B. Ba file trung gian phục vụ phân tích sống sót và mô hình**

| File | Vai trò |
|---|---|
| `group_a_survival.csv` | Dữ liệu sống sót Nhóm A, bao gồm xử lý delayed entry cho nhóm cài cũ |
| `group_b_landmark_survival.csv` | Dữ liệu mô hình Nhóm B sau Landmark 14 ngày |
| `feature_columns.csv` | Danh sách biến được phép sử dụng trong mô hình |

**C. Bảy bảng phụ trợ cho EDA, mô hình và Power BI**

| File | Vai trò |
|---|---|
| `event_param_profile.csv` | Bảng hồ sơ/thống kê tham số sự kiện |
| `level_difficulty.csv` | Bảng phục vụ phân tích level và độ khó |
| `dim_user_pbi.csv` | Bảng người dùng phục vụ mô hình dữ liệu/Power BI |
| `hazard_interval.csv` | Hazard theo khoảng thời gian, kèm thông tin person-time và khoảng tin cậy |
| `km_curve.csv` | Ước lượng Kaplan–Meier theo thời gian và các nhóm |
| `ml_cox_hr.csv` | Hazard Ratio và kết quả liên quan của các biến Cox dùng cho báo cáo |
| `ml_model_performance.csv` | Bảng tổng hợp hiệu năng mô hình |

> **Lưu ý:** Không phải cả 15 file đều là dữ liệu thô. Năm bảng ở nhóm A là đầu vào của notebook chuẩn bị dữ liệu; các bảng còn lại là dữ liệu đã xử lý hoặc bảng kết quả/phụ trợ cho phân tích và Dashboard.

Nguồn dữ liệu công khai: [Google Analytics – BigQuery sample dataset for a gaming app](https://developers.google.com/analytics/bigquery/app-gaming-demo-dataset).

## 4. Quy trình và phương pháp

1. **Chuẩn bị dữ liệu:** đọc năm bảng đầu vào, kiểm tra tính toàn vẹn giữa các bảng, xử lý bản ghi trùng và chuẩn hóa ngày/múi giờ.
2. **Định nghĩa churn:** người dùng được gán sự kiện rời bỏ theo định nghĩa vận hành khi không có hoạt động chủ động trong ít nhất **14 ngày liên tiếp**.
3. **Delayed entry (left-truncation):** xử lý người cài ứng dụng trước thời điểm bắt đầu cửa sổ quan sát, tránh giả định rằng toàn bộ thời gian từ lúc cài đặt đã được theo dõi.
4. **Landmark Analysis:** sử dụng hành vi trong ngày 0–13 để mô hình hóa kết cục sau mốc ngày 14, hạn chế sử dụng thông tin tương lai làm biến dự báo.
5. **Phân tích khám phá:** Kaplan–Meier, hazard theo person-time, so sánh mô tả theo nền tảng/quốc gia/thiết bị và phân khúc tương tác; hiệu chỉnh Holm khi thực hiện nhiều phép so sánh theo thiết kế nghiên cứu.
6. **Mô hình:** Cox Proportional Hazards với các bộ biến khác nhau và Random Survival Forest làm mô hình đối chiếu.
7. **Đánh giá:** repeated stratified 5-fold cross-validation, lặp 10 lần, tổng 50 fold; sử dụng C-index, AUC phụ thuộc thời gian, Brier Score/Integrated Brier Score (IBS) và calibration tại 30 ngày sau Landmark.
8. **Trực quan hóa:** xuất bảng kết quả để dùng trong các trang Power BI: Tổng quan, Retention & Survival, Gameplay & Level, và Phân khúc & ML.

## 5. Kết quả chính

### 5.1. Rời bỏ tập trung ở giai đoạn đầu

- **378/680 người (55,6%)** trong cohort cài mới đã xác minh có `T = 1`, nghĩa là chỉ ghi nhận hoạt động trong ngày lịch cài đặt theo cách định nghĩa thời gian của nghiên cứu.
- Ước lượng Kaplan–Meier: **S(14) = 20,6%** và **S(30) = 12,0%**.
- Hazard rate giảm mạnh sau ngày đầu. Đây là tỷ suất sự kiện trên person-time, không phải xác suất churn trong một ngày.

### 5.2. Delayed entry ảnh hưởng đến ước lượng ở nhóm cài cũ

Ở nhóm người đã cài ứng dụng trước khi bắt đầu cửa sổ quan sát, trung vị sống sót ước lượng là **22 ngày khi xử lý delayed entry**, so với **415 ngày nếu bỏ qua entry**. Đây là minh họa về sai lệch tiềm tàng khi xem người cài cũ như đã được theo dõi từ ngày cài đặt dù dữ liệu không bao phủ giai đoạn đó.

### 5.3. Cox M3 tốt nhất trong so sánh hiện tại, nhưng hiệu năng dự báo còn hạn chế

| Chỉ số | Kết quả |
|---|---:|
| C-index Cox M3 (trung bình ± SD qua 50 fold) | **0,6203 ± 0,0625** |
| AUC động Cox M3 tại 30 ngày (trung bình ± SD) | **0,727 ± 0,099** |
| IBS Cox M3 | **0,2064** |
| IBS mô hình Null Kaplan–Meier | **0,2268** |
| Mức giảm IBS tương đối so với Null | **9,0%** |
| C-index RSF với bộ biến M3 | **0,5805 ± 0,0703** |

Trong kết quả hiện tại, RSF chưa vượt Cox M3. C-index của Cox M3 cho thấy khả năng phân biệt rủi ro còn hạn chế. So sánh qua cross-validation mang tính mô tả.

### 5.4. Những biến có liên quan thống kê trong Cox M3

| Biến | HR | p-value | Cách diễn giải |
|---|---:|---:|---|
| `log2_active_days` | 0,591 | 0,013 | Mức hoạt động ban đầu cao hơn có liên quan với hazard thấp hơn, khi giữ các biến khác không đổi. |
| `log2_screens` | 1,584 | 0,003 | Số màn hình khác nhau có liên quan với hazard cao hơn; không chứng minh việc xem nhiều màn hình gây churn. |
| `india` | 2,254 | 0,009 | Biến chỉ báo India có hazard ước lượng cao hơn nhóm tham chiếu trong mô hình; cần diễn giải theo bối cảnh mẫu. |

Đây là các mối liên quan có điều kiện trong dữ liệu quan sát, không phải bằng chứng nhân quả. Việc lựa chọn biến và quy mô mẫu nhỏ giới hạn mức độ khái quát của kết luận.

### 5.5. Phân khúc tương tác không phải thang xếp hạng rủi ro mặc định

Tỷ lệ rời bỏ không tăng hoặc giảm đơn điệu theo mức tương tác. Tên phân khúc mô tả hành vi trong giai đoạn đầu, không nên mặc định là thứ hạng rủi ro churn.

