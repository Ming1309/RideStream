## A. Nguồn dữ liệu (Data Sources)

Dữ liệu đầu vào của hệ thống không chỉ là các bản ghi đơn giản, mà được thiết kế để mô phỏng sự tương tác phức tạp (Correlation) giữa môi trường, thời gian và hành vi con người. Dữ liệu kết hợp 3 đặc tính cốt lõi trong Big Data: Streaming (Tốc độ cao), Micro-batch (Định kỳ) và Static (Tĩnh).

### 1. Luồng sự kiện vận hành & Giao thông (Ride Events & Traffic Telemetry)

Đây là nguồn dữ liệu cốt lõi quan trọng nhất, được sinh ra từ hệ thống giả lập Backend bằng PostgreSQL. Điểm đặc biệt là hệ thống không chỉ lưu lại sự kiện giao dịch mà còn sinh ra dữ liệu viễn thám học (Telemetry).

- **Bản chất:** Dữ liệu giao dịch (High-throughput Transactional) và Dữ liệu chuỗi thời gian liên tục (High-frequency Telemetry).
- **Cách thức tạo dữ liệu:** Một script Python Data Generator sử dụng thư viện `Faker`. Script liên tục thực hiện lệnh `INSERT/UPDATE` giả lập toàn bộ vòng đời cuốc xe.
- **Logic giả lập hành vi thực tế (Traffic Logic):** 
  - Vận tốc di chuyển của tài xế được lập trình biến thiên dựa trên Tọa độ địa lý và Khung giờ hệ thống. 
  - *Ví dụ:* Script sẽ tự động sinh vận tốc thấp (< 15km/h) cho các cuốc xe có tọa độ ở trọng tâm Quận 1 vào khung giờ 07:30 - 09:00, và sinh vận tốc cao (40-50km/h) cho vùng ngoại thành hoặc giờ thấp điểm.
- **Mục tiêu:** Giả lập trực tiếp dữ liệu GPS để stream processing engine (như Spark) tính toán Chỉ số kẹt xe (Real-time Congestion Index) và đo lường tham số trễ (Latency).

### 2. Dữ liệu bối cảnh môi trường (External Weather API)

Trong nghiệp vụ gọi xe, thời tiết tác động trực tiếp lên hệ số giá (Surge Pricing) và Nhu cầu (Demand).

- **Bản chất:** Dữ liệu kéo định kỳ (Polling / Micro-batch Data).
- **Nguồn cung cấp:** API từ **OpenWeatherMap**.
- **Cách thức thu thập:** Apache Airflow kích hoạt vòng lặp (DAGs) tự động gọi API mỗi 10 phút, sau đó đẩy trực tiếp chuỗi JSON payload vào một topic Kafka (ví dụ: `weather_api_events`).
- **Vai trò:** Kết hợp chặt chẽ với dữ liệu giao thông thông qua **Stream-Stream Join (cùng Watermarking)** trên Spark để giải bài toán Định giá động (Dynamic Pricing). Mô hình kinh doanh thực tế diễn ra ngay tức thì khi: Mưa to nhận từ Kafka + Kẹt xe từ Telemetry -> Hệ số giá tăng mạnh.

### 3. Dữ liệu Không gian & Danh mục (Spatial & Metadata)

Để hệ thống chuyển biến dữ liệu "thô" (vĩ độ, kinh độ, timestamp) thành "ngữ cảnh" (quận nào, phân loại thời gian gì), chúng ta cần mô hình hóa dữ liệu chiều (Dimensional Data).

- **Bản chất:** Dữ liệu tĩnh (Static Data / Dimension Data).
- **Dữ liệu Không gian (Spatial Dimension):** Các file định dạng **GeoJSON** chứa tọa độ đa giác vẽ ranh giới Quận/Huyện TP.HCM. Sử dụng qua các hàm phân tích không gian (GIS) trên SQL để khoanh vùng khu vực ùn tắc.
- **Danh mục Thời gian (Temporal Dimension):** Một tệp metadata định nghĩa các khoảng thời gian đặc thù:
  - `Morning_Rush`: 07:00 - 09:00
  - `Evening_Rush`: 16:30 - 19:00
  - `Weekend_Peak`: Tối thứ 7 và Chủ nhật
- **Vai trò:** Cung cấp baseline để gán nhãn tự động (Automated Classification) cho các sự kiện trực tiếp trên luồng stream, mở đường cho phân tích xu hướng giá và nhu cầu.

> **💡 Điểm nhấn kiến trúc (Architecture Highlights):**
> Nhờ việc bổ sung tham số Traffic Telemetry và Temporal Dimension, thiết kế của hệ thống đã vượt giao diện thu thập dữ liệu thuần túy để trở thành **Xử lý sự kiện phức tạp có nhận thức ngữ cảnh (Context-aware Complex Event Processing)**. Sự kiện A (Thời gian) ảnh hưởng dữ liệu B (Vận tốc), và cả hai kết hợp biến thiên C (Thời tiết) tạo ra Target D (Surge Pricing Ratio). Đây là kiến trúc Data Modeling thực tế được các công ty Ride-hailing áp dụng.

