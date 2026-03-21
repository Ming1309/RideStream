**Tên dự án:** RideStream - Real-time CDC & Analytics Platform
**Lĩnh vực:** Data Engineering & Full-stack Development

## 1. Đặt vấn đề (Problem Statement)

Trong các hệ thống di chuyển đô thị hiện đại, dữ liệu được sinh ra liên tục với khối lượng khổng lồ. Thách thức lớn nhất đối với các DE là làm sao thu thập sự thay đổi trạng thái của các cuốc xe theo thời gian thực từ cơ sở dữ liệu vận hành (OLTP) mà không làm ảnh hưởng đến hiệu năng hệ thống, đồng thời kết hợp chúng với dữ liệu bối cảnh (thời tiết, giao thông) để đưa ra các quyết định kinh doanh như định giá động (Dynamic Pricing) với độ trễ thấp nhất.

Dự án **RideStream** được đề xuất nhằm giải quyết bài toán trên bằng cách xây dựng End-to-End Data Pipeline áp dụng kỹ thuật Change Data Capture (CDC).

## 2. Mục tiêu dự án (Objectives)

- **Về mặt kỹ thuật (Data Pipeline):** Thiết kế và triển khai một kiến trúc Lambda/Kappa kết hợp luồng xử lý Streaming và Batch. Hệ thống phải có khả năng chịu tải cao, bắt sự kiện theo thời gian thực từ Database và thực hiện các phép biến đổi không gian (Spatial Data).
- **Về mặt kỹ thuật (Application Layer):** Xây dựng lớp Backend API (Node.js) kết nối với PostgreSQL (dữ liệu đã xử lý bởi Spark), sử dụng cơ chế **LISTEN/NOTIFY** để nhận sự kiện thời gian thực và đẩy xuống Frontend qua **Socket.io**. Cung cấp REST API phục vụ truy vấn dữ liệu phân tích. Phát triển Frontend Dashboard (React) cho phép đội ngũ vận hành giám sát, phân tích và ra quyết định kinh doanh trực quan.
- **Về mặt vận hành (DevOps):** Đóng gói toàn bộ các dịch vụ bằng container (Docker) để đảm bảo tính nhất quán giữa các môi trường, thiết lập luồng CI/CD và triển khai trên hạ tầng điện toán đám mây.
- **Về mặt nghiệp vụ:** Xây dựng mô hình dữ liệu đa chiều (Star Schema) phục vụ việc trực quan hóa bản đồ nhiệt giao thông và theo dõi tỷ lệ chuyển đổi trạng thái của các cuốc xe.
    

## 3. Kiến trúc Hệ thống & Công nghệ (Architecture & Tech Stack)

Hệ thống được thiết kế phân lớp rõ ràng, tuân thủ các tiêu chuẩn công nghiệp hiện hành:

- **Lớp Sinh dữ liệu & Lưu trữ vận hành (Data Generation & OLTP):** Sử dụng **Python** giả lập hành vi đặt xe và vòng đời cuốc xe (Requested -> Accepted -> Completed). Dữ liệu giao dịch được ghi trực tiếp vào **PostgreSQL** (tích hợp PostGIS để xử lý tọa độ địa lý).
- **Lớp Thu thập thời gian thực (CDC Ingestion):** Sử dụng **Debezium** đọc Write-Ahead Logs (WAL) từ PostgreSQL để bắt các sự kiện thay đổi dữ liệu ở cấp độ dòng (row-level) và đẩy vào hàng đợi thông điệp.
- **Lớp Trạm trung chuyển (Message Broker):** **Apache Kafka** làm nhiệm vụ hứng luồng dữ liệu tốc độ cao, đảm bảo hệ thống không bị nghẽn tải.
- **Lớp Xử lý dữ liệu (Stream Processing):** **Apache Spark (Structured Streaming)** tiêu thụ dữ liệu từ Kafka, thực hiện làm sạch, bóc tách JSON và gom nhóm dữ liệu. Thực hiện **Stream-Stream Join** giữa luồng sự kiện cuốc xe và luồng thời tiết với cơ chế Watermarking để phát cảnh báo theo thời gian thực.
- **Lớp Điều phối (Orchestration & Ingestion):** **Apache Airflow** điều phối các tác vụ chạy định kỳ, gọi API (OpenWeatherMap) lấy dữ liệu thời tiết mỗi 10 phút và đẩy trực tiếp vào Topic Kafka.
- **Lớp Lưu trữ Đám mây (Cloud Data Platform):** Dữ liệu thô và đã xử lý được lưu trữ trên Google Cloud Storage, sau đó nạp vào **Google BigQuery** để làm Data Warehouse.
- **Lớp Ứng dụng Backend (Application API Layer):** Sử dụng **Node.js (Express/Fastify)** xây dựng REST API server phục vụ dữ liệu cho Frontend. Backend kết nối vào 3 điểm:
    - **PostgreSQL LISTEN/NOTIFY** (`pg`): Lắng nghe sự kiện từ database trigger khi Spark ghi dữ liệu mới (cuốc xe đã xử lý, cảnh báo surge, cảnh báo fraud) → đẩy xuống Frontend theo thời gian thực qua **Socket.io**.
    - **PostgreSQL Query** (`pg`/`Prisma`): REST API phục vụ truy vấn dữ liệu chi tiết cuốc xe, thống kê real-time và dữ liệu trong ngày.
    - **BigQuery** (`@google-cloud/bigquery`): Query dữ liệu phân tích lịch sử quy mô lớn (thống kê theo tuần, tháng, quý) từ Data Warehouse phục vụ báo cáo và biểu đồ xu hướng.
