# Hướng dẫn cài đặt môi trường phát triển — Nụ Cười Em

Tài liệu này hướng dẫn dựng môi trường dev từ đầu trên máy cá nhân. Triển khai
production xem [`DEPLOYMENT.md`](DEPLOYMENT.md).

> **Trạng thái repo:** đây là giai đoạn khởi tạo, nhiều file cấu hình trong repo
> vẫn **rỗng** (`requirements.txt`, `package.json`, `docker-compose.yml`,
> `Makefile`, `pytest.ini`, `.nvmrc`…). [Phụ lục §11](#11-phụ-lục--nội-dung-tối-thiểu-cho-file-cấu-hình)
> cung cấp nội dung tối thiểu để điền vào. Khi các file đó đã có nội dung trên
> nhánh `dev`, bạn chỉ cần làm theo §3–§6.

---

## Mục lục

1. [Yêu cầu hệ thống](#1-yêu-cầu-hệ-thống)
2. [Tổng quan monorepo](#2-tổng-quan-monorepo)
3. [Clone repository](#3-clone-repository)
4. [Cài đặt Backend](#4-cài-đặt-backend)
5. [Cài đặt Frontend](#5-cài-đặt-frontend)
6. [Chạy song song cả hai](#6-chạy-song-song-cả-hai)
7. [Kiểm tra cài đặt thành công](#7-kiểm-tra-cài-đặt-thành-công)
8. [Lệnh thường dùng](#8-lệnh-thường-dùng)
9. [Cấu hình IDE](#9-cấu-hình-ide)
10. [Xử lý sự cố](#10-xử-lý-sự-cố)
11. [Phụ lục — nội dung tối thiểu cho file cấu hình](#11-phụ-lục--nội-dung-tối-thiểu-cho-file-cấu-hình)

---

## 1. Yêu cầu hệ thống

| Công cụ | Phiên bản | Bắt buộc | Ghi chú |
|---------|-----------|:--------:|---------|
| [Python](https://www.python.org/downloads/) | **3.13** (3.12 cũng chạy) | ✔ | Ghi trong `backend/.python-version` |
| [Node.js](https://nodejs.org/) | **22 LTS** (tối thiểu 20) | ✔ | Ghi trong `.nvmrc` |
| [Git](https://git-scm.com/) | 2.40+ | ✔ | |
| MongoDB | **6.0+** | ✔ | Chọn Atlas (§4.4a) **hoặc** Docker (§4.4b) |
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | 24+ | ✘ | Chỉ cần nếu chạy MongoDB local |
| [mongosh](https://www.mongodb.com/try/download/shell) | 2.x | ✘ | Tiện để soi dữ liệu |
| `make` | — | ✘ | Không có sẵn trên Windows, xem §6.2 |

**Kiểm tra nhanh:**

```bash
python --version
node --version
git --version
docker --version
```

Nếu thiếu Node.js, cài qua [nvm-windows](https://github.com/coreybutler/nvm-windows)
(Windows) hoặc [nvm](https://github.com/nvm-sh/nvm) (macOS/Linux) để dùng đúng
phiên bản trong `.nvmrc`:

```bash
nvm install 22
nvm use 22
```

### 1.1. Quy tắc tương thích quan trọng

> **`django-mongodb-backend` phải cùng minor version với Django.**
> Django 5.2 ↔ `django-mongodb-backend` 5.2.x. Nâng Django 5.2 → 6.0 mà không nâng
> backend tương ứng sẽ lỗi ngay khi khởi động. Luôn nâng hai gói này cùng nhau.

---

## 2. Tổng quan monorepo

```
nu-cuoi-em-web/
├── frontend/     React + Vite          → chạy ở http://localhost:5173
├── backend/      Django + DRF          → chạy ở http://localhost:8000
├── docker/       docker-compose MongoDB cho dev local
├── deploy/       Nginx, Gunicorn, systemd (chỉ dùng cho production)
└── DOC/          Tài liệu dự án
```

Frontend và backend là **hai tiến trình riêng biệt**, giao tiếp qua REST API.
Trong dev, frontend gọi thẳng `http://localhost:8000/api` (khai báo ở
`VITE_API_URL`) và backend cho phép origin `http://localhost:5173` qua CORS.

---

## 3. Clone repository

```bash
git clone https://github.com/HieuXiao/nu-cuoi-em-web.git
cd nu-cuoi-em-web
git checkout dev
```

Luôn làm việc từ nhánh `dev`, không commit trực tiếp lên `main`.
Quy trình nhánh và commit xem [`CONTRIBUTING.md`](../../CONTRIBUTING.md).

---

## 4. Cài đặt Backend

### 4.1. Tạo môi trường ảo

**Windows (PowerShell)**

```powershell
cd backend
py -3.13 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Nếu PowerShell chặn script (`... cannot be loaded because running scripts is disabled`):

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

**macOS / Linux**

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
```

Dấu hiệu thành công: dấu nhắc lệnh có tiền tố `(.venv)`.

### 4.2. Cài dependencies

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt -r requirements-dev.txt
```

`requirements.txt` chứa gói chạy production; `requirements-dev.txt` chứa công cụ
dev (pytest, ruff, black). Nội dung gợi ý ở [§11.3](#113-backendrequirementstxt).

### 4.3. Tạo file `.env`

```bash
cp .env.example .env      # PowerShell: Copy-Item .env.example .env
```

Mở `backend/.env` và điền:

| Biến | Bắt buộc | Ghi chú |
|------|:--------:|---------|
| `DJANGO_SECRET_KEY` | ✔ | Sinh bằng lệnh bên dưới, **không dùng lại giữa các môi trường** |
| `MONGODB_URI` | ✔ | Xem §4.4 |
| `MONGODB_DB_NAME` | ✔ | Để `nu_cuoi_em` |
| `DJANGO_DEBUG` | ✔ | `True` khi dev |
| `CORS_ALLOWED_ORIGINS` | ✔ | `http://localhost:5173` |
| `CLOUDINARY_*` | ✘ | Bỏ trống thì tính năng upload ảnh sẽ lỗi, phần còn lại vẫn chạy |
| `EMAIL_API_KEY` | ✘ | Bỏ trống → email in ra console thay vì gửi thật |

Sinh `DJANGO_SECRET_KEY`:

```bash
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

> `.env` đã nằm trong `.gitignore`. **Không bao giờ commit file này** —
> xem checklist bảo mật trong [`CONTRIBUTING.md`](../../CONTRIBUTING.md).

### 4.4. Chuẩn bị MongoDB

Chọn **một** trong hai cách.

#### 4.4a. MongoDB Atlas (khuyến nghị — giống production)

1. Tạo tài khoản tại [mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)
   và tạo cluster **M0 (free tier, 512MB)**, chọn region gần Việt Nam
   (Singapore `ap-southeast-1`).
2. **Database Access** → tạo user với quyền `readWrite` trên database `nu_cuoi_em`.
3. **Network Access** → thêm IP hiện tại của bạn. Nếu IP nhà mạng thay đổi liên
   tục, dùng `0.0.0.0/0` **chỉ cho cluster dev**, tuyệt đối không cho production.
4. **Connect → Drivers → Python** để lấy chuỗi kết nối, dán vào `MONGODB_URI`:

   ```
   MONGODB_URI=mongodb+srv://nce_dev:<password>@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority
   ```

> Mật khẩu chứa ký tự đặc biệt (`@ : / ? # [ ] %`) phải được **URL-encode**.
> Ví dụ `p@ss:word` → `p%40ss%3Aword`. Đây là nguyên nhân phổ biến nhất của lỗi
> `Authentication failed`.

#### 4.4b. MongoDB local bằng Docker

```bash
docker compose -f docker/docker-compose.yml up -d
docker compose -f docker/docker-compose.yml logs -f mongo   # Ctrl+C để thoát
```

Trong `backend/.env`:

```
MONGODB_URI=mongodb://localhost:27017/?directConnection=true&replicaSet=rs0
```

> MongoDB chạy dạng **replica set 1 node** (`--replSet rs0`) chứ không phải node
> đơn lẻ. Lý do: node đơn lẻ không hỗ trợ transaction, và ta muốn môi trường dev
> giống Atlas. File compose ở [§11.5](#115-dockerdocker-composeyml) đã tự khởi tạo
> replica set trong `healthcheck`.

Dừng và xoá dữ liệu local:

```bash
docker compose -f docker/docker-compose.yml down -v
```

### 4.5. Cấu hình `django-mongodb-backend`

Trong `backend/config/settings/base.py`, phần database phải có dạng:

```python
import os
from pathlib import Path

import django_mongodb_backend
from dotenv import load_dotenv

BASE_DIR = Path(__file__).resolve().parent.parent.parent
load_dotenv(BASE_DIR / ".env")        # nạp backend/.env; không ghi đè biến đã có sẵn

DATABASES = {
    "default": django_mongodb_backend.parse_uri(
        os.environ["MONGODB_URI"],
        db_name=os.environ["MONGODB_DB_NAME"],
    ),
}

DEFAULT_AUTO_FIELD = "django_mongodb_backend.fields.ObjectIdAutoField"

# Các app contrib của Django cần migration riêng dùng ObjectId làm khoá chính.
MIGRATION_MODULES = {
    "admin": "mongo_migrations.admin",
    "auth": "mongo_migrations.auth",
    "contenttypes": "mongo_migrations.contenttypes",
}
```

Bốn điểm dễ sai:

- **`load_dotenv()` phải chạy trước** mọi `os.environ[...]`, nếu không `.env`
  không có tác dụng và `MONGODB_URI` báo `KeyError`.
- **Không** viết `ENGINE` thủ công — `parse_uri()` tự dựng toàn bộ dict.
- **`DEFAULT_AUTO_FIELD`** bắt buộc, nếu thiếu Django sẽ cố dùng `BigAutoField`
  và migration sẽ hỏng.
- **`MIGRATION_MODULES`** trỏ tới thư mục `backend/mongo_migrations/` — cần tồn
  tại kèm `__init__.py`. Nếu không có, chạy `migrate` sẽ lỗi ở app `contenttypes`.

> Cú pháp cấu hình có thể đổi giữa các minor version. Khi nâng cấp, đối chiếu lại
> với tài liệu chính thức của phiên bản đang cài
> (`pip show django-mongodb-backend`).

### 4.6. Chạy migration

```bash
python manage.py makemigrations
python manage.py migrate
```

Kiểm tra collection đã được tạo (nếu có `mongosh`):

```bash
mongosh "$MONGODB_URI" --eval "use nu_cuoi_em; db.getCollectionNames()"
```

Danh sách collection kỳ vọng xem
[`DATABASE_SCHEMA.md` §1.4](DATABASE_SCHEMA.md#14-danh-sách-collection).

### 4.7. Tạo tài khoản quản trị

```bash
python manage.py createsuperuser
```

Nhập **email** (không phải username) — `apps.accounts` dùng email làm định danh
đăng nhập. Tài khoản này có `role = "superadmin"`, toàn quyền.

### 4.8. Nạp dữ liệu mẫu (tuỳ chọn)

```bash
python scripts/seed_data.py
```

Script tạo vài sự kiện, bài viết, khoản quyên góp và đơn tình nguyện để giao diện
không trống trơn khi dev.

> Script **xoá sạch dữ liệu nghiệp vụ** trước khi nạp lại. Chỉ chạy trên database
> dev, không bao giờ trỏ vào cluster production.

### 4.9. Khởi động backend

```bash
python manage.py runserver
```

API chạy ở `http://localhost:8000/api`, Django Admin ở `http://localhost:8000/django-admin/`.

> Django Admin **không** đặt ở `/admin/` vì `/admin/*` là route của trang quản trị
> React — ở production cả hai chạy chung một domain. Đường dẫn lấy từ biến
> `DJANGO_ADMIN_URL` (mặc định `django-admin/`), xem
> [`ARCHITECTURE.md` §4.4](ARCHITECTURE.md#44-đường-dẫn).

---

## 5. Cài đặt Frontend

### 5.1. Cài dependencies

```bash
cd frontend
nvm use            # đọc .nvmrc, bỏ qua nếu không dùng nvm
npm ci             # dùng npm install nếu chưa có package-lock.json
```

### 5.2. Tạo file `.env`

```bash
cp .env.example .env      # PowerShell: Copy-Item .env.example .env
```

```
VITE_API_URL=http://localhost:8000/api
VITE_APP_NAME=Nụ Cười Em
```

> Vite chỉ đưa biến có tiền tố **`VITE_`** vào bundle. Sửa `.env` xong **phải
> khởi động lại dev server** — Vite không hot-reload biến môi trường.
>
> Mọi biến `VITE_*` đều **hiện trong mã nguồn phía trình duyệt**. Không bao giờ
> đặt secret, API key hay token vào đây.

### 5.3. Khởi động frontend

```bash
npm run dev
```

Mở `http://localhost:5173`.

---

## 6. Chạy song song cả hai

### 6.1. Hai terminal (cách đơn giản nhất)

| Terminal | Thư mục | Lệnh |
|----------|---------|------|
| 1 | `backend/` | `.venv\Scripts\Activate.ps1` rồi `python manage.py runserver` |
| 2 | `frontend/` | `npm run dev` |

### 6.2. Dùng `make` (macOS / Linux / Git Bash có make)

```bash
make dev
```

> `make` **không có sẵn trên Windows**. Cài qua
> `winget install GnuWin32.Make`, hoặc dùng cách §6.1, hoặc dùng script npm ở §6.3.

### 6.3. Dùng npm workspaces từ thư mục gốc

Nếu `package.json` ở root đã cấu hình workspaces + `concurrently`:

```bash
npm install
npm run dev
```

---

## 7. Kiểm tra cài đặt thành công

Chạy lần lượt, cả 4 bước đều phải đạt:

**1. Backend sống và kết nối được database**

```bash
curl http://localhost:8000/api/health
```

Kỳ vọng `"status": "ok"` **và** `"database": "ok"`. Nếu `database: "error"` →
sai `MONGODB_URI` hoặc chưa mở IP trên Atlas.

**2. Đăng nhập được**

```bash
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"admin@nucuoiem.org\",\"password\":\"mat-khau-cua-ban\"}"
```

Kỳ vọng `200` với `data.access` và `data.refresh`.

**3. Frontend gọi được API** — mở `http://localhost:5173`, bật DevTools tab
Network. Request tới `/api/settings/public` phải trả `200`. Nếu thấy lỗi CORS →
kiểm tra `CORS_ALLOWED_ORIGINS` trong `backend/.env`.

**4. Đăng nhập được vào trang quản trị** — vào `http://localhost:5173/admin/login`
bằng tài khoản đã tạo ở §4.7.

---

## 8. Lệnh thường dùng

Chạy trong `backend/` với venv đã kích hoạt:

| Việc cần làm | Lệnh |
|--------------|------|
| Chạy test | `pytest` |
| Test kèm coverage | `pytest --cov=apps --cov-report=html` (mở `htmlcov/index.html`) |
| Chạy test một app | `pytest apps/events` |
| Lint | `ruff check .` |
| Tự sửa lỗi lint | `ruff check . --fix` |
| Format | `ruff format .` |
| Tạo migration | `python manage.py makemigrations` |
| Xem SQL/thao tác của migration | `python manage.py sqlmigrate <app> <số>` |
| Django shell | `python manage.py shell` |
| Công bố sự kiện đã hẹn giờ | `python manage.py publish_scheduled_events` |
| Dọn refresh token hết hạn | `python manage.py flushexpiredtokens` |
| Tính lại bộ đếm bị lệch | `python manage.py recalculate_counters` |

Chạy trong `frontend/`:

| Việc cần làm | Lệnh |
|--------------|------|
| Dev server | `npm run dev` |
| Build production | `npm run build` |
| Xem thử bản build | `npm run preview` |
| Lint | `npm run lint` |
| Format | `npm run format` |
| Test | `npm run test` |

> Các management command `publish_scheduled_events`, `recalculate_counters` được
> mô tả trong [`DATABASE_SCHEMA.md`](DATABASE_SCHEMA.md) — nếu chưa được cài đặt
> trong `apps/*/management/commands/`, lệnh sẽ báo `Unknown command`.

---

## 9. Cấu hình IDE

### 9.1. VS Code — extension nên cài

| Extension | Dùng để |
|-----------|---------|
| Python + Pylance | Hoàn thiện mã Python, trỏ đúng interpreter |
| Ruff | Lint & format Python theo cấu hình repo |
| ESLint | Lint JavaScript/JSX |
| Prettier | Format JS/JSX/CSS theo `.prettierrc` |
| Tailwind CSS IntelliSense | Gợi ý class Tailwind |
| EditorConfig for VS Code | Áp dụng `.editorconfig` |

### 9.2. `.vscode/settings.json` gợi ý

```json
{
  "python.defaultInterpreterPath": "backend/.venv/Scripts/python.exe",
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["backend"],
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true
  },
  "[javascript][javascriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
  },
  "files.exclude": { "**/__pycache__": true, "**/.pytest_cache": true }
}
```

Trên macOS/Linux đổi interpreter thành `backend/.venv/bin/python`.
Thư mục `.vscode/` không được commit — mỗi người tự cấu hình máy mình.

---

## 10. Xử lý sự cố

| Triệu chứng | Nguyên nhân & cách xử lý |
|-------------|--------------------------|
| `Activate.ps1 cannot be loaded because running scripts is disabled` | Chạy `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` rồi kích hoạt lại venv |
| `ServerSelectionTimeoutError` khi `migrate` | IP chưa được thêm vào Network Access của Atlas, hoặc container MongoDB chưa chạy (`docker compose ps`) |
| `Authentication failed` với Atlas | Mật khẩu trong `MONGODB_URI` chưa được URL-encode (§4.4a), hoặc user chưa có quyền trên `nu_cuoi_em` |
| `MongoServerError: not primary` (Docker) | Replica set chưa khởi tạo. Chạy `docker compose exec mongo mongosh --eval "rs.initiate()"` |
| `ImproperlyConfigured: settings.DATABASES ENGINE` | Thiếu `parse_uri()` hoặc `MONGODB_URI` rỗng — kiểm tra `.env` đã được nạp chưa (§4.5) |
| `migrate` lỗi ở app `contenttypes` | Thiếu `MIGRATION_MODULES` hoặc thư mục `backend/mongo_migrations/` (§4.5) |
| `django.db.utils.NotSupportedError` khi dùng transaction | MongoDB đang chạy node đơn lẻ. Dùng replica set (§4.4b) |
| Trình duyệt báo lỗi CORS | `CORS_ALLOWED_ORIGINS` trong `backend/.env` chưa có `http://localhost:5173`; sửa xong phải khởi động lại backend |
| Frontend gọi API ra `undefined/api/...` | `VITE_API_URL` chưa được đặt, hoặc đã sửa `.env` nhưng chưa restart `npm run dev` |
| Đăng nhập xong bị đá ra ngay | Access token hết hạn quá nhanh — kiểm tra `JWT_ACCESS_TOKEN_LIFETIME_MINUTES`, và đồng hồ hệ thống có bị lệch không |
| `Port 8000 is already in use` | `netstat -ano \| findstr :8000` rồi `taskkill /PID <pid> /F` (Windows); hoặc `python manage.py runserver 8001` |
| `Port 5173 is already in use` | `npm run dev -- --port 5174` |
| `make: command not found` | `make` không có sẵn trên Windows — dùng §6.1 hoặc §6.3 |
| Upload ảnh trả `502 UPLOAD_FAILED` | Chưa điền `CLOUDINARY_*` trong `.env`, hoặc sai API secret |
| Email không được gửi khi test | Bình thường khi `EMAIL_API_KEY` rỗng — nội dung email in ra console của backend |
| `ModuleNotFoundError` sau khi `git pull` | Có dependency mới: chạy lại `pip install -r requirements.txt` và `npm ci` |

Nếu vẫn bí, bật log chi tiết bằng `DJANGO_DEBUG=True` và đọc traceback đầy đủ
trong terminal của backend.

---

## 11. Phụ lục — nội dung tối thiểu cho file cấu hình

Các file dưới đây hiện đang rỗng trong repo. Đây là nội dung tối thiểu để môi
trường dev chạy được; điều chỉnh khi dự án phát triển thêm.

> **Về phiên bản:** các số phiên bản dưới đây là mốc tham khảo tại thời điểm viết
> tài liệu. Sau khi cài lần đầu, chốt lại bằng `pip freeze` / `package-lock.json`
> để cả team dùng đúng một bộ. Riêng cặp Django ↔ `django-mongodb-backend` phải
> tuân thủ quy tắc ở [§1.1](#11-quy-tắc-tương-thích-quan-trọng).

### 11.1. `.nvmrc`

```
22
```

### 11.2. `backend/.python-version`

```
3.13
```

### 11.3. `backend/requirements.txt`

```
Django~=5.2.0
django-mongodb-backend~=5.2.0
djangorestframework~=3.16
djangorestframework-simplejwt~=5.5
django-cors-headers~=4.7
django-filter~=25.1
python-dotenv~=1.1
cloudinary~=1.44
nh3~=0.2
gunicorn~=23.0
```

`nh3` dùng để sanitize HTML từ WYSIWYG trước khi lưu — bắt buộc, xem
[`API.md` §5.3](API.md#53-post-apiarticlessubmit--khách-gửi-bài).

### 11.4. `backend/requirements-dev.txt`

```
-r requirements.txt
pytest~=8.3
pytest-django~=4.9
pytest-cov~=6.0
ruff~=0.9
model-bakery~=1.20
```

### 11.5. `docker/docker-compose.yml`

```yaml
services:
  mongo:
    image: mongo:8
    container_name: nce-mongo
    restart: unless-stopped
    command: ["--replSet", "rs0", "--bind_ip_all"]
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: >
        mongosh --quiet --eval "
          try { rs.status().ok }
          catch (e) { rs.initiate({_id:'rs0', members:[{_id:0, host:'localhost:27017'}]}) }
        "
      interval: 10s
      timeout: 10s
      retries: 10
      start_period: 20s

volumes:
  mongo_data:
```

### 11.6. `backend/pytest.ini`

```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.development
python_files = test_*.py
testpaths = apps
addopts = -ra --strict-markers
```

### 11.7. `Makefile` (thư mục gốc)

```makefile
.PHONY: dev dev-backend dev-frontend test lint mongo-up mongo-down

dev:                    # chạy song song backend + frontend, Ctrl+C dừng cả hai
	$(MAKE) -j2 dev-backend dev-frontend

dev-backend:
	cd backend && python manage.py runserver

dev-frontend:
	cd frontend && npm run dev

test:
	cd backend && pytest
	cd frontend && npm run test

lint:
	cd backend && ruff check .
	cd frontend && npm run lint

mongo-up:
	docker compose -f docker/docker-compose.yml up -d

mongo-down:
	docker compose -f docker/docker-compose.yml down
```

### 11.8. `frontend/package.json` — phần `scripts`

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "format": "prettier --write \"src/**/*.{js,jsx,css}\"",
    "test": "vitest run"
  }
}
```

---

## 12. Tài liệu liên quan

- [`ARCHITECTURE.md`](ARCHITECTURE.md) — kiến trúc tổng thể hệ thống
- [`DATABASE_SCHEMA.md`](DATABASE_SCHEMA.md) — cấu trúc collection và index
- [`API.md`](API.md) — hợp đồng REST API
- [`DEPLOYMENT.md`](DEPLOYMENT.md) — triển khai lên VPS
- [`CONTRIBUTING.md`](../../CONTRIBUTING.md) — quy trình nhánh, commit, checklist bảo mật
