# Hướng dẫn cài đặt viROS MCP Server

Hướng dẫn này tập trung vào luồng cài đặt phổ biến: **MCP client + ROS MCP Server chạy trên laptop/desktop**, còn **rosbridge chạy trên máy robot**.

## 1. Điều kiện cần

### Máy người dùng

- Python 3.10 trở lên;
- một MCP client tương thích;
- tài khoản/model AI phù hợp với client;
- kết nối mạng tới máy robot.

### Máy robot

- ROS 1 hoặc ROS 2 đã hoạt động;
- `rosbridge_server`;
- các node/topic/service/action cần dùng đã được launch.

## 2. Kiến trúc mạng

```text
Laptop / Desktop                         Robot
────────────────────                    ─────────────────────
Claude / Codex / Gemini                 ROS / ROS 2
ChatGPT / Cursor                        rosbridge_server
        │                                       ▲
        ▼ MCP                                   │ ROS graph
ROS MCP Server ───── WebSocket :9090 ───────────┘
```

Nếu hai thành phần chạy cùng máy, có thể dùng `127.0.0.1:9090`.

Nếu chạy khác máy, dùng địa chỉ IP LAN của robot, ví dụ `192.168.1.50:9090`.

## 3. Cài ROS MCP Server

### Cách khuyến nghị: `uvx`

Cài `uv`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Kiểm tra:

```bash
uv --version
```

ROS MCP Server có entry point `ros-mcp`. MCP client có thể chạy trực tiếp bằng:

```bash
uvx ros-mcp --transport=stdio
```

### Cài bằng pip

```bash
python -m pip install ros-mcp
```

Sau đó:

```bash
ros-mcp --help
```

### Chạy từ source

```bash
git clone https://github.com/Base27-CVNSS/viros-mcp-server.git
cd viros-mcp-server
```

Nếu dùng `uv`:

```bash
uv sync
uv run ros-mcp --help
```

> Repo Việt hóa giữ nguyên tên package/entry point `ros-mcp` để tương thích với upstream và MCP registry.

## 4. Cấu hình MCP client

### Claude Code

```bash
claude mcp add ros-mcp -- uvx ros-mcp --transport=stdio
```

Khởi động lại Claude Code nếu cần, sau đó kiểm tra server đã xuất hiện trong danh sách MCP.

### Client khác

Tài liệu cấu hình chi tiết cho từng client nằm tại:

- [Claude Code](../install/clients/claude-code.md)
- [Codex CLI](../install/clients/codex-cli.md)
- [Gemini CLI](../install/clients/gemini-cli.md)
- [Claude Desktop](../install/clients/claude-desktop.md)
- [ChatGPT](../install/clients/chatgpt.md)
- [Cursor](../install/clients/cursor.md)
- [Robot MCP Client](../install/clients/robot-mcp-client.md)
- [Custom MCP Client](../install/clients/custom-client.md)

Các file cấu hình thường yêu cầu command tương đương:

```text
uvx ros-mcp --transport=stdio
```

## 5. Cài rosbridge trên robot

### ROS 2

Thay `<ros-distro>` bằng distro đang dùng, ví dụ `humble` hoặc `jazzy`:

```bash
sudo apt update
sudo apt install ros-<ros-distro>-rosbridge-server
```

Source môi trường ROS/workspace:

```bash
source /opt/ros/<ros-distro>/setup.bash
```

Nếu có workspace riêng:

```bash
source ~/ros_ws/install/setup.bash
```

Khởi chạy:

```bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
```

Mặc định rosbridge thường lắng nghe ở cổng `9090`.

### ROS 1

Cài package phù hợp với distro ROS 1 rồi launch `rosbridge_websocket` theo tài liệu distro tương ứng. Xem tài liệu upstream tại [rosbridge.md](../install/rosbridge.md) để theo sát tên package và launch command mới nhất trong repo.

## 6. Xác định IP của robot

Trên Linux:

```bash
hostname -I
```

hoặc:

```bash
ip addr
```

Ví dụ robot có IP:

```text
192.168.1.50
```

