# Khắc phục sự cố viROS MCP Server

Tài liệu này ưu tiên chẩn đoán theo lớp: **MCP client → package `ros-mcp` → mạng → rosbridge → ROS graph → controller robot**. Không nên debug tất cả cùng lúc.

## 1. MCP Server không xuất hiện trong client

### Dấu hiệu

- không thấy `ros-mcp` trong danh sách MCP server;
- client không hiển thị tool/resource của ROS;
- server khởi tạo rồi biến mất.

### Kiểm tra

```bash
uvx ros-mcp --help
```

Nếu lệnh trên lỗi, vấn đề nằm ở package/runtime trước khi liên quan tới ROS.

Thử:

1. đóng hoàn toàn MCP client rồi mở lại;
2. kiểm tra log khởi tạo MCP;
3. xác nhận command/config dùng đúng `uvx ros-mcp --transport=stdio`;
4. kiểm tra Python/uv có trong `PATH` của môi trường mà client sử dụng.

## 2. Connection refused / timeout tới rosbridge

### Dấu hiệu

- `Connection refused`;
- timeout;
- không tạo được session;
- AI không đọc được ROS graph.

### Phía robot

Kiểm tra ROS:

```bash
# ROS 2
ros2 topic list

# ROS 1
rostopic list
```

Kiểm tra tiến trình rosbridge:

```bash
ps aux | grep rosbridge
```

Kiểm tra cổng:

```bash
ss -ltnp | grep 9090
```

### Từ máy người dùng

Linux/macOS:

```bash
nc -vz <robot-ip> 9090
```

Windows PowerShell:

```powershell
Test-NetConnection <robot-ip> -Port 9090
```

Nếu ping thành công nhưng port 9090 không tới được, kiểm tra firewall hoặc rosbridge chưa listen trên interface phù hợp.

## 3. Sai địa chỉ IP

Trên robot:

```bash
hostname -I
```

Nếu máy có nhiều interface, có thể xuất hiện nhiều IP cho Wi-Fi, Ethernet, Docker, VPN. Hãy chọn IP mà laptop thực sự định tuyến tới được.

Ví dụ:

```text
192.168.1.50    ← LAN Wi-Fi
172.17.0.1      ← Docker bridge, thường không dùng từ máy khác
100.x.x.x       ← có thể là VPN/Tailscale
```

## 4. WSL trên Windows

Kiểm tra distro:

```powershell
wsl --list --verbose
```

Trong cấu hình MCP, dùng đúng tên distro nếu command được chạy qua WSL.

Khuyến nghị đặt source code trong filesystem Linux:

```text
/home/<user>/projects/
```

thay vì:

```text
/mnt/c/Users/...
```

vì mount Windows có thể chậm và gây khác biệt permission/file watching.

Kiểm tra trực tiếp trong WSL:

```bash
uvx ros-mcp --help
```

## 5. HTTP transport không hoạt động

Khởi chạy thủ công:

```bash
uvx ros-mcp --transport streamable-http --host 127.0.0.1 --port 9000
```

Kiểm tra port:

```bash
netstat -tulpn | grep :9000
```

Kiểm tra endpoint:

```bash
curl http://localhost:9000/mcp
```

Nếu truy cập từ máy khác, cần cân nhắc bind address, firewall và xác thực/network boundary. Không nên public MCP endpoint không bảo vệ ra Internet.

## 6. MCP kết nối được nhưng không thấy topic/service

Kiểm tra trực tiếp trên robot:

```bash
ros2 node list
ros2 topic list
ros2 service list
ros2 action list
```

Nếu ROS CLI cũng không thấy, lỗi không nằm ở MCP.

Nếu ROS CLI thấy nhưng MCP không thấy, kiểm tra:

- rosbridge/rosapi đã hoạt động đầy đủ;
- ROS distro và interface type;
- namespace;
- ROS_DOMAIN_ID trong ROS 2;
- network/container boundary;
- các service rosapi cần thiết.

## 7. AI gọi sai message type

Không để mô hình đoán schema message tùy biến.

Quy trình đúng:

```text
1. tìm topic/service/action
2. lấy type
3. đọc interface/schema
4. tạo payload
5. validate
6. mới gửi
```

