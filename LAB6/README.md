# LAB6 – Xác thực Active Directory và Chính sách truy cập trên pfSense

**Họ tên:** Lưu Gia Luân
**MSSV:** 1150080064
**Lớp:** 11_ĐH_CNPM1
**Tên lab:** LAB6 – pfSense AD Authentication & Access Policies

## Phiên bản môi trường

- Oracle VirtualBox 7.2.16r174877
- pfSense CE 2.7.2-RELEASE (amd64), build 20231206-2010
- Windows Server 2022 Standard Evaluation (Desktop Experience) – DC01
- Windows 11 25H2 – Client-A, Client-B 
- Ubuntu Server (dự kiến) – DMZ-WEB

## Mô hình mạng

Internet ── LAB6-WAN (NAT Network, 192.168.250.0/24)
│
[pfSense-LAB6]
┌───────┼───────┬────────────┐
MGMT LAN DMZ
(Host-only, (Internal, (Internal,
192.168.56.0/24) lab6-lan, lab6-dmz,
10.6.0.0/24) 172.16.6.0/24)


| Interface | IP | Mạng VirtualBox |
|---|---|---|
| WAN | DHCP (192.168.250.x) | NAT Network `LAB6-WAN` |
| MGMT | 192.168.56.254/24 | Host-only `VirtualBox Host-Only Ethernet Adapter` |
| LAN | 10.6.0.1/24 | Internal Network `lab6-lan` |
| DMZ | 172.16.6.1/24 | Internal Network `lab6-dmz` |

| Máy | IP | Vai trò |
|---|---|---|
| DC01 | 10.6.0.10/24 | AD DS + DNS, forest `lab6.test` |
| Client-A | 10.6.0.21/24 | Đại diện tài khoản `sv_a` |
| Client-B | 10.6.0.22/24 | Đại diện tài khoản `sv_b` |
| DMZ-WEB | 172.16.6.10/.20/.30/.40 | Nginx mô phỏng GENERAL/JOB/SOCIAL/RESTRICTED site |
| Máy thật | 192.168.56.1/24 | Quản trị WebGUI qua MGMT |

## Cách dựng môi trường

1. Tạo mạng ảo riêng cho LAB6, tách biệt hoàn toàn với LAB5: NAT Network `LAB6-WAN` (192.168.250.0/24), Internal Network `lab6-lan` và `lab6-dmz`; dùng chung Host-only Network có sẵn cho MGMT nhưng gán thêm IP phụ `192.168.56.1/24` lên card đó (không ảnh hưởng IP `10.0.0.100` của LAB5).
2. Tạo VM `pfSense-LAB6` (FreeBSD 64-bit, 2 GB RAM, 2 vCPU, ổ 16 GB) với 4 card mạng: Adapter 1 NAT Network `LAB6-WAN` (WAN), Adapter 2 Host-only (MGMT), Adapter 3 Internal `lab6-lan` (LAN), Adapter 4 Internal `lab6-dmz` (DMZ). Cài đặt (Auto ZFS, stripe).
3. Gán 4 interface qua console (`assign interfaces`): WAN=em0, LAN=em2, OPT1=em1 (MGMT), OPT2=em3 (DMZ); đặt IP từng interface qua menu console.
4. Chạy Setup Wizard qua WebGUI (`https://192.168.56.254`), đổi Description OPT1→MGMT, OPT2→DMZ.
5. Dựng DC01 (Windows Server 2022), đặt IP `10.6.0.10/24`, gateway `10.6.0.1`, cài AD DS, tạo forest `lab6.test`, thêm DNS Forwarder `8.8.8.8`, tạo OU `LAB6-Users` với hai user `sv_a`, `sv_b`, và service account `svc_pfsense_ldap` (không cấp quyền Domain Admin) để dùng làm LDAP bind.
6. Dựng Client-A, Client-B (Windows 11), gắn Internal Network `lab6-lan`, IP tĩnh `10.6.0.21`/`10.6.0.22`, DNS trỏ về DC01.
7. Dựng DMZ-WEB (Ubuntu + Nginx, gắn thêm 3 IP phụ .20/.30/.40 mô phỏng nhóm website).
8. Tích hợp LDAP vào pfSense (System → User Manager → Authentication Servers), kiểm tra qua Diagnostics → Authentication.
9. Cấu hình Captive Portal trên LAN, xác thực qua LDAP.
10. Tạo Alias (CLIENT_A, CLIENT_B, POLICY_CLIENTS, 4 site, WEB_PORTS) và Schedule (LUNCH_WINDOW, WORK_HOURS, TEST_NOW).
11. Thực hành 7 kịch bản chính sách firewall theo tài liệu.

