**Tên dự án:** RideStream - Real-time CDC & Analytics Platform
**Lĩnh vực:** Data Engineering 

## 1. Đặt vấn đề (Problem Statement)

Trong các hệ thống di chuyển đô thị hiện đại, dữ liệu được sinh ra liên tục với khối lượng khổng lồ. Thách thức lớn nhất đối với các DE là làm sao thu thập sự thay đổi trạng thái của các cuốc xe theo thời gian thực từ cơ sở dữ liệu vận hành (OLTP) mà không làm ảnh hưởng đến hiệu năng hệ thống, đồng thời kết hợp chúng với dữ liệu bối cảnh (thời tiết, giao thông) để đưa ra các quyết định kinh doanh như định giá động (Dynamic Pricing) với độ trễ thấp nhất.

Dự án **RideStream** được đề xuất nhằm giải quyết bài toán trên bằng cách xây dựng End-to-End Data Pipeline áp dụng kỹ thuật Change Data Capture (CDC).

## 2. Mục tiêu dự án (Objectives)

- **Về mặt kỹ thuật:** Thiết kế và triển khai một kiến trúc Lambda/Kappa kết hợp luồng xử lý Streaming và Batch. Hệ thống phải có khả năng chịu tải cao, bắt sự kiện theo thời gian thực từ Database và thực hiện các phép biến đổi không gian (Spatial Data).
- **Về mặt vận hành (DevOps):** Đóng gói toàn bộ các dịch vụ bằng container (Docker) để đảm bảo tính nhất quán giữa các môi trường, thiết lập luồng CI/CD và triển khai trên hạ tầng điện toán đám mây.
- **Về mặt nghiệp vụ:** Xây dựng mô hình dữ liệu đa chiều (Star Schema) phục vụ việc trực quan hóa bản đồ nhiệt giao thông và theo dõi tỷ lệ chuyển đổi trạng thái của các cuốc xe.
    

## 3. Kiến trúc Hệ thống & Công nghệ (Architecture & Tech Stack)

Hệ thống được thiết kế phân lớp rõ ràng, tuân thủ các tiêu chuẩn công nghiệp hiện hành:

- **Lớp Sinh dữ liệu & Lưu trữ vận hành (Data Generation & OLTP):** Sử dụng **Python** giả lập hành vi đặt xe và vòng đời cuốc xe (Requested -> Accepted -> Completed). Dữ liệu giao dịch được ghi trực tiếp vào **PostgreSQL** (tích hợp PostGIS để xử lý tọa độ địa lý).
- **Lớp Thu thập thời gian thực (CDC Ingestion):** Sử dụng **Debezium** đọc Write-Ahead Logs (WAL) từ PostgreSQL để bắt các sự kiện thay đổi dữ liệu ở cấp độ dòng (row-level) và đẩy vào hàng đợi thông điệp.
- **Lớp Trạm trung chuyển (Message Broker):** **Apache Kafka** làm nhiệm vụ hứng luồng dữ liệu tốc độ cao, đảm bảo hệ thống không bị nghẽn tải.
- **Lớp Xử lý dữ liệu (Stream Processing):** **Apache Spark (Structured Streaming)** tiêu thụ dữ liệu từ Kafka, thực hiện làm sạch, bóc tách JSON và gom nhóm dữ liệu. Thực hiện **Stream-Stream Join** giữa luồng sự kiện cuốc xe và luồng thời tiết với cơ chế Watermarking để phát cảnh báo theo thời gian thực.
- **Lớp Điều phối (Orchestration & Ingestion):** **Apache Airflow** điều phối các tác vụ chạy định kỳ, gọi API (OpenWeatherMap) lấy dữ liệu thời tiết mỗi 10 phút và đẩy trực tiếp vào Topic Kafka.
- **Lớp Lưu trữ Đám mây (Cloud Data Platform):** Dữ liệu thô và đã xử lý được lưu trữ trên , sau đó nạp vào **Google BigQuery** để làm Data Warehouse.

## 4. Thiết kế Mô hình Dữ liệu (Data Modeling)

Dữ liệu trong Data Warehouse sẽ được tổ chức theo mô hình Star Schema để tối ưu hóa truy vấn phân tích:

- **Fact_Ride_Events:** Bảng sự kiện lưu trữ thông tin cuốc xe (ID, tọa độ đón/trả, giá tiền, thời gian, trạng thái).
    
- **Dim_Time:** Bảng chiều thời gian (Giờ, Ngày, Tháng, Năm, Thứ, Cuối tuần).
    
- **Dim_Location:** Bảng chiều không gian (ID Quận, Tên Quận, Ranh giới hành chính).
    
- **Dim_Weather:** Bảng bối cảnh thời tiết (Nhiệt độ, Lượng mưa, Trạng thái thời tiết).
    

## 5. Tiêu chí Đánh giá & Bài toán Phân tích (Deliverables & Use Cases)

Dự án được coi là thành công khi xuất ra được Dashboard (qua Power BI hoặc Grafana) giải quyết được các câu hỏi kinh doanh:

1. **Surge Pricing Alert:** Hệ thống có khả năng bắn cảnh báo nhân hệ số giá (1.5x) khi phát hiện khu vực có lượng yêu cầu đặt xe tăng đột biến kết hợp với lượng mưa > 20mm trong 10 phút gần nhất.
    
2. **Conversion & Drop-off Analysis:** Tính toán được tỷ lệ tài xế hủy cuốc xe theo từng quận và theo điều kiện thời tiết thực tế.
    

## 6. Kế hoạch Triển khai (Roadmap)

- **Giai đoạn 1:** Khởi tạo hạ tầng Docker Compose (PostgreSQL, Kafka, Debezium). Viết script Data Generator sinh dữ liệu vào Database và cấu hình Debezium bắt luồng thay đổi.
    
- **Giai đoạn 2:** Viết các ứng dụng PySpark xử lý luồng Streaming từ Kafka, thực hiện các phép biến đổi cơ bản và đẩy dữ liệu mẫu ra console để kiểm thử.
    
- **Giai đoạn 3:** Thiết lập Apache Airflow, viết DAG gọi API thời tiết đẩy vào Kafka. Thực hiện Stream-Stream Join trên Spark. Tích hợp các luồng dữ liệu lên môi trường Cloud (GCP).
    
- **Giai đoạn 4:** Xây dựng mô hình Star Schema trên Cloud Data Warehouse, kết nối công cụ BI để vẽ biểu đồ và hoàn thiện tài liệu kỹ thuật.