ROS 2:

```bash
ros2 topic info /topic_name
ros2 interface show package/msg/Type
```

ROS 1:

```bash
rostopic info /topic_name
rosmsg show package/Type
```

## 8. Publish thành công nhưng robot không chuyển động

Đây là lỗi thường bị hiểu nhầm là “MCP không điều khiển được robot”. Publish thành công chỉ chứng minh message đã đi vào một topic; không chứng minh controller cuối cùng chấp nhận lệnh.

Kiểm tra:

- controller có active không;
- có topic điều khiển cấp cao khác không;
- robot có watchdog/enable switch không;
- node lifecycle đã `active` chưa;
- command có bị safety layer ghi đè không;
- tốc độ có nằm trong giới hạn không;
- có mux/priority arbiter chặn nguồn lệnh không;
- robot yêu cầu action/service thay vì topic hay không.

## 9. Parameter không hoạt động

Một số chức năng parameter/action trong kiến trúc repo chủ yếu liên quan ROS 2.

Kiểm tra trực tiếp:

```bash
ros2 param list
ros2 param get /node_name parameter_name
```

Nếu node không khai báo parameter hoặc parameter read-only, MCP không thể vượt qua ràng buộc đó.

## 10. Camera/ảnh không hiển thị

Kiểm tra:

```bash
ros2 topic list | grep image
ros2 topic info /camera/image_raw
```

Các nguyên nhân phổ biến:

- dùng nhầm raw/compressed topic;
- encoding không được xử lý;
- frame quá lớn;
- băng thông Wi-Fi thấp;
- image topic có QoS không phù hợp;
- client MCP không render content ảnh như mong đợi.

## 11. Lỗi ROS 1/ROS 2 khác nhau

Không giả định service/type giống nhau giữa ROS 1 và ROS 2. Khi có lỗi type hoặc rosapi, luôn xác định distro và phiên bản ROS trước.

Ví dụ prompt an toàn:

```text
Hãy nhận diện ROS version trước. Sau đó chỉ dùng interface tương thích
với phiên bản đã phát hiện. Nếu không chắc type, hãy dừng và đọc metadata.
```

## 12. Debug theo tầng

| Tầng | Câu hỏi | Kiểm tra |
|---|---|---|
| MCP client | client có load server? | log client, danh sách MCP |
| Package | `ros-mcp` chạy được? | `uvx ros-mcp --help` |
| Network | tới robot được? | `ping`, `Test-NetConnection`, `nc` |
| Rosbridge | cổng mở? | `ss`, process, log rosbridge |
| ROS graph | node/topic tồn tại? | `ros2 ... list` / ROS 1 CLI |
| Interface | type/schema đúng? | `ros2 interface show` |
| Controller | lệnh có được chấp nhận? | controller state/log |
| Physical safety | actuator có bị interlock? | E-stop, enable, safety PLC/controller |

## 13. Bộ lệnh debug nhanh

```bash
# MCP
uvx ros-mcp --help

# ROS 2
ros2 node list
ros2 topic list
ros2 service list
ros2 action list

# ROS 1
rosnode list
rostopic list
rosservice list

# Linux network/process
ps aux | grep rosbridge
ss -ltnp | grep 9090
```

Windows:

```powershell
Test-NetConnection <robot-ip> -Port 9090
wsl --list --verbose
```

## 14. Khi cần mở issue

Chuẩn bị tối thiểu:

- hệ điều hành;
- ROS distro/ROS 1 hay ROS 2;
- MCP client;
- cách cài `ros-mcp`;
- robot/simulator;
- IP topology (không đăng bí mật/public credential);
- log lỗi đầy đủ;
- bước tái hiện;
- kết quả mong đợi và kết quả thực tế.

Issue của bản Việt hóa có thể mở tại repo này; lỗi upstream thuần túy nên đối chiếu thêm với dự án gốc `robotmcp/ros-mcp-server`.

## 15. Liên kết

- [Trung tâm tiếng Việt](README.md)
- [Cài đặt tiếng Việt](cai-dat.md)
- [Kiến trúc tiếng Việt](kien-truc.md)
- [Troubleshooting upstream](../install/troubleshooting.md)
