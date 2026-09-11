# Hướng dẫn triển khai Production — Nụ Cười Em

Tài liệu mô tả cách dựng VPS, cấu hình dịch vụ, triển khai tự động, sao lưu và
vận hành website ở môi trường production. Môi trường dev xem [`SETUP.md`](SETUP.md);
lý do đằng sau các lựa chọn xem [`ARCHITECTURE.md`](ARCHITECTURE.md).

> **Domain trong tài liệu:** `nucuoiem.org` là domain ví dụ, khớp với
> `backend/.env.example`. Thay bằng domain thật khi đã mua — xem ghi chú về lựa
> chọn `.com` / `.org` trong [`Phan-tich-chi-phi-van-hanh.md`](../project/Phan-tich-chi-phi-van-hanh.md).
>
> **Trạng thái repo:** các file trong `deploy/` và `.github/workflows/` hiện đang
> rỗng. Nội dung chuẩn của chúng được đặc tả ngay trong tài liệu này (§6, §7, §10, §12).

---

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Chuẩn bị tài khoản & dịch vụ](#2-chuẩn-bị-tài-khoản--dịch-vụ)
3. [Thiết lập VPS lần đầu](#3-thiết-lập-vps-lần-đầu)
4. [Cấu trúc thư mục trên VPS](#4-cấu-trúc-thư-mục-trên-vps)
5. [Biến môi trường production](#5-biến-môi-trường-production)
6. [Gunicorn & systemd](#6-gunicorn--systemd)
7. [Nginx](#7-nginx)
8. [Cloudflare](#8-cloudflare)
9. [Lần triển khai đầu tiên](#9-lần-triển-khai-đầu-tiên)
10. [CI/CD với GitHub Actions](#10-cicd-với-github-actions)
11. [Rollback](#11-rollback)
12. [Sao lưu & khôi phục](#12-sao-lưu--khôi-phục)
13. [Giám sát & log](#13-giám-sát--log)
14. [Bảo trì định kỳ](#14-bảo-trì-định-kỳ)
15. [Xử lý sự cố](#15-xử-lý-sự-cố)

---

## 1. Tổng quan

```
                      Cloudflare (DNS, SSL, CDN)
                                 │  HTTPS
                                 ▼
┌────────────────────── VPS Ubuntu 24.04 — 1 vCPU / 1GB RAM / 20GB SSD ──────────────────────┐
│                                                                                            │
│  Nginx :443 ──┬── /, /assets/*               ──► /srv/nu-cuoi-em/current/frontend/dist     │
│               ├── /django-static/*          ──► /srv/nu-cuoi-em/current/backend/staticfiles│
│               └── /api/*, /share/*,          ──► unix:/run/nu-cuoi-em/gunicorn.sock        │
│                   /sitemap.xml, /django-admin/        │                                    │
│                                                        ▼                                    │
│                                         Gunicorn (nu-cuoi-em-api.service)                  │
│                                                                                            │
│  systemd timers:  scheduler (5 phút) · maintenance (03:00) · backup (02:30)                │
└────────────────────────────────────────────────────────────────────────────────────────────┘
                 │                          │                          │
                 ▼                          ▼                          ▼
        MongoDB Atlas M0            Cloudinary / Resend        Google Drive (backup)
```

| Thành phần | Giá trị |
|-----------|---------|
| Hệ điều hành | Ubuntu Server 24.04 LTS |
| Người dùng chạy ứng dụng | `nce` (không mật khẩu, chỉ SSH key) |
| Thư mục ứng dụng | `/srv/nu-cuoi-em` |
| Múi giờ | `Asia/Ho_Chi_Minh` |
| Triển khai | Tự động khi merge vào `main` (§10) |

---

## 2. Chuẩn bị tài khoản & dịch vụ

Hoàn thành danh sách này **trước** khi động vào VPS.

| # | Việc | Ghi chú |
|---|------|---------|
| 1 | Thuê VPS | Ubuntu 24.04, tối thiểu 1 vCPU / 1GB RAM / 20GB SSD, IP tĩnh |
| 2 | Mua domain | Ghi lại ngày hết hạn vào lịch bảo trì (§14) |
| 3 | Tạo tài khoản Cloudflare, thêm domain | Đổi nameserver tại nhà đăng ký domain sang Cloudflare |
| 4 | Tạo cluster Atlas **production** riêng | Project khác với cluster dev; M0, region Singapore |
| 5 | Tạo user DB `nce_prod` | Quyền `readWrite` chỉ trên database `nu_cuoi_em` |
| 6 | Atlas Network Access | **Chỉ** IP của VPS. Không bao giờ dùng `0.0.0.0/0` cho production |
| 7 | Tài khoản Cloudinary | Lấy `cloud_name`, `api_key`, `api_secret` |
| 8 | Tài khoản Resend, xác minh domain | Thêm các bản ghi TXT/MX Resend cung cấp vào Cloudflare, chế độ **DNS only** |
| 9 | Tạo API key Resend | Quyền "Sending access", không cấp "Full access" |
| 10 | Tài khoản Google Drive dùng chung của nhóm | Nơi lưu backup ngoài VPS (§12) |
| 11 | Tài khoản UptimeRobot | Giám sát uptime (§13) |
| 12 | Trình quản lý mật khẩu dùng chung (VD Bitwarden) | Lưu bản sao `.env` production và mọi mật khẩu dịch vụ |

---

## 3. Thiết lập VPS lần đầu

Các lệnh chạy trên VPS với quyền `sudo`. Thay `hieu` bằng tên tài khoản quản trị của bạn.

### 3.1. Cập nhật hệ thống, múi giờ

```bash
sudo apt update && sudo apt full-upgrade -y
sudo timedatectl set-timezone Asia/Ho_Chi_Minh
sudo hostnamectl set-hostname nce-prod
```

### 3.2. Tài khoản quản trị & SSH

```bash
# Đăng nhập bằng root lần đầu, tạo tài khoản quản trị
adduser hieu
usermod -aG sudo hieu
mkdir -p /home/hieu/.ssh
cp ~/.ssh/authorized_keys /home/hieu/.ssh/   # hoặc dán public key của bạn
chown -R hieu:hieu /home/hieu/.ssh && chmod 700 /home/hieu/.ssh && chmod 600 /home/hieu/.ssh/authorized_keys
```

Mở **một terminal mới**, xác nhận đăng nhập được bằng `hieu` + SSH key, rồi mới khoá SSH:

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
sudo systemctl restart ssh
```

> Giữ nguyên cổng SSH 22. Ubuntu 24.04 kích hoạt SSH qua `ssh.socket`, đổi cổng
> trong `sshd_config` sẽ không có tác dụng nếu không sửa cả socket — dễ tự khoá
> mình ngoài VPS. Key-only + fail2ban là đủ.

### 3.3. Swap 2GB

Bắt buộc với VPS 1GB RAM — thiếu swap, `pip install` hoặc `mongodump` có thể bị OOM kill.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

### 3.4. Cài gói

```bash
sudo apt install -y nginx git curl ufw fail2ban unattended-upgrades rclone software-properties-common

# Python 3.13 (Ubuntu 24.04 mặc định là 3.12, repo ghim 3.13 trong backend/.python-version)
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt install -y python3.13 python3.13-venv python3.13-dev
```

**MongoDB Database Tools** (cho `mongodump` / `mongorestore`): tải gói `.deb` cho
Ubuntu 24.04 x86_64 tại trang *MongoDB Command Line Database Tools Download*, rồi:

```bash
sudo apt install -y ./mongodb-database-tools-ubuntu2404-x86_64-*.deb
mongodump --version
```

### 3.5. Tự động cập nhật bảo mật

```bash
sudo dpkg-reconfigure -plow unattended-upgrades   # chọn Yes
```

### 3.6. Firewall

Chỉ mở 80/443 cho **dải IP của Cloudflare** — truy cập thẳng vào IP VPS bị chặn,
kẻ tấn công không vượt được lớp Cloudflare.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH

for ip in $(curl -s https://www.cloudflare.com/ips-v4) $(curl -s https://www.cloudflare.com/ips-v6); do
  sudo ufw allow proto tcp from "$ip" to any port 80,443 comment 'cloudflare'
done

sudo ufw enable
sudo ufw status numbered
```

fail2ban trên Ubuntu bật sẵn jail `sshd`, kiểm tra bằng `sudo fail2ban-client status sshd`.

### 3.7. Người dùng ứng dụng `nce`

```bash
sudo adduser --system --group --home /srv/nu-cuoi-em --shell /bin/bash nce
sudo mkdir -p /srv/nu-cuoi-em/{releases,shared,backups/daily} /srv/nu-cuoi-em/.ssh /var/cache/nu-cuoi-em
sudo chown -R nce:nce /srv/nu-cuoi-em /var/cache/nu-cuoi-em
sudo chmod 700 /srv/nu-cuoi-em/.ssh
```

Cho `nce` quyền `sudo` **chỉ** với vài lệnh cần cho deploy:

```bash
sudo tee /etc/sudoers.d/nce-deploy > /dev/null <<'EOF'
nce ALL=(root) NOPASSWD: /usr/bin/systemctl restart nu-cuoi-em-api, /usr/bin/systemctl is-active nu-cuoi-em-api
EOF
sudo chmod 440 /etc/sudoers.d/nce-deploy
sudo visudo -c
```

---

## 4. Cấu trúc thư mục trên VPS

```
/srv/nu-cuoi-em/
├── current -> releases/20260911-093000-abc1234     Symlink tới bản đang chạy
├── releases/                                        Giữ 3 bản gần nhất
│   └── 20260911-093000-abc1234/
│       ├── backend/            Mã Django (+ .env -> ../../../shared/.env)
│       │   └── staticfiles/    Kết quả collectstatic
│       ├── deploy/             Cấu hình Nginx, systemd, script
│       ├── frontend/dist/      React đã build (từ GitHub Actions)
│       └── venv/               Virtualenv riêng của bản này
├── shared/
│   └── .env                    Biến môi trường production (chmod 600)
├── backups/daily/              mongodump 7 ngày gần nhất
└── .ssh/authorized_keys        Public key của GitHub Actions

/var/cache/nu-cuoi-em/          FileBasedCache của Django
/run/nu-cuoi-em/gunicorn.sock   Unix socket (systemd tự tạo)
/etc/ssl/cloudflare/            Origin Certificate
```

**Vì sao mỗi bản có venv riêng:** rollback về bản cũ là chuyển symlink rồi restart
— không phải cài lại dependency, và bản cũ chắc chắn chạy đúng như lúc trước.
Mỗi venv ~150MB, giữ 3 bản là ~500MB, không đáng kể trên ổ 20GB.

---

## 5. Biến môi trường production

Tạo `/srv/nu-cuoi-em/shared/.env` (quyền `600`, chủ sở hữu `nce`):

```bash
sudo -u nce nano /srv/nu-cuoi-em/shared/.env
sudo chmod 600 /srv/nu-cuoi-em/shared/.env
```

```
DJANGO_SETTINGS_MODULE=config.settings.production
DJANGO_SECRET_KEY=<chuỗi ngẫu nhiên RIÊNG cho production>
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=nucuoiem.org,www.nucuoiem.org
DJANGO_ADMIN_URL=django-admin/
CSRF_TRUSTED_ORIGINS=https://nucuoiem.org
CORS_ALLOWED_ORIGINS=https://nucuoiem.org

MONGODB_URI=mongodb+srv://nce_prod:<password>@<cluster-prod>.mongodb.net/?retryWrites=true&w=majority
MONGODB_DB_NAME=nu_cuoi_em

JWT_ACCESS_TOKEN_LIFETIME_MINUTES=30
JWT_REFRESH_TOKEN_LIFETIME_DAYS=7

CLOUDINARY_CLOUD_NAME=<...>
CLOUDINARY_API_KEY=<...>
CLOUDINARY_API_SECRET=<...>

EMAIL_API_KEY=<...>
DEFAULT_FROM_EMAIL=no-reply@nucuoiem.org

FRONTEND_URL=https://nucuoiem.org
SENTRY_DSN=
```

| Khác biệt so với dev | Lý do |
|----------------------|-------|
| `DJANGO_DEBUG=False` | Bật lại là lộ traceback, biến môi trường và cấu trúc code ra ngoài |
| `DJANGO_SECRET_KEY` riêng | Dùng chung key với dev = ai có `.env` dev đều giả mạo được token production |
| `DJANGO_ADMIN_URL` | Django Admin không được ở `/admin/` vì đè route React ([ARCHITECTURE §4.4](ARCHITECTURE.md#44-đường-dẫn)) |
| `CORS_ALLOWED_ORIGINS` | Production cùng origin nên thực tế không cần CORS; giữ để chặn origin lạ |

> **Hai biến mới** so với `backend/.env.example` hiện tại: `DJANGO_ADMIN_URL` và
> `SENTRY_DSN`. Cần bổ sung vào `.env.example` khi viết `config/settings/`.
>
> File này **không có giá trị trong ngoặc kép**, **không có `export`**, và không
> được `source` bằng bash — nó được đọc bởi cả `python-dotenv` (qua symlink
> `backend/.env` trong mỗi release) lẫn `EnvironmentFile=` của systemd, cú pháp
> phải hợp lệ với cả hai.

Sau khi tạo, **lưu một bản sao vào trình quản lý mật khẩu của nhóm**. Mất file này
mà không có bản sao thì phải xoay vòng toàn bộ secret.

### 5.1. Cấu hình Django production cần có

Trong `backend/config/settings/production.py`:

```python
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SECURE_SSL_REDIRECT = False          # Cloudflare + Nginx đã chuyển hướng; bật ở đây dễ gây vòng lặp
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

STATIC_URL = "/django-static/"
STATIC_ROOT = BASE_DIR / "staticfiles"

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.filebased.FileBasedCache",
        "LOCATION": "/var/cache/nu-cuoi-em",
    }
}

REST_FRAMEWORK["NUM_PROXIES"] = 1    # Lấy IP thật từ X-Forwarded-For do Nginx đặt
```

`NUM_PROXIES = 1` kết hợp với cấu hình real IP ở §7 là **bắt buộc** — thiếu nó,
DRF throttle theo IP của Cloudflare và một người spam sẽ khoá form của mọi người.

---

## 6. Gunicorn & systemd

### 6.1. `deploy/gunicorn.conf.py`

```python
bind = "unix:/run/nu-cuoi-em/gunicorn.sock"
umask = 0o007                  # socket đọc/ghi được bởi group www-data (Nginx)

workers = 2
worker_class = "gthread"
threads = 4                    # API chủ yếu chờ mạng tới Atlas → thread tận dụng tốt thời gian chờ
timeout = 30
graceful_timeout = 20
keepalive = 5

max_requests = 1000            # tái sinh worker định kỳ, chặn rò rỉ bộ nhớ tích luỹ
max_requests_jitter = 100

preload_app = False            # PyMongo không an toàn khi fork sau khi đã mở kết nối

accesslog = "-"                # ra stdout → journald
errorlog = "-"
loglevel = "info"
```

Vì sao **2 worker × 4 thread** thay vì công thức `2 × CPU + 1`: mỗi worker Django
chiếm ~90MB RAM; 3 worker sync vừa tốn RAM hơn vừa xử lý đồng thời kém hơn khi
mỗi truy vấn phải chờ 30–50ms tới Singapore ([ARCHITECTURE §4.6](ARCHITECTURE.md#46-độ-trễ-tới-database)).

### 6.2. `deploy/systemd/nu-cuoi-em-api.service`

```ini
[Unit]
Description=Nu Cuoi Em - Django API (Gunicorn)
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
NotifyAccess=main
User=nce
Group=www-data
RuntimeDirectory=nu-cuoi-em
RuntimeDirectoryMode=0750
WorkingDirectory=/srv/nu-cuoi-em/current/backend
EnvironmentFile=/srv/nu-cuoi-em/shared/.env
ExecStart=/srv/nu-cuoi-em/current/venv/bin/gunicorn -c /srv/nu-cuoi-em/current/deploy/gunicorn.conf.py config.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5
KillMode=mixed
TimeoutStopSec=30

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/cache/nu-cuoi-em
MemoryMax=450M

[Install]
WantedBy=multi-user.target
```

`MemoryMax=450M` là chốt an toàn: nếu có rò rỉ bộ nhớ, systemd kill và khởi động
lại riêng Gunicorn thay vì để kernel OOM kill ngẫu nhiên cả Nginx hay sshd.

### 6.3. `deploy/systemd/nu-cuoi-em-scheduler.service` + `.timer`

```ini
# nu-cuoi-em-scheduler.service
[Unit]
Description=Nu Cuoi Em - publish scheduled events and articles
After=network-online.target

[Service]
Type=oneshot
User=nce
Group=www-data
WorkingDirectory=/srv/nu-cuoi-em/current/backend
EnvironmentFile=/srv/nu-cuoi-em/shared/.env
ExecStart=/srv/nu-cuoi-em/current/venv/bin/python manage.py publish_scheduled_events
TimeoutStartSec=120
```

```ini
# nu-cuoi-em-scheduler.timer
[Unit]
Description=Run publish_scheduled_events every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true
Unit=nu-cuoi-em-scheduler.service

[Install]
WantedBy=timers.target
```

### 6.4. Timer bảo trì và backup *(cần tạo thêm file)*

Repo hiện chỉ có timer `scheduler`. Cần thêm 4 file sau vào `deploy/systemd/`:

```ini
# nu-cuoi-em-maintenance.service
[Unit]
Description=Nu Cuoi Em - daily maintenance
After=network-online.target

[Service]
Type=oneshot
User=nce
Group=www-data
WorkingDirectory=/srv/nu-cuoi-em/current/backend
EnvironmentFile=/srv/nu-cuoi-em/shared/.env
ExecStart=/srv/nu-cuoi-em/current/venv/bin/python manage.py cleanup_data
ExecStart=/srv/nu-cuoi-em/current/venv/bin/python manage.py recalculate_counters
ExecStart=/srv/nu-cuoi-em/current/venv/bin/python manage.py flushexpiredtokens
TimeoutStartSec=600
```

```ini
# nu-cuoi-em-maintenance.timer
[Unit]
Description=Daily maintenance at 03:00

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
Unit=nu-cuoi-em-maintenance.service

[Install]
WantedBy=timers.target
```

```ini
# nu-cuoi-em-backup.service
[Unit]
Description=Nu Cuoi Em - MongoDB backup
After=network-online.target

[Service]
Type=oneshot
User=nce
EnvironmentFile=/srv/nu-cuoi-em/shared/.env
ExecStart=/srv/nu-cuoi-em/current/deploy/scripts/backup.sh
TimeoutStartSec=900
```

```ini
# nu-cuoi-em-backup.timer
[Unit]
Description=Daily MongoDB backup at 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
Unit=nu-cuoi-em-backup.service

[Install]
WantedBy=timers.target
```

### 6.5. Cài đặt unit

```bash
sudo cp /srv/nu-cuoi-em/current/deploy/systemd/*.service /srv/nu-cuoi-em/current/deploy/systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now nu-cuoi-em-api
sudo systemctl enable --now nu-cuoi-em-scheduler.timer nu-cuoi-em-maintenance.timer nu-cuoi-em-backup.timer
systemctl list-timers 'nu-cuoi-em-*'
```

> Unit systemd và cấu hình Nginx **không tự cài lại khi deploy** — cố ý, để một
> lần deploy lỗi không làm hỏng Nginx. Khi sửa file trong `deploy/systemd/` hoặc
> `deploy/nginx/`, chạy lại lệnh cài tương ứng bằng tay. Script deploy (§10.3) sẽ
> in cảnh báo nếu phát hiện file trên VPS khác file trong repo.

---

## 7. Nginx

### 7.1. Snippet dùng chung

**`/etc/nginx/snippets/cloudflare-realip.conf`** — lấy IP thật của người dùng
thay vì IP Cloudflare. Sinh tự động từ danh sách chính thức:

```bash
{
  echo "# Sinh tu https://www.cloudflare.com/ips — $(date -I)"
  for ip in $(curl -s https://www.cloudflare.com/ips-v4) $(curl -s https://www.cloudflare.com/ips-v6); do
    echo "set_real_ip_from $ip;"
  done
  echo "real_ip_header CF-Connecting-IP;"
} | sudo tee /etc/nginx/snippets/cloudflare-realip.conf
```

**`/etc/nginx/snippets/nce-proxy.conf`**

```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $remote_addr;   # ghi đè, không nối thêm — chặn giả mạo IP
proxy_set_header X-Forwarded-Proto https;
proxy_redirect off;
proxy_read_timeout 30s;
```

**`/etc/nginx/snippets/nce-security-headers.conf`**

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
add_header Strict-Transport-Security "max-age=15552000" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https://res.cloudinary.com https://i.ytimg.com; frame-src https://www.youtube-nocookie.com https://www.youtube.com https://www.google.com; connect-src 'self'; font-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'" always;
```

> **Bẫy của Nginx:** `add_header` ở cấp `server` **không được kế thừa** vào
> `location` nào có `add_header` riêng. Vì vậy snippet header bảo mật phải được
> `include` lại trong mọi location có `add_header` (xem `/assets/` bên dưới).
>
> Nếu bật Google Analytics (`site_settings.seo.google_analytics_id`) hoặc thêm
> font từ Google Fonts, phải bổ sung domain tương ứng vào CSP, nếu không trình
> duyệt sẽ chặn im lặng.

### 7.2. `deploy/nginx/nu-cuoi-em.conf`

```nginx
upstream nce_api {
    server unix:/run/nu-cuoi-em/gunicorn.sock fail_timeout=0;
}

# Bot mạng xã hội không chạy JavaScript → cần HTML có sẵn thẻ Open Graph
map $http_user_agent $is_social_bot {
    default 0;
    ~*(facebookexternalhit|facebot|zalo|twitterbot|linkedinbot|slackbot|telegrambot|discordbot|whatsapp) 1;
}

server {
    listen 80;
    listen [::]:80;
    server_name nucuoiem.org www.nucuoiem.org;
    return 301 https://nucuoiem.org$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.nucuoiem.org;
    ssl_certificate     /etc/ssl/cloudflare/origin.pem;
    ssl_certificate_key /etc/ssl/cloudflare/origin.key;
    return 301 https://nucuoiem.org$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name nucuoiem.org;

    ssl_certificate     /etc/ssl/cloudflare/origin.pem;
    ssl_certificate_key /etc/ssl/cloudflare/origin.key;

    include snippets/cloudflare-realip.conf;
    include snippets/nce-security-headers.conf;

    root  /srv/nu-cuoi-em/current/frontend/dist;
    index index.html;

    client_max_body_size 6m;        # upload ảnh tối đa 5MB + phần đầu multipart

    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;

    # ── API & các route do Django phục vụ ─────────────────────────
    location /api/ {
        include snippets/nce-proxy.conf;
        proxy_pass http://nce_api;
    }

    location /share/ {
        include snippets/nce-proxy.conf;
        proxy_pass http://nce_api;
    }

    location = /sitemap.xml {
        include snippets/nce-proxy.conf;
        proxy_pass http://nce_api;
    }

    location /django-admin/ {
        # allow <IP tin cậy>;   # tuỳ chọn: chỉ cho vài IP truy cập
        # deny all;
        include snippets/nce-proxy.conf;
        proxy_pass http://nce_api;
    }

    location /django-static/ {
        alias /srv/nu-cuoi-em/current/backend/staticfiles/;
        expires 30d;
        access_log off;
    }

    # ── Asset của Vite: tên file có hash → cache vĩnh viễn ───────
    location /assets/ {
        include snippets/nce-security-headers.conf;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        try_files $uri =404;
        access_log off;
    }

    # ── Trang chi tiết: bot mạng xã hội → Django, người dùng → React ─
    location ~ ^/(events|news)/[^/]+/?$ {
        error_page 418 = @share;
        if ($is_social_bot) { return 418; }
        try_files $uri /index.html;
    }

    location @share {
        rewrite ^/(.*)$ /share/$1 break;
        include snippets/nce-proxy.conf;
        proxy_pass http://nce_api;
    }

    # ── index.html không được cache, để bản deploy mới có hiệu lực ngay ─
    location = /index.html {
        include snippets/nce-security-headers.conf;
        add_header Cache-Control "no-cache" always;
    }

    # ── Mọi route còn lại do React Router xử lý ───────────────────
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### 7.3. Cài đặt

```bash
sudo cp /srv/nu-cuoi-em/current/deploy/nginx/nu-cuoi-em.conf /etc/nginx/sites-available/
sudo ln -sfn /etc/nginx/sites-available/nu-cuoi-em.conf /etc/nginx/sites-enabled/nu-cuoi-em.conf
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

**Luôn chạy `nginx -t` trước `reload`.** Cấu hình lỗi mà reload thẳng thì Nginx
giữ cấu hình cũ, nhưng nếu lỡ `restart` thì cả website sập.

---

## 8. Cloudflare

### 8.1. DNS

| Loại | Tên | Giá trị | Proxy |
|------|-----|---------|:-----:|
| `A` | `@` | IP của VPS | 🟠 Proxied |
| `CNAME` | `www` | `nucuoiem.org` | 🟠 Proxied |
| `TXT` / `MX` | theo Resend | theo Resend | ⚪ DNS only |

### 8.2. SSL/TLS

1. **SSL/TLS → Overview → Full (strict).** Không bao giờ dùng *Flexible* — nó
   làm kết nối Cloudflare → VPS không mã hoá và gây vòng lặp chuyển hướng.
2. **SSL/TLS → Origin Server → Create Certificate** — RSA, hostnames
   `nucuoiem.org, *.nucuoiem.org`, thời hạn 15 năm. Lưu lên VPS:

   ```bash
   sudo mkdir -p /etc/ssl/cloudflare
   sudo nano /etc/ssl/cloudflare/origin.pem    # dán Origin Certificate
   sudo nano /etc/ssl/cloudflare/origin.key    # dán Private Key
   sudo chmod 600 /etc/ssl/cloudflare/origin.key
   ```

   Private key chỉ hiện **một lần** — lưu thêm vào trình quản lý mật khẩu.
3. **Edge Certificates**: bật *Always Use HTTPS*, *Automatic HTTPS Rewrites*,
   Minimum TLS Version = 1.2.

### 8.3. Cache Rules

| Thứ tự | Điều kiện | Hành động |
|:------:|-----------|-----------|
| 1 | URI path bắt đầu bằng `/api/` **hoặc** `/share/` **hoặc** `/django-admin/` | **Bypass cache** |
| 2 | URI path bắt đầu bằng `/assets/` | Eligible for cache, Edge TTL 1 tháng |

Quy tắc 1 là **bắt buộc**: API dùng chung đường dẫn cho khách và admin, cache ở
CDN có thể trả dữ liệu admin cho khách ([ARCHITECTURE §4.5](ARCHITECTURE.md#45-cache)).

### 8.4. Bảo mật

- Để mặc định *Security Level*. Nếu bật *Bot Fight Mode*, sau đó phải thử chia sẻ
  một link sự kiện lên Facebook — nếu xem trước bị mất ảnh/tiêu đề, xem
  **Security → Events** để biết Cloudflare có chặn crawler không.
- Có thể thêm WAF rule chặn quốc gia khác cho `/django-admin/` nếu cần.

---

## 9. Lần triển khai đầu tiên

Thứ tự quan trọng — unit systemd phải tồn tại trước khi workflow deploy chạy.

**Bước 1 — Hoàn tất §2 và §3.**

**Bước 2 — Tạo `.env` production** theo §5.

**Bước 3 — Tạo SSH key cho GitHub Actions** (trên máy cá nhân, không đặt passphrase):

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f nce_deploy
```

Chép `nce_deploy.pub` vào `/srv/nu-cuoi-em/.ssh/authorized_keys` trên VPS
(chủ sở hữu `nce`, quyền `600`). Nội dung `nce_deploy` (private) đưa vào GitHub
Secrets ở bước 5, sau đó **xoá file private key khỏi máy**.

**Bước 4 — Cài cấu hình tạm từ repo** (chưa có `current`):

```bash
sudo -u nce git clone --depth 1 --branch main https://github.com/HieuXiao/nu-cuoi-em-web.git /tmp/nce
sudo cp /tmp/nce/deploy/systemd/*.service /tmp/nce/deploy/systemd/*.timer /etc/systemd/system/
sudo cp /tmp/nce/deploy/nginx/nu-cuoi-em.conf /etc/nginx/sites-available/
sudo ln -sfn /etc/nginx/sites-available/nu-cuoi-em.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo systemctl daemon-reload
sudo systemctl enable nu-cuoi-em-api        # enable, CHƯA start
sudo rm -rf /tmp/nce
```

Tạo 3 snippet Nginx ở §7.1 và Origin Certificate ở §8.2, rồi `sudo nginx -t`.

**Bước 5 — Cấu hình GitHub** (Settings → Environments → tạo `production`):

| Secret | Giá trị |
|--------|---------|
| `VPS_HOST` | IP của VPS |
| `VPS_PORT` | `22` |
| `VPS_USER` | `nce` |
| `VPS_SSH_KEY` | Nội dung file `nce_deploy` (private key) |
| `VPS_KNOWN_HOSTS` | Kết quả `ssh-keyscan -p 22 <IP VPS>` |

Bật branch protection cho `main`: bắt buộc PR, bắt buộc CI pass, ít nhất 1 approve
(khớp [`CONTRIBUTING.md`](../../CONTRIBUTING.md)).

**Bước 6 — Chạy deploy lần đầu:** GitHub → Actions → *Deploy* → *Run workflow*
trên nhánh `main`.

**Bước 7 — Sau khi deploy xanh:**

```bash
sudo systemctl reload nginx
sudo systemctl enable --now nu-cuoi-em-scheduler.timer nu-cuoi-em-maintenance.timer nu-cuoi-em-backup.timer

# Tạo tài khoản superadmin
sudo -u nce bash -c 'cd /srv/nu-cuoi-em/current/backend && ../venv/bin/python manage.py createsuperuser'
```

**Bước 8 — Kiểm tra:**

- [ ] `https://nucuoiem.org` hiện trang chủ
- [ ] `https://nucuoiem.org/api/health` trả `"database": "ok"`
- [ ] Đăng nhập được `https://nucuoiem.org/admin/login`
- [ ] `https://nucuoiem.org/django-admin/` hiện trang đăng nhập Django có CSS
- [ ] `curl -I http://<IP VPS>` **bị từ chối** (firewall chỉ nhận Cloudflare)
- [ ] Chia sẻ một link sự kiện lên Facebook → xem trước có ảnh & tiêu đề
      (dùng *Sharing Debugger* của Facebook để kiểm tra)
- [ ] Gửi thử form liên hệ → email thông báo tới hộp thư quản trị
- [ ] `systemctl list-timers 'nu-cuoi-em-*'` hiện đủ 3 timer

---

## 10. CI/CD với GitHub Actions

```
PR vào dev/main ──► ci.yml: lint + test + build (backend & frontend)
push vào dev    ──► ci.yml
merge vào main  ──► deploy.yml: gọi ci.yml → build frontend → đóng gói → scp → deploy.sh trên VPS
```

### 10.1. `.github/workflows/ci.yml`

```yaml
name: CI

on:
  pull_request:
    branches: [dev, main]
  push:
    branches: [dev]
  workflow_call:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    env:
      DJANGO_SETTINGS_MODULE: config.settings.development
      DJANGO_SECRET_KEY: ci-only-not-a-secret
      MONGODB_URI: mongodb://localhost:27017/?directConnection=true&replicaSet=rs0
      MONGODB_DB_NAME: nu_cuoi_em_test
    steps:
      - uses: actions/checkout@v4

      - name: Start MongoDB (replica set 1 node)
        working-directory: .
        run: |
          docker run -d --name mongo -p 27017:27017 mongo:8 --replSet rs0 --bind_ip_all
          for i in $(seq 1 30); do
            docker exec mongo mongosh --quiet --eval "db.adminCommand('ping')" && break
            sleep 1
          done
          docker exec mongo mongosh --quiet --eval "rs.initiate({_id:'rs0',members:[{_id:0,host:'localhost:27017'}]})"

      - uses: actions/setup-python@v5
        with:
          python-version-file: backend/.python-version
          cache: pip
          cache-dependency-path: backend/requirements*.txt

      - run: pip install -r requirements-dev.txt
      - run: ruff check .
      - run: ruff format --check .
      - run: python manage.py makemigrations --check --dry-run
      - run: pytest --cov=apps --cov-report=term-missing

  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run test
      - run: npm run build
        env:
          VITE_API_URL: /api
```

`makemigrations --check` làm CI đỏ nếu ai đó sửa model mà quên tạo migration.

### 10.2. `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: deploy-production
  cancel-in-progress: false      # không bao giờ huỷ một lần deploy đang chạy dở

jobs:
  ci:
    uses: ./.github/workflows/ci.yml

  deploy:
    needs: ci
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Build frontend
        working-directory: frontend
        env:
          VITE_API_URL: /api
          VITE_APP_NAME: Nụ Cười Em
        run: |
          npm ci
          npm run build

      - name: Package release
        run: |
          RELEASE="$(date -u +%Y%m%d-%H%M%S)-${GITHUB_SHA::7}"
          echo "RELEASE=$RELEASE" >> "$GITHUB_ENV"
          git archive --format=tar HEAD backend deploy > release.tar
          tar -rf release.tar frontend/dist
          gzip release.tar

      - name: Configure SSH
        env:
          SSH_KEY: ${{ secrets.VPS_SSH_KEY }}
          KNOWN_HOSTS: ${{ secrets.VPS_KNOWN_HOSTS }}
        run: |
          mkdir -p ~/.ssh
          printf '%s\n' "$SSH_KEY" > ~/.ssh/id_ed25519
          printf '%s\n' "$KNOWN_HOSTS" > ~/.ssh/known_hosts
          chmod 600 ~/.ssh/id_ed25519

      - name: Upload and deploy
        env:
          HOST: ${{ secrets.VPS_HOST }}
          PORT: ${{ secrets.VPS_PORT }}
          USER: ${{ secrets.VPS_USER }}
        run: |
          scp -P "$PORT" release.tar.gz "$USER@$HOST:/tmp/$RELEASE.tar.gz"
          ssh -p "$PORT" "$USER@$HOST" "bash -s -- $RELEASE" < deploy/scripts/deploy.sh
```

Frontend build với `VITE_API_URL=/api` (đường dẫn tương đối) — cùng origin với
website nên không cần CORS, và bản build không gắn cứng tên domain.

### 10.3. `deploy/scripts/deploy.sh` *(cần tạo)*

Chạy trên VPS dưới quyền `nce`.

```bash
#!/usr/bin/env bash
set -euo pipefail

APP=/srv/nu-cuoi-em
RELEASE="$1"
ARCHIVE="/tmp/$RELEASE.tar.gz"
DIR="$APP/releases/$RELEASE"
PY=python3.13
DOMAIN=nucuoiem.org

echo "==> Giải nén $RELEASE"
mkdir -p "$DIR"
tar -xzf "$ARCHIVE" -C "$DIR"
ln -sfn "$APP/shared/.env" "$DIR/backend/.env"

echo "==> Tạo virtualenv và cài dependency"
"$PY" -m venv "$DIR/venv"
"$DIR/venv/bin/pip" install --quiet --upgrade pip
"$DIR/venv/bin/pip" install --quiet -r "$DIR/backend/requirements.txt"

echo "==> Kiểm tra, migrate, collectstatic"
cd "$DIR/backend"                     # settings tự nạp backend/.env (symlink) qua python-dotenv
"$DIR/venv/bin/python" manage.py check --deploy --fail-level ERROR
"$DIR/venv/bin/python" manage.py migrate --noinput
"$DIR/venv/bin/python" manage.py collectstatic --noinput --verbosity 0

echo "==> Chuyển sang bản mới"
PREVIOUS="$(readlink -f "$APP/current" || true)"
ln -sfn "$DIR" "$APP/current"
sudo /usr/bin/systemctl restart nu-cuoi-em-api

echo "==> Health check"
for i in $(seq 1 15); do
  if curl -fsS --unix-socket /run/nu-cuoi-em/gunicorn.sock -H "Host: $DOMAIN" \
       http://localhost/api/health > /dev/null; then
    echo "==> Deploy thành công: $RELEASE"
    ls -1dt "$APP"/releases/* | tail -n +4 | xargs -r rm -rf
    rm -f "$ARCHIVE"

    diff -q "$DIR/deploy/nginx/nu-cuoi-em.conf" /etc/nginx/sites-available/nu-cuoi-em.conf > /dev/null \
      || echo "!! CẢNH BÁO: cấu hình Nginx trong repo khác trên VPS — cài lại thủ công (DEPLOYMENT §7.3)"
    for unit in "$DIR"/deploy/systemd/*; do
      diff -q "$unit" "/etc/systemd/system/$(basename "$unit")" > /dev/null 2>&1 \
        || echo "!! CẢNH BÁO: $(basename "$unit") khác trên VPS — cài lại thủ công (DEPLOYMENT §6.5)"
    done
    exit 0
  fi
  sleep 2
done

echo "!! Health check THẤT BẠI" >&2
if [ -n "$PREVIOUS" ] && [ "$PREVIOUS" != "$DIR" ]; then
  echo "!! Rollback về $PREVIOUS" >&2
  ln -sfn "$PREVIOUS" "$APP/current"
  sudo /usr/bin/systemctl restart nu-cuoi-em-api
fi
exit 1
```

> Script **không** `source .env` bằng bash: `MONGODB_URI` chứa `&`, bash sẽ hiểu
> là chạy nền và biến không được gán. Django tự đọc `.env` qua `python-dotenv`;
> các unit systemd đọc qua `EnvironmentFile=` — cả hai đều hiểu đúng giá trị không bọc nháy.

### 10.4. Quy tắc migration an toàn

Migration chạy **trước** khi chuyển symlink, tức là trong vài chục giây code cũ
chạy trên schema mới. Mọi migration phải **tương thích ngược**:

| Được phép trong một lần deploy | Không được |
|-------------------------------|------------|
| Thêm trường có giá trị mặc định | Đổi tên trường |
| Thêm index | Xoá trường code cũ còn đọc |
| Thêm collection | Đổi kiểu dữ liệu trường đang dùng |

Việc phá vỡ tương thích phải tách hai lần deploy: (1) code mới đọc được cả hai
dạng + migration dữ liệu, (2) dọn dạng cũ.

---

## 11. Rollback

### 11.1. Tự động

`deploy.sh` tự quay về bản trước nếu health check thất bại trong 30 giây.

### 11.2. Thủ công — khi lỗi chỉ lộ ra sau khi deploy xanh

**Cách ưu tiên** — giữ `main` luôn trùng với production:

```bash
git revert <commit lỗi>
# mở PR → merge vào main → deploy tự chạy
```

**Cách khẩn cấp** — khi cần quay lại ngay trong vài giây:

```bash
ls -1t /srv/nu-cuoi-em/releases/                          # xem các bản còn giữ
sudo -u nce ln -sfn /srv/nu-cuoi-em/releases/<bản-trước> /srv/nu-cuoi-em/current
sudo systemctl restart nu-cuoi-em-api
```

Sau đó vẫn phải `git revert` trên `main`, nếu không lần deploy kế tiếp sẽ đưa lỗi trở lại.

> Rollback code **không** rollback database. Migration đã chạy vẫn giữ nguyên —
> đó là lý do quy tắc tương thích ngược ở §10.4 là bắt buộc. Khôi phục database
> từ backup (§12.3) là biện pháp cuối cùng vì mất mọi dữ liệu phát sinh sau bản backup.

---

## 12. Sao lưu & khôi phục

**Atlas M0 không có backup tự động.** Dự án tự sao lưu bằng `mongodump`.

| Dữ liệu | Cách sao lưu | Tần suất | Giữ |
|---------|-------------|----------|-----|
| MongoDB | `mongodump` → VPS → Google Drive | Hằng ngày 02:30 | 7 ngày trên VPS, 30 ngày trên Drive |
| `.env` production | Bản sao thủ công trong trình quản lý mật khẩu | Mỗi lần sửa | Vĩnh viễn |
| Origin Certificate + key | Trình quản lý mật khẩu | Một lần | Vĩnh viễn |
| Ảnh gốc | Thư mục Google Drive của nhóm (Cloudinary free không có backup) | Sau mỗi sự kiện | Vĩnh viễn |
| Mã nguồn & cấu hình | GitHub | Liên tục | — |

### 12.1. `deploy/scripts/backup.sh` *(cần tạo)*

```bash
#!/usr/bin/env bash
# Chạy bởi nu-cuoi-em-backup.service — MONGODB_URI, MONGODB_DB_NAME đến từ EnvironmentFile
set -euo pipefail

DIR=/srv/nu-cuoi-em/backups/daily
FILE="$DIR/nce-$(date +%Y%m%d).archive.gz"
REMOTE=gdrive:nu-cuoi-em-backups

mkdir -p "$DIR"
mongodump --uri="$MONGODB_URI" --db="$MONGODB_DB_NAME" --gzip --archive="$FILE" --quiet

rclone copy "$FILE" "$REMOTE"
rclone delete "$REMOTE" --min-age 30d

find "$DIR" -name '*.archive.gz' -mtime +7 -delete
echo "Backup OK: $FILE ($(du -h "$FILE" | cut -f1))"
```

Đẩy lên Drive **mỗi ngày**, không phải mỗi tuần — nếu VPS chết, backup nằm trên
chính VPS đó cũng mất theo.

### 12.2. Cấu hình rclone

VPS không có trình duyệt, cần xác thực Google Drive từ máy cá nhân:

```bash
# Trên máy cá nhân (đã cài rclone)
rclone authorize "drive"          # đăng nhập Google, copy token in ra

# Trên VPS
sudo -u nce rclone config         # n → tên "gdrive" → storage "drive" → dán token khi được hỏi
sudo -u nce rclone lsd gdrive:    # kiểm tra
sudo -u nce rclone mkdir gdrive:nu-cuoi-em-backups
```

### 12.3. Khôi phục

```bash
# Lấy file backup (từ VPS hoặc Drive)
rclone copy gdrive:nu-cuoi-em-backups/nce-20260910.archive.gz .

# Khôi phục — --drop xoá collection hiện có trước khi nạp
mongorestore --uri="$MONGODB_URI" --gzip --archive=nce-20260910.archive.gz \
  --nsInclude='nu_cuoi_em.*' --drop
```

> `--drop` xoá dữ liệu hiện tại. Luôn chạy `mongodump` một bản của trạng thái hiện
> tại trước khi restore, kể cả khi dữ liệu đang hỏng.

### 12.4. Diễn tập khôi phục hằng tháng

Backup chưa từng được thử khôi phục thì coi như không có backup. Mỗi tháng:

```bash
docker compose -f docker/docker-compose.yml up -d
mongorestore --uri="mongodb://localhost:27017/?directConnection=true" --gzip \
  --archive=nce-<ngày>.archive.gz --nsInclude='nu_cuoi_em.*' --drop
mongosh "mongodb://localhost:27017/nu_cuoi_em?directConnection=true" --eval "db.events.countDocuments()"
```

Dữ liệu này chứa thông tin cá nhân thật — **xoá container và file sau khi diễn tập**
(`docker compose ... down -v`).

---

## 13. Giám sát & log

### 13.1. Uptime

UptimeRobot (free) — monitor HTTP(s) tới `https://nucuoiem.org/api/health`, chu kỳ
5 phút, cảnh báo qua email và Telegram. Endpoint trả `503` khi mất kết nối
database nên monitor theo status code là đủ.

### 13.2. Xem log

| Cần xem | Lệnh |
|---------|------|
| Log Django/Gunicorn trực tiếp | `journalctl -u nu-cuoi-em-api -f` |
| Lỗi trong 1 giờ qua | `journalctl -u nu-cuoi-em-api --since "1 hour ago" -p err` |
| Tác vụ nền hôm nay | `journalctl -u 'nu-cuoi-em-*' --since today` |
| Lần chạy timer kế tiếp | `systemctl list-timers 'nu-cuoi-em-*'` |
| Unit đang lỗi | `systemctl --failed` |
| Truy cập Nginx | `sudo tail -f /var/log/nginx/access.log` |
| Lỗi Nginx | `sudo tail -f /var/log/nginx/error.log` |
| RAM, swap | `free -h` |
| Ổ đĩa | `df -h /` và `du -sh /srv/nu-cuoi-em/*` |

Giới hạn dung lượng journald để log không làm đầy ổ 20GB:

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=200M\n' | sudo tee /etc/systemd/journald.conf.d/size.conf
sudo systemctl restart systemd-journald
```

Nginx đã có `logrotate` mặc định (giữ 14 ngày).

### 13.3. Theo dõi lỗi (tuỳ chọn)

Điền `SENTRY_DSN` để gửi exception về Sentry (gói miễn phí đủ cho lưu lượng dự
án). Cấu hình `send_default_pii=False` — không gửi email, IP người dùng lên Sentry.

---

## 14. Bảo trì định kỳ

| Tần suất | Việc |
|----------|------|
| **Hằng tuần** | Kiểm tra UptimeRobot không có sự cố chưa xử lý · Drive có đủ backup 7 ngày gần nhất · `systemctl --failed` trống |
| **Hằng tháng** | Diễn tập khôi phục (§12.4) · Xem mức dùng Cloudinary / Atlas / Resend (theo [`Phan-tich-chi-phi-van-hanh.md`](../project/Phan-tich-chi-phi-van-hanh.md)) · Khởi động lại nếu có `/var/run/reboot-required` · Rà danh sách tài khoản admin, vô hiệu hoá người đã rời nhóm |
| **Hằng quý** | Cập nhật dependency (`pip list --outdated`, `npm outdated`), đọc security release của Django · Sinh lại danh sách IP Cloudflare (§3.6, §7.1) · Dọn release và file thừa |
| **Hằng năm** | Gia hạn domain (nhắc trước 30 ngày) · Gia hạn VPS · Xoay vòng SSH key của GitHub Actions và mật khẩu DB · Rà lại ngân sách |

| Mốc cần nhớ | Ngày hết hạn |
|-------------|--------------|
| Domain | _điền khi mua_ |
| VPS | _điền khi thuê_ |
| Cloudflare Origin Certificate | _ngày tạo + 15 năm_ |

> **Cluster Atlas dev** không có kết nối trong 60 ngày sẽ bị Atlas tạm dừng. Cluster
> production luôn có lưu lượng nên không bị ảnh hưởng; nếu cluster dev bị dừng, bấm
> *Resume* trong giao diện Atlas.

---

## 15. Xử lý sự cố

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
|-------------|-----------------------|------------|
| **502 Bad Gateway** | Gunicorn không chạy hoặc Nginx không đọc được socket | `systemctl status nu-cuoi-em-api`; `ls -l /run/nu-cuoi-em/` — socket phải thuộc group `www-data` |
| **ERR_TOO_MANY_REDIRECTS** | Cloudflare đang ở chế độ *Flexible*, hoặc bật `SECURE_SSL_REDIRECT` | Đặt Cloudflare *Full (strict)*; `SECURE_SSL_REDIRECT = False` (§5.1) |
| **Error 526** từ Cloudflare | Origin Certificate sai / thiếu | Kiểm tra `/etc/ssl/cloudflare/`, `sudo nginx -t` |
| **Error 521/522** từ Cloudflare | Nginx không chạy hoặc firewall chặn IP Cloudflare | `systemctl status nginx`; `sudo ufw status` |
| **400 Bad Request** mọi request API | Domain không có trong `DJANGO_ALLOWED_HOSTS` | Sửa `.env`, restart API |
| **403 CSRF** khi đăng nhập Django Admin | Thiếu `CSRF_TRUSTED_ORIGINS=https://...` | Sửa `.env`, restart API |
| Django Admin mất CSS | Chưa `collectstatic` hoặc sai `alias` Nginx | Kiểm tra `backend/staticfiles/` của bản hiện tại |
| **413** khi upload ảnh | Ảnh > giới hạn | Frontend phải resize trước upload; kiểm tra `client_max_body_size` |
| Mọi người cùng bị **429** | Throttle đang đếm theo IP Cloudflare | Thiếu snippet `cloudflare-realip.conf` hoặc `NUM_PROXIES` (§5.1, §7.1) |
| `ServerSelectionTimeoutError` | IP VPS chưa có trong Atlas Network Access | Thêm IP, chờ 1–2 phút |
| Sự kiện hẹn giờ không tự công bố | Timer không chạy | `systemctl list-timers`; `journalctl -u nu-cuoi-em-scheduler` |
| Website chậm dần rồi treo | Hết RAM, đang swap nhiều | `free -h`; `journalctl -k \| grep -i oom`; kiểm tra `MemoryMax` |
| Chia sẻ Facebook không có ảnh | Bot không vào được `/share/` hoặc bị Cloudflare chặn | Thử bằng Facebook Sharing Debugger; `curl -A facebookexternalhit https://nucuoiem.org/events/<slug>` phải trả HTML có `og:image` |
| Deploy báo lỗi ở bước `pip install` | Hết RAM khi build gói | Kiểm tra swap đã bật (§3.3) |
| Deploy xanh nhưng trang vẫn là bản cũ | Trình duyệt/CDN cache `index.html` | Kiểm tra location `= /index.html` có `no-cache`; Cloudflare → *Purge Everything* |
| Backup không có trên Drive | Token rclone hết hạn | `sudo -u nce rclone lsd gdrive:`; nếu lỗi, làm lại §12.2 |
