# viROS MCP Server 🧠 ⇄ 🤖

<p align="center">
  <strong>Cầu nối Model Context Protocol (MCP) giữa trợ lý AI và hệ sinh thái ROS/ROS 2 — tài liệu hóa chuyên nghiệp bằng tiếng Việt.</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/ROS-Hỗ%20trợ-2ea44f" alt="ROS" />
  <img src="https://img.shields.io/badge/ROS%202-Hỗ%20trợ-2ea44f" alt="ROS 2" />
  <img src="https://img.shields.io/badge/Python-3.10%2B-3776AB" alt="Python 3.10+" />
  <img src="https://img.shields.io/badge/MCP-Tương%20thích-7B61FF" alt="MCP compatible" />
  <img src="https://img.shields.io/badge/Giấy%20phép-Apache%202.0-blue" alt="Apache-2.0" />
  <img src="https://img.shields.io/github/stars/Base27-CVNSS/viros-mcp-server?style=social" alt="GitHub stars" />
</p>

> **viROS MCP Server** là bản Việt hóa/tài liệu hóa của dự án nguồn mở [robotmcp/ros-mcp-server](https://github.com/robotmcp/ros-mcp-server). Mục tiêu của repo này là giúp cộng đồng Việt Nam tiếp cận ROS + MCP dễ hơn mà **không thay đổi các định danh giao thức, tên package hay API cốt lõi** của dự án gốc.

<!-- mcp-name: io.github.robotmcp/ros-mcp-server -->

<p align="center">
  <img src="https://github.com/robotmcp/ros-mcp-server/blob/main/docs/images/framework.png?raw=1" alt="Kiến trúc ROS MCP Server" />
</p>

## 🧭 viROS MCP Server là gì?

ROS-MCP-Server kết nối các mô hình ngôn ngữ lớn như **ChatGPT, Claude, Gemini, Codex** với robot đang chạy **ROS 1 hoặc ROS 2**. AI có thể khám phá trạng thái hệ thống, đọc dữ liệu cảm biến và tương tác với robot thông qua các primitive chuẩn của ROS mà không yêu cầu sửa mã nguồn ứng dụng robot hiện có.

Luồng tổng quát:

```text
Người dùng
   │ ngôn ngữ tự nhiên
   ▼
Trợ lý AI / MCP Client
   │ Model Context Protocol
   ▼
ROS MCP Server
   │ WebSocket / rosbridge
   ▼
ROS / ROS 2
   │
   ├─ Topics
   ├─ Services
   ├─ Actions
   ├─ Parameters
   ├─ Nodes
   └─ Camera / Sensor data
```

### Bản chất kỹ thuật

MCP **không thay thế ROS**. MCP là lớp giao tiếp để mô hình AI gọi công cụ theo schema có cấu trúc. ROS-MCP-Server chuyển các yêu cầu đó sang rosbridge/ROS, sau đó chuẩn hóa kết quả trả về cho AI.

Điểm quan trọng là hệ thống giữ nguyên mô hình vận hành quen thuộc của ROS:

- `topic` dùng cho luồng dữ liệu publish/subscribe;
- `service` dùng cho yêu cầu–phản hồi;
- `action` dùng cho tác vụ dài hoặc có feedback;
- `parameter` dùng cho cấu hình runtime;
- `node` mô tả các tiến trình/thành phần đang hoạt động;
- `rosbridge` cung cấp lớp WebSocket trung gian để MCP server không cần chạy trực tiếp bên trong robot.

## ✨ Vì sao dùng ROS-MCP?

- **Không cần sửa mã nguồn robot** — thông thường chỉ cần bổ sung `rosbridge` vào hệ ROS hiện có.
- **Giao tiếp hai chiều** — AI có thể vừa quan sát trạng thái vừa thực hiện thao tác được cấp quyền.
- **Ngữ cảnh ROS đầy đủ** — khám phá topic, service, action, parameter, node và kiểu dữ liệu, kể cả interface tùy biến.
- **Điều khiển bằng ngôn ngữ tự nhiên** — chuyển ý định con người thành chuỗi thao tác ROS có cấu trúc.
- **Tương thích nhiều MCP client** — Claude Code/Desktop, Codex CLI, Gemini CLI, ChatGPT, Cursor và client MCP tùy biến.
- **Hỗ trợ nhiều thế hệ ROS** — dùng được với ROS 2 (Jazzy, Humble và các distro khác) cũng như ROS 1.
- **Không khóa nhà cung cấp mô hình** — kiến trúc MCP tách lớp robot khỏi LLM/client.

## 🧱 Kiến trúc 4 lớp

| Lớp | Vai trò |
|---|---|
| **AI / MCP Client** | Hiểu yêu cầu người dùng, chọn tool/resource phù hợp và diễn giải kết quả |
| **MCP Server** | Cung cấp schema tool, resource và quy tắc thao tác với ROS |
| **rosbridge** | Chuyển đổi giữa WebSocket/JSON và hệ giao tiếp ROS |
| **Robot / ROS Graph** | Node, topic, service, action, parameter, sensor và actuator thực tế |

Tài liệu kiến trúc chuyên sâu: [docs/architecture.md](docs/architecture.md).  
Tổng quan tiếng Việt: [docs/vi/README.md](docs/vi/README.md).

## 🚀 Bắt đầu nhanh

### 1. Chuẩn bị MCP client trên máy người dùng

Ví dụ với Claude Code:

```bash
# Cài uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Thêm ROS MCP Server vào Claude Code
claude mcp add ros-mcp -- uvx ros-mcp --transport=stdio
```

### 2. Cài rosbridge trên máy robot

```bash
sudo apt update
sudo apt install ros-<ros-distro>-rosbridge-server
```

Khởi chạy ROS 2:

```bash
source /<duong-dan-workspace>/install/setup.bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
```

### 3. Yêu cầu AI kết nối robot

Ví dụ prompt:

```text
Kết nối tới robot tại 192.168.1.50:9090. Hãy kiểm tra kết nối,
nhận diện phiên bản ROS, sau đó liệt kê các topic và service đang hoạt động.
Không thực hiện lệnh điều khiển actuator cho đến khi tôi xác nhận.
```

📘 Xem hướng dẫn đầy đủ: [docs/install/installation.md](docs/install/installation.md)

## 🖥️ Mô hình triển khai

Thông thường hệ thống gồm hai máy trong cùng mạng LAN:

```text
┌──────────────────────────────┐       WebSocket       ┌──────────────────────────┐
│ Laptop / Desktop             │ ◄───────────────────► │ Máy tính trên robot       │
│                              │      TCP :9090        │                          │
│ MCP Client + LLM             │                       │ ROS / ROS 2              │
│ ROS MCP Server               │                       │ rosbridge_server          │
└──────────────────────────────┘                       └──────────────────────────┘
```

Có thể chạy tất cả trên cùng một máy. Khi truy cập từ xa qua Internet nên ưu tiên **VPN/Tailscale/WireGuard** thay vì công khai cổng rosbridge trực tiếp ra Internet.

## 🧰 Năng lực chính

Tùy phiên bản upstream, MCP server có thể cung cấp các nhóm công cụ sau:

- kiểm tra kết nối và nhận diện ROS;
- khám phá ROS graph;
- đọc/ghi topic;
- gọi service;
- gửi và theo dõi action goal;
- đọc/thay đổi parameter;
- nhận ảnh camera và dữ liệu cảm biến;
- tải robot specification;
- truy xuất resource metadata để AI hiểu hệ robot trước khi thao tác.

> Tên tool, resource URI, package `ros-mcp` và namespace Python vẫn giữ nguyên tiếng Anh để bảo đảm tương thích kỹ thuật.

## 🎥 Ví dụ ứng dụng

### Robot thao tác di động trong NVIDIA Isaac Sim

Lệnh được nhập bằng ngôn ngữ tự nhiên; MCP client sử dụng ROS-MCP để điều khiển robot mô phỏng.

<p align="center">
  <img src="https://github.com/robotmcp/ros-mcp-server/blob/main/docs/images/result.gif?raw=1" alt="ROS MCP với NVIDIA Isaac Sim" />
</p>

### Unitree Go2

ROS-MCP có thể đưa dữ liệu camera vào ngữ cảnh của AI và gọi các giao diện ROS để robot phản hồi theo yêu cầu ngôn ngữ tự nhiên.  
[Video minh họa upstream](https://youtu.be/RW9_FgfxWzs?si=8bdhpHNYaupzi9q3)

### Chẩn đoán robot công nghiệp

AI có thể duyệt ROS graph, xác định interface tùy biến, đọc trạng thái và hỗ trợ kỹ sư kiểm thử/debug bằng ngôn ngữ tự nhiên. Việc cho phép gọi lệnh thay đổi trạng thái robot cần được kiểm soát bằng phân quyền và quy trình xác nhận của con người.

## 🔐 An toàn khi dùng AI với robot

Kết nối LLM với robot là bài toán **cyber-physical**, vì một tool call sai có thể gây chuyển động thật. Khuyến nghị:

1. chạy thử trên simulator trước;
2. phân tách tool chỉ đọc và tool thay đổi trạng thái;
3. yêu cầu xác nhận con người trước lệnh actuator nguy hiểm;
4. dùng network allowlist/firewall/VPN;
5. không public rosbridge `:9090` trực tiếp ra Internet;
6. cấu hình giới hạn vận tốc, vùng làm việc và emergency stop ở tầng robot;
7. lưu log MCP/tool call phục vụ truy vết;
8. coi prompt và nội dung sensor bên ngoài là dữ liệu không tin cậy.

## 📚 Tài liệu

| Nội dung | Liên kết |
|---|---|
| 🇻🇳 Trung tâm tài liệu tiếng Việt | [docs/vi/README.md](docs/vi/README.md) |
| Cài đặt | [docs/install/installation.md](docs/install/installation.md) |
| Rosbridge | [docs/install/rosbridge.md](docs/install/rosbridge.md) |
| Kết nối robot | [docs/install/connect.md](docs/install/connect.md) |
| Khắc phục sự cố | [docs/install/troubleshooting.md](docs/install/troubleshooting.md) |
| Kiến trúc | [docs/architecture.md](docs/architecture.md) |
| Kiểm thử | [docs/testing.md](docs/testing.md) |
| Ví dụ | [examples](examples) |

## 🌐 Thuật ngữ Việt hóa

| Thuật ngữ gốc | Cách dùng trong tài liệu Việt |
|---|---|
| topic | topic / kênh dữ liệu |
| publish | phát dữ liệu |
| subscribe | đăng ký nhận dữ liệu |
| service | dịch vụ ROS |
| action | tác vụ ROS có trạng thái/feedback |
| node | nút ROS |
| parameter | tham số runtime |
| rosbridge | cầu nối WebSocket ↔ ROS |
| tool | công cụ MCP |
| resource | tài nguyên MCP |
| prompt | chỉ dẫn/yêu cầu cho mô hình |

Các từ khóa dùng trực tiếp trong API, command line và code **không dịch**.

## 🤝 Đóng góp

Đóng góp được khuyến khích ở các hướng:

- sửa lỗi và cập nhật tài liệu;
- cải thiện bản dịch tiếng Việt;
- bổ sung ví dụ ROS/ROS 2 thực tế;
- kiểm thử robot/simulator phổ biến tại Việt Nam;
- tăng cường an toàn, permission và human-in-the-loop;
- bổ sung tài liệu cho sinh viên, phòng lab và kỹ sư tự động hóa.

Xem hướng dẫn upstream: [docs/contributing.md](docs/contributing.md).

## 🧬 Nguồn gốc và ghi công

Repo này dựa trên dự án nguồn mở **ROS MCP Server** của cộng đồng [robotmcp](https://github.com/robotmcp/ros-mcp-server). Các tác giả và thông tin package gốc được giữ nguyên trong `pyproject.toml` để bảo toàn lịch sử, attribution và khả năng tương thích.

Bản Việt hóa trong repo `Base27-CVNSS/viros-mcp-server` tập trung vào lớp tài liệu, diễn giải kỹ thuật và trải nghiệm tiếp cận cho cộng đồng Việt Nam.

## 📜 Giấy phép

Dự án sử dụng [Apache License 2.0](LICENSE). Khi phân phối hoặc sửa đổi, cần giữ nguyên các thông báo bản quyền và điều khoản giấy phép áp dụng từ dự án gốc.
