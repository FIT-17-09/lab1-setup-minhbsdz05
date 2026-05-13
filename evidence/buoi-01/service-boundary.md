# Service Boundary của nhóm

## 1. Thông tin nhóm

- Tên nhóm: Nhóm 8
- Lớp: CNTT 17-09
- Thành viên: Nguyễn Xuân Phúc, Nguyễn Thị Hảo Ngân, Hoàng Anh Minh, Nguyễn Kiều Phong.
- Service nhóm phụ trách: Dịch vụ tiếp nhận dữ liệu IoT
- Sản phẩm tổng thể của lớp: Hệ thống Giám sát và Phân tích dữ liệu IoT tập trung

## 2. Actor

Ai tương tác với hệ thống/service?

- Thiết bị IoT: Gửi dữ liệu telemetry (nhiệt độ, độ ẩm, vị trí...) về hệ thống.

- Cổng vào: Các thiết bị trung gian tập hợp dữ liệu từ nhiều node cảm biến trước khi gửi lên Cloud.

- Hệ thống Quản trị: Tương tác để cấu hình các ngưỡng tiếp nhận hoặc quản lý danh sách thiết bị được phép kết nối.

## 3. System Boundary

Nhóm em xây phần nào?

- Nhóm tập trung vào việc xây dựng bộ phận tiếp nhận dữ liệu hiệu năng cao, đảm bảo không làm mất mát dữ liệu khi có lượng lớn thiết bị gửi tin cùng lúc.

Phần nhóm kiểm soát:

- Giao thức tiếp nhận (MQTT Broker, HTTP/REST API).
- Bộ đệm dữ liệu (Message Queue như Kafka hoặc RabbitMQ) để tránh nghẽn cổ chai.
- Logic xác thực thiết bị (Device Authentication) dựa trên API Key/Token.
- Chuẩn hóa dữ liệu thô (Data Parsing) về định dạng chung của hệ thống.

Phần nhóm chỉ tích hợp:

- Dịch vụ xác thực tập trung của lớp (Identity Service).
- Cơ sở dữ liệu Time-series (InfluxDB/TimescaleDB) do nhóm khác quản lý.

## 4. Service Boundary

Service của nhóm có trách nhiệm gì?

- Duy trì kết nối ổn định với hàng ngàn thiết bị đồng thời.

- Kiểm tra tính hợp lệ của gói tin (Validation).

- Chuyển đổi giao thức (ví dụ: từ MQTT sang Message lưu trong Queue).

- Ghi log trạng thái hoạt động của thiết bị (Online/Offline).

Service KHÔNG làm gì?

- Không thực hiện phân tích dữ liệu (Analytics).

- Không vẽ biểu đồ hiển thị (Visualization).

- Không quản lý thông tin cá nhân của người dùng (User Management).

- Không lưu trữ dữ liệu lịch sử lâu dài (Long-term Storage).

## 5. Input / Output

### Input

- Gói tin JSON từ thiết bị (Ví dụ: {"device_id": "sensor_01", "temp": 25.5, "timestamp": 1715386140}).

- Thông tin xác thực (Header Token hoặc MQTT Username/Password).

### Output

- Phản hồi xác nhận tiếp nhận (ACK - Acknowledgment).

- Dữ liệu đã chuẩn hóa đẩy vào Message Broker cho các service khác tiêu thụ.

- Thông báo lỗi nếu dữ liệu không hợp lệ hoặc thiết bị chưa được đăng ký.

## 6. API dự kiến

| Method | Endpoint                | Mục đích                                        |
| ------ | ----------------------- | ----------------------------------------------- |
| GET    | /health                 | Kiểm tra trạng thái sống/chết của service       |
| POST   | /v1/telemetry           | Tiếp nhận dữ liệu từ thiết bị qua HTTP/HTTPS    |
| GET    | /v1/devices/{id}/status | Kiểm tra thiết bị này có đang kết nối hay không |
| POST   | /v1/config/ingestion    | Cấu hình các quy tắc lọc dữ liệu thô            |

## 7. Phụ thuộc service khác

Service này gọi đến service nào?

- Service này gọi đến: Identity Service (để xác thực thiết bị).

- Service này đẩy dữ liệu cho: Storage Service và Alert Service (thông qua Message Broker).

- Service nào gọi đến service này: Device Management Service (để cập nhật cấu hình tiếp nhận).

- Service nào gọi đến service này?

