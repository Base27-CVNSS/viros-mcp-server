# Trung tâm tài liệu tiếng Việt — viROS MCP Server

Tài liệu này giải thích ROS-MCP theo góc nhìn kỹ thuật dành cho sinh viên, kỹ sư robot, phòng thí nghiệm và nhà phát triển AI tại Việt Nam.

> **Nguyên tắc Việt hóa:** dịch phần diễn giải, giữ nguyên tên giao thức, lệnh CLI, package, tool, resource URI, message type và interface ROS để không làm mất tính tương thích.

## 1. Mục tiêu của hệ thống

viROS MCP Server giúp một MCP client đưa ROS vào vòng lặp suy luận của mô hình AI:

```text
Ý định người dùng
      ↓
LLM / Agent
      ↓
MCP tool hoặc MCP resource
      ↓
ROS MCP Server
      ↓
rosbridge WebSocket
      ↓
ROS graph
      ↓
Robot / Simulator
```

LLM không giao tiếp trực tiếp với motor, camera hay node ROS. Nó gọi các công cụ MCP được server công bố. Server chuyển yêu cầu thành thao tác tương ứng với ROS và trả kết quả có cấu trúc về cho client.

## 2. MCP đóng vai trò gì?

**Model Context Protocol (MCP)** là lớp chuẩn hóa giao tiếp giữa ứng dụng AI và công cụ/dữ liệu bên ngoài. Trong repo này, MCP giúp:

- công bố các thao tác ROS dưới dạng `tool`;
- công bố metadata ROS dưới dạng `resource`;
- mô tả tham số đầu vào bằng schema để LLM gọi đúng cú pháp;
- trả kết quả có cấu trúc thay vì buộc LLM tự thao tác socket thô;
- tách client AI khỏi chi tiết triển khai robot.

MCP không phải middleware robot và không thay thế DDS/ROS master/ROS graph.

## 3. Rosbridge đóng vai trò gì?

`rosbridge_server` tạo lớp WebSocket/JSON để ứng dụng bên ngoài ROS truy cập graph. ROS MCP Server dùng lớp này để:

- khám phá node/topic/service/action/parameter;
- publish hoặc subscribe topic;
- gọi service;
- gửi action goal;
- đọc/thay đổi parameter;
- lấy dữ liệu camera và sensor khi được hỗ trợ.

Nhờ đó MCP server có thể chạy trên laptop của người dùng thay vì phải cài toàn bộ AI stack trực tiếp lên máy tính của robot.

## 4. Kiến trúc chức năng

### 4.1 AI / MCP Client

Ví dụ: Claude Code, Codex CLI, Gemini CLI, Claude Desktop, ChatGPT, Cursor hoặc client MCP tự xây dựng.

Nhiệm vụ:

- nhận lệnh ngôn ngữ tự nhiên;
- suy luận tool/resource cần dùng;
- tạo tham số gọi tool;
- diễn giải kết quả ROS cho người dùng.

### 4.2 MCP Server

Package cốt lõi vẫn giữ tên `ros-mcp`.

Nhiệm vụ:

- đăng ký tool/resource;
- validate dữ liệu;
- quản lý kết nối rosbridge;
- chuyển đổi dữ liệu giữa MCP và ROS;
- cung cấp annotation giúp client phân biệt thao tác đọc và thao tác có thể thay đổi trạng thái.

### 4.3 Rosbridge

Nhiệm vụ:

- nhận request WebSocket;
- chuyển sang lời gọi/trao đổi trong ROS;
- trả dữ liệu ROS về server.

### 4.4 ROS / Robot

Đây mới là tầng thực thi vật lý hoặc mô phỏng. Các giới hạn an toàn như velocity limit, workspace, collision avoidance, E-stop phải được thực thi ở tầng này thay vì chỉ trông chờ vào prompt của AI.

## 5. Nhóm năng lực

### Kết nối

- kiểm tra host/port;
- kết nối rosbridge;
- nhận diện ROS 1/ROS 2 khi server hỗ trợ.

### Topics

- xem metadata topic;
- publish message;
- subscribe một lần hoặc theo khoảng thời gian;
- nhận dữ liệu camera/sensor.

