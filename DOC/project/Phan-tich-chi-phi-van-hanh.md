# Phân tích chi phí vận hành — Nụ Cười Em

Tài liệu phân tích chi phí duy trì website hằng tháng/năm, giới hạn của các dịch
vụ miễn phí, và kế hoạch xử lý khi vượt giới hạn.

| Hạng mục | Giá trị |
|----------|---------|
| Ngân sách được duyệt | **100.000đ / tháng** |
| Chi phí ước tính | **≈ 81.600đ / tháng** (979.000đ / năm) |
| Còn dư | ≈ 18.400đ / tháng |
| Chi phí phát triển | 0đ — đội ngũ tình nguyện |
| Kiến trúc tham chiếu | [`ARCHITECTURE.md`](../technical/ARCHITECTURE.md) |

> **Về giá của dịch vụ bên thứ ba:** giới hạn free tier và giá gói trả phí trong
> tài liệu là số tham khảo tại thời điểm viết, nhà cung cấp có thể thay đổi bất
> cứ lúc nào. **Kiểm tra lại trên trang giá chính thức** trước khi đăng ký và
> trong mỗi lần rà soát hằng quý (§10).

---

## 1. Tóm tắt

Website chỉ tốn tiền cho **hai thứ: VPS và domain**. Mọi dịch vụ còn lại
(database, lưu ảnh, email, CDN, CI/CD, giám sát, backup) dùng gói miễn phí, và
mức sử dụng dự kiến chỉ chiếm **dưới 25%** giới hạn của từng gói.

Rủi ro chi phí lớn nhất không nằm ở tháng bình thường mà ở **tháng có bài viết lan
truyền mạnh**: băng thông ảnh trên Cloudinary có thể tiến sát giới hạn. Kiến trúc
đã có biện pháp giảm thiểu (§4.1), và §7 mô tả phương án dự phòng.

---

## 2. Chi phí định kỳ

| Hạng mục | Giải pháp | Đơn giá | Quy ra tháng | Quy ra năm |
|----------|-----------|---------|-------------:|-----------:|
| Hosting | VPS Việt Nam 1 vCPU / 1GB RAM / 20GB SSD | 50.000đ/tháng | 50.000đ | 600.000đ |
| Domain | `.com`, giá gia hạn thực tế | 379.000đ/năm | 31.600đ | 379.000đ |
| Database | MongoDB Atlas M0 × 2 cluster (dev + production) | Miễn phí | 0đ | 0đ |
| Lưu trữ ảnh | Cloudinary Free | Miễn phí | 0đ | 0đ |
| Video | YouTube | Miễn phí | 0đ | 0đ |
| Email | Resend Free | Miễn phí | 0đ | 0đ |
| CDN, DNS, SSL | Cloudflare Free | Miễn phí | 0đ | 0đ |
| CI/CD | GitHub Actions (repo công khai) | Miễn phí | 0đ | 0đ |
| Giám sát uptime | UptimeRobot Free | Miễn phí | 0đ | 0đ |
| Backup ngoài VPS | Google Drive (15GB miễn phí) | Miễn phí | 0đ | 0đ |
| Thanh toán quyên góp | Chuyển khoản ngân hàng + VietQR | 0% phí | 0đ | 0đ |
| **Tổng** | | | **≈ 81.600đ** | **979.000đ** |

Giá domain dùng **giá gia hạn**, không dùng giá khuyến mãi năm đầu — tránh bất ngờ
khi sang năm thứ hai.

---

## 3. Dịch vụ miễn phí — giới hạn và mức sử dụng dự kiến