## B. Cấu trúc Dữ liệu Thu thập (Data Ingestion Schemas & Payloads)

Hệ thống xử lý hai luồng dữ liệu chính với định dạng JSON. Dưới đây là cấu trúc chi tiết của các gói tin (payloads) khi được nạp vào hệ thống:

### 1. Luồng dữ liệu Thời gian thực: Gói tin CDC Cuốc xe (Ride Event Payload)

Khi có sự thay đổi (INSERT/UPDATE) trong PostgreSQL, Debezium sẽ chụp lại dòng dữ liệu đó và đẩy vào Kafka dưới dạng một chuỗi JSON. Dưới đây là cấu trúc gói tin đại diện cho một cuốc xe ở trạng thái `completed` đã được trích xuất  để Spark Streaming xử lý:

```json
{
  "payload": {
    "ride_id": "RIDE-8f72a1b9",
    "user_id": "USR-10945",
    "driver_id": "DRV-5542",
    "service_type": "Car",
    "pickup_lat": 10.7769,
    "pickup_lon": 106.7009,
    "dropoff_lat": 10.7966,
    "dropoff_lon": 106.7128,
    "status": "completed",
    "estimated_fare_vnd": 45000.00,
    "created_at": "2026-03-13T08:00:00Z",
    "updated_at": "2026-03-13T08:25:12Z"
  }
}
```

**Từ điển dữ liệu (Data Dictionary):**

| Trường dữ liệu (Field) | Kiểu dữ liệu (Type) | Mô tả chi tiết (Description) |
| :--- | :--- | :--- |
| `ride_id` | String (UUID) | Mã định danh duy nhất của cuốc xe. |
| `user_id` / `driver_id` | String | Mã khách hàng và mã tài xế nhận cuốc. |
| `service_type` | String | Phân loại dịch vụ (VD: RideBike, RideCar, RideDelivery). |
| `pickup_lat` / `lon` | Decimal | Tọa độ điểm đón (Vĩ độ / Kinh độ). |
| `status` | String | Trạng thái cuốc xe (`requested`, `accepted`, `completed`, `cancelled`). |
| `estimated_fare_vnd` | Float | Cước phí dự kiến tính bằng VNĐ. |
| `updated_at` | Timestamp | Thời gian xảy ra sự kiện thay đổi trạng thái cuối cùng. |

### 2. Luồng dữ liệu Định kỳ: Gói tin API Thời tiết (Weather API Response)

Dữ liệu được Apache Airflow kéo về từ OpenWeatherMap API mỗi 10 phút. Gói tin trả về là dạng JSON lồng nhau (Nested JSON), yêu cầu kỹ năng giải nén (Explode/Flatten) trong quá trình xử lý ETL.

```json
{
  "coord": {
    "lon": 106.6667,
    "lat": 10.75
  },
  "weather": [
    {
      "id": 501,
      "main": "Rain",
      "description": "moderate rain",
      "icon": "10d"
    }
  ],
  "main": {
    "temp": 28.5,
    "feels_like": 32.1,
    "temp_min": 27.0,
    "temp_max": 29.5,
    "pressure": 1010,
    "humidity": 85
  },
  "visibility": 8000,
  "wind": {
    "speed": 4.1,
    "deg": 250
  },
  "rain": {
    "1h": 3.2
  },
  "dt": 1710313200,
  "name": "Ho Chi Minh City",
  "cod": 200
}
```

**Từ điển dữ liệu (Data Dictionary - Các trường cần trích xuất):**

