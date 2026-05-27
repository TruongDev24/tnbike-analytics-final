# Thống Nhất Bike — B2B Demand Forecasting & Dealer Analytics

Dự án phân tích dữ liệu B2B cho nhà phân phối xe đạp **Thống Nhất**, bao gồm dự báo doanh thu, phân tích xu hướng SKU/màu sắc, phân đoạn đại lý theo RFM, dự báo churn và tổng hợp insights bằng Gemini AI.

---

## Kết quả chính

| Chỉ số | Giá trị |
|--------|---------|
| Dữ liệu lịch sử | 17,031 giao dịch, 5 tháng (2025-01 → 2026-02) |
| Số đại lý | 702 |
| Số SKU | 247 |
| Tổng doanh thu (5 tháng) | 68.6 tỷ VND |
| Mô hình dự báo tốt nhất | Holt-Winters ETS — sMAPE 63.8%, MAE 2.47 tỷ |
| Dự báo Q2/2026 (T4–T6) | 25.6 → 29.6 → 33.5 tỷ ≈ **88.7 tỷ** tổng |
| ROC-AUC churn (LR) | 0.28 (dữ liệu B2B chu kỳ không đều) |
| BG-NBD CLV 90 ngày | TB 0.59 đơn/đại lý |
| Email đơn hàng tháng 3/2026 | 1,132 email → 40.8 tỷ VND |

---

## Cấu trúc dự án

```
├── data/
│   ├── raw/
│   │   ├── Database_6.zip          # SQL schema + dữ liệu lịch sử (9 bảng)
│   │   └── Emails_Files_5.zip      # 1,132 email đơn hàng tháng 3/2026
│   ├── fact_sales.csv              # Bảng fact chính (18,163 dòng, gồm email)
│   ├── order_line.csv              # Chi tiết dòng đơn hàng
│   ├── sales_order.csv             # Đơn hàng (2,759 đơn)
│   ├── email_log.csv               # Log parsing email (1,132 dòng)
│   └── sales_action_list_Q2.csv    # Top 20 đại lý ưu tiên chăm sóc Q2
├── notebooks/
│   ├── pipeline_clean.ipynb        # ETL: zip → SQLite → CSV
│   ├── 00_eda.ipynb                # Khám phá dữ liệu tổng quan
│   ├── 01_q1_sales_forecast.ipynb  # Dự báo doanh thu Q2/2026
│   ├── 02_q2_color_sku.ipynb       # Xu hướng màu sắc & SKU
│   ├── 03_q3_dealer_activity.ipynb # RFM + Churn + CLV đại lý
│   └── 04_llm_insights.ipynb       # Tổng hợp insights bằng Gemini AI
└── .gitignore
```

---

## Pipeline dữ liệu

```
Database_6.zip          Emails_Files_5.zip
      │                        │
      ▼                        ▼
  SQL schema              Parse 1,132 .eml
  + INSERT data           (3 định dạng regex)
      │                        │
      └──────────┬─────────────┘
                 ▼
           SQLite DB (tnbike.db)
                 │
     ┌───────────┴────────────┐
     ▼                        ▼
fact_sales.csv           order_line.csv
(JOIN đầy đủ:            sales_order.csv
 product_name,           email_log.csv
 color, line_name,
 group_code)
```

`pipeline_clean.ipynb` xử lý toàn bộ pipeline từ zip đến CSV. Chạy lại:

```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/pipeline_clean.ipynb
```

---

## Notebooks

### 00 — EDA (`00_eda.ipynb`)
Khám phá dữ liệu: doanh thu theo tháng, theo nhóm sản phẩm, phân phối đại lý, heatmap tỉnh thành.

### 01 — Dự báo doanh thu (`01_q1_sales_forecast.ipynb`)
So sánh 4 mô hình: Naive, Linear Regression, Holt-Winters ETS, SARIMA, LightGBM.

| Mô hình | sMAPE | MAE | RMSE |
|---------|-------|-----|------|
| Holt-Winters ETS | **63.8%** | 2.47 tỷ | 3.46 tỷ |
| Linear Regression | 88.9% | 6.56 tỷ | 7.86 tỷ |
| SARIMA(1,1,0) | 96.2% | 7.25 tỷ | 7.93 tỷ |
| LightGBM | 96.5% | 3.27 tỷ | 3.57 tỷ |

Dự báo ETS: T4/2026 = 25.6 tỷ, T5 = 29.6 tỷ, T6 = 33.5 tỷ.

### 02 — Màu sắc & SKU (`02_q2_color_sku.ipynb`)
Phân tích thị phần màu sắc theo tháng, dự báo nhu cầu màu, xu hướng tăng/giảm từng SKU.

### 03 — Hoạt động đại lý (`03_q3_dealer_activity.ipynb`)
- **RFM segmentation**: 702 đại lý → Champions / Loyal / Potential / New / At-Risk / Lost
- **Logistic Regression** dự báo churn (time-series split, không dùng random split)
- **BG-NBD + Gamma-Gamma** ước tính CLV 90 ngày (TB 0.59 đơn/đại lý)
- Xuất `sales_action_list_Q2.csv`: top 20 đại lý doanh thu cao + churn > 60%

### 04 — LLM Insights (`04_llm_insights.ipynb`)
Gọi Gemini API (`gemini-2.5-flash-lite`) để tổng hợp 5 nhóm insights actionable cho CEO từ kết quả các notebooks trước.

Thay Gemini key:
```bash
export GEMINI_API_KEY='your-key'
jupyter nbconvert --to notebook --execute --inplace notebooks/04_llm_insights.ipynb
```

---

## Cài đặt

```bash
pip install pandas numpy matplotlib scikit-learn statsmodels lightgbm lifetimes google-genai nbformat
```

Yêu cầu thêm `unrar-free` để giải nén email:
```bash
apt-get install unrar-free  # Ubuntu/Debian
```

---

## Ghi chú kỹ thuật

**Dữ liệu tháng 3/2026**: 1,132 email đơn hàng được parse từ file `.eml`, không có chi tiết SKU nên `product_code = 'UNKNOWN'`. Tháng này bị loại khỏi training data các model nhưng được ghi nhận trong `fact_sales.csv`.

**Tại sao sMAPE thay vì MAPE**: Doanh thu B2B có tháng gần bằng 0 (trước khi kênh đại lý kích hoạt), khiến MAPE phân kỳ. sMAPE scale-invariant và ổn định hơn.

**Churn ROC-AUC thấp (0.28)**: Đặc thù B2B — đại lý mua theo chu kỳ dài không đều, không giống B2C. Recency vẫn là feature quan trọng nhất nhưng tín hiệu yếu trên 5 tháng dữ liệu.
