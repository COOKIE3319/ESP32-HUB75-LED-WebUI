# ESP32 HUB75 LED WebUI

ESP32 HUB75 LED 矩阵屏的 Web 前端图片上传工具。

## 功能

- **设备连接**：输入 ESP32 IP 地址，检测连接状态
- **显示调节**：连接后可实时调整 Gamma（暗部/中间调）
- **图片选择**：点击选择或拖放 PNG 图片
- **图片预览**：原图预览 + LED 屏幕像素化模拟预览 (5x 放大)
- **图片上传**：自动将 PNG → RGB888 → Base64 编码 → 上传到 ESP32
- **清空屏幕**：一键清空 LED 显示内容
- **IP 记忆**：自动保存上次连接的 IP 到 localStorage

## 技术栈

- Vue 3 (Composition API)
- Vite 5
- 纯 CSS (无 UI 框架，暗色主题)

## 图片要求

- 格式：PNG
- 尺寸：128×32 像素
- 色彩：RGB (自动提取，忽略 Alpha 通道)

## 开发

```bash
# 安装依赖
npm install

# 启动开发服务器
npm run dev

# 构建生产版本
npm run build
```

开发服务器默认运行在 `http://localhost:5173`。

## 使用方法

1. 启动开发服务器 `npm run dev`
2. 浏览器打开 `http://localhost:5173`
3. 在「设备连接」区域输入 ESP32 的 IP 地址，点击「检测连接」
4. 在「图片上传」区域选择或拖放一张 128×32 的 PNG 图片
5. 确认预览无误后，点击「上传到 LED 屏幕」

## 工作原理

```
PNG 文件 → Canvas 绘制 → getImageData() 提取 RGBA
→ 去除 Alpha 得到 RGB888 (12288 字节)
→ Base64 编码 → POST 到 ESP32 /upload 接口
→ ESP32 解码后写入帧缓冲区 → LED 屏幕显示
```

## 配套固件

见 [ESP32-HUB75-LED](../ESP32-HUB75-LED) 项目。
