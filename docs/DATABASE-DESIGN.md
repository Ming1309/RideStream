# DATABASE DESIGN — RideStream

> **Hệ thống:** RideStream — Real-time CDC & Analytics Platform  
> **Loại CSDL:** PostgreSQL (OLTP + PostGIS)  
> **Phiên bản tài liệu:** 1.0  
> **Cập nhật lần cuối:** 2026-03-30

---

## Tổng quan

Cơ sở dữ liệu PostgreSQL của RideStream gồm hai nhóm bảng chính:

| Nhóm                     | Mục đích                                                 | Bảng                                                                                          |
| :----------------------- | :------------------------------------------------------- | :-------------------------------------------------------------------------------------------- |
| **OLTP (Vận hành)**      | Lưu trữ dữ liệu giao dịch thô, là nguồn cho Debezium CDC | `users`, `drivers`, `vehicles`, `rides`, `rides_status_log`, `payments`                       |
| **Processed (Đã xử lý)** | Dữ liệu Spark đã xử lý, dùng cho Backend API và NOTIFY   | `live_district_metrics`, `live_system_kpis`, `live_weather_snapshot`                          |

### Sơ đồ thiết kế 

![[RideStream-db-design.drawio.png]]

---

## 1. Bảng `users` — Thông tin Hành khách

**Ý nghĩa:** Lưu trữ thông tin các hành khách đăng ký sử dụng dịch vụ gọi xe.

| Tên trường     | Kiểu dữ liệu   | Ràng buộc               | Ý nghĩa                                                                        |
| :------------- | :------------- | :---------------------- | :----------------------------------------------------------------------------- |
| `user_id`      | `VARCHAR(20)`  | **PK**, NOT NULL        | Mã định danh duy nhất của hành khách. Định dạng: `USR-{số}` (VD: `USR-10945`). |
| `full_name`    | `VARCHAR(100)` | NOT NULL                | Họ và tên đầy đủ của hành khách.                                               |
| `phone_number` | `VARCHAR(15)`  | NOT NULL, UNIQUE        | Số điện thoại đăng ký tài khoản. Dùng để xác định danh tính và liên lạc.       |
| `created_at`   | `TIMESTAMPTZ`  | NOT NULL, DEFAULT NOW() | Thời điểm hành khách đăng ký tài khoản.                                        |

---

## 2. Bảng `drivers` — Thông tin Tài xế

**Ý nghĩa:** Lưu trữ hồ sơ các tài xế đối tác đăng ký trên nền tảng.

| Tên trường        | Kiểu dữ liệu   | Ràng buộc                       | Ý nghĩa                                                                             |
| :---------------- | :------------- | :------------------------------ | :---------------------------------------------------------------------------------- |
| `driver_id`       | `VARCHAR(20)`  | **PK**, NOT NULL                | Mã định danh duy nhất của tài xế. Định dạng: `DRV-{số}` (VD: `DRV-5542`).           |
| `full_name`       | `VARCHAR(100)` | NOT NULL                        | Họ và tên đầy đủ của tài xế.                                                        |
| `phone_number`    | `VARCHAR(15)`  | NOT NULL, UNIQUE                | Số điện thoại liên lạc của tài xế.                                                  |
| `driving_license` | `VARCHAR(20)`  | NOT NULL, UNIQUE                | Số bằng lái xe hợp lệ.                                                              |
| `rating`          | `NUMERIC(3,2)` | DEFAULT 5.00, CHECK (0.00–5.00) | Điểm đánh giá trung bình của tài xế từ hành khách (thang 5 sao).                    |
| `status`          | `VARCHAR(20)`  | NOT NULL                        | Trạng thái hoạt động hiện tại của tài xế. Giá trị: `online` \| `offline` \| `busy`. |
| `created_at`      | `TIMESTAMPTZ`  | NOT NULL, DEFAULT NOW()         | Thời điểm tài xế đăng ký vào nền tảng.                                              |

---

## 3. Bảng `vehicles` — Thông tin Phương tiện

**Ý nghĩa:** Lưu trữ thông tin phương tiện mà mỗi tài xế đăng ký để hoạt động trên nền tảng. Mỗi tài xế có thể đăng ký một hoặc nhiều phương tiện.

