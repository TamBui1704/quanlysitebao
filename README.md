# 📊 Hệ thống Phân tích Nhuận bút & Hiệu quả Tòa soạn — VnExpress

> **End-to-end Data Engineering pipeline** tự động hóa thu thập, xử lý và cung cấp dữ liệu nhuận bút, hiệu suất bài viết và traffic của toàn bộ hệ thống site thuộc báo VnExpress — sẵn sàng cho phân tích và báo cáo nghiệp vụ.

---

## 🎯 Mục tiêu dự án

Hệ thống được xây dựng nhằm giải quyết bài toán **quản lý chi phí nhuận bút** và **đánh giá hiệu quả làm việc** của đội ngũ Cộng tác viên (CTV) và Biên tập viên (BTV) trên quy mô lớn, đồng thời cung cấp góc nhìn đa chiều về **hiệu quả vận hành** từng site, từng mục nội dung dựa trên dữ liệu traffic thực tế.

| Vấn đề nghiệp vụ | Giải pháp |
|---|---|
| Theo dõi chi phí nhuận bút thủ công, dễ sai sót | Tự động hóa ETL từ hệ thống chấm nhuận → DWH |
| Khó đánh giá năng suất CTV/BTV theo thời gian | Dashboard phân tích số bài viết, giá trị nhuận bút theo kỳ |
| Thiếu liên kết giữa chi phí và hiệu quả traffic | Kết hợp dữ liệu nhuận bút + traffic trên cùng semantic model |
| Báo cáo rời rạc, tốn thời gian tổng hợp | Một nguồn dữ liệu duy nhất (Single Source of Truth) trên DWH |

---

## 🏗️ Kiến trúc hệ thống

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                             │
│                                                                 │
│   ┌──────────────────────────────────────────────────────┐      │
│   │  SQL Server — Hệ thống Chấm Nhuận bút (OLTP)         │      │
│   │  · Tiền nhuận bút CTV / BTV                          │      │
│   │  · Số lượng bài viết theo mục / site                 │      │
│   │  · Traffic các site thuộc VnExpress                  │      │
│   └──────────────────────────────────────────────────────┘      │
└────────────────────────────┬────────────────────────────────────┘
                             │  Extract
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STAGING LAYER (SQL Server)                     │
│   · Raw data landing zone                                        │
│   · Schema bám sát cấu trúc nguồn                               │
│   · Hỗ trợ incremental load & full load                         │
└────────────────────────────┬────────────────────────────────────┘
                             │  Transform & Load
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│               DATA WAREHOUSE (SQL Server — Snowflake Schema)     │
│                                                                  │
│   Fact Tables                  Dimension Tables                  │
│   · fact_nhuan_but            · dim_nhan_su  (CTV / BTV)        │
│   · fact_bai_viet             · dim_site                        │
│   · fact_traffic              · dim_muc                         │
│                               · dim_date                        │
└────────────────────────────┬────────────────────────────────────┘
                             │  Semantic Modeling
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              SEMANTIC LAYER — SSAS Tabular Model                 │
│   · Measures: Tổng nhuận bút, Chi phí/bài, Traffic/mục...      │
│   · KPIs, Time Intelligence (MTD / YTD / MoM)                  │
│   · Row-level Security theo phân quyền                          │
└─────────────────────────────────────────────────────────────────┘
         (BI Tools kết nối trực tiếp SSAS qua Live Connection)
```

---

## 🔧 Công nghệ sử dụng

| Layer | Công nghệ | Vai trò |
|---|---|---|
| **Orchestration & ETL** | SQL Server Integration Services (SSIS) | Điều phối toàn bộ pipeline Extract → Transform → Load |
| **Data Warehouse** | SQL Server | Lưu trữ dữ liệu lịch sử theo mô hình Snowflake Schema (Bông tuyết) |
| **Semantic Layer** | SQL Server Analysis Services (SSAS) — Tabular | Xây dựng business model, DAX measures, KPIs |

---

## 📦 Cấu trúc dự án

```
quanlysitebao/
├── KhoaLuan/                  # Project SSIS (ETL & DWH)
│   ├── KhoaLuan.sln           # Solution Visual Studio
│   ├── ETL_DWH.dtproj         # Project Integration Services
│   ├── *.dtsx                 # Các package SSIS (Extract, Transform, Load)
│   ├── *.conmgr               # Nguồn kết nối (Connection Managers)
│   └── Project.params         # Tham số dự án
│
├── KhoaLuan_Tabular/          # Project SSAS (Semantic Layer)
│   ├── KhoaLuan_Tabular.sln   # Solution Visual Studio
│   ├── KhoaLuan_Tabular.smproj # Project Analysis Services Tabular
│   └── Model.bim              # File định nghĩa Tabular Model (Tables, Measures, Roles)
```

---

## 🔄 Chi tiết Pipeline ETL (SSIS)

### Luồng chính

```
[Nguồn: SQL Server OLTP]
        │
        ▼  Extract (Full / Incremental)
