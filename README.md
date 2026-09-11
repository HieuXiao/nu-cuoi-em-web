# Nụ Cười Em — Website

Nền tảng kết nối cộng đồng, mang nghệ thuật và cảm xúc đến trẻ em mắc hội chứng
rối loạn phổ tự kỷ tại lớp trẻ "Gia Đình" (TP. Buôn Ma Thuột, Đắk Lắk).

## Kiến trúc

| Thành phần | Công nghệ |
|-----------|-----------|
| Frontend | React + Vite, React Router, Tailwind CSS |
| Backend | Django + Django REST Framework |
| Database | MongoDB Atlas (`django-mongodb-backend`) |
| Auth | JWT (djangorestframework-simplejwt) |
| Media | Cloudinary |
| Email | SendGrid / Resend |
| Hạ tầng | 1 VPS Việt Nam — Nginx (static + reverse proxy) + Gunicorn (systemd) |

Repo dạng monorepo: `frontend/` và `backend/` tách riêng, deploy chung trên một VPS.

## Cấu trúc thư mục

```
frontend/   Ứng dụng React (Vite)
backend/    Django project (config/ + apps/)
deploy/     Cấu hình Nginx, Gunicorn, systemd
docker/     docker-compose MongoDB cho dev local
DOC/        Tài liệu dự án (project/ + technical/)
```

## Bắt đầu

Xem [`DOC/technical/SETUP.md`](DOC/technical/SETUP.md) để cài đặt môi trường dev,
và [`CONTRIBUTING.md`](CONTRIBUTING.md) để biết quy trình nhánh & commit.

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
make dev            # chạy song song frontend + backend
```

## Tài liệu

| Tài liệu | Nội dung |
|----------|----------|
| [`ARCHITECTURE.md`](DOC/technical/ARCHITECTURE.md) | Kiến trúc tổng thể, luồng nghiệp vụ, quyết định kỹ thuật |
| [`DATABASE_SCHEMA.md`](DOC/technical/DATABASE_SCHEMA.md) | Cấu trúc các collection MongoDB, index |
| [`API.md`](DOC/technical/API.md) | Hợp đồng REST API giữa frontend và backend |
| [`SETUP.md`](DOC/technical/SETUP.md) | Cài đặt môi trường phát triển |
| [`DEPLOYMENT.md`](DOC/technical/DEPLOYMENT.md) | Triển khai VPS, CI/CD, sao lưu, vận hành |
| [`Cau-truc-Repository.md`](DOC/project/Cau-truc-Repository.md) | Vai trò từng thư mục, quy ước đặt code, việc cần làm trước khi code |
| [`Phan-tich-chi-phi-van-hanh.md`](DOC/project/Phan-tich-chi-phi-van-hanh.md) | Chi phí vận hành, giới hạn free tier |

## License

[MIT](LICENSE) © 2025 Nụ Cười Em
