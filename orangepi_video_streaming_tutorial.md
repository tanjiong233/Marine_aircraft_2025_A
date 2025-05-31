# 香橙派Zero 2W Python图传完整教程

## 前置准备

### 硬件要求
- 香橙派Zero 2W开发板
- Micro SD卡（至少16GB，Class 10）
- USB摄像头或CSI摄像头
- 5V/2A USB-C电源适配器
- 网线（可选，用于有线连接）

### 软件要求
- Ubuntu 22.04 Server版镜像（无桌面版）
- SD卡烧录工具（如Balena Etcher）

---

## 第一阶段：系统安装和基础配置

### 1.1 烧录系统镜像

**步骤：**
1. 下载香橙派Zero 2W专用的Ubuntu 22.04镜像
```bash
# 官方下载地址示例
wget http://www.orangepi.org/downloadresources/orangepizero2w/ubuntu22.04/ubuntu-server-22.04.img.xz
```

2. 使用Balena Etcher烧录到SD卡

**验证方法：**
- 烧录完成后，SD卡应显示两个分区：boot和rootfs
- 检查文件完整性：`ls -la /media/用户名/boot/`

### 1.2 首次启动和基础配置

**步骤：**
1. 插入SD卡，连接显示器和键盘
2. 上电启动（首次启动约需2-3分钟）
3. 默认登录信息：
   - 用户名：orangepi
   - 密码：orangepi

**验证方法：**
```bash
# 检查系统信息
cat /etc/os-release
uname -a
```
预期输出包含："Ubuntu 22.04" 和 "aarch64"

### 1.3 更新系统密码和配置

**步骤：**
```bash
# 修改默认密码
passwd

# 更新系统包列表
sudo apt update
```

**验证方法：**
```bash
# 检查更新状态
sudo apt list --upgradable
```

---

## 第二阶段：网络连接配置

### 2.1 WiFi连接设置

**步骤：**
```bash
# 1. 扫描可用WiFi网络
sudo iwlist wlan0 scan | grep ESSID

# 2. 编辑网络配置文件
sudo nano /etc/netplan/01-network-manager-all.yaml
```

**配置文件内容：**
```yaml
network:
  version: 2
  renderer: NetworkManager
  wifis:
    wlan0:
      dhcp4: true
      access-points:
        "你的WiFi名称":
          password: "你的WiFi密码"
```

**应用配置：**
```bash
# 应用网络配置
sudo netplan apply

# 重启网络服务
sudo systemctl restart NetworkManager
```

**验证方法：**
```bash
# 检查网络状态
ip addr show wlan0
ping -c 4 8.8.8.8

# 检查网络速度
wget -O /dev/null http://speedtest.wdc01.softlayer.com/downloads/test10.zip
```
预期结果：应显示IP地址且能ping通外网

### 2.2 有线网络配置（使用扩展板）

**步骤：**
```bash
# 检查以太网接口
ip link show

# 配置有线网络
sudo nano /etc/netplan/01-network-manager-all.yaml
```

**添加以太网配置：**
```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eth0:
      dhcp4: true
```

**验证方法：**
```bash
# 检查有线连接
ethtool eth0
ping -I eth0 -c 4 8.8.8.8
```

---

## 第三阶段：摄像头硬件配置

### 3.1 USB摄像头配置

**步骤：**
```bash
# 1. 安装摄像头相关包
sudo apt install v4l-utils ffmpeg

# 2. 检测摄像头设备
lsusb
ls /dev/video*
```

**验证方法：**
```bash
# 查看摄像头信息
v4l2-ctl --list-devices
v4l2-ctl --device=/dev/video0 --list-formats-ext

# 测试摄像头捕获
ffmpeg -f v4l2 -i /dev/video0 -t 5 -y test.mp4
ls -la test.mp4
```
预期结果：应生成5秒的测试视频文件

### 3.2 CSI摄像头配置（如果使用）

**步骤：**
```bash
# 检查CSI摄像头模块
sudo modprobe sun6i_csi
dmesg | grep csi

# 检查设备节点
ls /dev/video*
```

**验证方法：**
```bash
# 测试CSI摄像头
v4l2-ctl --device=/dev/video0 --list-formats
```