| Dịch vụ | Giới hạn gói miễn phí (tham khảo) | Dự kiến dùng | Tỷ lệ | Mức rủi ro |
|---------|-----------------------------------|--------------|------:|:----------:|
| MongoDB Atlas M0 | 512MB lưu trữ | ~40MB năm đầu | ~8% | Thấp |
| Cloudinary | 25 credit/tháng | ~4–6 credit/tháng bình thường | ~20% | **Trung bình** |
| Resend | 3.000 email/tháng, 100 email/ngày | ~100–250 email/tháng | ~8% | Thấp (tháng) · Trung bình (ngày cao điểm) |
| Cloudflare | Không giới hạn băng thông CDN | — | — | Thấp |
| GitHub Actions | Không giới hạn phút với repo công khai | ~300 phút/tháng | — | Thấp |
| UptimeRobot | 50 monitor, chu kỳ 5 phút | 1 monitor | 2% | Thấp |
| Google Drive | 15GB (dùng chung với Gmail) | < 200MB backup | ~1% | Thấp |

Chi tiết cách ước tính ở §4.

---

## 4. Ước tính mức sử dụng

### 4.1. Cloudinary

Cloudinary tính theo **credit**. Theo cách tính hiện hành, 1 credit tương đương
một trong ba: **1GB lưu trữ**, **1GB băng thông**, hoặc **1.000 lượt biến đổi ảnh**.

**Giả định:** 10.000 lượt xem trang/tháng; năm đầu có ~60 sự kiện và ~200 bài viết.

| Hạng mục | Cách tính | Credit/tháng |
|----------|-----------|-------------:|
| Lưu trữ | 60 sự kiện × 20 ảnh × 500KB + 200 bài × 3 ảnh × 400KB ≈ 0,9GB | ~0,9 (tăng dần theo năm) |
| Băng thông | 10.000 lượt xem × ~300KB ảnh/trang ≈ 3GB | ~3 |
| Biến đổi | ~125 ảnh mới/tháng × 4 kích thước + ảnh OG | ~0,6 |
| **Tổng** | | **~4,5** |

**Kịch bản lan truyền** — một bài viết được chia sẻ mạnh, 50.000 lượt xem trong tháng:

| Mức tối ưu ảnh | Ảnh/trang | Băng thông | Tổng credit | So với giới hạn 25 |
|----------------|----------:|-----------:|------------:|:------------------:|
| Đã tối ưu (resize trước upload, `f_auto,q_auto`, lazy-load, ảnh danh sách `w_400`) | ~300KB | ~15GB | ~16 | An toàn |
| Không tối ưu (ảnh gốc từ điện thoại, tải hết ảnh ngay) | ~2MB | ~100GB | ~100 | **Vượt 4 lần** |