| Tên trường      | Kiểu dữ liệu   | Ràng buộc                              | Ý nghĩa                                                                                  |
| :-------------- | :------------- | :------------------------------------- | :--------------------------------------------------------------------------------------- |
| `vehicle_id`    | `SERIAL`       | **PK**, NOT NULL                       | Mã định danh tự tăng duy nhất của phương tiện.                                           |
| `driver_id`     | `VARCHAR(20)`  | **FK** → `drivers.driver_id`, NOT NULL | Liên kết phương tiện với tài xế sở hữu.                                                  |
| `type`          | `VARCHAR(20)`  | NOT NULL                               | Loại phương tiện. Giá trị: `Bike` \| `Car` . Quyết định loại dịch vụ tài xế có thể nhận. |
| `license_plate` | `VARCHAR(15)`  | NOT NULL, UNIQUE                       | Biển số xe đã đăng ký. Là định danh pháp lý của phương tiện.                             |
| `brand_model`   | `VARCHAR(100)` | NOT NULL                               | Hãng xe và tên dòng xe (VD: `Honda Wave Alpha 110`).                                     |
| `color`         | `VARCHAR(30)`  | NOT NULL                               | Màu sắc của phương tiện. Hỗ trợ hành khách nhận diện.                                    |

---

## 4. Bảng `rides` — Thông tin Cuốc xe 

**Ý nghĩa:** Bảng trung tâm của hệ thống OLTP. Mỗi dòng đại diện cho một yêu cầu đặt xe, theo dõi toàn bộ vòng đời từ lúc yêu cầu đến khi hoàn thành hoặc hủy. Đây là nguồn dữ liệu chính của Debezium CDC.

| Tên trường                 | Kiểu dữ liệu            | Ràng buộc                                | Ý nghĩa                                                                                                                                                      |
| :------------------------- | :---------------------- | :--------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ride_id`                  | `VARCHAR(30)`           | **PK**, NOT NULL                         | Mã định danh duy nhất của cuốc xe. Định dạng: `RIDE-{UUID ngắn}` (VD: `RIDE-8f72a1b9`).                                                                      |
| `user_id`                  | `VARCHAR(20)`           | **FK** → `users.user_id`, NOT NULL       | Liên kết cuốc xe với hành khách đặt chuyến.                                                                                                                  |
| `driver_id`                | `VARCHAR(20)`           | **FK** → `drivers.driver_id`, NULLABLE   | Liên kết với tài xế nhận cuốc. `NULL` khi cuốc chưa có tài xế nhận (`requested`).                                                                            |
| `vehicle_id`               | `INTEGER`               | **FK** → `vehicles.vehicle_id`, NULLABLE | Phương tiện được sử dụng trong cuốc xe. `NULL` khi chưa có tài xế nhận.                                                                                      |
| `service_type`             | `VARCHAR(20)`           | NOT NULL                                 | Loại dịch vụ được sử dụng. Giá trị: `RideBike` \| `RideCar` \| `RideDelivery`.                                                                               |
| `pickup_location`          | `GEOMETRY(Point, 4326)` | NOT NULL                                 | Tọa độ địa lý điểm đón hành khách (PostGIS). Lưu dưới dạng `POINT(lon lat)`.                                                                                 |
| `dropoff_location`         | `GEOMETRY(Point, 4326)` | NOT NULL                                 | Tọa độ địa lý điểm trả hành khách (PostGIS).                                                                                                                 |
| `pickup_time`              | `TIMESTAMPTZ`           | NULLABLE                                 | Thời điểm thực tế tài xế đón hành khách. `NULL` khi cuốc chưa bắt đầu.                                                                                       |
| `dropoff_time`             | `TIMESTAMPTZ`           | NULLABLE                                 | Thời điểm thực tế tài xế trả hành khách. `NULL` khi cuốc chưa hoàn thành.                                                                                    |
| `estimated_distance_km`    | `NUMERIC(6,2)`          | NULLABLE                                 | Quãng đường ước tính (km) được tính toán khi cuốc được tạo.                                                                                                  |
| fare_amount                | `NUMERIC(12,2)`         | NULLABLE                                 | Cước phí của cuốc xe có thể bao gồm hệ số surge. `NULL` khi chưa hoàn thành.                                                                                 |
| `surge_multiplier_applied` | `NUMERIC(4,2)`          | NULLABLE                                 | Hệ số nhân giá surge áp dụng cho cuốc xe                                                                                                                     |
| `status`                   | `VARCHAR(20)`           | NOT NULL                                 | Trạng thái hiện tại của cuốc xe. Giá trị: `requested` \| `accepted` \| `ongoing` \| `completed` \| `cancelled`. Là trường kích hoạt Debezium CDC khi UPDATE. |
| `created_at`               | `TIMESTAMPTZ`           | NOT NULL, DEFAULT NOW()                  | Thời điểm hành khách tạo yêu cầu đặt xe.                                                                                                                     |
| `updated_at`               | `TIMESTAMPTZ`           | NOT NULL, DEFAULT NOW()                  | Thời điểm cuốc xe được cập nhật lần cuối (trigger tự động). Dùng để đo độ trễ pipeline (SLA Monitoring).                                                     |

**Vòng đời trạng thái (`status`):**

```
requested ──→ accepted ──→ ongoing ──→ completed
     │                                     
     └──────────────────→ cancelled