---

## 第四阶段：Python环境配置

### 4.1 安装Python和依赖包

**步骤：**
```bash
# 1. 确认Python版本
python3 --version

# 2. 安装pip
sudo apt install python3-pip python3-dev

# 3. 安装系统依赖
sudo apt install build-essential cmake pkg-config
sudo apt install libjpeg-dev libtiff5-dev libjasper-dev libpng-dev
sudo apt install libavcodec-dev libavformat-dev libswscale-dev libv4l-dev
sudo apt install libxvidcore-dev libx264-dev libgtk-3-dev
sudo apt install libatlas-base-dev gfortran
```

**验证方法：**
```bash
# 检查安装状态
pip3 --version
gcc --version
pkg-config --version
```

### 4.2 安装OpenCV和图传相关库

**步骤：**
```bash
# 1. 升级pip
python3 -m pip install --upgrade pip

# 2. 安装OpenCV
pip3 install opencv-python opencv-contrib-python

# 3. 安装图传相关库
pip3 install flask flask-socketio
pip3 install numpy pillow
pip3 install uvloop  # 异步IO优化
```

**验证方法：**
```bash
# 测试OpenCV安装
python3 -c "import cv2; print(cv2.__version__)"

# 测试Flask安装
python3 -c "import flask; print(flask.__version__)"

# 测试摄像头调用
python3 -c "
import cv2
cap = cv2.VideoCapture(0)
if cap.isOpened():
    print('摄像头连接成功')
    ret, frame = cap.read()
    if ret:
        print(f'成功捕获帧，尺寸: {frame.shape}')
    cap.release()
else:
    print('摄像头连接失败')
"
```

---

## 第五阶段：图传代码实现

### 5.1 基础MJPEG流式传输

**创建基础图传脚本：**
```bash
# 创建项目目录
mkdir ~/video_streaming
cd ~/video_streaming

# 创建主程序文件
nano stream_server.py
```

**stream_server.py内容：**
```python
#!/usr/bin/env python3
import cv2
import threading
import time
from flask import Flask, Response, render_template_string
import socket

class VideoStreamer:
    def __init__(self, src=0, width=640, height=480, fps=30):
        self.src = src
        self.width = width
        self.height = height
        self.fps = fps
        self.frame = None
        self.thread = None
        self.running = False
        
    def start(self):
        if self.thread is None:
            self.running = True
            self.thread = threading.Thread(target=self.update)
            self.thread.daemon = True
            self.thread.start()
            return self
        
    def update(self):
        cap = cv2.VideoCapture(self.src)
        cap.set(cv2.CAP_PROP_FRAME_WIDTH, self.width)
        cap.set(cv2.CAP_PROP_FRAME_HEIGHT, self.height)
        cap.set(cv2.CAP_PROP_FPS, self.fps)
        
        while self.running:
            ret, frame = cap.read()
            if ret:
                self.frame = frame.copy()
            time.sleep(1/self.fps)
            
        cap.release()
        
    def read(self):
        return self.frame
        
    def stop(self):
        self.running = False
        if self.thread is not None:
            self.thread.join()

app = Flask(__name__)
vs = VideoStreamer(src=0, width=640, height=480, fps=30)

def generate_frames():
    while True:
        frame = vs.read()
        if frame is not None:
            # 编码为JPEG
            ret, buffer = cv2.imencode('.jpg', frame, 
                                     [cv2.IMWRITE_JPEG_QUALITY, 85])
            if ret:
                frame_bytes = buffer.tobytes()
                yield (b'--frame\r\n'
                       b'Content-Type: image/jpeg\r\n\r\n' + frame_bytes + b'\r\n')
        time.sleep(0.033)  # 约30fps

@app.route('/')
def index():
    # 获取本机IP地址
    hostname = socket.gethostname()
    local_ip = socket.gethostbyname(hostname)
    
    return render_template_string('''
    <!DOCTYPE html>
    <html>
    <head>
        <title>香橙派图传</title>
        <style>
            body { font-family: Arial, sans-serif; text-align: center; }
            .info { margin: 20px; }
            img { border: 2px solid #333; }
        </style>
    </head>
    <body>
        <h1>香橙派Zero 2W 实时图传</h1>
        <div class="info">
            <p>访问地址: http://{{ ip }}:5000</p>
            <p>延迟: <span id="latency">检测中...</span></p>
        </div>
        <img src="{{ url_for('video_feed') }}" width="640" height="480">
        
        <script>
            // 简单延迟检测
            setInterval(function() {
                var start = Date.now();
                var img = new Image();
                img.onload = function() {
                    var latency = Date.now() - start;
                    document.getElementById('latency').textContent = latency + 'ms';
                };
                img.src = "{{ url_for('video_feed') }}?" + start;
            }, 1000);
        </script>
    </body>
    </html>
    ''', ip=local_ip)

@app.route('/video_feed')
def video_feed():
    return Response(generate_frames(),
                    mimetype='multipart/x-mixed-replace; boundary=frame')

if __name__ == '__main__':
    print("启动视频流服务器...")
    vs.start()
    time.sleep(2)  # 等待摄像头初始化
    
    # 获取并显示IP地址
    hostname = socket.gethostname()
    local_ip = socket.gethostbyname(hostname)
    print(f"图传服务器已启动！")
    print(f"本地访问: http://localhost:5000")
    print(f"网络访问: http://{local_ip}:5000")
    
    try:
        app.run(host='0.0.0.0', port=5000, debug=False, threaded=True)
    except KeyboardInterrupt:
        print("\n正在关闭服务器...")
        vs.stop()
```

