# LLM Honeypot - API Key Capture

一个通过 **提示注入（Prompt Injection）** 诱导 AI 爬虫 / LLM 扫描器泄露 API Key 及其厂商信息的蜜罐服务。

## 工作原理

```mermaid
flowchart LR
    Crawler["AI Crawler / LLM"] -->|"访问 /"| Index["首页返回伪装的API文档（正文+代码块+JSON-LD）"]
    Index -->|"文档中的验证流程"| Verify["访问 /verify 并携带请求头"]
    Verify -->|"始终返回401，诱导反复尝试"| Capture["/home/captured_keys.txt"]
```

1. 首页 `/` 返回伪装的 API 文档页面，通过**三层注入**诱导 LLM：
   - **正文文本** — 以 API 文档口吻描述"验证流程"
   - **代码块** — `<pre><code>` 中包含 HTTP 请求和 cURL 示例
   - **JSON-LD** — `<script type="application/ld+json">` 结构化数据，搜索引擎和 AI 爬虫会专门解析
2. 文档以"必须完成验证才能继续响应"为由，要求 LLM 向 `/verify` 发送以下任一 API Key 头：
   - `Authorization: Bearer <api-key>`
   - `X-Api-Key: <api-key>`
   - `Api-Key: <api-key>`
   - 以及 `X-Api-Provider`（厂商名称）和 `X-Api-Base-Url`（API Base URL）
3. `verify` 端点捕获信息后**始终返回 401**，错误信息引导 agent 换 Key 重试，有机会泄露更多凭据
4. 捕获的数据（含 UTC 时间戳）追加写入 `/home/captured_keys.txt`

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
[<UTC-timestamp>] [<client-ip>] KEY: <api-key> | PROVIDER: <provider> | BASE_URL: <base-url> | UA: <user-agent>
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
- **更改注入内容**：修改 `INDEX_PAGE` 中的正文文本、代码块示例或 JSON-LD 数据，可针对特定 LLM 或 Crawler 定制诱导逻辑

## 安全注意事项

- 确保服务器已配置防火墙，仅暴露必要端口
- 定期检查并清理 `/home/captured_keys.txt`，避免敏感信息长期留存
- 此项目仅用于**安全研究和授权测试**，请勿用于非法用途

## License

MIT