```

---

## 5. Bảng `rides_status_log` — Lịch sử thay đổi trạng thái cuốc xe

**Ý nghĩa:** Ghi lại lịch sử toàn bộ lần thay đổi trạng thái của mỗi cuốc xe. Phục vụ phân tích thời gian chờ (ETA), tỷ lệ chuyển đổi và phát hiện bất thường.

| Tên trường | Kiểu dữ liệu | Ràng buộc | Ý nghĩa |
| :--- | :--- | :--- | :--- |
| `log_id` | `SERIAL` | **PK**, NOT NULL | Mã định danh tự tăng của bản ghi lịch sử. |
| `ride_id` | `VARCHAR(30)` | **FK** → `rides.ride_id`, NOT NULL | Cuốc xe tương ứng với lần thay đổi trạng thái này. |
| `old_status` | `VARCHAR(20)` | NULLABLE | Trạng thái cũ trước khi thay đổi. `NULL` nếu là bản ghi đầu tiên (`requested`). |
| `new_status` | `VARCHAR(20)` | NOT NULL | Trạng thái mới sau khi thay đổi. |
| `change_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT NOW() | Thời điểm chính xác khi trạng thái thay đổi. Dùng để tính khoảng thời gian giữa các bước (`requested` → `accepted`, v.v.). |

---

## 6. Bảng `payments` — Thông tin Thanh toán

**Ý nghĩa:** Ghi lại thông tin giao dịch thanh toán cho mỗi cuốc xe đã hoàn thành. Quan hệ 1:1 với bảng `rides`.

| Tên trường | Kiểu dữ liệu | Ràng buộc | Ý nghĩa |
| :--- | :--- | :--- | :--- |
| `payment_id` | `SERIAL` | **PK**, NOT NULL | Mã định danh tự tăng duy nhất của giao dịch thanh toán. |
| `ride_id` | `VARCHAR(30)` | **FK** → `rides.ride_id`, NOT NULL, UNIQUE | Cuốc xe được thanh toán. UNIQUE đảm bảo mỗi cuốc chỉ có một bản ghi thanh toán. |
| `payment_method` | `VARCHAR(30)` | NOT NULL | Phương thức thanh toán. Giá trị: `cash` \| `momo` \| `zalopay` \| `vnpay` \| `credit_card`. |
| `amount` | `NUMERIC(12,2)` | NOT NULL | Số tiền thực tế được thanh toán (VNĐ). Phải khớp với `final_cost` ở bảng `rides`. |
| `payment_status` | `VARCHAR(20)` | NOT NULL | Trạng thái của giao dịch thanh toán. Giá trị: `pending` \| `success` \| `failed` \| `refunded`. |
| `transaction_time` | `TIMESTAMPTZ` | NOT NULL, DEFAULT NOW() | Thời điểm giao dịch thanh toán được thực hiện. |

---

## 7. Bảng `live_district_metrics` — Chỉ số Thời gian thực theo Quận

**Ý nghĩa:** Bảng Processed — do Spark Streaming ghi sau khi xử lý. Lưu trữ các chỉ số tổng hợp theo quận/khu vực, phục vụ hiển thị bản đồ nhiệt (heatmap) và cảnh báo Surge Pricing trên Dashboard. Được cập nhật theo chu kỳ (mỗi 1–5 phút). Khi INSERT/UPDATE, trigger sẽ kích hoạt `pg_notify` gửi đến Backend.