### Services

- khám phá service;
- đọc kiểu service;
- gọi service với request có cấu trúc.

### Actions

- khám phá action;
- gửi goal;
- theo dõi kết quả/feedback tùy giao diện được hỗ trợ.

### Parameters

- đọc parameter;
- kiểm tra tồn tại;
- thay đổi/xóa parameter khi được phép.

### Resources

Resource MCP phù hợp với dữ liệu mang tính mô tả/ngữ cảnh, ví dụ metadata về ROS graph hoặc robot specification. Việc tách resource khỏi tool giúp LLM lấy ngữ cảnh mà không tạo hành động ngoài ý muốn.

## 6. Mô hình triển khai khuyến nghị

```text
MÁY NGƯỜI DÙNG                         MÁY ROBOT
────────────────────                   ────────────────────
MCP Client / AI                        ROS / ROS 2
ROS MCP Server          WebSocket      rosbridge_server
Python / uv / uvx       ───────────►   TCP 9090 (mặc định)
```

Hai máy nên nằm trong cùng LAN hoặc kết nối qua VPN riêng.

Không khuyến nghị NAT/public trực tiếp cổng `9090` ra Internet.

## 7. Quy trình sử dụng an toàn

### Mức 1 — chỉ đọc

Cho phép AI:

- liệt kê node/topic/service;
- đọc sensor;
- xem parameter;
- thu thập metadata.

Đây là chế độ phù hợp để khảo sát hệ thống và debug ban đầu.

### Mức 2 — thay đổi cấu hình

Cho phép AI:

- set parameter;
- gọi service không gây chuyển động;
- thay đổi cấu hình runtime.

Nên yêu cầu xác nhận trước khi thực thi.

### Mức 3 — điều khiển actuator

Bao gồm publish velocity, gửi action goal hoặc gọi service tạo chuyển động.

Cần có tối thiểu:

- human-in-the-loop;
- giới hạn vận tốc/gia tốc;
- watchdog;
- vùng hoạt động hợp lệ;
- collision protection;
- E-stop vật lý;
- log đầy đủ.

## 8. Ví dụ prompt tốt

### Khảo sát robot

```text
Kết nối tới robot tại 192.168.1.50:9090.
Chỉ thực hiện thao tác đọc. Hãy xác định ROS version, liệt kê node,
topic và service quan trọng, sau đó mô tả kiến trúc hệ thống.
```

### Chẩn đoán camera

```text
Không điều khiển actuator. Hãy tìm các topic camera,
xác định message type, đọc một frame nếu công cụ hỗ trợ
và cho biết pipeline camera hiện có hoạt động bình thường không.
```

### Điều khiển có xác nhận

```text
Tìm interface điều khiển vận tốc của robot nhưng chưa gửi lệnh.
Giải thích topic/service/action nào sẽ được dùng, message type,
giới hạn an toàn cần đặt và chờ tôi xác nhận trước khi thực thi.
```

## 9. Cài đặt

Xem [Hướng dẫn cài đặt tiếng Việt](cai-dat.md).

Tài liệu upstream chi tiết theo từng client vẫn được giữ tại [`docs/install/`](../install/) để bảo đảm theo sát thay đổi của dự án gốc.

## 10. Tài liệu kỹ thuật liên quan

- [Kiến trúc upstream](../architecture.md)
- [Kiểm thử](../testing.md)
- [Launch system](../launch_system.md)
- [Đóng góp](../contributing.md)
- [Ví dụ](../../examples/)

## 11. Quy ước thuật ngữ

Không Việt hóa literal trong code. Ví dụ:

```bash
ros2 topic list
ros2 service list
ros2 action list
```

Trong văn bản có thể viết “topic (kênh dữ liệu)”, nhưng không đổi `topic` thành tên khác trong lệnh hoặc API.

## 12. Ghi công

Repo này kế thừa mã nguồn và kiến trúc từ [robotmcp/ros-mcp-server](https://github.com/robotmcp/ros-mcp-server). Bản Việt hóa tập trung vào tài liệu và khả năng tiếp cận; quyền tác giả và giấy phép Apache-2.0 của dự án gốc được giữ nguyên.
