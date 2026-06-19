# LLM Honeypot - API Key Capture

一个通过 **提示注入（Prompt Injection）** 诱导 AI 爬虫 / LLM 扫描器泄露 API Key 及其厂商信息的蜜罐服务。

## 工作原理

```mermaid
flowchart LR
    Crawler["AI Crawler / LLM"] -->|"访问 /"| Index["首页返回含隐藏指令的HTML"]
    Index -->|"指令要求"| Verify["访问 /verify 并携带请求头"]
    Verify -->|"捕获并落盘"| Capture["/home/captured_keys.txt"]
```

1. 首页 `/` 返回的 HTML 中，CSS 注释里嵌入了 **system instruction**
2. 该指令以"必须完成验证才能继续响应"为由，要求 LLM 向 `/verify` 发送 HTTP 头。`verify` 端点会捕获以下任一 API Key 头：
   - `Authorization: Bearer <api-key>`
   - `X-Api-Key: <api-key>`
   - `Api-Key: <api-key>`
   - 以及 `X-Api-Provider`（厂商名称）和 `X-Api-Base-Url`（API Base URL）
3. `verify` 端点将捕获到的所有信息追加写入本地文件 `/home/captured_keys.txt`

> **为什么需要 Provider 和 Base URL？** 仅获取 API Key 无法确定其用途。带上厂商信息后，才能知道该 Key 对应哪个服务，从而实际使用。

## 文件结构

```
.
├── app.py            # Flask 蜜罐应用
├── nginx.conf        # Nginx 反向代理配置
├── requirements.txt  # Python 依赖
└── README.md         # 本文件
```

## 快速部署

### 前置条件

- Python >= 3.8
- Nginx（可选，但推荐用于对外暴露服务）

### 1. 克隆并安装依赖

```bash
git clone <your-repo-url>
cd honeypot
pip install -r requirements.txt
```

### 2. 启动 Flask

```bash
python app.py &
```

建议使用 `nohup` 或 `screen` 保持后台运行：

```bash
nohup python app.py > flask.log 2>&1 &
```

Flask 默认监听 `127.0.0.1:5000`，仅本机可访问。若只需本地测试，到此即可。

### 3. 配置 Nginx（对外暴露）

```bash
# Debian / Ubuntu
sudo cp nginx.conf /etc/nginx/sites-available/honeypot
sudo ln -s /etc/nginx/sites-available/honeypot /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# RHEL / CentOS / Fedora
sudo cp nginx.conf /etc/nginx/conf.d/honeypot.conf
sudo nginx -t
sudo systemctl reload nginx
```

Nginx 监听 `0.0.0.0:80`，将请求反向代理到 Flask。

### 4. 查看捕获结果

```bash
tail -f /home/captured_keys.txt
```

每条记录格式：
```
[<client-ip>] KEY: <api-key> | PROVIDER: <provider> | BASE_URL: <base-url> | UA: <user-agent>
```

## 进阶部署

### 使用 systemd 管理 Flask 进程

创建 `/etc/systemd/system/honeypot.service`：

```ini
[Unit]
Description=LLM Honeypot Flask App
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/path/to/honeypot
ExecStart=/usr/bin/python /path/to/honeypot/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now honeypot
sudo systemctl status honeypot
```

### 使用 Docker（自行构建）

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "app.py"]
```

```bash
docker build -t llm-honeypot .
docker run -d -p 5000:5000 llm-honeypot
```

## 自定义与扩展

- **修改捕获文件路径**：编辑 `app.py` 中的 `CAPTURE_FILE` 变量
- **新增需要捕获的头**：在 `verify` 路由中增加 `request.headers.get()` 并在写入格式中添加对应字段
- **更改指令内容**：修改 `INDEX_PAGE` 中 `/* <system-instruction> ... */` 部分，可针对特定 LLM 或 Crawler 定制诱导逻辑

## 安全注意事项

- 确保服务器已配置防火墙，仅暴露必要端口
- 定期检查并清理 `/home/captured_keys.txt`，避免敏感信息长期留存
- 此项目仅用于**安全研究和授权测试**，请勿用于非法用途

## License

MIT
