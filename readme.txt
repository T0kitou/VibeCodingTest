# Python 抓取 B 站直播弹幕并常驻桌面显示工作流

这份文档描述了如何使用 Python 开发一个抓取 Bilibili 直播间弹幕，并将其以透明窗口形式常驻桌面显示的应用程序。

## 1. 环境准备 (Environment Setup)

需要安装 Python 以及以下第三方库：

*   **Python 版本**: 建议 Python 3.8+
*   **网络请求与 WebSocket**: 用于连接 B 站弹幕服务器
    *   `aiohttp`: 异步 HTTP 请求（推荐）
    *   `brotli`: 用于解压 B 站的 WebSocket 数据包
*   **GUI 界面**: 用于在桌面显示弹幕
    *   `PyQt5` 或 `PySide6`: 功能强大的 GUI 库，支持透明背景和窗口置顶
    *   (备选) `tkinter`: Python 自带，但处理透明背景较为麻烦

### 安装命令
```bash
pip install aiohttp brotli PyQt5
```

## 2. 核心模块设计

### 模块 A: 弹幕获取 (Danmaku Fetcher)
B 站直播弹幕使用 WebSocket 协议。

1.  **获取真实 RoomID**: B 站 URL 中的房间号可能是短号，需要通过 API (https://api.live.bilibili.com/room/v1/Room/room_init?id={short_id}) 获取真实 `room_id`。
2.  **获取 WebSocket 地址**: 调用 API 获取直播间 WS 地址和认证 Token。
3.  **建立连接**:
    *   发送认证包 (Auth Packet)。
    *   发送心跳包 (Heartbeat Packet): 每 30 秒发送一次，防止断连。
4.  **数据解析**:
    *   接收二进制数据。
    *   协议头部解析 (Header Protocol)。
    *   数据解压 (通常使用 Brotli 或 Zlib)。
    *   JSON 解析提取弹幕内容 (`cmd` 为 `DANMU_MSG`)。

### 模块 B: 桌面 UI 显示 (Desktop Overlay)
使用 PyQt5 实现一个透明、穿透鼠标的悬浮窗。

1.  **窗口属性设置**:
    *   `FramelessWindowHint`: 无边框。
    *   `WindowStaysOnTopHint`: 窗口置顶。
    *   `WA_TranslucentBackground`: 背景透明。
    *   (可选) 鼠标穿透: 设置窗口样式使得鼠标点击穿透到下方应用。
2.  **弹幕渲染**:
    *   使用 `QLabel` 或自定义绘制 (`QPainter`) 显示文本。
    *   实现滚动效果或淡入淡出效果。
    *   设置字体描边或阴影，确保在不同背景色下清晰可见。

## 3. 开发步骤 (Workflow)

1.  **初始化项目**: 创建 `main.py`, `danmaku_client.py`, `overlay_window.py`。
2.  **编写 WebSocket 客户端**:
    *   实现连接、发送心跳、接收消息循环。
    *   封装回调函数，当收到新弹幕时触发。
3.  **编写 GUI 窗口**:
    *   创建一个 PyQt 窗口。
    *   配置透明和置顶属性。
    *   定义一个 `add_danmaku(text)` 方法更新 UI。
4.  **整合**:
    *   在 `main.py` 中启动 GUI 线程。
    *   使用 `QThread` 或 `asyncio` 集成 WebSocket 客户端，避免阻塞 UI 线程。
    *   将收到的弹幕通过信号 (Signal/Slot) 传递给 GUI 进行显示。

## 4. 示例代码片段 (伪代码)

```python
# 伪代码逻辑示意
import sys
from PyQt5.QtWidgets import QApplication, QLabel, QWidget
from PyQt5.QtCore import Qt

class DanmakuWindow(QWidget):
    def __init__(self):
        super().__init__()
        # 设置无边框、置顶、透明
        self.setWindowFlags(Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint | Qt.Tool)
        self.setAttribute(Qt.WA_TranslucentBackground)
        
        self.label = QLabel("等待弹幕...", self)
        self.label.setStyleSheet("color: white; font-size: 20px; font-weight: bold;")
    
    def update_text(self, text):
        self.label.setText(text)
        self.label.adjustSize()

# 主流程
if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = DanmakuWindow()
    window.show()
    
    # 在这里启动 WebSocket 线程连接 B 站
    # on_message_received -> window.update_text(msg)
    
    sys.exit(app.exec_())
```

## 5. 进阶功能
*   **过滤规则**: 屏蔽特定关键词或等级较低的用户。
*   **样式自定义**: 允许用户修改字体大小、颜色、透明度。
*   **历史记录**: 保存弹幕到本地日志文件。

## 6. 参考资料
*   Bilibili Live API 文档 (非官方)
*   PyQt5 官方文档