| Trường trích xuất (Parsed Field) | Nguồn cấp (JSON Path)     | Kiểu dữ liệu | Ý nghĩa                                               |
| :------------------------------- | :------------------------ | :----------- | :---------------------------------------------------- |
| `city_name`                      | `name`                    | String       | Tên khu vực lấy thời tiết (TP.HCM).                   |
| `lon` / `lat`                    | `coord.lon` / `coord.lat` | Float        | Tọa độ địa lý (Kinh độ / Vĩ độ).                      |
| `weather_condition`              | `weather[0].main`         | String       | Trạng thái thời tiết chính (Rain, Clear, Clouds).     |
| `weather_description`            | `weather[0].description`  | String       | Mô tả chi tiết thời tiết.                             |
| `temperature_celsius`            | `main.temp`               | Float        | Nhiệt độ đo được (độ C).                              |
| `feels_like_celsius`             | `main.feels_like`         | Float        | Nhiệt độ cảm nhận thực tế (độ C).                     |
| `temp_min` / `temp_max`          | `main.temp_min` / `max`   | Float        | Nhiệt độ thấp nhất và cao nhất (độ C).                |
| `pressure`                       | `main.pressure`           | Integer      | Áp suất khí quyển (hPa).                              |
| `humidity`                       | `main.humidity`           | Integer      | Độ ẩm (% ).                                           |
| `visibility`                     | `visibility`              | Integer      | Tầm nhìn xa (mét).                                    |
| `wind_speed`                     | `wind.speed`              | Float        | Tốc độ gió (m/s).                                     |
| `wind_degree`                    | `wind.deg`                | Integer      | Hướng gió (độ).                                       |
| `rain_volume_1h`                 | `rain.1h`                 | Float        | Lượng mưa trong 1 giờ qua (mm).                       |
| `timestamp`                      | `dt`                      | Timestamp    | Thời điểm lấy dữ liệu (có thể convert từ Unix Epoch). |
## C. Xác định các bài toán được đặt ra 
#### Business Requirement #1: Phân tích và Tối ưu hóa vận hành (Operations Analytics)
- **Theo dõi mật độ chuyến đi:** Giám sát số lượng cuốc xe theo từng trạng thái (`requested`, `accepted`, `completed`, `cancelled`) theo các khung thời gian thực (5 phút, 15 phút, 1 giờ).
- **Phân tích hiệu suất tài xế:** Xác định tỷ lệ nhận cuốc (Acceptance Rate) và tỷ lệ hủy cuốc (Cancellation Rate) của tài xế theo từng khu vực địa lý.
- **Xác định "Điểm nóng" (Hotspots):** Định vị các khu vực có nhu cầu đặt xe cao vượt trội so với lượng tài xế hiện có tại một thời điểm nhất định.
- **Phân tích thời gian chờ (ETA):** Đo lường khoảng thời gian từ lúc khách hàng `requested` đến khi tài xế chuyển trạng thái sang `ongoing` để tối ưu hóa việc điều phối.
#### Business Requirement #2: Phân tích tác động bối cảnh và Định giá động (Contextual & Surge Pricing Analytics)

- **Theo dõi biến động nhu cầu theo thời tiết:** Phân tích sự thay đổi lượng request đặt xe khi trạng thái thời tiết thay đổi (ví dụ: nhu cầu thay đổi bao nhiêu % khi trời bắt đầu mưa).
    
- **Xác định các yếu tố ảnh hưởng đến giá:** Phân tích mối tương quan giữa lượng mưa (`rain_volume`), nhiệt độ (`feels_like`) và hệ số giá (`price_multiplier`) được áp dụng.
    
- **Phân tích doanh thu theo điều kiện môi trường:** So sánh tổng giá trị giao dịch (GMV) giữa các ngày thời tiết bình thường và các ngày có thời tiết cực đoan.
    
- **Tối ưu hóa chiến dịch khuyến mãi:** Đề xuất các khu vực cần tăng cường tài xế hoặc áp dụng mã giảm giá dựa trên dự báo thời tiết và dữ liệu lịch sử.
#### Business Requirement #3: Phân tích Lưu lượng và Hiệu suất di chuyển

- **Xác định Chỉ số kẹt xe (Congestion Index):** Tính toán độ trễ dựa trên sự chênh lệch giữa thời gian di chuyển thực tế và thời gian lý tưởng.

- **Phân tích sự biến động theo khung giờ:** So sánh hiệu suất hoàn thành chuyến (Completion Rate) giữa giờ cao điểm và giờ thấp điểm.

- **Dự báo khu vực ùn tắc:** Sử dụng dữ liệu lịch sử để cảnh báo trước các khu vực thường xuyên kẹt xe vào các thứ trong tuần (ví dụ: sáng thứ Hai tại các cửa ngõ thành phố).

#### Business Requirement #4: Phát hiện gian lận và Bất thường (Fraud Detection & Anomaly Analytics)

- **Phát hiện GPS Spoofing / Route Deviation:** Nhận diện các cuốc xe có trạng thái `ongoing` nhưng tọa độ (`lat/lon`) không thay đổi hoặc có vận tốc di chuyển bất khả thi (ví dụ: > 150km/h trong nội thành).

- **Kiểm soát lạm dụng khuyến mãi (Promo Abuse):** Theo dõi các hành vi bất thường như một thiết bị đặt và hủy cuốc liên tục, hoặc cuốc xe diễn ra với quãng đường cực kỳ ngắn (dưới 100m) nhưng vẫn ghi nhận trạng thái `completed`.

- **Cảnh báo tính khả dụng của dữ liệu (Data Latency / SLA Monitoring):** Theo dõi độ trễ luồng truyền dữ liệu bằng cách so sánh thời gian thay đổi trạng thái gốc (`updated_at`) và thời gian dữ liệu đáp vào Kafka/Spark, đảm bảo đảm bảo tính thời gian thực (ví dụ: < 5 giây).
