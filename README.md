# Thực hành: WAF trên Nginx với ModSecurity và OWASP CRS

**Họ tên sinh viên:** Nguyễn Gia Vĩ  
**Mã số sinh viên:** 24021673  
**Môn học:** Kiến trúc phần mềm – Thiết kế cho bảo mật

## 1. Mục tiêu

Bài thực hành dựng một **Web Application Firewall (WAF)** bằng Nginx, ModSecurity và bộ luật OWASP Core Rule Set (CRS) ở phía trước ứng dụng OWASP Juice Shop. Sau khi chạy hệ thống, thực hiện các phép thử trong môi trường cục bộ để kiểm tra khả năng phát hiện và chặn request có dấu hiệu SQL Injection, đồng thời xác nhận request đăng nhập hợp lệ vẫn có thể đi qua WAF.

Các mục tiêu chính:

- Khởi chạy Juice Shop và WAF bằng Docker Compose.
- So sánh cùng một payload khi truy cập trực tiếp vào ứng dụng và khi đi qua WAF.
- Quan sát phản hồi `403 Forbidden` khi request bị chặn.
- Kiểm tra log của ModSecurity/OWASP CRS để xác định luật phát hiện và luật đánh giá chặn request.

> **Phạm vi kiểm thử:** Tất cả thử nghiệm trong README này được thực hiện với OWASP Juice Shop chạy cục bộ trên máy. Không sử dụng payload kiểm thử trên website hoặc hệ thống không thuộc quyền kiểm thử của bạn.

## 2. Mô hình hệ thống

Hệ thống gồm hai service được khai báo trong `docker-compose.yml`:

```text
                     Truy cập trực tiếp (không qua WAF)
Browser  ----------------------------------------------> Juice Shop
          http://localhost:3000                          web:3000

                     Truy cập qua WAF
Browser  ---> Nginx + ModSecurity + OWASP CRS  ---------> Juice Shop
          http://localhost:8080             BACKEND     web:3000
```

| Thành phần | Vai trò | Cổng trên máy host |
|---|---|---:|
| `web` | Ứng dụng OWASP Juice Shop dùng để kiểm thử | `3000` |
| `waf` | Nginx tích hợp ModSecurity và OWASP CRS, kiểm tra request trước khi chuyển tiếp | `8080` |

Hai địa chỉ:

- `http://localhost:3000`: truy cập **trực tiếp** vào Juice Shop, không đi qua WAF.
- `http://localhost:8080`: truy cập Juice Shop **thông qua WAF**.

## 3. Cấu hình Docker Compose

Nội dung `docker-compose.yml`:

```yaml
services:
  web:
    image: bkimminich/juice-shop
    ports:
      - "3000:3000"

  waf:
    image: owasp/modsecurity-crs:nginx
    ports:
      - "8080:8080"
    environment:
      BACKEND: "http://web:3000"
      MODSEC_RULE_ENGINE: "On"
      BLOCKING_PARANOIA: "1"
```

## 4. Khởi chạy hệ thống

### Bước 1: Kiểm tra Docker

Mở Docker Desktop và chờ Docker Engine khởi động. Mở PowerShell tại thư mục project, sau đó chạy:

```powershell
docker --version
docker compose version
docker info
```

Các lệnh trên lần lượt kiểm tra Docker, Docker Compose và khả năng kết nối đến Docker Engine.

### Bước 2: Kiểm tra cấu hình và khởi chạy container

Tại thư mục chứa `docker-compose.yml`, chạy:

```powershell
docker compose config
docker compose up -d
docker compose ps
```

- `docker compose config`: kiểm tra cấu hình Compose.
- `docker compose up -d`: tải image nếu cần và khởi chạy các container ở chế độ nền.
- `docker compose ps`: kiểm tra trạng thái các service.

Ảnh minh chứng khởi chạy hệ thống:

![Khởi chạy Docker Compose và kiểm tra các service](imageSteps/1%20khoi%20chay.png)

## 5. Thử nghiệm 1 - Tấn công khi không có WAF

### Bước 1: Truy cập trực tiếp vào Juice Shop

Mở trình duyệt và truy cập `http://localhost:3000`, sau đó mở trang **Login** và nhập tài khoản email như bên dưới, mật khẩu bất kì

```text
' OR 1=1--
```

Ảnh minh chứng nhập payload khi truy cập cổng 3000:

![Nhập payload SQL Injection khi truy cập trực tiếp, không qua WAF](imageSteps/2.1%20tan%20cong%20khi%20khong%20co%20waf%20%281%29.png)

### Bước 2: Quan sát kết quả