MCP server sẽ kết nối tới:

```text
ws://192.168.1.50:9090
```

## 7. Kiểm tra mạng trước khi dùng AI

Từ laptop:

```bash
ping 192.168.1.50
```

Kiểm tra port trên Linux/macOS:

```bash
nc -vz 192.168.1.50 9090
```

Trên Windows PowerShell:

```powershell
Test-NetConnection 192.168.1.50 -Port 9090
```

Nếu ping được nhưng port đóng, kiểm tra:

- rosbridge đã launch chưa;
- firewall trên robot;
- robot và laptop có cùng subnet/VPN hay không;
- rosbridge có bind vào interface phù hợp hay không.

## 8. Kết nối bằng ngôn ngữ tự nhiên

Sau khi MCP client nhận server, dùng prompt theo thứ tự an toàn:

```text
Kết nối tới robot tại 192.168.1.50:9090.
Chỉ dùng các thao tác đọc.
1. Kiểm tra kết nối.
2. Nhận diện ROS version.
3. Liệt kê node, topic và service.
4. Tóm tắt trạng thái hệ thống.
Không publish, set parameter, gọi service thay đổi trạng thái hoặc gửi action goal.
```

Nếu kết nối ổn, mới chuyển sang thao tác điều khiển.

## 9. Kiểm tra ROS phía robot

### ROS 2

```bash
ros2 node list
ros2 topic list
ros2 service list
ros2 action list
```

Kiểm tra topic cụ thể:

```bash
ros2 topic info /ten_topic
```

Kiểm tra kiểu message:

```bash
ros2 interface show <package/msg/Type>
```

### ROS 1

```bash
rosnode list
rostopic list
rosservice list
```

## 10. Lỗi thường gặp

### MCP client không thấy server

Kiểm tra:

```bash
uvx ros-mcp --help
```

Nếu command này lỗi, xử lý Python/package trước khi debug ROS.

### Không kết nối được `:9090`

Kiểm tra rosbridge, firewall và địa chỉ IP.

### AI thấy topic nhưng gọi sai message

Yêu cầu AI đọc metadata/interface trước khi publish. Không nên để mô hình đoán field của message tùy biến.

### Robot không phản ứng dù publish thành công

Có thể nguyên nhân nằm ở tầng robot:

- controller chưa active;
- topic không phải interface điều khiển cuối cùng;
- lifecycle node chưa chuyển trạng thái;
- hardware safety/controller hierarchy chặn lệnh;
- QoS hoặc ROS domain khác nhau;
- robot yêu cầu service/action thay vì topic.

### Camera không hiển thị

Kiểm tra message type, encoding, kích thước frame, compressed/raw topic và băng thông mạng.

Xem thêm [troubleshooting upstream](../install/troubleshooting.md).

## 11. Kết nối qua Internet

Không nên mở trực tiếp TCP `9090` ra Internet.

Ưu tiên:

- WireGuard;
- Tailscale;
- VPN site-to-site;
- SSH tunnel trong môi trường kiểm soát.

Sau khi tạo mạng riêng, MCP server có thể dùng IP VPN của robot giống như IP LAN.

## 12. Checklist trước khi điều khiển robot thật

- [ ] Đã thử trên simulator hoặc robot ở chế độ an toàn.
- [ ] Có E-stop hoạt động.
- [ ] Có giới hạn vận tốc/gia tốc ở controller.
- [ ] MCP client hiểu tool nào chỉ đọc và tool nào thay đổi trạng thái.
- [ ] Không public rosbridge trực tiếp ra Internet.
- [ ] Có người giám sát khi thử lệnh chuyển động.
- [ ] Đã kiểm tra topic/service/action type trước khi gửi dữ liệu.
- [ ] Có log để truy vết tool call và lỗi.

## 13. Bước tiếp theo

- [Trung tâm tài liệu tiếng Việt](README.md)
- [Kiến trúc upstream](../architecture.md)
- [Ví dụ robot](../../examples/)
- [Tài liệu cài đặt upstream](../install/installation.md)