[Staging DB]  ←── Control Flow: Error Handling, Logging
        │
        ▼  Transform (Stored Procedures)
        │   · Chuẩn hóa tên nhân sự, mã mục
        │   · Tính toán nhuận bút quy đổi
        │   · Ánh xạ site/mục theo hierarchy VnExpress
        │
        ▼  Load (SCD Type 2 cho Dimensions)
[DWH — Snowflake Schema]
```

### Các SSIS Package chính

| Package | Mô tả | Tần suất |
|---|---|---|
| `Dim *.dtsx` | Các package Extract & Load Dimensions (Channel, Employee, Job, Post Type, Ranking...) | Hàng ngày (qua SQL Server Agent Job) |
| `Fact Royalty.dtsx` | Transform & Load fact chi phí nhuận bút vào DWH | Hàng ngày (qua SQL Server Agent Job) |
| `Fact View Channel.dtsx` | Transform & Load fact traffic theo kênh/site | Hàng ngày (qua SQL Server Agent Job) |
| `Fact View Page.dtsx` | Transform & Load fact traffic theo bài viết/trang | Hàng ngày (qua SQL Server Agent Job) |
| `MasterPackage.dtsx` | Orchestrate toàn bộ pipeline ETL (chạy tuần tự `Dim` → `Fact`) | Chạy định kỳ hàng ngày bằng Agent Job |
---

## 📐 Mô hình dữ liệu (Galaxy Schema - Chòm sao sự kiện)

Dựa trên cấu trúc Tabular Model thực tế:

```text
[ Fact Tables ]                  [ Dimension Tables ]
Fact Royalty     ───────(*:1)──  Dim Date
                 ───────(*:1)──  Dim Channel
                 ───────(*:1)──  Dim Employee Detail
                 ───────(*:1)──  Dim Job
                 ───────(*:1)──  Dim Post Detail
                 ───────(*:1)──  Dim Post Type
                 ───────(*:1)──  Dim Ranking
                 ───────(*:1)──  Dim Royalty Status

Fact View Channel───────(*:1)──  Dim Channel
                 ───────(*:1)──  Dim Date

Fact View Page   ───────(*:1)──  Dim Date

[ Phân quyền - Row Level Security ]
· User Channel Permissions
· Users
```

### Key Measures (DAX — SSAS Tabular)

```dax
-- Tổng số lượng bài viết đã chấm nhuận
Post Code Count := DISTINCTCOUNT('Fact Royalty'[Post Key])

-- Tổng chi phí nhuận bút (chưa bao gồm thuế)
Total Royalties Excluding Tax := SUM('Fact Royalty'[Royalty Excluding Tax ( VNĐ)])

-- Tổng lượt xem theo Channel
Total View Channel := SUM('Fact View Channel'[Total View])

-- Tổng lượt xem theo Page/Bài viết
Total View Page := SUM('Fact View Page'[Total View])

-- Tăng trưởng nhuận bút so với tháng trước (MoM)
MoM Royalty % :=
    VAR PrevMonth = CALCULATE([Total Royalties Excluding Tax],
                    DATEADD('Dim Date'[Date], -1, MONTH))
    RETURN DIVIDE([Total Royalties Excluding Tax] - PrevMonth, PrevMonth)
```

---

---



## 📈 Giá trị mang lại

| Chỉ số | Trước | Sau |
|---|---|---|
| Thời gian tổng hợp báo cáo nhuận bút | ~2 ngày/tháng (thủ công) | Tự động, cập nhật hàng ngày |
| Độ chính xác dữ liệu | Phụ thuộc thao tác thủ công | Xác thực tự động trong pipeline |
| Khả năng phân tích đa chiều | Hạn chế (Excel pivot) | Đầy đủ trên SSAS Tabular — sẵn sàng kết nối BI tool bất kỳ |
| Liên kết chi phí ↔ hiệu quả | Không có | Semantic model tích hợp nhuận bút + traffic |

---

## 👨‍💻 Tác giả

**Tâm Bùi **
Data Engineer

> Dự án này thể hiện năng lực thiết kế và triển khai end-to-end Data Engineering pipeline trong môi trường doanh nghiệp thực tế, bao gồm toàn bộ stack: **ETL (SSIS)** → **Data Warehouse (SQL Server — Snowflake Schema)** → **Semantic Layer (SSAS Tabular / DAX)**.

---

*© VnExpress — Internal Data Platform*