Trong lần thử này, thao tác đăng nhập trực tiếp đã đưa người dùng vào giao diện tài khoản admin, cho thấy nguy cơ khi ứng dụng có lỗ hổng SQL Injection và request không đi qua lớp WAF.

![Kết quả đăng nhập sau phép thử trực tiếp không qua WAF](imageSteps/2.2%20tan%20cong%20khi%20khong%20co%20waf%20%282%29.png)

## 6. Thử nghiệm 2 - Gửi cùng payload qua WAF

### Bước 1: Truy cập qua cổng WAF

Mở `http://localhost:8080`, vào trang **Login**, rồi nhập lại cùng payload ở ô Email và một giá trị bất kỳ ở ô Password.

![Nhập cùng payload khi truy cập qua WAF ở cổng 8080](imageSteps/3.1%20tan%20cong%20khi%20co%20waf%20%281%29.png)

### Bước 2: Quan sát kết quả bị chặn

Trong ảnh minh chứng, phản hồi `403 Forbidden` từ Nginx xuất hiện tại khu vực đăng nhập. Đây là dấu hiệu request đã bị WAF chặn thay vì được chuyển tiếp và xử lý như một request đăng nhập bình thường.

![Phản hồi 403 Forbidden khi gửi payload qua WAF](imageSteps/3.2%20tan%20cong%20khi%20co%20waf%20%282%29.png)

## 7. Thử nghiệm 3 - Đăng nhập hợp lệ qua WAF

WAF không có mục tiêu chặn mọi request mà cần cho phép các request hợp lệ tiếp tục đến ứng dụng. Để kiểm tra điều này, vẫn sử dụng địa chỉ `http://localhost:8080`, nhưng nhập thông tin đăng nhập hợp lệ:

Ảnh minh chứng nhập thông tin hợp lệ qua WAF:

![Nhập thông tin đăng nhập hợp lệ qua WAF](imageSteps/3.3%20dang%20nhap%20tai%20khoan%20dung%20%28co%20waf%29.png)

Kết quả ghi lại cho thấy đăng nhập thành công và ứng dụng có thể được sử dụng bình thường qua cổng 8080.

![Giao diện Juice Shop sau khi đăng nhập thành công qua WAF](imageSteps/3.4%20dang%20nhap%20thanh%20cong%20%28co%20waf%29.png)

## 8. Kiểm tra log của WAF và kết quả thực nghiệm

| Trường hợp | Địa chỉ truy cập | Kết quả quan sát |
|---|---|---|
| SQL Injection, không qua WAF | `http://localhost:3000` | Trong lần thử được ghi lại, đăng nhập thành công và hiển thị giao diện admin. |
| SQL Injection, qua WAF | `http://localhost:8080` | Request bị chặn; log ghi nhận luật `942100`, luật đánh giá `949110` và HTTP `403`. |
| Đăng nhập hợp lệ, qua WAF | `http://localhost:8080` | Đăng nhập thành công và sử dụng giao diện ứng dụng bình thường. |

### Kết luận

Thử nghiệm minh họa vai trò của WAF như một lớp bảo vệ đứng trước ứng dụng web. Khi truy cập trực tiếp vào Juice Shop, payload SQL Injection trong phép thử đã khai thác được hành vi dễ bị tổn thương của ứng dụng. Khi gửi cùng payload qua cổng WAF, ModSecurity/OWASP CRS phát hiện request bằng luật `942100`; luật `949110` đánh giá tổng điểm vi phạm đạt ngưỡng và request nhận phản hồi `403 Forbidden`. Một lần đăng nhập hợp lệ qua WAF cũng được ghi lại để kiểm tra rằng WAF không đơn thuần chặn mọi request.

Kết quả này phản ánh hành vi trong môi trường lab và không có nghĩa WAF có thể phát hiện hoặc ngăn chặn tất cả kiểu tấn công. Cấu hình và luật WAF cần được kiểm thử, giám sát và điều chỉnh phù hợp khi sử dụng trong hệ thống thực tế.

## 9. Cấu trúc thư mục

```text
nginx-waf-lab/
├── docker-compose.yml
├── README.md
└── imageSteps/
    ├── 1 khoi chay.png
    ├── 2.1 tan cong khi khong co waf (1).png
    ├── 2.2 tan cong khi khong co waf (2).png
    ├── 3.1 tan cong khi co waf (1).png
    ├── 3.2 tan cong khi co waf (2).png
    ├── 3.3 dang nhap tai khoan dung (co waf).png
    └── 3.4 dang nhap thanh cong (co waf).png
```