| Tên trường          | Kiểu dữ liệu   | Ràng buộc              | Ý nghĩa                                                                                                                                                                |
| :------------------ | :------------- | :--------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `district_id`       | `VARCHAR(10)`  | **PK**, NOT NULL       | Mã định danh của quận/huyện. VD: `Q1`, `Q3`, `THD` (Thủ Đức).                                                                                                          |
| `district_name`     | `VARCHAR(100)` | NOT NULL               | Tên đầy đủ của quận/huyện (VD: `Quận 1`, `Thành phố Thủ Đức`).                                                                                                         |
| `demand_count`      | `INTEGER`      | NOT NULL, DEFAULT 0    | Số lượng yêu cầu đặt xe (`requested`) đang chờ trong khu vực tại thời điểm cập nhật.                                                                                   |
| `supply_count`      | `INTEGER`      | NOT NULL, DEFAULT 0    | Số lượng tài xế đang trực tuyến (`online`) và sẵn sàng trong khu vực.                                                                                                  |
| `surge_multiplier`  | `NUMERIC(4,2)` | NOT NULL, DEFAULT 1.00 | Hệ số nhân giá surge hiện tại áp dụng cho khu vực. `1.00` = không surge; `1.5` = giá tăng 50%. Được tính từ tỷ lệ `demand_count / supply_count` × điều kiện thời tiết. |
| `weather_condition` | `VARCHAR(30)`  | NULLABLE               | Điều kiện thời tiết hiện tại tại khu vực (join từ luồng thời tiết). VD: `Rain`, `Clear`, `Clouds`.                                                                     |
| `updated_at`        | `TIMESTAMPTZ`  | NOT NULL               | Thời điểm Spark cập nhật bản ghi này. Dùng để kiểm soát freshness của dữ liệu.                                                                                         |

---

## 8. Bảng `live_system_kpis` — Chỉ số KPI Toàn hệ thống

**Ý nghĩa:** Bảng Processed — do Spark Streaming ghi sau khi xử lý. Lưu trữ các chỉ số KPI tổng hợp của toàn bộ nền tảng tại một thời điểm nhất định. Phục vụ phần "Overview" trên Dashboard vận hành. Thường chỉ có một dòng dữ liệu được cập nhật liên tục (dạng singleton), hoặc lưu theo snapshot thời gian.

| Tên trường | Kiểu dữ liệu | Ràng buộc | Ý nghĩa |
| :--- | :--- | :--- | :--- |
| `id` | `SERIAL` | **PK**, NOT NULL | Mã định danh của bản ghi KPI (hoặc snapshot ID). |
| `total_ongoing_rides` | `INTEGER` | NOT NULL, DEFAULT 0 | Tổng số cuốc xe đang trong trạng thái `ongoing` (đang di chuyển) trên toàn hệ thống. |
| `total_active_drivers` | `INTEGER` | NOT NULL, DEFAULT 0 | Tổng số tài xế đang trực tuyến (`online` hoặc `busy`) trên toàn hệ thống. |
| `avg_wait_time_seconds` | `NUMERIC(8,2)` | NULLABLE | Thời gian chờ trung bình (giây) từ lúc hành khách đặt xe (`requested`) đến khi tài xế nhận cuốc (`accepted`) trong cửa sổ thời gian gần nhất. |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | Thời điểm Spark cập nhật bản ghi KPI này. |

---

## 9. Bảng `live_weather_snapshot` — Snapshot Thời tiết Thời gian thực

**Ý nghĩa:** Bảng Processed — do Spark Streaming ghi sau khi xử lý luồng thời tiết từ Kafka (nguồn: Airflow → OpenWeatherMap API). Lưu mỗi snapshot thời tiết mới nhất nhận được, phục vụ Join với luồng cuốc xe để tính Surge Pricing và hiển thị điều kiện thời tiết hiện tại lên Dashboard. Không thay thế `dim_weather` ở DWH — đây là bảng **operational** phục vụ real-time.

| Tên trường            | Kiểu dữ liệu   | Ràng buộc               | Ý nghĩa                                                                                                                                 |
| :-------------------- | :------------- | :---------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- |
| `snapshot_id`         | `SERIAL`       | **PK**, NOT NULL        | Mã định danh tự tăng của snapshot thời tiết.                                                                                            |
| `weather_condition`   | `VARCHAR(30)`  | NOT NULL                | Trạng thái thời tiết chính. Giá trị: `Clear` \| `Clouds` \| `Rain` \| `Thunderstorm` \| `Drizzle` \| `Haze`.                            |
| `temperature_celsius` | `NUMERIC(5,2)` | NOT NULL                | Nhiệt độ thực đo (°C) tại thời điểm snapshot.                                                                                           |
| `rain_volume_1h`      | `NUMERIC(6,2)` | NULLABLE                | Lượng mưa tích lũy trong 1 giờ qua (mm). `NULL` nếu không có mưa. Ngưỡng > 20mm kích hoạt Surge Pricing Alert.                         |
| `updated_at`          | `TIMESTAMPTZ`  | NOT NULL, DEFAULT NOW() | Thời điểm Spark ghi snapshot này — tương ứng với timestamp trong payload API. Dùng để Watermark join với luồng cuốc xe.                  |

