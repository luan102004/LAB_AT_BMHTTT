# LAB7 – Xây dựng chính sách Web Protection trên mô hình pfSense + Web Proxy
 
- **Họ tên:** Lưu Gia Luân
- **MSSV:** 1150080064
- **Lớp:** CNPM1
- **Tên lab:** LAB7 – pfSense Web Protection (pfSense + Squid + E2Guardian + ClamAV)
## Phiên bản môi trường
 
| Thành phần | Phiên bản / cấu hình |
|---|---|
| Ảo hóa | Oracle VirtualBox (máy thật Windows) |
| pfSense-LAB6 | pfSense CE – LAN 10.6.0.1/24, DMZ 172.16.6.1/24, MGMT 192.168.56.254 |
| WEB-PROXY | Ubuntu Server 24.04 LTS – 10.6.0.30/24, Squid 6.14 (127.0.0.1:3128), E2Guardian 5.3.5 (10.6.0.30:8080), ClamAV |
| DMZ-WEB | Ubuntu 24.04 + nginx – 172.16.6.10 (và 172.16.6.11 cho entertainment/games), 13 vhost `*.lab7.test` |
| DC01 | Windows Server AD/DNS – 10.6.0.10, zone `lab7.test` |
| Client-A | Windows 10 1809 Pro – 10.6.0.21, proxy 10.6.0.30:8080 |
 
## Cách dựng môi trường
 
1. Dùng lại topology LAB 6 (pfSense, DC01, Client-A).
2. Tạo VM WEB-PROXY (Ubuntu 24.04), đặt IP tĩnh 10.6.0.30/24, GW 10.6.0.1, DNS 10.6.0.10; cài Squid, E2Guardian, ClamAV.
3. Clone WEB-PROXY thành DMZ-WEB, đổi IP 172.16.6.10/24 (+172.16.6.11), cài nginx, tạo 13 vhost và file test (`eicar.com.txt`, `10MB.bin`, `100MB.bin`, `110MB.bin`).
4. Tạo zone `lab7.test` trên DC01 (A record trỏ 172.16.6.10; entertainment/games trỏ 172.16.6.11).
5. Cấu hình pfSense: alias (`LAB7_CLIENTS_LUUGIALUAN`, `LAB7_PROXY_`, `WEB_PORTS`, `MAIL_PORTS`), schedule `LAB7_ENT_60M`, ruleset LAN ép client đi qua proxy, chặn direct 80/443, tắt rule "Default allow", thêm explicit deny.
6. Cấu hình E2Guardian: `bypass = 300`, `reportinglevel = 3`, bật clamd (`contentscanexceptions = on`), danh sách block/warning/exception, template trang chặn LAB 7. Squid: `reply_body_max_size 100 MB`.
7. Client-A: bật proxy 10.6.0.30:8080 và kiểm thử từng kịch bản.
## Các tình huống đã thực hiện
 
| Kịch bản | Kết quả | Ghi chú |
|---|---|---|
| 1 – Baseline (normal.lab7.test) | Đạt | Qua proxy vào được; tắt proxy bị pfSense chặn TCP/80 |
| 2 – Geo-Sim (china, mideast) | Đạt | Hard block, không có nút bypass |
| 3 – Job Search (jobs) | Đạt | Hard block |
| 4 – Warning (weapons, politics, religion) | Đạt | Trang cảnh báo, Acknowledge mở tạm 5 phút (`bypass = 300`) |
| 5 – Education | Đạt | Nằm trong `exceptionsitelist`, không cảnh báo |
| 6 – Entertainment/Game | Đạt | Schedule 60 phút (07:59–08:59 ngày 27/10): trong khung vào được, ngoài khung bị chặn; là time-window policy, không phải quota |
| 7 – Streaming | Đạt | Cảnh báo + bypass tạm |
| 8 – Specific game (blockedgame) | Đạt | Hard block |
| 9 – Antivirus (EICAR) | Đạt | ClamAV chặn: `Eicar-Test-Signature` |
| 10 – Size (100MB / 110MB) | Đạt | 100MB → HTTP 200; 110MB → HTTP 403 |
| HTTPS inspection (mở rộng) | Không thực hiện | Phần tùy chọn |
 
## Lỗi gặp phải và cách khắc phục
 
| Lỗi | Nguyên nhân | Cách khắc phục |
|---|---|---|
| E2Guardian không khởi động ("one port per auth plugin") | Lệnh `sed` bỏ comment nhầm các dòng ví dụ | Khôi phục từ file `.bak`, dùng hàm `setkv` sửa đúng khóa cần đổi |
| EICAR không bị chặn / báo ERROR khi quét | clamd không đọc được file tạm; site exception không bị quét | `usermod -aG e2guardian clamav`, bật `contentscanexceptions = on` |
| Hard block vẫn bypass được | `site.story` có `checknobypasslists` ghi đè, danh sách non-bypass thiếu | Comment dòng đó, thêm domain hard block vào `domainsnobypass` |
| Trang hard block có nút Acknowledge không dùng được | Template luôn hiện nút | CSS ẩn nút khi `href` rỗng |
| `reply_body_max_size` lỗi cú pháp | Dùng cú pháp cũ `104857600 allow all` | Dùng `reply_body_max_size 100 MB` |
| Schedule không áp được lên entertainment/games | Client chỉ nói chuyện với proxy nên pfSense không thấy hostname | Tách site sang IP 172.16.6.11, dùng rule Pass có schedule + rule Reject ngoài khung |
| Rule 2 không hoạt động; rule 5 không có schedule | Dest port để `*`; schedule bỏ trống | Sửa port 8080, gán schedule `LAB7_ENT_60M` |
| Schedule sai ngày | Chọn nhầm ngày 14/24 | Sửa lại đúng ngày 27/10, 07:59–08:59 |
| VM Linux treo khi boot (smpboot) | 2 vCPU trên máy này | `--cpus 1 --ioapic on` |
| Không SSH/scp được vào VM | Chỉ có card Internal Network | Thêm card NAT + port forward 2222/2223, tắt route/DNS của card NAT bằng netplan |
| Quên mật khẩu admin pfSense | – | Console menu mục 3 để reset |
| Windows 11 không cài được | Yêu cầu tối thiểu 2 core | Dùng Windows 10 1809 Pro cho Client-A |
 
