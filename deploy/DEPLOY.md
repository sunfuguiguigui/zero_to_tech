# 部署说明（Nginx 静态托管 + FastAPI）

## 为什么之前会「分析失败：404」

三个原因叠在一起，按发现顺序：

1. **部署的是旧产物。** `out/` 是 2026-11-03 打的，而 `InputCard.jsx` 接后端是 2026-11-11 改的，
   旧产物里根本没有 `fetch`、没有 `/api/analyze`。
2. **`NEXT_PUBLIC_API_BASE_URL` 是构建期注入。** 原来 `.env.local` 里写的是
   `http://localhost:8000`，它会被编译成死字符串进 JS。上线后访客的浏览器会去请求
   **访客自己电脑**的 8000 端口。
3. **Nginx 没配 `/api/` 路由。** 请求打到 Nginx 后被当成静态文件去找，返回 `404.html`。
   因为响应体是 HTML 不是 JSON，前端 `res.json().catch(() => ({}))` 兜底成空对象，
   于是提示是你看到的「分析失败：404」而不是 FastAPI 的「Not Found」。

第 3 条正是你这次遇到的。判断依据：FastAPI 的 404 一定带 `{"detail":"Not Found"}`，
页面会显示 `Not Found`；显示 `分析失败：404` 说明响应不是 JSON，来自静态服务器。

## 改了什么

| 文件 | 改动 |
| --- | --- |
| `components/InputCard.jsx` | API 基址改为「环境变量 → 开发环境 localhost:8000 → 生产环境相对路径」 |
| `components/TextLabView.jsx` | 同上 |
| `components/HomeView.jsx` | 同上 |
| `.env.local` | 原来那行 `NEXT_PUBLIC_API_BASE_URL=http://localhost:8000` 注释掉了（开发时靠代码兜底） |
| `backend/main.py` | `ALLOWED_ORIGINS` 缺 `.env` 时不再 `AttributeError`；清掉了重复的 `init_db()` 和第二个 `app = FastAPI()` |
| `backend/storage.py` | `history.db` 改成基于文件位置的绝对路径，换工作目录启动也不会写到别处 |
| `deploy/nginx.conf` | 新增：`try_files $uri.html` 修 /text-lab 404，`location /api/` 反代到 8000 |

前端现在生产环境的请求是相对路径 `/api/...`，**同源**，所以：

- 不需要 CORS 白名单
- Cookie 的 `samesite=lax` 正常工作，历史记录能按会话分开
- 以后换服务器 IP/域名不用重新打包

## 上线步骤

### 1. 本地重新构建（必须做，产物是死文件）

```bash
cd C:\Users\Hefin\VScode\zero_to_tech
npm run build
```

产物在 `out/`。

### 2. 上传产物

```bash
rsync -avz --delete out/ user@服务器:/var/www/zero-to-tech/out/
```

`--delete` 是为了清掉旧的旧 chunk 文件；第一次上传可以不加。

### 3. 后端跑在 8000（只监听本机，不要暴露到公网）

```bash
cd /var/www/zero-to-tech/backend
pip install -r requirements.txt
uvicorn main:app --host 127.0.0.1 --port 8000
```

长期跑用 systemd：

```ini
# /etc/systemd/system/zero-to-tech-api.service
[Unit]
Description=zero-to-tech FastAPI
After=network.target

[Service]
WorkingDirectory=/var/www/zero-to-tech/backend
ExecStart=/usr/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=always
User=www-data

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now zero-to-tech-api
```

### 4. Nginx

```bash
sudo cp deploy/nginx.conf /etc/nginx/sites-available/zero-to-tech
sudo ln -s /etc/nginx/sites-available/zero-to-tech /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

记得改 `server_name` 和 `root` 两处。

## 验证

```bash
# 后端直连，应返回 JSON
curl -s http://127.0.0.1:8000/api/profile

# 走 Nginx，应与上面一致（这一步返回 404 就是 location /api/ 没生效）
curl -s -i http://你的域名/api/profile

# 静态页映射，应返回 200 而不是 404
curl -s -o /dev/null -w "%{http_code}\n" http://你的域名/text-lab

# POST 分析接口
curl -s -X POST http://你的域名/api/analyze \
  -H "Content-Type: application/json" \
  -d '{"text":"今天的风很轻"}'
```

浏览器里 F12 → Network：点「开始分析」时请求应该是 `http://你的域名/api/analyze`，
状态码 200，响应是 JSON；如果 URL 里还有 `localhost:8000`，说明 `out/` 没重新构建。
