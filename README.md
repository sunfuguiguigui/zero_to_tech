# zero-to-tech-6-6 · 前端替换文件（模块 6.6 配套代码）

这一节前端要改的都在 `components/` 和 `css/` 下，这里只放**改动的文件**，方便你直接覆盖，不用一行行敲。其余文件（`app/`、`data/site.js`、其它 css）都不用动。

> 这不是一个能独立跑的工程，是"替换用的文件"。后端是你在模块 6 一路搭起来的那个。

## 改了哪些文件

```
components/
  HistoryModal.jsx  ← 新增：历史记录弹窗，用 .map() 把数组逐条列出来
  ResultCard.jsx    ← 只加了右上角那个"历史记录"按钮
  TextLabView.jsx   ← 多两份 state（历史数据、弹窗开关），点开时才去 GET /api/history
css/
  lab.css           ← 新增按钮与弹窗的样式
```

历史放在**弹窗**里，而不是在页面底部再加一张卡——这样不会把文字实验室的页面撑得很长，对原来的两卡布局打扰最小。

弹窗是**点开的那一刻**才去请求 `/api/history` 的，没必要每次进页面都拉一遍。

## 怎么用

课件到"把历史显示出来"这一步时，用这些文件覆盖你 `~/zero-to-tech/` 下的同名文件即可。

```bash
git clone https://github.com/joylibo/zero-to-tech-demos.git
cp zero-to-tech-demos/zero-to-tech-6-6/components/*.jsx ~/zero-to-tech/components/
cp zero-to-tech-demos/zero-to-tech-6-6/css/lab.css      ~/zero-to-tech/css/
```

## 两点说明

- **时间显示做了本地化。** 后端存的是 UTC（6.3 立的规矩），`HistoryModal.jsx` 里的 `formatTime()` 负责转成你所在时区再显示——所以列表上看到的是本地时间，不是数据库里那串 `+00:00`。
- **这一版还没有会话。** 谁来访问，`/api/history` 返回的都是同一份"全部历史"——这正是课件要你亲眼看见的问题。跟着课件往下做，加上 cookie 会话之后，两处 `fetch` 还要各加一句 `credentials: "include"`。

## 部署到服务器

前提：服务器上已装好 Python 3.10+、Node.js 18+ 和 Nginx。

**1. 拉取代码**

```bash
cd ~/zero-to-tech
git pull
```

**2. 前端：装依赖、写配置、构建**

```bash
npm install
cp .env.example .env.production   # 按下面「配置说明」填好（线上留空）
npm run build                     # 产物进 out/，由 Nginx 提供服务
```

**3. 后端：建环境、装依赖、写配置**

```bash
cd backend
python3 -m venv --prompt=zero-to-tech .venv   # 首次部署才需要
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env              # 按下面「配置说明」填好
```

**4. 后端：委托给 systemd**（首次部署才需要）

新建 `/etc/systemd/system/zero-backend.service`：

```ini
[Unit]
Description=zero-to-tech backend

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/zero-to-tech/backend
ExecStart=/home/ubuntu/zero-to-tech/backend/.venv/bin/fastapi run --host 127.0.0.1
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start zero-backend     # 启动
sudo systemctl enable zero-backend    # 开机自启
sudo systemctl status zero-backend    # 看到 active (running) 就对了
```

**5. Nginx：静态文件 ＋ `/api/` 反向代理**（首次部署才需要）

编辑 `/etc/nginx/sites-enabled/default`：

```nginx
server {
    listen 80 default_server;
    server_name _;

    root /home/ubuntu/zero-to-tech/out;
    index index.html;

    location / {
        try_files $uri $uri.html $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8000;     # 末尾不要加斜杠

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

**6. 安全组 / 防火墙**

只放行 **22（SSH）** 和 **80（HTTP）**。**8000 不要放行**——后端只监听 `127.0.0.1`，由 Nginx 转发进去。

**7. 验证**

```bash
curl localhost/api/profile        # 服务器本机，注意不带端口
```

浏览器访问 `http://服务器IP`，做一次分析、看一眼历史记录。换个无痕窗口再试，两边的历史应该互相看不到。

## 日常发布

```bash
cd ~/zero-to-tech
git pull

npm run build                          # 前端改了才需要
sudo systemctl restart zero-backend    # 后端改了才需要
```

## 常用运维命令

```bash
sudo systemctl status zero-backend     # 看状态
sudo systemctl restart zero-backend    # 重启
sudo systemctl stop zero-backend       # 停掉
journalctl -u zero-backend -f          # 实时看日志（Ctrl+C 退出）
journalctl -u zero-backend -n 50       # 看最后 50 行

sudo nginx -t                          # 改完 Nginx 配置先验语法
sudo systemctl reload nginx            # 再让它重读
```
| 键 | 说明 | 本地 | 线上 |
| --- | --- | --- | --- |
| `NEXT_PUBLIC_API_BASE_URL` | 后端接口地址 | `http://localhost:8000` | **留空**——前后端同源，代码里走相对路径 |