## Các tình huống đã thực hiện

| Tình huống | Kết quả | Ghi chú |
|---|---|---|
| B.1–B.3 Dựng pfSense-LAB6 | Hoàn tất | 4 interface đúng IP, rule Pass MGMT đã tạo |
| B.4 Dựng DC01 | Hoàn tất | Forest `lab6.test`, OU `LAB6-Users`, đủ 3 tài khoản |
| B.5 Client-A | Đang thực hiện | Đang cài Windows 11 |
| B.5 Client-B | Chưa thực hiện | |
| B.6 DMZ-WEB | Chưa thực hiện | |
| C – Tích hợp LDAP | Chưa thực hiện | Đã có Bind DN: `CN=svc_pfsense_ldap,CN=Users,DC=lab6,DC=test` |
| D – Captive Portal | Chưa thực hiện | |
| E – Alias | Chưa thực hiện | |
| F – Schedule | Chưa thực hiện | |
| G.1–G.7 – 7 kịch bản | Chưa thực hiện | |

## Lỗi gặp phải và cách khắc phục

- **Không vào được WebGUI qua MGMT (`https://192.168.56.254`) dù interface đã đúng IP** → kiểm tra `ifconfig em1` trong pfSense xác nhận interface UP/active bình thường; dùng `pfctl -d` tạm tắt firewall để xác minh nguyên nhân là **OPT1/MGMT mặc định không có rule Pass nào** (khác LAN vốn có sẵn Anti-Lockout Rule); khắc phục bằng cách tạo rule Pass thủ công trên tab OPT1 (Source = OPT1 subnets, Destination = Any).
- **Card Host-only thứ hai (tạo riêng cho MGMT) không thông mạng dù cấu hình đúng** → sau nhiều bước loại trừ (kiểm tra route, ARP, firewall Windows, driver binding) vẫn không xác định được nguyên nhân gốc; chuyển hướng dùng chung card Host-only đã có của LAB5, chỉ thêm một IP phụ `192.168.56.1/24` lên cùng card đó — giải quyết được vấn đề mà không ảnh hưởng tới LAB5.
- **Lỗi `unknown file system` / `Cannot find the pool label for 'gpt'` khi boot pfSense-LAB6 lần đầu** → do tháo ISO cài đặt quá sớm (ngay lúc chọn Reboot) khiến quá trình ghi ZFS chưa hoàn tất; khắc phục bằng cách cài lại từ đầu, lần này để máy tự chạy sau khi Reboot và chỉ tháo ISO khi xác nhận đã boot đúng vào hệ thống hoặc quay lại trình cài.
- **Máy Client-A (Windows 11) đứng màn hình đen, `screenshotpng` báo `Unsupported resolution 0x0`** → đọc `VBox.log` thấy firmware EFI dừng ở `DXE_AP`; khắc phục bằng cách giảm từ 2 vCPU xuống 1 vCPU (nghi do xung đột với kiến trúc CPU lai P-core/E-core của máy chủ), áp dụng cách đã từng hiệu quả với máy DC01 ở LAB5.
- **Trình cài Windows 11 báo "failed to boot" do bỏ lỡ dòng "Press any key to boot from CD or DVD"** → khắc phục bằng cách Reset VM và bấm phím ngay khi dòng chữ xuất hiện (chỉ chờ vài giây).
- **Wizard pfSense bị `ERR_CONNECTION_TIMED_OUT` ở bước Reload cuối cùng** → do bước Reload tự động bật lại firewall trong khi rule MGMT chưa được tạo, lặp lại đúng vấn đề mất kết nối MGMT; khắc phục bằng `pfctl -d` tạm thời rồi tạo rule chính thức ngay sau khi vào được WebGUI.