- **Lớp Giao diện (Frontend Dashboard):** Sử dụng **React** kết hợp **Leaflet** (bản đồ tương tác) xây dựng Operations Dashboard cho phép quản lý vận hành theo dõi hoạt động hệ thống theo thời gian thực, bao gồm: bản đồ nhiệt, biểu đồ phân tích, bảng cảnh báo và báo cáo.

## 4. Thiết kế Mô hình Dữ liệu (Data Modeling)

Dữ liệu trong Data Warehouse sẽ được tổ chức theo mô hình Star Schema để tối ưu hóa truy vấn phân tích:

- **Fact_Ride_Events:** Bảng sự kiện lưu trữ thông tin cuốc xe (ID, tọa độ đón/trả, giá tiền, thời gian, trạng thái).
    
- **Dim_Time:** Bảng chiều thời gian (Giờ, Ngày, Tháng, Năm, Thứ, Cuối tuần).
    
- **Dim_Location:** Bảng chiều không gian (ID Quận, Tên Quận, Ranh giới hành chính).
    
- **Dim_Weather:** Bảng bối cảnh thời tiết (Nhiệt độ, Lượng mưa, Trạng thái thời tiết).
    

## 5. Tiêu chí Đánh giá & Bài toán Phân tích (Deliverables & Use Cases)

Dự án được coi là thành công khi đáp ứng được các tiêu chí sau:

**A. Data Pipeline (Power BI / Grafana):**

1. **Surge Pricing Alert:** Hệ thống có khả năng bắn cảnh báo nhân hệ số giá (1.5x) khi phát hiện khu vực có lượng yêu cầu đặt xe tăng đột biến kết hợp với lượng mưa > 20mm trong 10 phút gần nhất.
    
2. **Conversion & Drop-off Analysis:** Tính toán được tỷ lệ tài xế hủy cuốc xe theo từng quận và theo điều kiện thời tiết thực tế.

**B. Application Layer (Web Dashboard):**

3. **Real-time Operations Dashboard:** Giao diện web hiển thị số liệu cuốc xe cập nhật theo thời gian thực (qua WebSocket), bao gồm bản đồ nhiệt khu vực có nhu cầu cao.

4. **Surge & Fraud Alert System:** Hệ thống cảnh báo tức thì trên web khi phát hiện sự kiện surge pricing hoặc hành vi gian lận, nhận qua cơ chế PostgreSQL LISTEN/NOTIFY → Socket.io.

5. **Analytics & Reporting:** Các trang phân tích cho phép lọc dữ liệu theo quận, khung giờ, điều kiện thời tiết. Hỗ trợ báo cáo thống kê theo tuần/tháng/quý từ BigQuery Data Warehouse.

## 6. Kế hoạch Triển khai (Roadmap)

- **Giai đoạn 1 (Hạ tầng & Data Source):** Khởi tạo hạ tầng Docker Compose (PostgreSQL, Kafka, Debezium). Viết script Data Generator sinh dữ liệu vào Database và cấu hình Debezium bắt luồng thay đổi.
    
- **Giai đoạn 2 (Stream Processing):** Viết các ứng dụng PySpark xử lý luồng Streaming từ Kafka, thực hiện các phép biến đổi cơ bản và đẩy dữ liệu mẫu ra console để kiểm thử.

- **Giai đoạn 3 (Orchestration & Cloud):** Thiết lập Apache Airflow, viết DAG gọi API thời tiết đẩy vào Kafka. Thực hiện Stream-Stream Join trên Spark. Tích hợp các luồng dữ liệu lên môi trường Cloud (GCP).

- **Giai đoạn 4 (Data Warehouse & BI):** Xây dựng mô hình Star Schema trên Cloud Data Warehouse, kết nối công cụ BI để vẽ biểu đồ.

- **Giai đoạn 5 (Backend API):** Khởi tạo project Node.js (Express/Fastify). Xây dựng REST API kết nối PostgreSQL (bảng processed) và BigQuery (thống kê tuần/tháng). Thiết lập database triggers + LISTEN/NOTIFY để nhận sự kiện real-time. Triển khai Socket.io server đẩy dữ liệu xuống client. Thiết lập Authentication (JWT).

- **Giai đoạn 6 (Frontend Dashboard):** Phát triển React Dashboard bao gồm: trang tổng quan real-time với bản đồ nhiệt (Leaflet), trang cảnh báo Surge/Fraud, trang phân tích & báo cáo. Kết nối WebSocket và REST API. Hoàn thiện tài liệu kỹ thuật.