**验证方法：**
```bash
# 运行图传服务器
cd ~/video_streaming
python3 stream_server.py

# 检查服务是否启动
netstat -an | grep 5000

# 测试本地访问
curl -I http://localhost:5000
```

### 5.2 高级功能实现

**创建增强版图传脚本：**
```bash
nano advanced_stream.py
```

**advanced_stream.py内容：**
```python
#!/usr/bin/env python3
import cv2
import threading
import time
import json
import base64
from flask import Flask, Response, render_template_string, request, jsonify
from flask_socketio import SocketIO, emit
import socket
import psutil
import subprocess

class AdvancedVideoStreamer:
    def __init__(self, src=0):
        self.src = src
        self.cap = None
        self.frame = None
        self.thread = None
        self.running = False
        self.fps = 30
        self.quality = 85
        self.resolution = (640, 480)
        self.frame_count = 0
        self.start_time = time.time()
        
    def start(self):
        if self.thread is None:
            self.running = True
            self.thread = threading.Thread(target=self.update)
            self.thread.daemon = True
            self.thread.start()
            time.sleep(1)  # 等待初始化
            return self
        
    def update(self):
        self.cap = cv2.VideoCapture(self.src)
        self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, self.resolution[0])
        self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, self.resolution[1])
        self.cap.set(cv2.CAP_PROP_FPS, self.fps)
        
        while self.running:
            ret, frame = self.cap.read()
            if ret:
                # 添加时间戳和帧计数
                timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
                cv2.putText(frame, f"Frame: {self.frame_count}", 
                           (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
                cv2.putText(frame, timestamp, 
                           (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
                
                self.frame = frame.copy()
                self.frame_count += 1
            time.sleep(1/self.fps)
            
        if self.cap:
            self.cap.release()
        
    def read(self):
        return self.frame
        
    def get_stats(self):
        elapsed = time.time() - self.start_time
        actual_fps = self.frame_count / elapsed if elapsed > 0 else 0
        return {
            'fps': round(actual_fps, 2),
            'frames': self.frame_count,
            'resolution': self.resolution,
            'quality': self.quality
        }
        
    def set_resolution(self, width, height):
        self.resolution = (width, height)
        if self.cap:
            self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, width)
            self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, height)
            
    def set_quality(self, quality):
        self.quality = max(10, min(100, quality))
        
    def stop(self):
        self.running = False
        if self.thread is not None:
            self.thread.join()

app = Flask(__name__)
app.config['SECRET_KEY'] = 'orangepi_streaming'
socketio = SocketIO(app, cors_allowed_origins="*")
vs = AdvancedVideoStreamer(src=0)

def get_system_info():
    """获取系统信息"""
    cpu_percent = psutil.cpu_percent(interval=1)
    memory = psutil.virtual_memory()
    disk = psutil.disk_usage('/')
    
    # 获取网络信息
    try:
        result = subprocess.run(['iwconfig'], capture_output=True, text=True)
        wifi_info = "已连接" if "ESSID" in result.stdout else "未连接"
    except:
        wifi_info = "未知"
    
    # 获取温度信息
    try:
        with open('/sys/class/thermal/thermal_zone0/temp', 'r') as f:
            temp = int(f.read()) / 1000
    except:
        temp = 0
    
    return {
        'cpu_percent': cpu_percent,
        'memory_percent': memory.percent,
        'disk_percent': disk.percent,
        'wifi_status': wifi_info,
        'temperature': temp
    }

@app.route('/')
def index():
    hostname = socket.gethostname()
    try:
        local_ip = socket.gethostbyname(hostname)
    except:
        local_ip = "获取中..."
    
    return render_template_string('''
    <!DOCTYPE html>
    <html>
    <head>
        <title>香橙派高级图传系统</title>
        <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.0.0/socket.io.js"></script>
        <style>
            body { 
                font-family: Arial, sans-serif; 
                margin: 0; 
                background: #f0f0f0; 
            }
            .header { 
                background: #333; 
                color: white; 
                padding: 10px; 
                text-align: center; 
            }
            .container { 
                display: flex; 
                gap: 20px; 
                padding: 20px; 
            }
            .video-panel { 
                flex: 2; 
                background: white; 
                padding: 20px; 
                border-radius: 8px; 
                text-align: center; 
            }
            .control-panel { 
                flex: 1; 
                background: white; 
                padding: 20px; 
                border-radius: 8px; 
            }
            .stats { 
                background: #e8f4f8; 
                padding: 15px; 
                border-radius: 5px; 
                margin: 10px 0; 
            }
            .control-group { 
                margin: 15px 0; 
            }
            .control-group label { 
                display: block; 
                margin-bottom: 5px; 
                font-weight: bold; 
            }
            input[type="range"], select { 
                width: 100%; 
                margin: 5px 0; 
            }
            button { 
                background: #007cba; 
                color: white; 
                border: none; 
                padding: 10px 20px; 
                border-radius: 5px; 
                cursor: pointer; 
                width: 100%; 
                margin: 5px 0; 
            }
            button:hover { 
                background: #005a87; 
            }
            #video-stream { 
                border: 2px solid #333; 
                max-width: 100%; 
                height: auto; 
            }
            .status-indicator { 
                display: inline-block; 
                width: 10px; 
                height: 10px; 
                border-radius: 50%; 
                margin-right: 5px; 
            }
            .status-online { background: #4CAF50; }
            .status-offline { background: #f44336; }
        </style>
    </head>
    <body>
        <div class="header">
            <h1>🍊 香橙派Zero 2W 高级图传系统</h1>
            <p>访问地址: http://{{ ip }}:5000 | 
            <span class="status-indicator status-online"></span>在线</p>
        </div>
        
        <div class="container">
            <div class="video-panel">
                <h2>实时视频流</h2>
                <img id="video-stream" src="{{ url_for('video_feed') }}" alt="视频加载中...">
                
                <div class="stats">
                    <h3>📊 流媒体统计</h3>
                    <div id="stream-stats">加载中...</div>
                </div>
            </div>
            
            <div class="control-panel">
                <h2>🎛️ 控制面板</h2>
                
                <div class="control-group">
                    <label>分辨率:</label>
                    <select id="resolution" onchange="changeResolution()">
                        <option value="640x480" selected>640x480</option>
                        <option value="800x600">800x600</option>
                        <option value="1280x720">1280x720</option>
                        <option value="320x240">320x240 (低延迟)</option>
                    </select>
                </div>
                
                <div class="control-group">
                    <label>图像质量: <span id="quality-value">85</span>%</label>
                    <input type="range" id="quality" min="10" max="100" value="85" 
                           oninput="changeQuality(this.value)">
                </div>
                
                <div class="control-group">
                    <button onclick="restartStream()">🔄 重启流媒体</button>
                    <button onclick="captureFrame()">📸 截图</button>
                </div>
                
                <div class="stats">
                    <h3>🖥️ 系统状态</h3>
                    <div id="system-stats">加载中...</div>
                </div>
                
                <div class="stats">
                    <h3>🌐 网络信息</h3>
                    <div id="network-info">加载中...</div>
                </div>
            </div>
        </div>

        <script>
            var socket = io();
            
            function updateStreamStats() {
                fetch('/api/stats')
                    .then(response => response.json())
                    .then(data => {
                        document.getElementById('stream-stats').innerHTML = 
                            `实时帧率: ${data.fps} FPS<br>
                             总帧数: ${data.frames}<br>
                             分辨率: ${data.resolution[0]}x${data.resolution[1]}<br>
                             质量: ${data.quality}%`;
                    });
            }
            
            function updateSystemStats() {
                fetch('/api/system')
                    .then(response => response.json())
                    .then(data => {
                        document.getElementById('system-stats').innerHTML = 
                            `CPU使用率: ${data.cpu_percent}%<br>
                             内存使用率: ${data.memory_percent}%<br>
                             磁盘使用率: ${data.disk_percent}%<br>
                             温度: ${data.temperature}°C`;
                    });
            }
            
            function changeResolution() {
                var resolution = document.getElementById('resolution').value;
                var [width, height] = resolution.split('x').map(Number);
                
                fetch('/api/resolution', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({width: width, height: height})
                })
                .then(response => response.json())
                .then(data => {
                    if(data.success) {
                        setTimeout(() => location.reload(), 1000);
                    }
                });
            }
            
            function changeQuality(value) {
                document.getElementById('quality-value').textContent = value;
                
                fetch('/api/quality', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({quality: parseInt(value)})
                });
            }
            
            function restartStream() {
                fetch('/api/restart', {method: 'POST'})
                .then(() => {
                    setTimeout(() => location.reload(), 2000);
                });
            }
            
            function captureFrame() {
                window.open('/api/capture', '_blank');
            }
            
            // 定时更新统计信息
            setInterval(updateStreamStats, 1000);
            setInterval(updateSystemStats, 5000);
            
            // 初始加载
            updateStreamStats();
            updateSystemStats();
        </script>
    </body>
    </html>
    ''', ip=local_ip)

@app.route('/video_feed')
def video_feed():
    def generate():
        while True:
            frame = vs.read()
            if frame is not None:
                ret, buffer = cv2.imencode('.jpg', frame, 
                                         [cv2.IMWRITE_JPEG_QUALITY, vs.quality])
                if ret:
                    yield (b'--frame\r\n'
                           b'Content-Type: image/jpeg\r\n\r\n' + 
                           buffer.tobytes() + b'\r\n')
            time.sleep(0.033)
    
    return Response(generate(),
                    mimetype='multipart/x-mixed-replace; boundary=frame')

@app.route('/api/stats')
def api_stats():
    return jsonify(vs.get_stats())

@app.route('/api/system')
def api_system():
    return jsonify(get_system_info())

@app.route('/api/resolution', methods=['POST'])
def api_resolution():
    data = request.json
    vs.set_resolution(data['width'], data['height'])
    return jsonify({'success': True})

@app.route('/api/quality', methods=['POST'])
def api_quality():
    data = request.json
    vs.set_quality(data['quality'])
    return jsonify({'success': True})

@app.route('/api/restart', methods=['POST'])
def api_restart():
    vs.stop()
    time.sleep(1)
    vs.start()
    return jsonify({'success': True})

@app.route('/api/capture')
def api_capture():
    frame = vs.read()
    if frame is not None:
        timestamp = time.strftime("%Y%m%d_%H%M%S")
        filename = f"capture_{timestamp}.jpg"
        cv2.imwrite(f"/tmp/{filename}", frame)
        
        ret, buffer = cv2.imencode('.jpg', frame)
        if ret:
            return Response(buffer.tobytes(), 
                          mimetype='image/jpeg',
                          headers={'Content-Disposition': f'attachment; filename={filename}'})
    
    return "截图失败", 500

if __name__ == '__main__':
    print("🚀 启动高级图传服务器...")
    vs.start()
    time.sleep(2)
    
    hostname = socket.gethostname()
    try:
        local_ip = socket.gethostbyname(hostname)
    except:
        local_ip = "127.0.0.1"
    
    print(f"✅ 服务器启动成功！")
    print(f"📱 本地访问: http://localhost:5000")
    print(f"🌐 网络访问: http://{local_ip}:5000")
    print(f"🎥 摄像头状态: {'正常' if vs.read() is not None else '异常'}")
    
    try:
        socketio.run(app, host='0.0.0.0', port=5000, debug=False)
    except KeyboardInterrupt:
        print("\n🛑 正在关闭服务器...")
        vs.stop()
```

