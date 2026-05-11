# Service Boundary của nhóm

## 1. Thông tin nhóm

- Tên nhóm: Nhóm 8
- Lớp: CNTT 17-09
- Thành viên: Nguyễn Xuân Phúc, Nguyễn Thị Hảo Ngân, Hoàng Anh Minh, Nguyễn Kiều Phong
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
    subgraph "Thiết bị ngoại vi"
        Sensors[Cảm biến/Thiết bị IoT]
        GW[IoT Gateway]
    end

    subgraph "Ingestion Service (Nhóm đảm nhiệm)"
        MQTT[MQTT Broker/HTTP API]
        Validator[Bộ kiểm tra & Chuẩn hóa]
        Queue[Message Queue - Kafka/RabbitMQ]
    end

    subgraph "Các Service khác trong lớp"
        Auth[Identity Service]
        Storage[Storage Service]
        Analysis[Analytics Service]
    end

    Sensors -- "Dữ liệu thô" --> MQTT
    GW -- "Dữ liệu thô" --> MQTT
    MQTT --> Validator
    Validator -- "Xác thực" --> Auth
    Validator --> Queue
    Queue --> Storage
    Queue --> Analysis
```