**Ghi chú vận hành:**
- Bảng này được Spark INSERT thêm dòng mới mỗi 10 phút (theo chu kỳ DAG Airflow).
- Backend/Spark đọc dòng mới nhất (`ORDER BY updated_at DESC LIMIT 1`) để lấy điều kiện thời tiết hiện tại.
- Khi INSERT, trigger `pg_notify('weather_update', ...)` sẽ thông báo cho Backend đẩy update xuống Frontend.

---

## 10. Quan hệ giữa các bảng (Relationships)

| From | Cardinality | To | Mô tả |
| :--- | :---: | :--- | :--- |
| `users` | 1 : N | `rides` | Một hành khách có thể đặt nhiều cuốc xe |
| `drivers` | 1 : N | `rides` | Một tài xế có thể nhận nhiều cuốc xe |
| `drivers` | 1 : N | `vehicles` | Một tài xế có thể đăng ký nhiều phương tiện |
| `vehicles` | 1 : N | `rides` | Một phương tiện có thể được dùng cho nhiều cuốc xe |
| `rides` | 1 : N | `rides_status_log` | Một cuốc xe có nhiều bản ghi lịch sử trạng thái |
| `rides` | 1 : 1 | `payments` | Mỗi cuốc xe (hoàn thành) có đúng một bản ghi thanh toán |
| `live_weather_snapshot` | Độc lập | *(Spark join)* | Không có FK trực tiếp; Spark join theo `updated_at` Watermark với luồng cuốc xe |

---

## 11. Chiến lược Indexing

| Bảng | Cột được Index | Loại Index | Lý do |
| :--- | :--- | :--- | :--- |
| `rides` | `status` | B-tree | Truy vấn thường xuyên filter theo trạng thái |
| `rides` | `created_at` | B-tree | Truy vấn theo khoảng thời gian (time-range queries) |
| `rides` | `user_id`, `driver_id` | B-tree | JOIN và lookup theo khóa ngoại |
| `rides` | `pickup_location`, `dropoff_location` | GiST (PostGIS) | Không gian — xác định cuốc xe theo quận, tìm hotspot |
| `rides_status_log` | `ride_id`, `change_at` | B-tree | Lịch sử trạng thái theo cuốc xe và thời gian |
| `live_district_metrics` | `updated_at` | B-tree | Kiểm tra freshness dữ liệu |
| `live_weather_snapshot` | `updated_at` | B-tree | Lấy snapshot mới nhất và Watermark join |

---

## 12. PostgreSQL Triggers & NOTIFY Channels

Khi Spark ghi dữ liệu vào các bảng **Processed**, các trigger PostgreSQL tự động gửi sự kiện qua `pg_notify` để Backend Node.js lắng nghe và đẩy xuống Frontend qua Socket.io.

| Bảng kích hoạt          | Sự kiện               | NOTIFY Channel        | Backend Event         | Mô tả                          |
| :---------------------- | :-------------------- | :-------------------- | :-------------------- | :----------------------------- |
| `rides`                 | INSERT                | `new_ride`            | `ride:new`            | Cuốc xe mới được tạo           |
| `rides`                 | UPDATE (cột `status`) | `ride_status_changed` | `ride:status_changed` | Trạng thái cuốc xe thay đổi    |
| `live_district_metrics` | INSERT / UPDATE       | `surge_alert`         | `alert:surge`         | Cập nhật hệ số surge theo quận |
| `live_system_kpis`      | INSERT / UPDATE       | `kpi_update`          | `kpi:update`          | Cập nhật KPI toàn hệ thống     |
| `live_weather_snapshot` | INSERT                | `weather_update`      | `weather:update`      | Snapshot thời tiết mới từ Spark |

---

## 13. Ghi chú về Kiểu dữ liệu địa lý (PostGIS)

Các trường tọa độ trong bảng `rides` sử dụng kiểu `GEOMETRY` của extension **PostGIS**:

```sql
-- Cài đặt extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Ví dụ tạo cột tọa độ
pickup_location  GEOMETRY(Point, 4326),  -- SRID 4326 = WGS84 (GPS)
dropoff_location GEOMETRY(Point, 4326)
```

- **SRID 4326** là hệ tọa độ WGS84 — chuẩn GPS quốc tế.
- Cho phép dùng các hàm không gian như `ST_Within()`, `ST_Distance()`, `ST_Contains()` để xác định cuốc xe thuộc quận nào, đo khoảng cách thực tế, v.v.