**验证方法：**
```bash
# 运行高级图传服务器
python3 advanced_stream.py

# 在另一个终端检查进程
ps aux | grep python3

# 检查网络连接
ss -tuln | grep 5000

# 测试API接口
curl http://localhost:5000/api/stats
curl http://localhost:5000/api/system
```

---

## 第六阶段：性能优化和调试

### 6.1 系统性能优化

**步骤：**
```bash
# 1. 调整GPU内存分配
sudo nano /boot/config.txt
# 添加或修改: gpu_mem=128

# 2. 优化网络缓冲区
echo 'net.core.rmem_max = 134217728' | sudo tee -a /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 3. 设置CPU调度器为性能模式
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

**验证方法：**
```bash
# 检查GPU内存
vcgencmd get_mem gpu

# 检查网络参数
sysctl net.core.rmem_max
sysctl net.core.wmem_max

# 检查CPU频率
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

### 6.2 创建系统监控脚本

**步骤：**
```bash
# 创建监控脚本
nano ~/video_streaming/monitor.py
```

**monitor.py内容：**
```python
#!/usr/bin/env python3
import psutil
import time
import subprocess
import json

def get_detailed_stats():
    """获取详细的系统统计信息"""
    stats = {
        'timestamp': time.strftime('%Y-%m-%d %H:%M:%S'),
        'cpu': {
            'percent': psutil.cpu_percent(interval=1),
            'freq': psutil.cpu_freq()._asdict() if psutil.cpu_freq() else None,
            'count': psutil.cpu_count(),
            'load_avg': list(psutil.getloadavg())
        },
        'memory': psutil.virtual_memory()._asdict(),
        'disk': psutil.disk_usage('/')._asdict(),
        'network': {},
        'temperature': {}
    }
    
    # 网络统计
    net_io = psutil.net_io_counters()
    stats['network'] = {
        'bytes_sent': net_io.bytes_sent,
        'bytes_recv': net_io.bytes_recv,
        'packets_sent': net_io.packets_sent,
        'packets_recv': net_io.packets_recv
    }
    
    # 温度信息
    try:
        with open('/sys/class/thermal/thermal_zone0/temp', 'r') as f:
            stats['temperature']['cpu'] = int(f.read()) / 1000
    except:
        stats['temperature']['cpu'] = None
    
    # GPU内存信息
    try:
        result = subprocess.run(['vcgencmd', 'get_mem', 'gpu'], 
                               capture_output=True, text=True)
        if result.returncode == 0:
            gpu_mem = result.stdout.strip().split('=')[1]
            stats['gpu_memory'] = gpu_mem
    except:
        stats['gpu_memory'] = None
    
    return stats

def monitor_performance(duration=60, interval=5):
    """监控性能指定时间"""
    print(f"开始监控系统性能 {duration} 秒，每 {interval} 秒记录一次")
    print("-" * 80)
    
    start_time = time.time()
    samples = []
    
    while time.time() - start_time < duration:
        stats = get_detailed_stats()
        samples.append(stats)
        
        print(f"[{stats['timestamp']}] "
              f"CPU: {stats['cpu']['percent']:5.1f}% | "
              f"内存: {stats['memory']['percent']:5.1f}% | "
              f"温度: {stats['temperature']['cpu'] or 0:5.1f}°C")
        
        time.sleep(interval)
    
    # 保存结果到文件
    with open(f"/tmp/performance_log_{int(start_time)}.json", 'w') as f:
        json.dump(samples, f, indent=2)
    
    print(f"\n监控完成，结果已保存到 /tmp/performance_log_{int(start_time)}.json")
    return samples

if __name__ == '__main__':
    import sys
    
    if len(sys.argv) > 1:
        duration = int(sys.argv[1])
    else:
        duration = 60
        
    monitor_performance(duration)
```

