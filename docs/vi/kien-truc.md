# Kiến trúc viROS MCP Server

Tài liệu này diễn giải kiến trúc `ros-mcp` bằng tiếng Việt, bám theo cấu trúc hiện tại của dự án gốc. Mục tiêu là giúp người đọc hiểu **dòng dữ liệu, ranh giới trách nhiệm và điểm mở rộng** trước khi chỉnh sửa code hoặc kết nối robot thật.

## 1. Tổng quan

ROS MCP Server là một MCP server xây dựng trên **FastMCP**, cung cấp `tools`, `resources` và `prompts` để một MCP client tương tác với hệ ROS thông qua **rosbridge WebSocket**.

```text
┌──────────────────────┐
│ AI / MCP Client      │
│ Claude / GPT / ...   │
└──────────┬───────────┘
           │ MCP
           ▼
┌──────────────────────┐
│ FastMCP Server       │
│ ros_mcp/main.py      │
├──────────────────────┤
│ Tools                │
│ Resources            │
│ Prompts              │
└──────────┬───────────┘
           │ WebSocket
           ▼
┌──────────────────────┐
│ rosbridge_server     │
└──────────┬───────────┘
           │ ROS API
           ▼
┌──────────────────────┐
│ ROS / ROS 2 Graph    │
│ Robot / Simulator    │
└──────────────────────┘
```

## 2. Nguyên tắc thiết kế

Kiến trúc upstream theo năm nguyên tắc chính:

1. **Modular Design** — chia tool/resource/prompt thành module theo chức năng;
2. **Separation of Concerns** — tách lớp giao tiếp, tài nguyên, prompt và utility;
3. **Reusability** — dùng chung WebSocket manager và utility;
4. **Extensibility** — có pattern rõ ràng để thêm tool/resource/prompt;
5. **Library-First** — có thể import `ros_mcp` vào MCP server khác thay vì chỉ chạy như CLI độc lập.

## 3. Cấu trúc package

```text
ros_mcp/
├── main.py
├── tools/
│   ├── actions.py
│   ├── connection.py
│   ├── images.py
│   ├── nodes.py
│   ├── parameters.py
│   ├── robot_config.py
│   ├── services.py
│   └── topics.py
├── resources/
│   ├── robot_specs.py
│   └── ros_metadata.py
├── prompts/
│   ├── test_actions_tools.py
│   ├── test_connection_tools.py
│   ├── test_nodes_tools.py
│   ├── test_parameters_tools.py
│   ├── test_server_tools.py
│   ├── test_services_tools.py
│   └── test_topics_tools.py
└── utils/
    ├── config_utils.py
    ├── network_utils.py
    └── websocket.py
```

Ngoài package chính còn có:

```text
server.py
robot_specifications/
docs/
examples/
```

## 4. Entry point

`ros_mcp/main.py` chịu trách nhiệm:

- tạo instance `FastMCP`;
- khởi tạo `WebSocketManager`;
- đăng ký toàn bộ tool;
- đăng ký resource;
- đăng ký prompt;
- cung cấp entry point cho package `ros-mcp`.

Mẫu kiến trúc:

```python
mcp = FastMCP("ros-mcp-server")
ws_manager = WebSocketManager(
    ROSBRIDGE_IP,
    ROSBRIDGE_PORT,
    default_timeout=5.0,
)

register_all_tools(mcp, ws_manager, ...)
register_all_resources(mcp, ws_manager)
register_all_prompts(mcp)
```

## 5. Tool layer

Tool là giao diện hành động mà LLM có thể gọi qua MCP.

Theo tài liệu kiến trúc upstream hiện tại có **31 tool** thuộc tám nhóm:

| Nhóm | Số lượng | Vai trò |
|---|---:|---|
| Connection | 2 | kết nối và kiểm tra mạng/robot |
| Robot Config | 3 | robot specification và nhận diện ROS |
| Topics | 8 | khám phá, subscribe, publish topic |
| Services | 4 | khám phá và gọi service |
| Nodes | 2 | khám phá/kiểm tra node |
| Parameters | 6 | quản lý parameter, chủ yếu ROS 2 |
| Actions | 5 | khám phá và thực thi action, ROS 2 |
| Images | 1 | xử lý/hiển thị ảnh |

### Tool wrapper

Một tool thường gồm:

```python
@mcp.tool(description="...")
def tool_name(param1: str, param2: int = 0) -> dict:
    with ws_manager:
        # giao tiếp rosbridge
        ...
    return {"result": ...}
```

Phần `description` rất quan trọng vì LLM dùng nó để quyết định **khi nào nên gọi tool**. Docstring lại phục vụ developer và IDE.

### Tool annotation và an toàn

Trong kiến trúc MCP hiện đại, annotation có thể giúp client nhận biết một tool thiên về đọc hay có khả năng thay đổi trạng thái. Tuy nhiên annotation **không phải cơ chế an toàn vật lý**. Lớp robot vẫn phải có giới hạn, interlock, watchdog và E-stop.

## 6. Resource layer

Resource cung cấp dữ liệu ngữ cảnh dạng đọc, phù hợp để LLM tìm hiểu hệ thống trước khi hành động.