## 8. Sơ đồ minh họa

```mermaid
flowchart TD
    %% Định nghĩa CSS cho các component để dễ nhìn
    classDef ingestion fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px;
    classDef external fill:#f5f5f5,stroke:#9e9e9e,stroke-width:1px;
    classDef other fill:#e8f5e9,stroke:#4caf50,stroke-width:1px;

    subgraph EdgeLayer ["1. Thiết bị ngoại vi (Edge Layer)"]
        direction TB
        Sensors["Cảm biến/Thiết bị IoT\n(Nhiệt độ, Độ ẩm, GPS...)"]
        GW["IoT Gateway\n(Gom cụm & Tiền xử lý tại biên)"]
    end

    subgraph IngestionService ["2. Dịch vụ Tiếp nhận dữ liệu (Ingestion Service - Nhóm phụ trách)"]
        direction TB
        LB["Load Balancer / API Gateway\n(Điều phối lưu lượng)"]
        
        subgraph Adapters ["Protocol Adapters (Giao thức kết nối)"]
            direction LR
            MQTT["MQTT Broker\n(Mosquitto/EMQX)"]
            HTTP["HTTP/REST API"]
            CoAP["CoAP Endpoint"]
        end
        
        AuthFilter["Security Filter\n(Xác thực Token/Chứng chỉ)"]
        RateLimit["Rate Limiter\n(Chống Spam/DDoS)"]
        
        subgraph Pipeline ["Data Processing Pipeline (Tiền xử lý)"]
            direction TB
            Validator["Schema Validator\n(Kiểm tra tính hợp lệ dữ liệu)"]
            Normalizer["Data Normalizer\n(Chuẩn hóa định dạng JSON chung)"]
        end
        
        subgraph Queues ["Message Queues (Hàng đợi)"]
            direction LR
            Kafka["Main Message Bus\n(Kafka Topics / RabbitMQ)"]
            DLQ["Dead Letter Queue (DLQ)\n(Lưu trữ dữ liệu lỗi/rác)"]
        end
    end

    subgraph CoreServices ["3. Các Service khác trong hệ thống tổng thể"]
        direction TB
        Auth["Identity & Device Mgt\n(Quản lý thiết bị & User)"]
        
        subgraph StorageLayer ["Storage Service (Lưu trữ)"]
            direction LR
            TSDB["Time-Series DB\n(Dữ liệu nóng)"]
            DataLake["Data Lake\n(Dữ liệu lạnh/Backup)"]
        end
        
        subgraph AnalyticsLayer ["Analytics Service (Phân tích)"]
            direction LR
            Realtime["Stream Processing\n(Phân tích thời gian thực)"]
            Batch["Batch Processing\n(Báo cáo thống kê)"]
        end
    end

    %% --- Kết nối luồng dữ liệu ---
    
    %% Từ thiết bị đến Ingestion
    Sensors -- "Dữ liệu thô (MQTT/CoAP)" --> LB
    GW -- "Dữ liệu thô (HTTPS/MQTT)" --> LB
    
    %% Phân luồng giao thức
    LB --> MQTT & HTTP & CoAP
    MQTT & HTTP & CoAP --> AuthFilter
    
    %% Xác thực
    AuthFilter -- "Xác thực thiết bị" <--> Auth
    AuthFilter -- "Hợp lệ" --> RateLimit
    AuthFilter -. "Từ chối" .-> Drop("Drop Connection")
    
    %% Xử lý luồng
    RateLimit --> Validator
    Validator -- "Đúng Schema" --> Normalizer
    Validator -- "Sai Schema / Lỗi" --> DLQ
    Normalizer --> Kafka
    
    %% Phân phối từ Queue ra toàn hệ thống (Fan-out)
    Kafka === "Tiêu thụ (Consume)" ===> TSDB
    Kafka === "Tiêu thụ (Consume)" ===> DataLake
    Kafka === "Tiêu thụ (Consume)" ===> Realtime
    Kafka === "Tiêu thụ (Consume)" ===> Batch

    %% Gán class màu sắc
    class EdgeLayer,Sensors,GW,Drop external;
    class IngestionService,LB,MQTT,HTTP,CoAP,AuthFilter,RateLimit,Validator,Normalizer,Kafka,DLQ,Adapters,Pipeline,Queues ingestion;
    class CoreServices,Auth,TSDB,DataLake,Realtime,Batch,StorageLayer,AnalyticsLayer other;
```