**运行监控：**
```bash
# 监控60秒
python3 ~/video_streaming/monitor.py 60

# 同时运行图传服务器（在另一个终端）
python3 ~/video_streaming/advanced_stream.py
```

### 6.3 创建自动启动服务

**步骤：**
```bash
# 创建systemd服务文件
sudo nano /etc/systemd/system/orangepi-streaming.service
```

**服务文件内容：**
```ini
[Unit]
Description=Orange Pi Video Streaming Service
After=network.target
Wants=network.target

[Service]
Type=simple
User=orangepi
Group=orangepi
WorkingDirectory=/home/orangepi/video_streaming
Environment=PATH=/usr/local/bin:/usr/bin:/bin
ExecStart=/usr/bin/python3 /home/orangepi/video_streaming/advanced_stream.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**启用服务：**
```bash
# 重新加载systemd配置
sudo systemctl daemon-reload

# 启用服务
sudo systemctl enable orangepi-streaming

# 启动服务
sudo systemctl start orangepi-streaming

# 检查服务状态
sudo systemctl status orangepi-streaming
```

**验证方法：**
```bash
# 检查服务状态
sudo systemctl is-active orangepi-streaming
sudo systemctl is-enabled orangepi-streaming

# 查看服务日志
sudo journalctl -u orangepi-streaming -f