Một số URI được tài liệu upstream mô tả:

```text
ros-mcp://ros-metadata/all
ros-mcp://ros-metadata/topics/all
ros-mcp://ros-metadata/services/all
ros-mcp://ros-metadata/nodes/all
ros-mcp://ros-metadata/actions/all
ros-mcp://robot-specs/get_verified_robots_list
```

Lợi ích của resource:

- tách dữ liệu mô tả khỏi action;
- giảm nguy cơ dùng tool thay đổi trạng thái chỉ để lấy metadata;
- cho phép client tạo context ROS trước khi lập kế hoạch.

## 7. Prompt layer

Prompt trong repo chủ yếu là hướng dẫn tương tác/kiểm thử, ví dụ:

```text
test-server-tools
test-connection-tools
test-topics-tools
test-services-tools
test-nodes-tools
test-parameters-tools
test-actions-tools
```

Prompt ở đây không phải “hệ điều khiển an toàn”. Nó là hướng dẫn để client/LLM kiểm thử tool có hệ thống.

## 8. Utility layer

### `utils/websocket.py`

`WebSocketManager` chịu trách nhiệm:

- quản lý vòng đời kết nối rosbridge;
- gửi request/nhận response;
- xử lý timeout/lỗi;
- cung cấp context manager cho tool/resource.

Đây là lớp quan trọng để tránh mỗi tool tự triển khai WebSocket riêng.

### `utils/network_utils.py`

Phục vụ:

- ping host;
- kiểm tra port;
- xử lý khác biệt nền tảng khi kiểm tra mạng.

### `utils/config_utils.py`

Phục vụ:

- đọc robot specification YAML;
- parse/validate cấu hình;
- liệt kê specification đã biết.

## 9. Dòng dữ liệu một lệnh điển hình

Ví dụ người dùng yêu cầu “liệt kê topic camera”:

```text
1. User gửi yêu cầu
2. LLM xác định cần lấy ROS metadata
3. MCP client gọi resource/tool phù hợp
4. ROS MCP Server tạo request rosbridge
5. WebSocketManager gửi JSON tới rosbridge
6. rosbridge truy vấn ROS graph
7. Kết quả trả về MCP server
8. MCP server chuẩn hóa response
9. LLM giải thích danh sách topic cho user
```

Nếu người dùng yêu cầu publish velocity, bước 3 chuyển từ đọc metadata sang tool có khả năng thay đổi trạng thái. Đây là nơi cần human-in-the-loop.

## 10. Tích hợp như thư viện

`ros_mcp` được thiết kế để đăng ký vào MCP server khác:

```python
from ros_mcp.tools import register_all_tools
from ros_mcp.resources import register_all_resources
from ros_mcp.prompts import register_all_prompts
from ros_mcp.utils.websocket import WebSocketManager

mcp = FastMCP("my-server")
ws_manager = WebSocketManager("127.0.0.1", 9090)

register_all_tools(mcp, ws_manager, rosbridge_ip="127.0.0.1", rosbridge_port=9090)
register_all_resources(mcp, ws_manager)
register_all_prompts(mcp)
```

Điều này phù hợp với kiến trúc agent lớn hơn, trong đó ROS chỉ là một subsystem bên cạnh database, vision, planning hoặc enterprise tools.

## 11. Điểm mở rộng

### Thêm tool

1. đặt implementation vào module đúng nhóm;
2. thêm `@mcp.tool`;
3. mô tả input/output rõ ràng;
4. đăng ký trong `register_*_tools()`;
5. kiểm thử với simulator và schema của MCP client.

### Thêm resource

1. thêm function resource;
2. chọn URI ổn định;
3. dùng `@mcp.resource`;
4. đăng ký trong resource registry;
5. ưu tiên dữ liệu chỉ đọc và có cấu trúc.

### Thêm prompt

1. tạo prompt mới trong `prompts/`;
2. dùng `@mcp.prompt`;
3. đăng ký vào `register_all_prompts()`.

## 12. Ranh giới an toàn kiến trúc

Không nên coi LLM là real-time controller.

Kiến trúc phù hợp hơn:

```text
LLM / Agent
   │ ý định cấp cao
   ▼
MCP + Policy Gate
   │ command đã validate
   ▼
ROS Planner / Controller
   │ control loop xác định
   ▼
Actuator
```

Các vòng điều khiển yêu cầu tần số cao, deadline nghiêm ngặt hoặc safety-certified nên nằm ở ROS controller/PLC/MCU, không đặt trong vòng suy luận LLM.

## 13. Phụ thuộc chính

- FastMCP;
- `websocket-client`;
- OpenCV/Pillow cho dữ liệu ảnh;
- JSON schema/tiện ích Python liên quan;
- `rosbridge_server` chạy ngoài package trên hệ ROS.

Chi tiết version cụ thể nên lấy từ `pyproject.toml` của repo hiện tại.

## 14. Tài liệu liên quan

- [Trung tâm tiếng Việt](README.md)
- [Cài đặt tiếng Việt](cai-dat.md)
- [Kiến trúc upstream](../architecture.md)
- [Kiểm thử upstream](../testing.md)
- [Launch system](../launch_system.md)