Đây là lý do [ARCHITECTURE §10](../technical/ARCHITECTURE.md#10-media) bắt buộc
resize ảnh trước khi upload, dùng bộ kích thước cố định, và **chỉ nhúng video từ
YouTube** — một video 1 phút trên Cloudinary có thể tốn vài chục MB mỗi lượt xem.

Lưu trữ tích luỹ qua các năm (năm 3 ≈ 2,7 credit) nhưng vẫn nhỏ so với băng thông.

### 4.2. Email (Resend)

| Loại email | Ước tính/tháng |
|-----------|---------------:|
| Cảm ơn người quyên góp (khi xác nhận) | 15–30 |
| Kết quả duyệt tình nguyện viên | 10–40 |
| Kết quả duyệt bài cộng đồng | 5–10 |
| Thông báo nội bộ cho admin (đơn mới, quyên góp, bài chờ duyệt, liên hệ) | 50–150 |
| Tạo tài khoản, đặt lại mật khẩu | < 5 |
| **Tổng** | **~100–250** |

**Rủi ro nằm ở giới hạn ngày (100 email/ngày), không phải giới hạn tháng.** Một bài
kêu gọi tình nguyện viên lan truyền có thể mang về 50–80 đơn trong một ngày, mỗi
đơn sinh một email thông báo cho admin. Biện pháp:

- Tắt `on_new_volunteer` trong cài đặt trong những đợt tuyển quân; admin xem
  thẳng trên dashboard.
- Hệ thống đã thiết kế để **lưu DB trước, gửi email sau**, và gửi lại được — vượt
  hạn mức không làm mất dữ liệu ([API §8.4](../technical/API.md#84-post-apivolunteersidapprove)).
- Hạn mức ngày được cấu hình ở `site_settings.email_settings.daily_quota`.

> **SendGrid** được nêu trong tài liệu mô tả dự án, nhưng gói miễn phí của SendGrid
> đã thay đổi chính sách (chuyển sang dùng thử có thời hạn). Kiểm tra lại trước khi
> chọn; mặc định dự án dùng **Resend**.

### 4.3. MongoDB Atlas

Ước tính chi tiết theo từng collection ở
[`DATABASE_SCHEMA.md` §15](../technical/DATABASE_SCHEMA.md#15-ước-tính-dung-lượng-free-tier-512mb):
**~35–40MB năm đầu**, tăng khoảng 30–40MB mỗi năm. Với 512MB, dự án ở trong free
tier ít nhất **10 năm** nếu giữ nguyên nguyên tắc ảnh không lưu trong database.

Mỗi project Atlas chỉ được một cluster M0, nên dev và production nằm ở **hai
project khác nhau** — cả hai đều miễn phí.

### 4.4. VPS

Ngân sách RAM chi tiết ở
[ARCHITECTURE §12.1](../technical/ARCHITECTURE.md#121-ngân-sách-ram-trên-vps-1gb):
dùng ~450–550MB lúc bình thường trên 1GB, cộng 2GB swap. Ổ đĩa 20GB dùng khoảng
3–4GB (hệ điều hành, 3 bản release, backup 7 ngày, log). Cấu hình này đủ cho
khoảng **50 người dùng đồng thời**.

---

## 5. Thay đổi so với tài liệu mô tả dự án v2.0

| Hạng mục | `.docx` v2.0 | Hiện tại | Ảnh hưởng chi phí |
|----------|--------------|----------|-------------------|
| Thanh toán | Stripe / PayOS, ~2–3% phí giao dịch | Chuyển khoản + VietQR, admin đối soát | **Tiết kiệm** — xem bên dưới |
| Database | MongoDB Atlas **hoặc** Firebase | MongoDB Atlas | Không đổi (0đ) |
| Process manager | PM2 | systemd | Không đổi |
| Email | SendGrid / Resend | Resend | Không đổi (0đ) |
| Video | Không nêu | Chỉ YouTube | Tránh rủi ro vượt Cloudinary |
| Backup | Không nêu | `mongodump` → Google Drive | 0đ — Atlas M0 **không có** backup tự động |

**Bỏ cổng thanh toán tiết kiệm nhiều hơn toàn bộ chi phí vận hành.** Giả sử nhận
50 triệu đồng quyên góp tiền mặt mỗi năm:

| | Qua cổng thanh toán (2–3%) | Chuyển khoản trực tiếp |
|--|---------------------------:|-----------------------:|
| Phí giao dịch/năm | 1.000.000đ – 1.500.000đ | 0đ |
| So với chi phí vận hành cả năm (979.000đ) | 102% – 153% | 0% |

Đánh đổi là admin phải đối soát sao kê bằng tay — ước tính 1–2 giờ/tháng với
quy mô hiện tại. Nếu sau này lượng quyên góp tăng mạnh và cần đối soát tự động,
xem xét lại các cổng hỗ trợ VietQR và so phí tại thời điểm đó.

---

## 6. Phân tích độ nhạy

### 6.1. Thuế VAT

Giá niêm yết của nhà cung cấp VPS/domain Việt Nam có thể **chưa gồm VAT 10%**.

| Kịch bản | VPS/tháng | Domain/tháng | Tổng/tháng | Trong ngân sách? |
|----------|----------:|-------------:|-----------:|:----------------:|
| Giá đã gồm VAT | 50.000đ | 31.600đ | 81.600đ | ✔ |
| VPS chưa gồm VAT | 55.000đ | 31.600đ | 86.600đ | ✔ |
| Cả hai chưa gồm VAT | 55.000đ | 34.700đ | 89.700đ | ✔ |

Hỏi rõ nhà cung cấp trước khi ký — cả ba kịch bản vẫn nằm trong ngân sách.

### 6.2. Lựa chọn domain: `.com` hay `.org`

Ngân sách tính theo `.com` (379.000đ/năm), nhưng `backend/.env.example` và tài liệu
kỹ thuật đang dùng ví dụ `nucuoiem.org` — **chưa chốt**.

Với VPS 50.000đ/tháng, phần ngân sách còn lại cho domain là 50.000đ/tháng, tức
**tối đa 600.000đ/năm** (540.000đ/năm nếu VPS chưa gồm VAT). Mọi lựa chọn domain
có giá gia hạn dưới mức này đều không vượt ngân sách.

| Tiêu chí | `.com` | `.org` |
|----------|--------|--------|
| Nhận diện | Phổ biến, người dùng dễ nhớ | Gắn với tổ chức phi lợi nhuận, tăng độ tin cậy khi kêu gọi quyên góp |
| Giá | Đã có báo giá (379.000đ/năm) | Cần xin báo giá gia hạn |

### 6.3. VPS tăng giá hoặc cần nâng cấp

| Kịch bản | Chi phí/tháng ước tính | Trong ngân sách? |
|----------|-----------------------:|:----------------:|
| Hiện tại (1GB RAM) | 81.600đ | ✔ |
| VPS tăng giá thêm 20% | 91.600đ | ✔ |
| Nâng lên 2GB RAM (giá thị trường tham khảo 80.000–150.000đ) | 111.600đ – 181.600đ | ✘ |

Nâng cấp VPS **vượt ngân sách**. Kiến trúc đã được tính để 1GB đủ dùng; chỉ cân
nhắc nâng khi giám sát cho thấy thường xuyên dùng swap nhiều (§10).

---

## 7. Khi vượt giới hạn miễn phí

Gói trả phí của các dịch vụ quốc tế đều **vượt xa ngân sách** (tỷ giá tham khảo
~26.000đ/USD). Chiến lược là **ở lại trong free tier**, và có sẵn phương án thay thế
thay vì nâng cấp.

| Dịch vụ | Dấu hiệu sắp vượt | Gói trả phí thấp nhất (tham khảo) | Phương án thay thế trong ngân sách |
|---------|-------------------|-----------------------------------|-----------------------------------|
| **Cloudinary** | Dashboard Usage > 70% giữa tháng | ~90–100 USD/tháng (≈ 2,3–2,6 triệu đồng) | (1) Rà ảnh quá lớn, giảm kích thước ảnh danh sách · (2) Chuyển ảnh sự kiện cũ sang lưu trên VPS, phục vụ qua Cloudflare cache (miễn phí, nhưng tốn ổ đĩa và phải tự backup) |
| **Resend** | Nhiều ngày chạm 100 email | ~20 USD/tháng (≈ 520.000đ) | (1) Tắt thông báo nội bộ, admin xem dashboard · (2) Gửi thông báo admin dạng tổng hợp cuối ngày · (3) Chuyển sang nhà cung cấp khác có free tier |
| **MongoDB Atlas** | Dung lượng > 400MB | Gói Flex, khoảng vài chục USD/tháng | Gần như không xảy ra (§4.3). Nếu có: rút ngắn TTL `activity_logs`, dọn bình luận spam |
| **VPS** | RAM thường xuyên > 85%, swap > 500MB | Nâng 2GB: vượt ngân sách (§6.3) | Giảm Gunicorn còn 1 worker × 8 thread; kiểm tra rò rỉ bộ nhớ |

> **Không** lách giới hạn bằng cách mở nhiều tài khoản miễn phí cho cùng một dịch
> vụ — vi phạm điều khoản sử dụng và có nguy cơ bị khoá toàn bộ tài khoản, mất dữ liệu.

---

## 8. Chi phí ẩn cần lưu ý

| Chi phí | Ghi chú |
|---------|---------|
| **Giá gia hạn VPS** | Nhiều nhà cung cấp giảm giá kỳ đầu. Hỏi rõ giá gia hạn trước khi thuê |
| **Thời gian của tình nguyện viên** | Không tốn tiền nhưng có thật: ~2–4 giờ/tháng cho đối soát quyên góp, kiểm duyệt nội dung, bảo trì hệ thống ([DEPLOYMENT §14](../technical/DEPLOYMENT.md#14-bảo-trì-định-kỳ)) |
| **Tài khoản ngân hàng nhận quyên góp** | Ngoài phạm vi website nhưng cần tính: phí duy trì tài khoản, phí SMS/OTT banking có thể 10.000–20.000đ/tháng tuỳ ngân hàng |
| **Thẻ thanh toán quốc tế** | Các gói miễn phí hiện dùng thường không đòi thẻ. Chỉ cần nếu phải nâng cấp dịch vụ quốc tế |
| **Mất domain do quên gia hạn** | Chi phí lấy lại domain đã hết hạn cao hơn nhiều lần giá gia hạn. Bật tự động gia hạn và đặt nhắc lịch trước 30 ngày |
| **Tỷ giá** | Chỉ ảnh hưởng nếu phải dùng dịch vụ tính bằng USD |

---

## 9. Tổng chi phí theo năm

| Năm | VPS | Domain | Dịch vụ khác | Tổng | Trung bình/tháng |
|-----|----:|-------:|-------------:|-----:|-----------------:|
| Năm 1 | 600.000đ | 379.000đ | 0đ | 979.000đ | 81.600đ |
| Năm 2 | 600.000đ | 379.000đ | 0đ | 979.000đ | 81.600đ |
| Năm 3 | 600.000đ | 379.000đ | 0đ | 979.000đ | 81.600đ |
| **3 năm** | | | | **2.937.000đ** | |

Giả định giá không đổi và ở lại trong free tier. Ngân sách 3 năm là 3.600.000đ,
còn dư ~663.000đ — đủ làm quỹ dự phòng cho VAT hoặc tăng giá.

---

## 10. Theo dõi chi phí hằng tháng

| Dịch vụ | Xem mức dùng ở đâu | Ngưỡng cảnh báo |
|---------|-------------------|:---------------:|
| Cloudinary | Dashboard → Usage | 70% credit trước ngày 20 |
| Resend | Dashboard → Emails / Usage | Chạm 80 email/ngày |
| MongoDB Atlas | Cluster → Metrics → Data Size | 400MB |
| VPS | `free -h`, `df -h` ([DEPLOYMENT §13.2](../technical/DEPLOYMENT.md#132-xem-log)) | RAM > 85%, ổ đĩa > 80% |
| Cloudflare | Analytics → Traffic | — (chỉ để hiểu lưu lượng) |

**Checklist hằng tháng** (gộp vào lịch bảo trì của
[DEPLOYMENT §14](../technical/DEPLOYMENT.md#14-bảo-trì-định-kỳ)):

- [ ] Ghi lại mức dùng Cloudinary, Resend, Atlas vào bảng theo dõi của nhóm
- [ ] So sánh với tháng trước — tăng đột biến thì tìm nguyên nhân
- [ ] Đối chiếu hoá đơn VPS/domain với bảng §2

**Hằng quý:** kiểm tra lại giới hạn free tier và giá trên trang chính thức của từng
dịch vụ; cập nhật tài liệu này nếu có thay đổi.

**Mốc gia hạn:**

| Hạng mục | Nhà cung cấp | Ngày gia hạn | Người phụ trách |
|----------|--------------|--------------|-----------------|
| VPS | _điền khi thuê_ | _…_ | _…_ |
| Domain | _điền khi mua_ | _…_ | _…_ |

---

## 11. Minh bạch chi phí vận hành

Website cam kết minh bạch tài chính với người quyên góp. Nếu chi phí vận hành
website được trả **từ tiền quyên góp**:

- Ghi nhận khoản chi này riêng, không lẫn với chi phí cho các hoạt động của các bé.
- Công khai trên trang Minh bạch (`/transparency`, nội dung lấy từ
  `site_settings.pages.transparency_commitment`): số tiền, hạng mục, kỳ thanh toán.
- Cập nhật khi chi phí thay đổi.

Nếu chi phí do thành viên nhóm tự đóng góp, ghi rõ điều đó trên cùng trang —
người quyên góp thường muốn biết 100% số tiền của họ đến được với các bé.
