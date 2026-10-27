# Tính năng chương trình tiền xử lý dữ liệu trên NPU



## 1. Cấu Trúc Lưới 8x8 Ô Cell & 3 Kênh Dữ Liệu

- **Khung hình đầu vào**: $256 \times 192$ RAW16.
- **Phân hoạch lưới**: $8 \times 8 = 64$ ô cell.
  - Mỗi ô cell có kích thước: **32 pixel (chiều ngang) $\times$ 24 pixel (chiều dọc)** ($768$ pixels/cell).
  - Tọa độ bao phủ của cell $[r, c]$:
    - $X \in [c \times 32 \dots (c+1) \times 32 - 1]$
    - $Y \in [r \times 24 \dots (r+1) \times 24 - 1]$
- **3 Tham số chính cho từng ô cell**:
  - **Kênh 0: $T_{\text{avg}}$** — Nhiệt độ trung bình của ô cell.
  - **Kênh 1: $T_{\text{max}}$** — Nhiệt độ đỉnh (cao nhất) trong ô cell.
  - **Kênh 2: $\Delta T = T_{\text{max}} - T_{\text{avg}}$** — Độ tương phản nhiệt cục bộ (phân biệt điểm cháy với môi trường nóng đồng đều).

---

## 2. Biểu đồ tổng quan

``` mermaid
flowchart TD
    A["Đầu vào RAW16<br/>[B, 1, 192, 256]<br/>giá trị biểu diễn bằng Float32"] --> B["Hiệu chuẩn nhiệt độ<br/>T = RAW / 64 - 273.15"]
    B --> C1["AvgPool 3×4<br/>192×256 → 64×64"]
    B --> C2["MaxPool 3×4<br/>192×256 → 64×64"]
    C1 --> D1["AvgPool 8×8<br/>64×64 → 8×8"]
    C2 --> D2["MaxPool 8×8<br/>64×64 → 8×8"]
    D1 --> E["T_avg<br/>[B, 1, 8, 8]"]
    D2 --> F["T_max<br/>[B, 1, 8, 8]"]
    E --> G["Delta_T = T_max − T_avg"]
    F --> G
    E --> H["Concat theo chiều channel"]
    F --> H
    G --> H
    H --> I["Đầu ra [B, 3, 8, 8]<br/>Ch0: T_avg · Ch1: T_max · Ch2: Delta_T"]
```

## 4. Biểu đồ các tầng và kích thước tensor

``` mermaid
flowchart LR
    A["RAW16<br/>1×192×256"] --> B["Hiệu chuẩn<br/>1×192×256"]
    B --> C["Pool tầng 1<br/>Avg / Max<br/>1×64×64"]
    C --> D["Pool tầng 2<br/>Avg / Max<br/>1×8×8"]
    D --> E["T_avg: 1×8×8"]
    D --> F["T_max: 1×8×8"]
    E --> G["Delta_T = T_max − T_avg"]
    F --> G
    E --> H["Concat"]
    F --> H
    G --> H
    H --> I["3×8×8<br/>64 ô × 3 đặc trưng"]
```

  ------------------------------------------------------------------------------------------
  Giai đoạn               Phép toán                                     Kích thước đầu ra 
  ----------------------- ------------------------------------------    -----------------------
  Đầu vào                 RAW16 được biểu diễn bằng `Float32`            `1×192×256`

  Hiệu chuẩn              `raw * (1/64) - 273.15`                        `1×192×256`

  Pool tầng 1             AvgPool và MaxPool, kernel/stride `3×4`        Mỗi nhánh: `1×64×64`

  Pool tầng 2             AvgPool và MaxPool, kernel/stride `8×8`        Mỗi nhánh: `1×8×8`

  Độ tương phản cục bộ    `T_max - T_avg`                               `1×8×8`

  Ghép kênh               `Concat([T_avg, T_max, Delta_T], dim=1)`      `3×8×8`
  ------------------------------------------------------------------------------------------


  ## 5. Biểu đồ phân nhánh đặc trưng nhiệt

``` mermaid
flowchart TD
    A["Ảnh nhiệt sau hiệu chuẩn"] --> B["Nhánh trung bình"]
    A --> C["Nhánh cực đại"]
    B --> D["T_avg: nhiệt độ trung bình mỗi ô"]
    C --> E["T_max: nhiệt độ cao nhất mỗi ô"]
    D --> F["Delta_T = T_max − T_avg"]
    E --> F
    D --> G["Bản đồ đặc trưng 8×8"]
    E --> G
    F --> G
    G --> H["Đầu vào cho tầng phân tích rủi ro / mô hình tiếp theo"]
```

`T_avg`, `T_max` và `Delta_T` là ba kênh đầu ra của ML1. Bản thân mã
nguồn này chưa có logic phân loại cháy, ngưỡng cảnh báo hoặc bộ phân
tích rủi ro.

## 6. Công thức và giới hạn diễn giải

-   Hiệu chuẩn: `T_C = RAW × 0.015625 − 273.15`.
-   Trung bình: `T_avg` được tính bằng hai tầng `AvgPool2d`.
-   Cực đại: `T_max` được tính bằng hai tầng `MaxPool2d`.
-   Độ chênh: `Delta_T = T_max − T_avg`.

**Giới hạn cần kiểm chứng trước khi triển khai:**

1.  Công thức RAW-to-Celsius là giả định hiệu chuẩn được ghi trong mã;
    cần xác nhận đúng định dạng dữ liệu và quy ước RAW của Tiny-1C.
2.  Mã hiện tại khai báo mô hình PyTorch và xuất ONNX. Việc các toán tử
    thực sự được gộp/fuse vào SRAM hay chạy trên NPU phải xác minh bằng
    log biên dịch và đo trên thiết bị.
3.  `dummy_input` dùng khi xuất ONNX là tensor ngẫu nhiên quanh giá trị
    19000; nên bổ sung kiểm thử bằng dữ liệu RAW16 thực tế và kiểm tra
    sai số sau biên dịch/quantization.
4.  Các sơ đồ này mô tả ML1 trích xuất đặc trưng, không chứng minh độ
    chính xác cảnh báo cháy hoặc thời gian suy luận.