# Quy trình đóng góp code — Nụ Cười Em

## Mô hình nhánh (Git Flow rút gọn)

| Nhánh | Vai trò | Nhận merge từ |
|-------|---------|---------------|
| `main` | Mã chạy production trên VPS. Được bảo vệ, chỉ merge qua PR. | `dev` (release), `hotfix/*` |
| `dev` | Nhánh tích hợp. Mọi tính năng gộp về đây trước. | `feature/*`, `setup` |
| `setup` | Khởi tạo cấu trúc dự án, cấu hình hạ tầng ban đầu. | — |
| `feature/<tên>` | Một tính năng / module. Cắt ra từ `dev`, xoá sau khi merge. | — |
| `hotfix/<tên>` | Sửa lỗi khẩn cấp trên production. Cắt từ `main`. | — |

### Nhánh feature theo module

```
feature/auth              # accounts, JWT, phân quyền
feature/events            # sự kiện + lịch tự động publish
feature/news              # bài viết + kiểm duyệt
feature/donations         # quyên góp + báo cáo
feature/volunteers        # đăng ký tình nguyện viên + email
feature/comments          # bình luận + kiểm duyệt
feature/public-site       # giao diện công khai (home, contact, layout)
feature/admin-dashboard   # dashboard quản trị
```

## Qui ước commit — Conventional Commits

```
<type>(<scope>): <mô tả ngắn, tiếng Việt không dấu hoặc tiếng Anh>

type: feat | fix | docs | style | refactor | test | chore | ci | build
scope: auth, events, news, donations, volunteers, comments, admin, deploy...
```

Ví dụ: `feat(events): them API loc su kien theo ngay`

## Luồng làm việc

```bash
git checkout dev && git pull
git checkout -b feature/events
# ... code + test ...
git push -u origin feature/events
# Mở Pull Request: feature/events -> dev
```

- PR phải pass CI (`.github/workflows/ci.yml`) trước khi merge.
- Ít nhất 1 review approve.
- Merge bằng **Squash and merge**, xoá nhánh feature sau khi merge.
- Release: mở PR `dev -> main`, sau khi merge sẽ tự deploy qua `deploy.yml`.

## Trước khi push — checklist bảo mật

- [ ] Không commit file `.env` (chỉ `.env.example` được commit).
- [ ] Không hardcode secret/token/mật khẩu/URI database trong code.
- [ ] Không commit khoá `.pem`, `.key`, file service-account.
- [ ] `git status` sạch, không có file lạ ngoài phạm vi task.