# 测试重启后自动启动
sudo reboot
# 重启后检查
sudo systemctl status orangepi-streaming
```

---

## 第七阶段：故障排除和优化建议

### 7.1 常见问题诊断

**摄像头无法访问：**
```bash
# 检查摄像头设备
ls -la /dev/video*
lsusb | grep -i camera

# 检查权限
sudo usermod -a -G video orangepi
groups orangepi

# 测试摄像头
v4l2-ctl --device=/dev/video0 --info
```

**网络连接问题：**
```bash
# 检查网络接口
ip addr show
iwconfig

# 重启网络服务
sudo systemctl restart NetworkManager

# 检查DNS解析
nslookup google.com
```

**性能问题诊断：**
```bash
# 实时监控系统资源
htop

# 检查内存使用
free -h
cat /proc/meminfo

# 检查温度
watch -n 1 'cat /sys/class/thermal/thermal_zone0/temp | awk "{print \$1/1000 \"°C\"}"'
```

### 7.2 性能调优建议

**低延迟配置：**
- 分辨率：320x240 或 640x480
- 帧率：15-30 FPS
- 质量：60-80%
- 使用有线网络连接

**高质量配置：**
- 分辨率：1280x720
- 帧率：30 FPS
- 质量：85-95%
- 确保充足的带宽

**节能配置：**
- 分辨率：640x480
- 帧率：15 FPS
- 质量：70%
- 使用WiFi连接

### 7.3 安全建议

**步骤：**
```bash
# 1. 修改默认端口
# 在代码中将5000改为其他端口

# 2. 添加简单认证
pip3 install flask-httpauth

# 3. 配置防火墙
sudo ufw enable
sudo ufw allow 5000/tcp  # 或你的自定义端口

# 4. 设置访问日志
sudo nano /etc/rsyslog.d/orangepi-streaming.conf
```

**验证方法：**
```bash
# 检查防火墙状态
sudo ufw status

# 检查开放端口
ss -tuln

# 测试外部访问
# 从另一台设备访问 http://香橙派IP:5000
```

---

## 总结

通过以上步骤，你已经成功在香橙派Zero 2W上搭建了一个完整的Python图传系统。系统具备以下特性：

✅ **基础功能**：
- 实时视频流传输
- Web界面访问
- 多设备同时观看

✅ **高级功能**：
- 动态分辨率调整
- 质量控制
- 系统监控
- 性能统计

✅ **系统优化**：
- 自动启动服务
- 性能监控
- 故障诊断工具

✅ **可扩展性**：
- 模块化代码结构
- API接口
- 配置灵活

现在你可以通过浏览器访问 `http://香橙派IP:5000` 来查看实时图传效果！
