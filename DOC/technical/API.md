# API Reference — Nụ Cười Em

Hợp đồng REST API giữa `frontend/` (React + Vite) và `backend/` (Django REST Framework).

| Hạng mục | Giá trị |
|----------|---------|
| Base URL (dev) | `http://localhost:8000/api` |
| Base URL (prod) | `https://nucuoiem.org/api` |
| Biến frontend | `VITE_API_URL` trong `frontend/.env` |
| Định dạng | `application/json; charset=utf-8` (trừ upload dùng `multipart/form-data`) |
| Xác thực | JWT Bearer — `djangorestframework-simplejwt` |
| Múi giờ | Mọi mốc thời gian là **ISO 8601 UTC** (`2026-03-15T01:00:00Z`) |

> **Trạng thái tài liệu:** đây là hợp đồng chuẩn (source of truth) cho
> `backend/apps/*/urls.py`, `views.py`, `serializers.py` và
> `frontend/src/services/*.js`. Thay đổi endpoint phải cập nhật file này trong
> cùng một Pull Request.
>
> Tên trường trong request/response bám sát [`DATABASE_SCHEMA.md`](DATABASE_SCHEMA.md).

---

## Mục lục

1. [Quy ước chung](#1-quy-ước-chung)
2. [Xác thực & phân quyền](#2-xác-thực--phân-quyền)
3. [Người dùng quản trị](#3-người-dùng-quản-trị--apiusers)
4. [Sự kiện](#4-sự-kiện--apievents)
5. [Bài viết](#5-bài-viết--apiarticles)
6. [Bình luận](#6-bình-luận--apicomments)
7. [Quyên góp](#7-quyên-góp--apidonations)
8. [Tình nguyện viên](#8-tình-nguyện-viên--apivolunteers)
9. [Liên hệ](#9-liên-hệ--apicontact)
10. [Cài đặt website](#10-cài-đặt-website--apisettings)
11. [Dashboard](#11-dashboard--apidashboard)
12. [Upload media](#12-upload-media--apiuploads)
13. [Health check](#13-health-check)
14. [Mã lỗi](#14-mã-lỗi)
15. [Giới hạn tần suất](#15-giới-hạn-tần-suất-throttling)
16. [Bảng tra endpoint](#16-bảng-tra-endpoint)

---

## 1. Quy ước chung

### 1.1. Cấu trúc response

Mọi response đi qua `apps/common/renderers.py` và có cùng một lớp vỏ:

**Thành công**

```json
{
  "success": true,
  "data": { },
  "message": ""
}
```

**Thành công, có phân trang**

```json
{
  "success": true,
  "data": [ ],
  "meta": {
    "page": 1,
    "page_size": 12,
    "total": 57,
    "total_pages": 5,
    "has_next": true,
    "has_previous": false
  }
}
```

**Lỗi**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Dữ liệu không hợp lệ.",
    "details": {
      "email": ["Email không đúng định dạng."],
      "amount": ["Số tiền phải lớn hơn 0."]
    }
  }
}
```

`details` chỉ xuất hiện với lỗi validation; các lỗi khác trả `details: null`.

### 1.2. HTTP status code

| Code | Khi nào dùng |
|------|--------------|
| `200 OK` | GET / PATCH / POST hành động thành công |
| `201 Created` | Tạo mới thành công (có `Location` header) |
| `204 No Content` | DELETE thành công |
| `400 Bad Request` | Dữ liệu sai định dạng hoặc vi phạm nghiệp vụ |
| `401 Unauthorized` | Thiếu token / token hết hạn hoặc sai |
| `403 Forbidden` | Đã đăng nhập nhưng không đủ quyền |
| `404 Not Found` | Không tìm thấy tài nguyên (hoặc chưa được công bố) |
| `409 Conflict` | Trùng dữ liệu duy nhất (`slug`, `email`, lượt thích trùng) |
| `413 Payload Too Large` | File upload vượt giới hạn |
| `429 Too Many Requests` | Vượt giới hạn tần suất, kèm header `Retry-After` |
| `500 Internal Server Error` | Lỗi không lường trước |

### 1.3. Phân trang, sắp xếp, tìm kiếm

Áp dụng cho mọi endpoint dạng danh sách (`apps/common/pagination.py`):

| Tham số | Kiểu | Mặc định | Ghi chú |
|---------|------|----------|---------|
| `page` | int | `1` | Trang hiện tại |
| `page_size` | int | `12` (admin: `20`) | Tối đa `100` |
| `ordering` | string | tuỳ resource | Tiền tố `-` để giảm dần, VD `-created_at` |
| `q` | string | — | Tìm kiếm toàn văn (dùng text index) |

### 1.4. Phần công khai vs phần quản trị

API dùng **chung một đường dẫn** cho cả hai, phân biệt bằng token:

- **Không có token** — chỉ đọc được nội dung đã công bố (`status = "published"`
  với sự kiện/bài viết, `status = "approved"` với bình luận). Các trường nhạy cảm
  (email, số điện thoại, ghi chú nội bộ) bị loại khỏi serializer.
- **Có token + đủ quyền** — thấy toàn bộ trạng thái, lọc được bằng `?status=`,
  và nhận thêm các trường quản trị.

Cột **Quyền** trong tài liệu dùng mã quyền định nghĩa ở
[`DATABASE_SCHEMA.md` §2.1](DATABASE_SCHEMA.md#21-vai-trò-role-và-quyền).
`—` nghĩa là công khai, không cần đăng nhập.

### 1.5. Ghi log

Các hành động `approve`, `reject`, `publish`, `confirm`, `delete` và thay đổi
cài đặt đều ghi một bản ghi vào `activity_logs`
([schema §11](DATABASE_SCHEMA.md#11-activity_logs--nhật-ký-thao-tác-quản-trị)).

---

## 2. Xác thực & phân quyền

### 2.1. Cơ chế

- Đăng nhập trả về cặp **access token** (mặc định 30 phút,
  `JWT_ACCESS_TOKEN_LIFETIME_MINUTES`) và **refresh token** (7 ngày,
  `JWT_REFRESH_TOKEN_LIFETIME_DAYS`).
- Mọi request cần xác thực gửi header:

  ```
  Authorization: Bearer <access_token>
  ```

- Đăng xuất đưa refresh token vào blacklist (`token_blacklist_blacklistedtoken`),
  token đó không dùng lại được.
- Frontend lưu token trong `AuthContext`; interceptor ở
  `frontend/src/services/api.js` tự gọi `/auth/refresh` khi gặp `401` rồi thử lại
  request một lần.

### 2.2. `POST /api/auth/login`

Quyền: `—`

**Request**

```json
{
  "email": "admin@nucuoiem.org",
  "password": "matkhau-cua-ban"
}
```

**Response `200`**

```json
{
  "success": true,
  "data": {
    "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "66f0a1b2c3d4e5f601000001",
      "email": "admin@nucuoiem.org",
      "full_name": "Ngô Minh Hiếu",
      "role": "superadmin",
      "avatar": null,
      "permissions": ["events.view", "events.create", "..."],
      "must_change_password": false
    }
  }
}
```

**Lỗi**

| Code | HTTP | Tình huống |
|------|------|-----------|
| `INVALID_CREDENTIALS` | `401` | Sai email hoặc mật khẩu |
| `ACCOUNT_DISABLED` | `403` | `is_active = false` |
| `THROTTLED` | `429` | Quá 5 lần đăng nhập sai / 5 phút / IP |

> Thông báo lỗi **không** phân biệt "email không tồn tại" và "sai mật khẩu",
> tránh dò tài khoản.

### 2.3. `POST /api/auth/refresh`

Quyền: `—`

```json
{ "refresh": "eyJhbGciOi..." }
```

Response `200` trả `{ "access": "...", "refresh": "..." }` (bật rotation, refresh
token cũ bị blacklist). Refresh token sai hoặc đã thu hồi → `401 TOKEN_INVALID`.

### 2.4. `POST /api/auth/logout`

Quyền: đã đăng nhập. Body `{ "refresh": "..." }` → `204 No Content`.

### 2.5. `GET /api/auth/me`

Quyền: đã đăng nhập. Trả về đối tượng `user` như ở §2.2, kèm `permissions` là
**quyền hiệu lực** (quyền mặc định của `role` ∪ `extra_permissions`).

### 2.6. `PATCH /api/auth/me`

Quyền: đã đăng nhập. Sửa được `full_name`, `phone`, `avatar`.
Không sửa được `email`, `role`, `extra_permissions`, `is_active`.

### 2.7. `POST /api/auth/change-password`

Quyền: đã đăng nhập.

```json
{
  "current_password": "...",
  "new_password": "...",
  "new_password_confirm": "..."
}
```

Thành công → `204`, đồng thời **blacklist toàn bộ refresh token** của tài khoản
và đặt `must_change_password = false`. Frontend phải đăng nhập lại.

Lỗi: `INVALID_CREDENTIALS` (`400`) nếu sai mật khẩu hiện tại;
`VALIDATION_ERROR` nếu mật khẩu mới không đạt (tối thiểu 8 ký tự, qua
`AUTH_PASSWORD_VALIDATORS` của Django).

---

## 3. Người dùng quản trị — `/api/users`

Quản lý tài khoản vận hành website. Không dùng cho khách truy cập.

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `GET` | `/api/users` | `users.manage` | Danh sách admin |
| `POST` | `/api/users` | `users.manage` | Tạo tài khoản |
| `GET` | `/api/users/{id}` | `users.manage` | Chi tiết |
| `PATCH` | `/api/users/{id}` | `users.manage` | Sửa thông tin / vai trò |
| `DELETE` | `/api/users/{id}` | `users.manage` | Vô hiệu hoá (đặt `is_active = false`) |
| `POST` | `/api/users/{id}/reset-password` | `users.manage` | Đặt lại mật khẩu, gửi email |

### 3.1. `GET /api/users`

Query: `role`, `is_active`, `q` (tìm theo `full_name` / `email`),
`ordering` (mặc định `-date_joined`).

```json
{
  "success": true,
  "data": [
    {
      "id": "66f0a1b2c3d4e5f601000001",
      "email": "admin@nucuoiem.org",
      "full_name": "Ngô Minh Hiếu",
      "phone": "0905123456",
      "avatar": null,
      "role": "superadmin",
      "extra_permissions": [],
      "is_active": true,
      "last_login": "2026-09-08T02:11:04Z",
      "date_joined": "2026-01-05T09:00:00Z"
    }
  ],
  "meta": { "page": 1, "page_size": 20, "total": 6, "total_pages": 1, "has_next": false, "has_previous": false }
}
```

### 3.2. `POST /api/users`

```json
{
  "email": "bientap@nucuoiem.org",
  "full_name": "Trần Thị Hoa",
  "phone": "0912000111",
  "role": "news_editor",
  "extra_permissions": ["comments.moderate"]
}
```

Mật khẩu **không** truyền từ client: hệ thống sinh mật khẩu ngẫu nhiên, gửi qua
email và đặt `must_change_password = true`. Response `201` trả về user (không
bao giờ chứa `password`).

Lỗi: `409 DUPLICATE_EMAIL` nếu email đã tồn tại.

### 3.3. `DELETE /api/users/{id}`

**Không xoá cứng** — chỉ đặt `is_active = false` và blacklist token của tài khoản.
Trả `204`. Không cho phép tự vô hiệu hoá chính mình (`400 SELF_ACTION_FORBIDDEN`)
và không cho vô hiệu hoá `superadmin` cuối cùng (`400 LAST_SUPERADMIN`).

---

## 4. Sự kiện — `/api/events`

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `GET` | `/api/events` | `—` | Danh sách sự kiện |
| `GET` | `/api/events/{slug}` | `—` | Chi tiết theo slug |
| `POST` | `/api/events/{id}/view` | `—` | Tăng lượt xem |
| `GET` | `/api/events/filters` | `—` | Giá trị có sẵn cho bộ lọc |
| `POST` | `/api/events` | `events.create` | Tạo sự kiện |
| `PATCH` | `/api/events/{id}` | `events.update` | Cập nhật |
| `DELETE` | `/api/events/{id}` | `events.delete` | Xoá |
| `POST` | `/api/events/{id}/publish` | `events.publish` | Công bố ngay / đặt lịch |
| `POST` | `/api/events/{id}/archive` | `events.publish` | Lưu trữ |

### 4.1. `GET /api/events`

**Query params** (khớp `apps/events/filters.py`):

| Tham số | Kiểu | Mô tả |
|---------|------|-------|
| `q` | string | Tìm trong `title`, `summary` |
| `event_type` | enum | `ve_tranh` \| `nan_dat_set` \| `van_nghe` \| `quyen_gop` \| `khac` |
| `school` | string | Lọc theo `school_name` |
| `from` | date | `start_at >= from` (`YYYY-MM-DD`) |
| `to` | date | `start_at <= to` |
| `featured` | bool | Chỉ sự kiện `is_featured` |
| `status` | enum | **Chỉ admin.** `draft` \| `scheduled` \| `published` \| `archived` |
| `ordering` | string | `-start_at` (mặc định), `start_at`, `-view_count` |

Khách vô danh luôn chỉ nhận `status = "published"`, bỏ qua tham số `status`.

**Response `200`** — mỗi phần tử là bản rút gọn:

```json
{
  "success": true,
  "data": [
    {
      "id": "66f0a1b2c3d4e5f602000001",
      "title": "Ngày hội vẽ tranh acrylic cùng lớp Gia Đình",
      "slug": "ngay-hoi-ve-tranh-acrylic-cung-lop-gia-dinh",
      "summary": "Buổi vẽ tranh acrylic dành cho 25 bé tại lớp trẻ Gia Đình.",
      "event_type": "ve_tranh",
      "school_name": "Lớp trẻ Gia Đình",
      "start_at": "2026-03-15T01:00:00Z",
      "end_at": "2026-03-15T04:30:00Z",
      "location": { "venue": "Lớp trẻ Gia Đình", "district": "TP. Buôn Ma Thuột", "province": "Đắk Lắk" },
      "cover_image": {
        "url": "https://res.cloudinary.com/nce/image/upload/v1/events/ve-tranh.jpg",
        "alt": "Các bé vẽ tranh acrylic",
        "width": 1600,
        "height": 900
      },
      "stats": { "children_helped": 25, "volunteers_joined": 12, "funds_raised": "3500000" },
      "is_featured": true,
      "view_count": 412,
      "comment_count": 7,
      "published_at": "2026-03-16T02:00:00Z"
    }
  ],
  "meta": { "page": 1, "page_size": 12, "total": 34, "total_pages": 3, "has_next": true, "has_previous": false }
}
```

### 4.2. `GET /api/events/{slug}`

Trả thêm `description` (HTML đã sanitize), `gallery[]`, `videos[]`,
`allow_comments`. Với admin, trả thêm `status`, `publish_at`, `created_by`,
`updated_by`.

Sự kiện chưa công bố → `404 NOT_FOUND` với khách vô danh (không tiết lộ sự tồn tại).

### 4.3. `POST /api/events/{id}/view`

Tăng `view_count` bằng `$inc`. Chống spam bằng khoá `visitor_key` trong cache
30 phút. Trả `200` với `{ "view_count": 413 }`. Không tính là lỗi nếu bị bỏ qua.

### 4.4. `GET /api/events/filters`

Trả dữ liệu để dựng `EventFilter.jsx` mà không cần hardcode:

```json
{
  "success": true,
  "data": {
    "event_types": [
      { "value": "ve_tranh", "label": "Vẽ tranh acrylic", "count": 12 },
      { "value": "van_nghe", "label": "Văn nghệ giao lưu", "count": 5 }
    ],
    "schools": ["Lớp trẻ Gia Đình", "Trường Tiểu học Nguyễn Du"],
    "date_range": { "min": "2025-06-01T00:00:00Z", "max": "2026-09-01T00:00:00Z" }
  }
}
```

### 4.5. `POST /api/events`

Quyền `events.create`.

```json
{
  "title": "Ngày hội vẽ tranh acrylic cùng lớp Gia Đình",
  "summary": "Buổi vẽ tranh acrylic dành cho 25 bé tại lớp trẻ Gia Đình.",
  "description": "<p>Sáng ngày 15/03...</p>",
  "event_type": "ve_tranh",
  "school_name": "Lớp trẻ Gia Đình",
  "location": {
    "venue": "Lớp trẻ Gia Đình",
    "address": "123 Lê Duẩn",
    "ward": "Tân Lập",
    "district": "TP. Buôn Ma Thuột",
    "province": "Đắk Lắk",
    "map_url": "https://maps.app.goo.gl/xxxx"
  },
  "start_at": "2026-03-15T01:00:00Z",
  "end_at": "2026-03-15T04:30:00Z",
  "cover_image": { "url": "...", "public_id": "events/ve-tranh", "alt": "...", "width": 1600, "height": 900 },
  "gallery": [],
  "videos": [{ "url": "https://youtu.be/xxxx", "provider": "youtube", "caption": "Tổng kết buổi vẽ" }],
  "stats": { "children_helped": 25, "volunteers_joined": 12, "funds_raised": "3500000" },
  "is_featured": true,
  "allow_comments": true
}
```

- `slug` tự sinh từ `title` (bỏ dấu). Truyền `slug` để ghi đè; trùng → `409 DUPLICATE_SLUG`.
- Sự kiện mới luôn ở `status = "draft"`; dùng `/publish` để công bố.
- `created_by` lấy từ token, client không được truyền.
- `description` được sanitize (bleach/nh3) trước khi lưu — thẻ `<script>`, thuộc
  tính `on*` bị loại bỏ.

### 4.6. `POST /api/events/{id}/publish`

```json
{ "publish_at": "2026-03-20T01:00:00Z" }
```

- Có `publish_at` ở tương lai → `status = "scheduled"`, job
  `publish_scheduled_events` sẽ công bố đúng giờ.
- Body rỗng hoặc `publish_at = null` → công bố ngay, đặt
  `status = "published"` và `published_at = now`.

Lỗi `400 INVALID_STATE` nếu sự kiện đang ở `archived`.

---

## 5. Bài viết — `/api/articles`

Gồm bài do admin viết và **bài do khách gửi** (`source = "community"`, luôn phải
qua kiểm duyệt).

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `GET` | `/api/articles` | `—` | Danh sách bài viết |
| `GET` | `/api/articles/{slug}` | `—` | Chi tiết theo slug |
| `POST` | `/api/articles/submit` | `—` | Khách gửi bài (vào hàng chờ duyệt) |
| `POST` | `/api/articles/{id}/view` | `—` | Tăng lượt xem |
| `POST` | `/api/articles/{id}/like` | `—` | Thích bài viết |
| `DELETE` | `/api/articles/{id}/like` | `—` | Bỏ thích |
| `GET` | `/api/articles/tags` | `—` | Danh sách nhãn kèm số lượng |
| `POST` | `/api/articles` | `articles.create` | Admin tạo bài |
| `PATCH` | `/api/articles/{id}` | `articles.update` | Sửa bài |
| `DELETE` | `/api/articles/{id}` | `articles.delete` | Xoá bài |
| `POST` | `/api/articles/{id}/approve` | `articles.review` | Duyệt bài |
| `POST` | `/api/articles/{id}/reject` | `articles.review` | Từ chối kèm ghi chú |
| `POST` | `/api/articles/{id}/publish` | `articles.review` | Công bố / đặt lịch |

### 5.1. `GET /api/articles`

| Tham số | Mô tả |
|---------|-------|
| `q` | Tìm trong `title`, `excerpt`, `content` |
| `tag` | Lọc theo một nhãn, VD `cau-chuyen` |
| `source` | **Chỉ admin.** `admin` \| `community` |
| `status` | **Chỉ admin.** `draft` \| `pending` \| `scheduled` \| `published` \| `rejected` \| `archived` |
| `event` | `related_event_id` — bài viết của một sự kiện |
| `featured` | Chỉ bài `is_featured` |
| `ordering` | `-published_at` (mặc định), `-view_count`, `-like_count` |

Response rút gọn (không kèm `content`):

```json
{
  "success": true,
  "data": [
    {
      "id": "66f0a1b2c3d4e5f603000001",
      "title": "Bức tranh đầu tiên của bé An",
      "slug": "buc-tranh-dau-tien-cua-be-an",
      "excerpt": "Sau ba buổi học vẽ, An đã hoàn thành bức tranh đầu tiên.",
      "cover_image": { "url": "...", "alt": "Bức tranh của bé An", "width": 1200, "height": 800 },
      "tags": ["cau-chuyen", "hoat-dong"],
      "author": { "display_name": "Nguyễn Thị Lan", "is_guest": true },
      "published_at": "2026-03-20T03:20:00Z",
      "view_count": 158,
      "like_count": 34,
      "comment_count": 3
    }
  ],
  "meta": { "page": 1, "page_size": 12, "total": 88, "total_pages": 8, "has_next": true, "has_previous": false }
}
```

> `author.email` **không bao giờ** xuất hiện trong response công khai
> ([schema §4.2](DATABASE_SCHEMA.md#42-embedded-authorinfo)). Admin nhận thêm
> `author.email`, `source`, `status`, `review`.

### 5.2. `GET /api/articles/{slug}`

Trả thêm `content` (HTML), `gallery[]`, `related_event`, `liked_by_me`
(dựa trên `visitor_key` gửi kèm header `X-Visitor-Key`).

Query `?include=related` trả thêm tối đa 3 bài cùng nhãn ở `data.related[]`.

### 5.3. `POST /api/articles/submit` — khách gửi bài

Quyền `—`. Không cần đăng nhập.

```json
{
  "title": "Bức tranh đầu tiên của bé An",
  "content": "<p>An là một cậu bé ít nói...</p>",
  "excerpt": "Sau ba buổi học vẽ, An đã hoàn thành bức tranh đầu tiên.",
  "tags": ["cau-chuyen"],
  "cover_image": { "url": "...", "public_id": "news/be-an", "alt": "Bức tranh của bé An" },
  "author": {
    "display_name": "Nguyễn Thị Lan",
    "email": "lan.nguyen@example.com"
  },
  "related_event_id": "66f0a1b2c3d4e5f602000001",
  "hp_website": ""
}
```

- Server **luôn ép** `source = "community"`, `status = "pending"`,
  `author.is_guest = true`, `author.user_id = null` — client không đổi được.
- `hp_website` là **honeypot**: có giá trị → trả `201` giả nhưng không lưu.
- `content` được sanitize; chỉ giữ `p, br, strong, em, u, h2, h3, ul, ol, li, blockquote, a, img, figure, figcaption`.
- Gửi email báo admin nếu `site_settings.email_settings.on_new_article = true`.

**Response `201`**

```json
{
  "success": true,
  "data": { "id": "66f0a1b2c3d4e5f603000009", "status": "pending" },
  "message": "Cảm ơn bạn! Bài viết đang chờ ban quản trị duyệt."
}
```

### 5.4. `POST /api/articles/{id}/approve`

Quyền `articles.review`.

```json
{
  "note": "Bài viết cảm động, đã chỉnh chính tả.",
  "edited_before_publish": true,
  "publish": true,
  "publish_at": null
}
```

Ghi `review` embedded (`decision = "approved"`, `reviewed_by_id`, `reviewed_at`).
`publish: true` → chuyển thẳng `published`; kèm `publish_at` tương lai →
`scheduled`; `publish: false` → giữ trạng thái chờ để admin sửa tiếp.

Gửi email báo kết quả cho `author.email` nếu bài là `community`.

### 5.5. `POST /api/articles/{id}/reject`

```json
{ "note": "Nội dung chưa phù hợp với tiêu chí của chương trình." }
```

`note` **bắt buộc** (`400 VALIDATION_ERROR` nếu rỗng) — nó được gửi kèm email cho
tác giả. Đặt `status = "rejected"`.

### 5.6. `POST /api/articles/{id}/like`

Quyền `—`. Header `X-Visitor-Key: <uuid từ localStorage>` là **bắt buộc**.

- `201` → thích thành công, trả `{ "like_count": 35, "liked": true }`.
- `409 ALREADY_LIKED` → cặp `(article_id, visitor_key)` đã tồn tại.

`DELETE /api/articles/{id}/like` bỏ thích, trả `{ "like_count": 34, "liked": false }`.

### 5.7. `GET /api/articles/tags`

```json
{
  "success": true,
  "data": [
    { "value": "cau-chuyen", "label": "Câu chuyện", "count": 42 },
    { "value": "hoat-dong", "label": "Hoạt động", "count": 31 },
    { "value": "hoc-tap", "label": "Học tập", "count": 12 }
  ]
}
```

---

## 6. Bình luận — `/api/comments`

Dùng chung cho sự kiện và bài viết theo mẫu polymorphic
(`target_type` + `target_id`). **Mọi bình luận mới đều ở `pending`.**

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `GET` | `/api/comments` | `—` | Bình luận đã duyệt của một đối tượng |
| `POST` | `/api/comments` | `—` | Gửi bình luận |
| `GET` | `/api/comments/pending` | `comments.moderate` | Hàng đợi kiểm duyệt |
| `POST` | `/api/comments/{id}/approve` | `comments.moderate` | Duyệt |
| `POST` | `/api/comments/{id}/reject` | `comments.moderate` | Từ chối |
| `POST` | `/api/comments/{id}/spam` | `comments.moderate` | Đánh dấu spam |
| `POST` | `/api/comments/{id}/reply` | `comments.moderate` | Trả lời với danh nghĩa BQT |
| `DELETE` | `/api/comments/{id}` | `comments.moderate` | Xoá vĩnh viễn |

### 6.1. `GET /api/comments`

| Tham số | Bắt buộc | Mô tả |
|---------|:--------:|-------|
| `target_type` | ✔ | `event` \| `article` |
| `target_id` | ✔ | `_id` của sự kiện / bài viết |
| `status` | ✘ | **Chỉ admin.** Mặc định công khai là `approved` |

```json
{
  "success": true,
  "data": [
    {
      "id": "66f0a1b2c3d4e5f604000001",
      "author": { "display_name": "Trần Văn Bình", "is_staff": false },
      "content": "Chương trình rất ý nghĩa, mong nhóm tổ chức thêm nhiều buổi nữa!",
      "created_at": "2026-03-16T15:22:00Z",
      "replies": [
        {
          "id": "66f0a1b2c3d4e5f604000002",
          "author": { "display_name": "Ban quản trị Nụ Cười Em", "is_staff": true },
          "content": "Cảm ơn bạn đã đồng hành!",
          "created_at": "2026-03-17T01:10:00Z"
        }
      ]
    }
  ],
  "meta": { "page": 1, "page_size": 20, "total": 7, "total_pages": 1, "has_next": false, "has_previous": false }
}
```

Phản hồi lồng đúng **một cấp** (`parent_id`). `author.email`, `ip_hash`,
`user_agent`, `moderation_note` không xuất hiện trong response công khai.

### 6.2. `POST /api/comments`

```json
{
  "target_type": "event",
  "target_id": "66f0a1b2c3d4e5f602000001",
  "author": { "display_name": "Trần Văn Bình", "email": "binh.tran@example.com" },
  "content": "Chương trình rất ý nghĩa!",
  "hp_website": ""
}
```

- `content` là **plain text**, tối đa 2000 ký tự; HTML bị escape khi hiển thị.
- Server ép `status = "pending"`, lưu `ip_hash` + `user_agent`.
- Từ chối nếu đối tượng có `allow_comments = false` → `403 COMMENTS_DISABLED`.
- Response `201`: `{ "id": "...", "status": "pending" }` kèm
  `message: "Bình luận của bạn đang chờ duyệt."`

### 6.3. `POST /api/comments/{id}/reply`

```json
{ "content": "Cảm ơn bạn đã đồng hành cùng chương trình!" }
```

Tạo bình luận con với `parent_id = {id}`, `author.user_id` = admin hiện tại,
`author.is_staff = true`, và `status = "approved"` ngay (không cần tự duyệt).

### 6.4. Ảnh hưởng tới bộ đếm

Chuyển sang `approved` → `$inc` `comment_count` của `events`/`articles` lên 1;
rời khỏi `approved` (reject/spam/delete) → giảm 1. Xem
[schema §12](DATABASE_SCHEMA.md#12-dữ-liệu-denormalized--cách-đồng-bộ).

---

## 7. Quyên góp — `/api/donations`

> Website **không xử lý thanh toán trực tuyến**. Người quyên góp chuyển khoản
> theo thông tin ngân hàng công khai, sau đó khai báo qua form; admin đối chiếu
> sao kê rồi xác nhận.

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `POST` | `/api/donations` | `—` | Khai báo khoản quyên góp |
| `GET` | `/api/donations/recent` | `—` | Danh sách gần đây (đã ẩn danh tính) |
| `GET` | `/api/donations` | `donations.view` | Danh sách đầy đủ |
| `GET` | `/api/donations/{id}` | `donations.view` | Chi tiết |
| `PATCH` | `/api/donations/{id}` | `donations.manage` | Sửa ghi chú, `is_public`, `received_at` |
| `POST` | `/api/donations/{id}/confirm` | `donations.manage` | Xác nhận đã nhận |
| `POST` | `/api/donations/{id}/reject` | `donations.manage` | Từ chối / không đối chiếu được |
| `GET` | `/api/donations/summary` | `donations.view` | Thống kê tài chính |
| `GET` | `/api/donations/export` | `reports.export` | Xuất CSV |

### 7.1. `POST /api/donations`

Quyền `—`.

```json
{
  "kind": "cash",
  "amount": "2000000",
  "method": "bank_transfer",
  "bank_reference": "FT26031512345678",
  "event_id": "66f0a1b2c3d4e5f602000001",
  "donor": {
    "full_name": "Nguyễn Văn An",
    "email": "an.nguyen@example.com",
    "phone": "0912345678",
    "is_anonymous": false,
    "message": "Chúc các bé luôn vui khoẻ!"
  },
  "hp_website": ""
}
```

Với quyên góp vật dụng:

```json
{
  "kind": "goods",
  "method": "in_kind",
  "items": [
    { "name": "Sách thiếu nhi", "quantity": 50, "unit": "cuốn", "estimated_value": "1500000" },
    { "name": "Màu acrylic", "quantity": 20, "unit": "bộ" }
  ],
  "donor": { "full_name": "Lê Thị Mai", "is_anonymous": true }
}
```

**Quy tắc validation**

| Điều kiện | Lỗi |
|-----------|-----|
| `kind = "cash"` mà `amount <= 0` | `VALIDATION_ERROR` trên `amount` |
| `kind = "goods"` mà `items` rỗng | `VALIDATION_ERROR` trên `items` |
| `amount` không phải số nguyên VND | `VALIDATION_ERROR` |

Server ép `status = "pending"`, sinh `code` dạng `NCE-2026-000123`, bỏ qua mọi
giá trị `status` / `confirmed_by` client gửi lên.

**Response `201`**

```json
{
  "success": true,
  "data": { "id": "66f0a1b2c3d4e5f605000001", "code": "NCE-2026-000123", "status": "pending" },
  "message": "Cảm ơn tấm lòng của bạn! Chúng tôi sẽ đối chiếu và xác nhận sớm nhất."
}
```

### 7.2. `GET /api/donations/recent`

Quyền `—`. Chỉ trả bản ghi `status = "confirmed"` **và** `is_public = true`.
Query: `limit` (mặc định `10`, tối đa `50`).

```json
{
  "success": true,
  "data": [
    { "donor_name": "Nguyễn V. A.", "kind": "cash", "amount": "2000000", "received_at": "2026-03-15" },
    { "donor_name": "Nhà hảo tâm ẩn danh", "kind": "goods", "items_summary": "50 cuốn sách, 20 bộ màu", "received_at": "2026-03-12" }
  ]
}
```

Endpoint này **không bao giờ** trả `donor.email`, `donor.phone`,
`bank_reference`, `admin_note`, `id`, `code`. Tên được viết tắt, hoặc thay bằng
`"Nhà hảo tâm ẩn danh"` khi `is_anonymous = true`
([schema §7.3](DATABASE_SCHEMA.md#73-quy-tắc-hiển-thị-công-khai)).

### 7.3. `GET /api/donations`

Quyền `donations.view`. Query: `status`, `kind`, `method`, `event`,
`from` / `to` (theo `received_at`), `q` (tên hoặc `code`),
`ordering` (mặc định `-created_at`).

Trả về document đầy đủ theo schema §7.5, kèm `event` rút gọn
(`{ id, title, slug }`) và `confirmed_by` (`{ id, full_name }`).

### 7.4. `POST /api/donations/{id}/confirm`

```json
{
  "received_at": "2026-03-15",
  "admin_note": "Đã đối chiếu sao kê Vietcombank ngày 15/03.",
  "is_public": true
}
```

Đặt `status = "confirmed"`, `confirmed_by_id`, `confirmed_at`; gửi thư cảm ơn nếu
có `donor.email` và `email_settings.on_new_donation`. Xác nhận lại khoản đã
`confirmed` → `400 INVALID_STATE`.

### 7.5. `GET /api/donations/summary`

Query: `from`, `to`, `group_by` (`month` | `event` | `kind`, mặc định `month`).

```json
{
  "success": true,
  "data": {
    "total_cash": "48500000",
    "total_goods_value": "12300000",
    "count_confirmed": 132,
    "count_pending": 4,
    "by_month": [
      { "period": "2026-01", "cash": "8200000", "goods_value": "1500000", "count": 21 },
      { "period": "2026-02", "cash": "6400000", "goods_value": "0", "count": 15 }
    ]
  }
}
```

### 7.6. `GET /api/donations/export`

Quyền `reports.export`. Query: `format=csv` (mặc định), `from`, `to`, `status`.

Trả file thay vì JSON:

```
Content-Type: text/csv; charset=utf-8-sig
Content-Disposition: attachment; filename="quyen-gop-2026-01-01_2026-09-08.csv"
```

Dùng BOM `utf-8-sig` để Excel trên Windows hiển thị đúng tiếng Việt.

---

## 8. Tình nguyện viên — `/api/volunteers`

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `POST` | `/api/volunteers` | `—` | Nộp đơn đăng ký |
| `GET` | `/api/volunteers/roles` | `—` | Danh sách vai trò để dựng form |
| `GET` | `/api/volunteers` | `volunteers.view` | Danh sách đơn |
| `GET` | `/api/volunteers/{id}` | `volunteers.view` | Chi tiết đơn |
| `PATCH` | `/api/volunteers/{id}` | `volunteers.review` | Đổi `status` sang `reviewing`, ghi chú |
| `POST` | `/api/volunteers/{id}/approve` | `volunteers.review` | Phê duyệt + gửi email |
| `POST` | `/api/volunteers/{id}/reject` | `volunteers.review` | Từ chối + gửi email |
| `POST` | `/api/volunteers/{id}/resend-email` | `volunteers.review` | Gửi lại email kết quả |
| `GET` | `/api/volunteers/export` | `reports.export` | Xuất CSV |

### 8.1. `POST /api/volunteers`

Quyền `—`.

```json
{
  "full_name": "Lê Thị Mai",
  "email": "mai.le@example.com",
  "phone": "0987654321",
  "date_of_birth": "2004-06-12",
  "gender": "female",
  "occupation": "Sinh viên ĐH Tây Nguyên",
  "address": "Buôn Ma Thuột, Đắk Lắk",
  "roles": ["teaching", "media"],
  "skills": ["vẽ acrylic", "chụp ảnh"],
  "availability": {
    "weekdays": [5, 6],
    "time_slots": ["morning", "afternoon"],
    "hours_per_week": 8,
    "available_from": "2026-04-01"
  },
  "experience": "Từng tham gia CLB tình nguyện của trường 2 năm.",
  "motivation": "Muốn đồng hành cùng các bé qua hoạt động vẽ tranh.",
  "hp_website": ""
}
```

**Validation**

| Điều kiện | Lỗi |
|-----------|-----|
| `roles` rỗng | `VALIDATION_ERROR` |
| `date_of_birth` khiến tuổi < 16 | `VALIDATION_ERROR` — `"Tình nguyện viên phải từ 16 tuổi trở lên."` |
| `phone` không đúng định dạng VN (`0[3\|5\|7\|8\|9]xxxxxxxx`) | `VALIDATION_ERROR` |
| Cùng `email` đã nộp đơn trong 30 ngày qua | `409 DUPLICATE_APPLICATION` |

Server ép `status = "new"`. Response `201`:
`{ "id": "...", "status": "new" }` kèm
`message: "Cảm ơn bạn đã đăng ký! Chúng tôi sẽ liên hệ trong 3–5 ngày."`

### 8.2. `GET /api/volunteers/roles`

```json
{
  "success": true,
  "data": [
    { "value": "event_organizer", "label": "Tổ chức sự kiện" },
    { "value": "teaching", "label": "Dạy học / hướng dẫn nghệ thuật" },
    { "value": "technical_support", "label": "Hỗ trợ kỹ thuật" },
    { "value": "fundraising", "label": "Gây quỹ" },
    { "value": "media", "label": "Truyền thông, chụp ảnh, dựng phim" },
    { "value": "other", "label": "Khác" }
  ]
}
```

### 8.3. `GET /api/volunteers`

Quyền `volunteers.view`. Query: `status`, `role`, `q` (tên / email / SĐT),
`from`, `to`, `ordering` (mặc định `-created_at`).

### 8.4. `POST /api/volunteers/{id}/approve`

```json
{
  "note": "Mời bạn tham gia buổi định hướng ngày 05/04.",
  "send_email": true
}
```

Đặt `status = "approved"`, `reviewed_by_id`, `reviewed_at`, `review_note`, rồi
gửi email và ghi kết quả vào `notification`
([schema §8.3](DATABASE_SCHEMA.md#83-embedded-notificationlog)).

**Response `200`**

```json
{
  "success": true,
  "data": {
    "id": "66f0a1b2c3d4e5f606000001",
    "status": "approved",
    "notification": { "sent_at": "2026-03-22T04:01:12Z", "channel": "email", "status": "sent", "error": "" }
  }
}
```

Nếu vượt hạn mức 100 email/ngày của free tier, API vẫn trả `200` (trạng thái đơn
đã đổi) nhưng `notification.status = "failed"` kèm `error`. Admin dùng
`/resend-email` để gửi lại — API **không** rollback việc phê duyệt chỉ vì lỗi email.

---

## 9. Liên hệ — `/api/contact`

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `POST` | `/api/contact` | `—` | Gửi tin nhắn liên hệ |
| `GET` | `/api/contact-messages` | `contact.view` | Hộp thư |
| `GET` | `/api/contact-messages/{id}` | `contact.view` | Chi tiết (tự đánh dấu `read`) |
| `PATCH` | `/api/contact-messages/{id}` | `contact.handle` | Đổi `status`, ghi `reply_note` |
| `DELETE` | `/api/contact-messages/{id}` | `contact.handle` | Xoá vĩnh viễn |

### 9.1. `POST /api/contact`

```json
{
  "full_name": "Phạm Quốc Hưng",
  "email": "hung.pham@example.com",
  "phone": "0933444555",
  "subject": "Đề nghị tài trợ vật tư mỹ thuật",
  "message": "Chào nhóm, công ty mình muốn tài trợ màu vẽ cho chương trình...",
  "hp_website": ""
}
```

Lưu vào `contact_messages` với `status = "new"` **và** gửi email tới
`site_settings.email_settings.notify_emails` nếu `on_new_contact = true`.
Lưu DB trước, gửi email sau — mất email không làm mất liên hệ.

Response `201` kèm `message: "Cảm ơn bạn, chúng tôi sẽ phản hồi sớm."`

---

## 10. Cài đặt website — `/api/settings`

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `GET` | `/api/settings/public` | `—` | Cấu hình hiển thị cho phần công khai |
| `GET` | `/api/settings` | `settings.manage` | Toàn bộ cấu hình |
| `PATCH` | `/api/settings` | `settings.manage` | Cập nhật (merge từng phần) |
| `POST` | `/api/settings/recalculate-stats` | `settings.manage` | Tính lại `impact_stats` ngay |

### 10.1. `GET /api/settings/public`

Quyền `—`. Được gọi một lần khi khởi động app, dùng cho `Header`, `Footer`,
`HeroBanner`, `ImpactStats`, trang quyên góp.

```json
{
  "success": true,
  "data": {
    "organization": {
      "name": "Nụ Cười Em",
      "short_name": "NCE",
      "tagline": "Mang nghệ thuật và cảm xúc đến trẻ em",
      "description": "...",
      "founded_year": 2025,
      "logo": { "url": "...", "alt": "Logo Nụ Cười Em" }
    },
    "contact": {
      "email": "lienhe@nucuoiem.org",
      "phone": "0905123456",
      "address": "TP. Buôn Ma Thuột, Đắk Lắk",
      "map_url": "https://maps.app.goo.gl/xxxx",
      "working_hours": "T2–T6, 08:00–17:00"
    },
    "social": { "facebook": "https://facebook.com/nucuoiem", "youtube": "", "tiktok": "", "instagram": "", "zalo": "" },
    "impact_stats": {
      "children_helped": 240,
      "events_held": 34,
      "volunteers_count": 86,
      "total_donations": "48500000",
      "last_calculated_at": "2026-09-08T00:00:00Z"
    },
    "bank_accounts": [
      {
        "bank_name": "Vietcombank",
        "account_number": "0123456789",
        "account_holder": "NHOM NU CUOI EM",
        "branch": "CN Đắk Lắk",
        "qr_image": { "url": "..." },
        "is_primary": true
      }
    ],
    "pages": { "about_us": "<p>...</p>", "privacy_policy": "<p>...</p>", "terms_of_use": "<p>...</p>", "transparency_commitment": "<p>...</p>" },
    "seo": { "meta_title": "Nụ Cười Em", "meta_description": "...", "og_image": { "url": "..." } },
    "maintenance_mode": false
  }
}
```

Endpoint này **chỉ** trả tài khoản ngân hàng có `is_primary = true`, và **không**
trả `email_settings` hay `updated_by`
([schema §10.9](DATABASE_SCHEMA.md#109-index)).

Khi `maintenance_mode = true`, mọi endpoint công khai khác trả
`503 MAINTENANCE_MODE`; endpoint quản trị vẫn hoạt động bình thường.

### 10.2. `PATCH /api/settings`

Merge theo từng nhóm — chỉ gửi phần cần đổi:

```json
{
  "contact": { "phone": "0905999888" },
  "email_settings": { "on_new_donation": false }
}
```

Nhóm không gửi lên giữ nguyên. Riêng `bank_accounts` là **thay thế toàn bộ mảng**,
không merge từng phần tử. Mọi trường HTML trong `pages` được sanitize.

---

## 11. Dashboard — `/api/dashboard`

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `GET` | `/api/dashboard/stats` | đã đăng nhập | Số liệu tổng quan |
| `GET` | `/api/dashboard/activity` | đã đăng nhập | Nhật ký hoạt động gần đây |

### 11.1. `GET /api/dashboard/stats`

Cấp dữ liệu cho `pages/admin/Dashboard.jsx` và `components/admin/StatCard.jsx`.

```json
{
  "success": true,
  "data": {
    "events": { "total": 34, "published": 30, "draft": 2, "scheduled": 2 },
    "articles": { "total": 88, "published": 74, "pending_review": 6, "draft": 8 },
    "comments": { "pending": 12, "approved_last_7d": 23 },
    "donations": { "count_confirmed": 132, "count_pending": 4, "total_cash": "48500000", "this_month_cash": "3200000" },
    "volunteers": { "total": 86, "new": 5, "approved": 71 },
    "contact_messages": { "new": 3 },
    "needs_attention": {
      "articles_pending": 6,
      "comments_pending": 12,
      "donations_pending": 4,
      "volunteers_new": 5,
      "contact_new": 3
    }
  }
}
```

`needs_attention` là nguồn dữ liệu cho badge số trên `AdminSidebar.jsx`.
Các số đếm chỉ tính trong phạm vi quyền của người gọi — tài khoản `moderator`
chỉ nhận khối `comments` và `contact_messages`, các khối khác là `null`.

### 11.2. `GET /api/dashboard/activity`

Query: `limit` (mặc định `20`, tối đa `100`), `actor`, `target_collection`.

```json
{
  "success": true,
  "data": [
    {
      "id": "66f0a1b2c3d4e5f60a000001",
      "actor": { "id": "66f0a1b2c3d4e5f601000001", "full_name": "Ngô Minh Hiếu", "email": "admin@nucuoiem.org" },
      "action": "approve",
      "target_collection": "articles",
      "target_id": "66f0a1b2c3d4e5f603000001",
      "summary": "Duyệt bài viết \"Bức tranh đầu tiên của bé An\"",
      "created_at": "2026-03-20T03:15:00Z"
    }
  ]
}
```

---

## 12. Upload media — `/api/uploads`

Ảnh và video **không lưu trong MongoDB** — backend nhận file, đẩy lên Cloudinary
qua `backend/services/cloudinary_service.py`, rồi trả về đối tượng `Image` để
client gắn vào `cover_image` / `gallery`.

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `POST` | `/api/uploads/image` | đã đăng nhập | Upload ảnh (admin) |
| `POST` | `/api/uploads/public-image` | `—` | Upload ảnh kèm bài viết của khách |
| `DELETE` | `/api/uploads/image` | `articles.update` hoặc `events.update` | Xoá ảnh trên Cloudinary |

### 12.1. `POST /api/uploads/image`

`Content-Type: multipart/form-data`

| Field | Bắt buộc | Mô tả |
|-------|:--------:|-------|
| `file` | ✔ | Ảnh, tối đa **5 MB** |
| `folder` | ✘ | `events` \| `news` \| `settings` \| `avatars` (mặc định `misc`) |
| `alt` | ✘ | Văn bản thay thế |

Định dạng chấp nhận: `jpg`, `jpeg`, `png`, `webp`. Server kiểm tra **magic bytes**,
không tin `Content-Type` do client khai.

**Response `201`**

```json
{
  "success": true,
  "data": {
    "url": "https://res.cloudinary.com/nce/image/upload/v1710000000/events/ve-tranh.jpg",
    "public_id": "events/ve-tranh",
    "alt": "Các bé vẽ tranh acrylic",
    "width": 1600,
    "height": 900,
    "bytes": 284512,
    "format": "jpg"
  }
}
```

Lỗi: `413 FILE_TOO_LARGE`, `400 UNSUPPORTED_FILE_TYPE`,
`502 UPLOAD_FAILED` (Cloudinary lỗi hoặc hết hạn mức 25GB).

### 12.2. `POST /api/uploads/public-image`

Dành cho khách gửi bài. Giới hạn chặt hơn: tối đa **2 MB**, **3 ảnh/giờ/IP**,
ép `folder = "news/community"`. Ảnh không được gắn vào bài viết nào trong
24 giờ sẽ bị job dọn dẹp xoá khỏi Cloudinary.

### 12.3. `DELETE /api/uploads/image`

```json
{ "public_id": "events/ve-tranh" }
```

Chỉ xoá được ảnh trong các folder do ứng dụng quản lý. Trả `204`.

---

## 13. Health check

`GET /api/health` — quyền `—`, không throttle. Dùng cho Nginx và giám sát VPS.

```json
{
  "success": true,
  "data": { "status": "ok", "database": "ok", "version": "1.0.0", "time": "2026-09-08T02:30:00Z" }
}
```

Mất kết nối MongoDB → `503` với `database: "error"`.

### 13.1. Route ngoài `/api`

Hai route do Django phục vụ nhưng trả HTML/XML thay vì JSON, không đi qua lớp vỏ
`{ success, data }`:

| Method | Route | Mô tả |
|--------|-------|-------|
| `GET` | `/share/events/{slug}` | HTML tối giản có thẻ Open Graph cho bot mạng xã hội |
| `GET` | `/share/news/{slug}` | Tương tự cho bài viết |
| `GET` | `/sitemap.xml` | Sitemap sự kiện & bài viết đã công bố |

**`/share/...`** chỉ được gọi khi Nginx nhận diện User-Agent là bot
(Facebook, Zalo…) — người dùng thật luôn nhận React app. Response chứa:

```html
<meta property="og:type" content="article">
<meta property="og:title" content="Ngày hội vẽ tranh acrylic cùng lớp Gia Đình">
<meta property="og:description" content="Buổi vẽ tranh acrylic dành cho 25 bé…">
<meta property="og:image" content="https://res.cloudinary.com/nce/image/upload/c_fill,w_1200,h_630,f_jpg,q_auto/events/ve-tranh.jpg">
<meta property="og:url" content="https://nucuoiem.org/events/ngay-hoi-ve-tranh-acrylic-cung-lop-gia-dinh">
<link rel="canonical" href="https://nucuoiem.org/events/ngay-hoi-ve-tranh-acrylic-cung-lop-gia-dinh">
```

- Slug không tồn tại hoặc chưa công bố → vẫn trả `200` với OG mặc định của
  website (từ `site_settings.seo`), **không** trả 404 — tránh để lộ nội dung
  chưa công bố và tránh bot cache lỗi. Route `/news/submit` cũng rơi vào trường hợp này.
- `og:image` luôn là JPEG 1200×630 (`f_jpg`): một số crawler không đọc được WebP/AVIF.
- Cache 10 phút trong FileBasedCache.

Cấu hình nhận diện bot ở [`DEPLOYMENT.md` §7.2](DEPLOYMENT.md#72-deploynginxnu-cuoi-emconf).

---

## 14. Mã lỗi

`error.code` là chuỗi ổn định để frontend xử lý; `error.message` là tiếng Việt
để hiển thị trực tiếp cho người dùng.

| Code | HTTP | Ý nghĩa |
|------|------|---------|
| `VALIDATION_ERROR` | `400` | Dữ liệu không hợp lệ, chi tiết ở `details` |
| `INVALID_STATE` | `400` | Thao tác không hợp lệ với trạng thái hiện tại |
| `SELF_ACTION_FORBIDDEN` | `400` | Không thể tự thao tác lên tài khoản của mình |
| `LAST_SUPERADMIN` | `400` | Không thể vô hiệu hoá `superadmin` cuối cùng |
| `INVALID_CREDENTIALS` | `401` | Sai email/mật khẩu |
| `TOKEN_EXPIRED` | `401` | Access token hết hạn — gọi `/auth/refresh` |
| `TOKEN_INVALID` | `401` | Token sai, hỏng hoặc đã bị thu hồi |
| `ACCOUNT_DISABLED` | `403` | Tài khoản bị khoá |
| `PERMISSION_DENIED` | `403` | Không đủ quyền |
| `COMMENTS_DISABLED` | `403` | Đối tượng đã tắt bình luận |
| `NOT_FOUND` | `404` | Không tìm thấy hoặc chưa công bố |
| `DUPLICATE_EMAIL` | `409` | Email đã tồn tại |
| `DUPLICATE_SLUG` | `409` | Slug đã tồn tại |
| `ALREADY_LIKED` | `409` | Đã thích bài viết này |
| `DUPLICATE_APPLICATION` | `409` | Đã nộp đơn tình nguyện trong 30 ngày qua |
| `FILE_TOO_LARGE` | `413` | File vượt giới hạn |
| `UNSUPPORTED_FILE_TYPE` | `400` | Định dạng file không được chấp nhận |
| `THROTTLED` | `429` | Vượt giới hạn tần suất |
| `MAINTENANCE_MODE` | `503` | Website đang bảo trì |
| `UPLOAD_FAILED` | `502` | Cloudinary lỗi |
| `INTERNAL_ERROR` | `500` | Lỗi hệ thống |

Toàn bộ được chuẩn hoá tại `apps/common/exceptions.py` (custom
`EXCEPTION_HANDLER` của DRF). Ở production (`DJANGO_DEBUG=False`),
`INTERNAL_ERROR` **không** trả traceback ra response.

---

## 15. Giới hạn tần suất (throttling)

Cấu hình bằng `DEFAULT_THROTTLE_RATES` của DRF, đếm theo IP với endpoint công khai
và theo tài khoản với endpoint quản trị.

| Scope | Giới hạn | Áp dụng cho |
|-------|----------|-------------|
| `anon_read` | 120/phút | Mọi GET công khai |
| `login` | 5/5 phút | `POST /api/auth/login` |
| `comment_create` | 5/giờ | `POST /api/comments` |
| `article_submit` | 3/giờ | `POST /api/articles/submit` |
| `donation_create` | 10/giờ | `POST /api/donations` |
| `volunteer_create` | 3/giờ | `POST /api/volunteers` |
| `contact_create` | 5/giờ | `POST /api/contact` |
| `public_upload` | 3/giờ | `POST /api/uploads/public-image` |
| `user` | 1000/giờ | Mọi endpoint đã xác thực |

Response `429` kèm header `Retry-After` (giây):

```json
{
  "success": false,
  "error": {
    "code": "THROTTLED",
    "message": "Bạn thao tác quá nhanh. Vui lòng thử lại sau 42 giây.",
    "details": null
  }
}
```

Ngoài throttling, mọi form công khai đều có trường **honeypot** `hp_website` —
bot điền vào sẽ nhận response thành công giả nhưng dữ liệu không được lưu.

### 15.1. CORS

Chỉ origin trong `CORS_ALLOWED_ORIGINS` được phép gọi API
(dev: `http://localhost:5173`; prod: domain chính thức). Cho phép header
`Authorization`, `Content-Type`, `X-Visitor-Key`. Không dùng cookie qua CORS —
token nằm trong header, `CORS_ALLOW_CREDENTIALS = False`.

---

## 16. Bảng tra endpoint

| Method | Endpoint | Quyền | Service frontend |
|--------|----------|-------|------------------|
| `POST` | `/api/auth/login` | `—` | `authService.js` |
| `POST` | `/api/auth/refresh` | `—` | `api.js` (interceptor) |
| `POST` | `/api/auth/logout` | đăng nhập | `authService.js` |
| `GET` `PATCH` | `/api/auth/me` | đăng nhập | `authService.js` |
| `POST` | `/api/auth/change-password` | đăng nhập | `authService.js` |
| `GET` `POST` | `/api/users` | `users.manage` | `userService.js` |
| `GET` `PATCH` `DELETE` | `/api/users/{id}` | `users.manage` | `userService.js` |
| `POST` | `/api/users/{id}/reset-password` | `users.manage` | `userService.js` |
| `GET` | `/api/events` | `—` | `eventService.js` |
| `GET` | `/api/events/{slug}` | `—` | `eventService.js` |
| `GET` | `/api/events/filters` | `—` | `eventService.js` |
| `POST` | `/api/events/{id}/view` | `—` | `eventService.js` |
| `POST` | `/api/events` | `events.create` | `eventService.js` |
| `PATCH` `DELETE` | `/api/events/{id}` | `events.update` / `events.delete` | `eventService.js` |
| `POST` | `/api/events/{id}/publish` | `events.publish` | `eventService.js` |
| `POST` | `/api/events/{id}/archive` | `events.publish` | `eventService.js` |
| `GET` | `/api/articles` | `—` | `newsService.js` |
| `GET` | `/api/articles/{slug}` | `—` | `newsService.js` |
| `GET` | `/api/articles/tags` | `—` | `newsService.js` |
| `POST` | `/api/articles/submit` | `—` | `newsService.js` |
| `POST` | `/api/articles/{id}/view` | `—` | `newsService.js` |
| `POST` `DELETE` | `/api/articles/{id}/like` | `—` | `newsService.js` |
| `POST` | `/api/articles` | `articles.create` | `newsService.js` |
| `PATCH` `DELETE` | `/api/articles/{id}` | `articles.update` / `articles.delete` | `newsService.js` |
| `POST` | `/api/articles/{id}/approve` | `articles.review` | `newsService.js` |
| `POST` | `/api/articles/{id}/reject` | `articles.review` | `newsService.js` |
| `POST` | `/api/articles/{id}/publish` | `articles.review` | `newsService.js` |
| `GET` `POST` | `/api/comments` | `—` | `commentService.js` |
| `GET` | `/api/comments/pending` | `comments.moderate` | `commentService.js` |
| `POST` | `/api/comments/{id}/approve` | `comments.moderate` | `commentService.js` |
| `POST` | `/api/comments/{id}/reject` | `comments.moderate` | `commentService.js` |
| `POST` | `/api/comments/{id}/spam` | `comments.moderate` | `commentService.js` |
| `POST` | `/api/comments/{id}/reply` | `comments.moderate` | `commentService.js` |
| `DELETE` | `/api/comments/{id}` | `comments.moderate` | `commentService.js` |
| `POST` | `/api/donations` | `—` | `donationService.js` |
| `GET` | `/api/donations/recent` | `—` | `donationService.js` |
| `GET` | `/api/donations` | `donations.view` | `donationService.js` |
| `GET` `PATCH` | `/api/donations/{id}` | `donations.view` / `donations.manage` | `donationService.js` |
| `POST` | `/api/donations/{id}/confirm` | `donations.manage` | `donationService.js` |
| `POST` | `/api/donations/{id}/reject` | `donations.manage` | `donationService.js` |
| `GET` | `/api/donations/summary` | `donations.view` | `donationService.js` |
| `GET` | `/api/donations/export` | `reports.export` | `donationService.js` |
| `POST` | `/api/volunteers` | `—` | `volunteerService.js` |
| `GET` | `/api/volunteers/roles` | `—` | `volunteerService.js` |
| `GET` | `/api/volunteers` | `volunteers.view` | `volunteerService.js` |
| `GET` `PATCH` | `/api/volunteers/{id}` | `volunteers.view` / `volunteers.review` | `volunteerService.js` |
| `POST` | `/api/volunteers/{id}/approve` | `volunteers.review` | `volunteerService.js` |
| `POST` | `/api/volunteers/{id}/reject` | `volunteers.review` | `volunteerService.js` |
| `POST` | `/api/volunteers/{id}/resend-email` | `volunteers.review` | `volunteerService.js` |
| `GET` | `/api/volunteers/export` | `reports.export` | `volunteerService.js` |
| `POST` | `/api/contact` | `—` | `settingsService.js` |
| `GET` | `/api/contact-messages` | `contact.view` | `settingsService.js` |
| `GET` `PATCH` `DELETE` | `/api/contact-messages/{id}` | `contact.view` / `contact.handle` | `settingsService.js` |
| `GET` | `/api/settings/public` | `—` | `settingsService.js` |
| `GET` `PATCH` | `/api/settings` | `settings.manage` | `settingsService.js` |
| `POST` | `/api/settings/recalculate-stats` | `settings.manage` | `settingsService.js` |
| `GET` | `/api/dashboard/stats` | đăng nhập | `api.js` |
| `GET` | `/api/dashboard/activity` | đăng nhập | `api.js` |
| `POST` | `/api/uploads/image` | đăng nhập | `api.js` |
| `POST` | `/api/uploads/public-image` | `—` | `api.js` |
| `DELETE` | `/api/uploads/image` | `events.update` / `articles.update` | `api.js` |
| `GET` | `/api/health` | `—` | — |
| `GET` | `/share/events/{slug}`, `/share/news/{slug}` | `—` (chỉ bot) | — |
| `GET` | `/sitemap.xml` | `—` | — |

---

## 17. Tài liệu liên quan

- [`DATABASE_SCHEMA.md`](DATABASE_SCHEMA.md) — cấu trúc collection, tên trường, index
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — kiến trúc tổng thể hệ thống
- [`SETUP.md`](SETUP.md) — cài đặt môi trường dev
- [`DEPLOYMENT.md`](DEPLOYMENT.md) — triển khai VPS, Nginx, cron job
