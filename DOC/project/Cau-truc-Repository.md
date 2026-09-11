# Cấu trúc Repository — Nụ Cười Em

Tài liệu mô tả vai trò của từng thư mục và file trong `nu-cuoi-em-web`, quy ước
đặt code mới, và danh sách những gì còn thiếu trước khi bắt đầu giai đoạn code.

| Hạng mục | Giá trị |
|----------|---------|
| Repository | `github.com/HieuXiao/nu-cuoi-em-web` |
| Mô hình | Monorepo — frontend và backend chung một repo, deploy chung một VPS |
| Visibility | Public |
| License | MIT |
| Nhánh chính | `main` (production), `dev` (tích hợp) — xem [`CONTRIBUTING.md`](../../CONTRIBUTING.md) |

> **Lưu ý:** mục 6 "Cấu trúc Repository" trong `Mo-ta-Du-an-Nu-Cuoi-Em.docx` mô tả
> cấu trúc Next.js + Express (`src/app/`, `controllers/`, `routes/`…) — **không còn
> đúng**. Tài liệu này mô tả cấu trúc thực tế của repo. Lý do thay đổi stack xem
> [`ARCHITECTURE.md` §3.1](../technical/ARCHITECTURE.md#31-khác-biệt-so-với-tài-liệu-mô-tả-dự-án-v20).

---

## 1. Tổng quan

```
nu-cuoi-em-web/
├── frontend/          Ứng dụng React + Vite (SPA)
├── backend/           Django project + Django REST Framework
├── deploy/            Cấu hình Nginx, Gunicorn, systemd, script deploy/backup
├── docker/            docker-compose MongoDB cho dev local
├── DOC/               Tài liệu dự án
│   ├── project/       Mô tả dự án, cấu trúc repo, chi phí vận hành
│   └── technical/     Kiến trúc, schema, API, cài đặt, triển khai
├── .github/workflows/ CI (test mỗi PR) và CD (deploy khi merge main)
├── Makefile           Lệnh tắt cho dev
├── package.json       Root — npm workspaces (tuỳ chọn)
└── README.md
```

**Nguyên tắc:**

- `frontend/` và `backend/` **độc lập hoàn toàn**: mỗi bên có dependency, `.env`,
  `.gitignore`, test riêng. Chúng chỉ biết nhau qua hợp đồng trong
  [`API.md`](../technical/API.md).
- Không có code dùng chung giữa hai bên. Enum (loại sự kiện, trạng thái…) được
  khai báo hai lần — `backend/apps/*/models.py` và `frontend/src/utils/constants.js`
  — và phải khớp [`DATABASE_SCHEMA.md`](../technical/DATABASE_SCHEMA.md).
- Mọi thứ chỉ chạy trên VPS nằm trong `deploy/`; mọi thứ chỉ chạy trên máy dev
  nằm trong `docker/` hoặc `Makefile`.

---

## 2. Backend — `backend/`

```
backend/
├── manage.py
├── pyproject.toml            Cấu hình ruff (lint + format)
├── pytest.ini                Cấu hình pytest-django
├── requirements.txt          Dependency production
├── requirements-dev.txt      Dependency dev/test (include requirements.txt)
├── .python-version           Phiên bản Python (3.13)
├── .env.example              Mẫu biến môi trường — file duy nhất về env được commit
│
├── config/                   Cấu hình Django project
│   ├── settings/
│   │   ├── base.py           Dùng chung: apps, DRF, JWT, database, throttling
│   │   ├── development.py    DEBUG, email console, CORS localhost, LocMemCache
│   │   └── production.py     HTTPS, FileBasedCache, logging
│   ├── urls.py               Gắn /api/*, /share/*, /sitemap.xml, Django Admin
│   ├── wsgi.py               Điểm vào Gunicorn
│   └── asgi.py
│
├── apps/                     Mỗi app một nghiệp vụ — xem §2.1
│   ├── common/
│   ├── accounts/
│   ├── events/
│   ├── news/
│   ├── comments/
│   ├── donations/
│   ├── volunteers/
│   └── sitesettings/
│
├── services/                 Tích hợp dịch vụ bên ngoài
│   ├── cloudinary_service.py Upload / xoá ảnh
│   └── email_service.py      Gửi email qua Resend/SendGrid; console khi dev
│
├── mongo_migrations/         (cần tạo) Migration của admin/auth/contenttypes cho MongoDB
└── scripts/
    └── seed_data.py          Nạp dữ liệu mẫu cho dev — KHÔNG chạy trên production
```

### 2.1. Các app

| App | Collection | Nhánh feature | Ghi chú |
|-----|-----------|---------------|---------|
| `common` | `activity_logs` | `feature/auth` (phần nền) | Làm **đầu tiên** — mọi app khác phụ thuộc |
| `accounts` | `users` | `feature/auth` | Custom user, JWT, vai trò & quyền |
| `events` | `events` | `feature/events` | Có management command hẹn giờ công bố |
| `news` | `articles`, `article_likes` | `feature/news` | Kiểm duyệt bài cộng đồng |
| `comments` | `comments` | `feature/comments` | Phụ thuộc `events`, `news` |
| `donations` | `donations` | `feature/donations` | Thống kê, xuất CSV |
| `volunteers` | `volunteers` | `feature/volunteers` | Có `services.py` gửi email |
| `sitesettings` | `site_settings`, `contact_messages` | `feature/public-site` | Singleton cấu hình + form liên hệ |

### 2.2. Bên trong một app

Ví dụ `apps/events/` — mọi app theo cùng khuôn:

| File | Chứa gì | Không được chứa |
|------|---------|-----------------|
| `models.py` | Model, choices, `Meta.db_table`, `Meta.indexes` | Logic gửi email, gọi API ngoài |
| `serializers.py` | Validate input; **tách** `…PublicSerializer` và `…AdminSerializer` | Truy vấn DB phức tạp |
| `views.py` | ViewSet/APIView: kiểm quyền, chọn serializer, gọi service | Nghiệp vụ nhiều bước |
| `services.py` | Nghiệp vụ có tác dụng phụ: đổi trạng thái, ghi `activity_logs`, gửi email | Xử lý HTTP request/response |
| `urls.py` | Router DRF của app | — |
| `filters.py` | Bộ lọc query string (`django-filter`) | — |
| `permissions.py` | Permission class riêng của app (nếu có) | — |
| `admin.py` | Đăng ký Django Admin (công cụ khẩn cấp) | — |
| `tasks.py` | Hàm tác vụ nền, được management command gọi | Vòng lặp chạy vô hạn |
| `management/commands/` | Điểm vào cho systemd timer | — |
| `tests/` | `test_api.py`, `test_services.py`… | — |
| `migrations/` | Migration do `makemigrations` sinh | Sửa tay migration đã merge |

### 2.3. `apps/common/`

| File | Vai trò |
|------|---------|
| `models.py` | `TimeStampedModel`, embedded `Image`, `Video`, `Location`; model `ActivityLog` |
| `renderers.py` | Bọc response thành `{ success, data, message, meta }` |
| `responses.py` | Helper tạo response thành công/lỗi thống nhất |
| `exceptions.py` | Custom exception handler → `{ success: false, error: { code, message, details } }` |
| `pagination.py` | Phân trang chuẩn (`page`, `page_size`, `meta`) |
| `permissions.py` | `HasPermission("events.publish")` — kiểm mã quyền |
| `views.py` *(cần tạo)* | `/api/health`, `/share/*`, `/sitemap.xml` |
| `management/commands/` *(cần tạo)* | `cleanup_data`, `recalculate_counters` |

> Management command là **điểm vào**, không bị app khác import, nên được phép
> import nhiều app cùng lúc. Quy tắc phụ thuộc một chiều
> ([ARCHITECTURE §4.2](../technical/ARCHITECTURE.md#42-các-app-và-trách-nhiệm))
> áp dụng cho `models.py` và `services.py`.

---

## 3. Frontend — `frontend/`

```
frontend/
├── index.html                Khung HTML, thẻ meta mặc định
├── package.json
├── vite.config.js            Alias @/ → src/, cấu hình dev server
├── tailwind.config.js        Màu thương hiệu, font
├── postcss.config.js
├── eslint.config.mjs
├── jsconfig.json             Alias @/ cho editor
├── vitest.config.js          Cấu hình test
├── vitest.setup.js           Setup Testing Library
├── .env.example              VITE_API_URL, VITE_APP_NAME
│
├── public/                   Copy nguyên vào dist/
│   ├── favicon.ico
│   ├── robots.txt            (cần tạo)
│   └── images/               Ảnh tĩnh của giao diện (không phải ảnh nội dung)
│
└── src/
    ├── main.jsx              Mount app, bọc Provider
    ├── App.jsx
    ├── routes/
    │   ├── index.jsx         Khai báo toàn bộ route
    │   └── ProtectedRoute.jsx
    ├── pages/                Một file = một route (§3.1)
    │   └── admin/
    ├── components/
    │   ├── common/           Button, Modal, Pagination, Loading
    │   ├── layout/           Header, Footer, PublicLayout, AdminLayout, AdminSidebar
    │   ├── admin/            DataTable, StatCard, WysiwygEditor
    │   ├── home/             HeroBanner, ImpactStats, CtaSection
    │   ├── events/           EventCard, EventDetailView, EventFilter
    │   ├── news/             ArticleCard, ArticleEditorPreview
    │   ├── donation/         DonationForm, DonationList
    │   └── volunteer/        VolunteerForm
    ├── context/              AuthContext, ToastContext
    ├── hooks/                useAuth, useFetch, useDebounce
    ├── services/             api.js + một file cho mỗi resource
    ├── utils/                constants, formatDate, validators
    ├── styles/index.css      Entry Tailwind
    └── __tests__/
```

### 3.1. Page ↔ route ↔ service

| Page | Route | Service |
|------|-------|---------|
| `Home.jsx` | `/` | `settingsService`, `eventService`, `newsService` |
| `Events.jsx` | `/events` | `eventService` |
| `EventDetail.jsx` | `/events/:slug` | `eventService`, `commentService` |
| `News.jsx` | `/news` | `newsService` |
| `NewsDetail.jsx` | `/news/:slug` | `newsService`, `commentService` |
| `NewsSubmit.jsx` *(cần tạo)* | `/news/submit` | `newsService` |
| `Donation.jsx` | `/donate` | `donationService`, `settingsService` |
| `Volunteer.jsx` | `/volunteer` | `volunteerService` |
| `Contact.jsx` | `/contact` | `settingsService` |
| `StaticPage.jsx` *(cần tạo)* | `/about`, `/privacy`, `/terms`, `/transparency` | `settingsService` |
| `NotFound.jsx` | `*` | — |
| `admin/Login.jsx` | `/admin/login` | `authService` |
| `admin/Dashboard.jsx` | `/admin` | `api.js` (dashboard) |
| `admin/EventsList.jsx`, `admin/EventForm.jsx` | `/admin/events…` | `eventService` |
| `admin/NewsList.jsx`, `admin/NewsForm.jsx` | `/admin/news…` | `newsService` |
| `admin/NewsReview.jsx` | `/admin/news/review` | `newsService` |
| `admin/Comments.jsx` | `/admin/comments` | `commentService` |
| `admin/Donations.jsx` | `/admin/donations` | `donationService` |
| `admin/Volunteers.jsx` | `/admin/volunteers` | `volunteerService` |
| `admin/ContactMessages.jsx` *(cần tạo)* | `/admin/messages` | `settingsService` |
| `admin/Users.jsx` | `/admin/users` | `userService` |
| `admin/Settings.jsx` | `/admin/settings` | `settingsService` |
| `admin/Profile.jsx` *(cần tạo)* | `/admin/profile` | `authService` |

Quyền cần cho từng route admin: [ARCHITECTURE §5.1](../technical/ARCHITECTURE.md#51-bản-đồ-route).

---

## 4. Hạ tầng & tự động hoá

```
deploy/
├── gunicorn.conf.py                    Cấu hình Gunicorn (socket, worker, thread)
├── nginx/
│   └── nu-cuoi-em.conf                 Virtual host production
├── systemd/
│   ├── nu-cuoi-em-api.service          Chạy Gunicorn
│   ├── nu-cuoi-em-scheduler.service    Công bố nội dung hẹn giờ
│   ├── nu-cuoi-em-scheduler.timer      ...mỗi 5 phút
│   ├── nu-cuoi-em-maintenance.service  (cần tạo) Dọn dữ liệu, tính lại bộ đếm
│   ├── nu-cuoi-em-maintenance.timer    (cần tạo) ...03:00 hằng ngày
│   ├── nu-cuoi-em-backup.service       (cần tạo) mongodump + đẩy Google Drive
│   └── nu-cuoi-em-backup.timer         (cần tạo) ...02:30 hằng ngày
└── scripts/
    ├── deploy.sh                       (cần tạo) Chạy trên VPS mỗi lần deploy
    └── backup.sh                       (cần tạo) Được backup.service gọi

docker/
└── docker-compose.yml                  MongoDB 8 replica set 1 node cho dev

.github/workflows/
├── ci.yml                              Lint + test + build, chạy mỗi PR
└── deploy.yml                          Build frontend, đóng gói, deploy khi merge main
```

Nội dung chuẩn của mọi file trên:
[`DEPLOYMENT.md`](../technical/DEPLOYMENT.md) (§6, §7, §10, §12) và
[`SETUP.md` §11](../technical/SETUP.md#11-phụ-lục--nội-dung-tối-thiểu-cho-file-cấu-hình).

---

## 5. File cấu hình ở thư mục gốc

| File | Vai trò |
|------|---------|
| `.gitignore` | Chặn `.env`, secret, `node_modules`, `.venv`, build, cache |
| `.gitattributes` | Chuẩn hoá xuống dòng LF; `*.conf`, `*.service`, `*.timer`, `Makefile` luôn LF vì chạy trên Linux |
| `.editorconfig` | UTF-8, LF, thụt lề 2 (JS/JSON/YAML) và 4 (Python) |
| `.prettierrc` | Format JS/JSX/CSS |
| `.nvmrc` | Phiên bản Node.js cho dev và CI |
| `package.json` | npm workspaces để chạy frontend từ gốc (tuỳ chọn) |
| `Makefile` | `make dev`, `make test`, `make lint`, `make mongo-up`… |
| `CONTRIBUTING.md` | Mô hình nhánh, Conventional Commits, checklist bảo mật |
| `LICENSE` | MIT |

---

## 6. Quy ước đặt tên

| Loại | Quy ước | Ví dụ |
|------|---------|-------|
| App Django | số nhiều, chữ thường | `events`, `volunteers` |
| Model | PascalCase, số ít | `Event`, `VolunteerApplication` |
| Collection | `snake_case` số nhiều qua `Meta.db_table` | `events`, `contact_messages` |
| Serializer | `<Model><Mục đích>Serializer` | `EventPublicSerializer`, `EventAdminSerializer` |
| Management command | động từ + đối tượng, `snake_case` | `publish_scheduled_events` |
| Test backend | `tests/test_<phạm vi>.py` | `tests/test_api.py`, `tests/test_services.py` |
| React component / page | PascalCase `.jsx` | `EventCard.jsx`, `NewsReview.jsx` |
| Hook | `use` + PascalCase `.js` | `useFetch.js` |
| Service frontend | `<resource>Service.js` | `eventService.js` |
| Test frontend | `<Tên>.test.jsx` cạnh file hoặc trong `__tests__/` | `EventCard.test.jsx` |
| Route URL | tiếng Anh, `kebab-case` | `/admin/news/review` |
| Slug nội dung | tiếng Việt không dấu, `kebab-case` | `ngay-hoi-ve-tranh-acrylic` |
| Nhãn hiển thị | tiếng Việt có dấu, chỉ trong `constants.js` | `"Vẽ tranh acrylic"` |

---

## 7. Thêm một tính năng mới — đặt code ở đâu

Ví dụ: thêm "Nhà tài trợ" (sponsors) hiển thị trên trang chủ.

1. **Tài liệu trước** — thêm collection vào `DATABASE_SCHEMA.md`, endpoint vào
   `API.md`, trong cùng PR với code.
2. **Backend** — `python manage.py startapp sponsors apps/sponsors`, thêm vào
   `INSTALLED_APPS`, viết `models.py` (có `db_table`), `serializers.py`,
   `views.py`, `urls.py`, gắn vào `config/urls.py`. Thêm mã quyền
   `sponsors.*` vào `apps/accounts/permissions.py`.
3. **Test backend** — `apps/sponsors/tests/test_api.py`, bao gồm test response
   công khai **không** chứa trường nội bộ.
4. **Frontend** — `services/sponsorService.js`, component trong
   `components/home/`, route admin trong `routes/index.jsx`, mục menu trong
   `AdminSidebar.jsx`, enum/nhãn trong `utils/constants.js`.
5. **Nhánh** — `feature/sponsors` cắt từ `dev`, PR về `dev`.

---

## 8. Việc cần làm trước khi bắt đầu code

Hiện trạng: repo có **đủ khung thư mục** nhưng **toàn bộ file code và cấu hình
đều rỗng (0 byte)**, trừ `.gitignore`, `.gitattributes`, hai file `.env.example`
và tài liệu. Danh sách dưới đây là những gì phải có để giai đoạn code chạy trơn tru.

### 8.1. Nền móng — làm trên nhánh `setup`, merge vào `dev` trước mọi feature

Không có những file này thì không ai chạy được project:

- [ ] `.nvmrc`, `backend/.python-version` — [SETUP §11.1–11.2](../technical/SETUP.md#111-nvmrc)
- [ ] `backend/requirements.txt`, `backend/requirements-dev.txt` — [SETUP §11.3–11.4](../technical/SETUP.md#113-backendrequirementstxt)
- [ ] `backend/pyproject.toml` (cấu hình ruff), `backend/pytest.ini`
- [ ] `backend/manage.py`, `backend/config/` (settings 3 file, `urls.py`, `wsgi.py`, `asgi.py`) — [SETUP §4.5](../technical/SETUP.md#45-cấu-hình-django-mongodb-backend)
- [ ] `backend/mongo_migrations/` cho `admin`, `auth`, `contenttypes`
- [ ] `apps.py` của 8 app (không có file này Django không nạp được app)
- [ ] `docker/docker-compose.yml` — [SETUP §11.5](../technical/SETUP.md#115-dockerdocker-composeyml)
- [ ] `frontend/package.json`, `vite.config.js`, `tailwind.config.js`, `postcss.config.js`, `eslint.config.mjs`, `jsconfig.json`, `vitest.config.js`, `vitest.setup.js`, `index.html`
- [ ] `.editorconfig`, `.prettierrc`, `Makefile`, `package.json` gốc
- [ ] `frontend/public/favicon.ico` — hiện là file 0 byte
- [ ] `.github/workflows/ci.yml` — [DEPLOYMENT §10.1](../technical/DEPLOYMENT.md#101-githubworkflowsciyml). Có CI sớm để mọi PR feature đều được kiểm tra
- [ ] Bổ sung `DJANGO_ADMIN_URL` và `SENTRY_DSN` vào `backend/.env.example`

### 8.2. Thứ tự làm các feature

Theo phụ thuộc giữa các app:

```
setup ──► feature/auth (common + accounts) ──┬──► feature/events ──┬──► feature/comments
                                             ├──► feature/news ────┘
                                             ├──► feature/donations
                                             ├──► feature/volunteers
                                             └──► feature/public-site (sitesettings)
                                                          │
                                  feature/admin-dashboard ◄┘ (sau khi các API đã có)
```

`feature/auth` phải xong trước vì nó chứa `apps.common` (renderer, pagination,
exception handler, base model) mà mọi app khác dùng.

### 8.3. File chưa có trong khung — cần tạo thêm

| File | Thuộc feature | Đặc tả |
|------|---------------|--------|
| `backend/apps/common/views.py` | `feature/auth` | Health check, share OG, sitemap — [API §13](../technical/API.md#13-health-check) |
| `backend/apps/common/management/commands/cleanup_data.py` | `feature/public-site` | [ARCHITECTURE §9](../technical/ARCHITECTURE.md#9-tác-vụ-nền) |
| `backend/apps/common/management/commands/recalculate_counters.py` | `feature/public-site` | [Schema §12](../technical/DATABASE_SCHEMA.md#12-dữ-liệu-denormalized--cách-đồng-bộ) |
| `backend/apps/*/services.py` (trừ `volunteers` đã có) | từng feature | [ARCHITECTURE §4.1](../technical/ARCHITECTURE.md#41-phân-lớp) |
| `frontend/src/pages/NewsSubmit.jsx` | `feature/news` | Khách gửi bài |
| `frontend/src/pages/StaticPage.jsx` | `feature/public-site` | Giới thiệu, chính sách, điều khoản, minh bạch |
| `frontend/src/pages/admin/ContactMessages.jsx` | `feature/admin-dashboard` | Hộp thư liên hệ |
| `frontend/src/pages/admin/Profile.jsx` | `feature/auth` | Đổi mật khẩu, bắt buộc khi `must_change_password` |
| `frontend/public/robots.txt` | `feature/public-site` | Chặn `/admin`, `/django-admin` |
| `deploy/scripts/deploy.sh`, `deploy/scripts/backup.sh` | trước lần deploy đầu | [DEPLOYMENT §10.3, §12.1](../technical/DEPLOYMENT.md#103-deployscriptsdeploysh-cần-tạo) |
| `deploy/systemd/nu-cuoi-em-{maintenance,backup}.{service,timer}` | trước lần deploy đầu | [DEPLOYMENT §6.4](../technical/DEPLOYMENT.md#64-timer-bảo-trì-và-backup-cần-tạo-thêm-file) |

### 8.4. Quyết định còn bỏ ngỏ

Các điểm này chưa chặn việc bắt đầu code, nhưng cần chốt trước khi làm tới phần liên quan:

| Quyết định | Phương án | Cần chốt trước |
|-----------|-----------|----------------|
| Domain | `.com` (đã tính trong ngân sách) hay `.org` (đang dùng trong `.env.example`) | Cấu hình email & deploy |
| Nhà cung cấp email | Resend (khuyến nghị) hay SendGrid | `feature/volunteers` |
| Thư viện WYSIWYG | Tiptap hay Quill — [ARCHITECTURE §5.5](../technical/ARCHITECTURE.md#55-trình-soạn-thảo-wysiwyg) | `feature/news` |
| Nhà cung cấp VPS | Theo tiêu chí trong [chi phí vận hành](Phan-tich-chi-phi-van-hanh.md) | Lần deploy đầu |
| Giữ `activity_logs`? | Giữ (minh bạch tài chính) hay bỏ cho gọn | `feature/auth` |
| Nguồn trả chi phí vận hành | Quỹ riêng của nhóm hay trích từ quyên góp (nếu trích phải công khai) | Trước khi công bố website |
