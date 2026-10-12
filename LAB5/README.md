# LAB5 – pfSense Firewall & DMZ

**Họ tên:** Lưu Gia Luân
**MSSV:** 1150080064
**Lớp:** 11_ĐH_CNPM1
**Tên lab:** LAB5 – pfSense Firewall & DMZ

## Phiên bản môi trường

- Oracle VirtualBox 7.2.16r174877
- pfSense CE 2.7.2-RELEASE (amd64), build 20231206-2010
- Windows Server 2022 Standard Evaluation (Domain Controller)
- Windows Server 2022 (DMZ-Web, IIS)
- Ubuntu Server (LAN-Test) 

## Mô hình mạng

Internet ── WAN (Bridged, DHCP) ── [pfSense] ── LAN 10.0.0.0/8 (Host-only)
└────── DMZ 172.16.0.0/16 (Internal Network “dmz-net”)


| Thiết bị | Interface | IP | Gateway | DNS |
|---|---|---|---|---|
| pfSense | WAN | DHCP | – | – |
| pfSense | LAN | 10.0.0.1/8 | – | – |
| pfSense | DMZ (OPT1) | 172.16.0.1/16 | – | – |
| Domain Controller (DC) | LAN | 10.0.0.2/8 | 10.0.0.1 | 10.0.0.2 |
| DMZ-Web | DMZ | 172.16.0.2/16 | 172.16.0.1 | 8.8.8.8 |
| LAN-Test | LAN | 10.0.0.3/8 | 10.0.0.1 | – |
| Máy thật (quản trị) | Host-only (VirtualBox) | 10.0.0.100/8 | *(để trống)* | *(để trống)* |

## Cách dựng môi trường

1. Tải file cài `pfSense-CE-2.7.2-RELEASE-amd64.iso.gz`, kiểm tra SHA-256 (khớp `883fb7bc...cd9e4`), giải nén bằng 7-Zip ra file `.iso`.
2. Cấu hình mạng VirtualBox: Host-only Network (`VirtualBox Host-Only Ethernet Adapter`) đặt `10.0.0.100/8`, tắt DHCP Server; tắt Cloudflare WARP trên máy thật để tránh xung đột dải `172.16.0.0/16`.
3. Tạo VM `pfSense` (BSD, FreeBSD 64-bit, 2 GB RAM, 2 vCPU, ổ 20 GB) với 3 card mạng: Adapter 1 Bridged (WAN), Adapter 2 Host-only (LAN), Adapter 3 Internal Network `dmz-net` (DMZ). Cài đặt (Auto ZFS, stripe), tháo ISO sau khi cài.
4. Gán card mạng trên console (`em0`=WAN, `em1`=LAN), đặt IP LAN `10.0.0.1/8` qua menu console.
5. Vào WebGUI `https://10.0.0.1` từ máy thật, chạy Setup Wizard, đổi mật khẩu admin.
6. Gán interface DMZ (OPT1 = `em2`), đặt `172.16.0.1/16`.
7. Kiểm tra Outbound NAT (Automatic), xác nhận Automatic Rules có `10.0.0.0/8` và `172.16.0.0/16`.
8. Chuẩn hóa rule LAN: Disable hai rule "Default allow LAN to any" (IPv4/IPv6), giữ Anti-Lockout Rule, tạo rule `Pass LAN net → Any`, Reset States sau mỗi thay đổi.
9. Dựng Domain Controller: tạo VM Windows Server (ban đầu thử chuyển từ máy VMware có sẵn qua `VBoxManage clonemedium`, gặp lỗi EFI kẹt ở `DXE_AP` khi 2 vCPU và quên mật khẩu Administrator → chuyển hướng cài mới từ ISO Windows Server 2022 Evaluation), đặt IP `10.0.0.2/8`, gateway `10.0.0.1`, DNS `10.0.0.2`, cài AD DS, tạo forest `vietnam.local`, thêm DNS Forwarder `8.8.8.8`.
10. Dựng DMZ-Web (Windows Server, không join domain, IP `172.16.0.2/16`, cài IIS) và LAN-Test (Ubuntu, IP `10.0.0.3/8`) khi cần cho từng tình huống.

## Các tình huống đã thực hiện

| Tình huống | Kết quả | Ghi chú |
|---|---|---|
| B9 – Rule LAN cơ bản (Pass LAN net → Any) | Khi rule đang bật: ping 8.8.8.8, Resolve-DnsName example.com, curl.exe -4 https://example.com từ DC đều thành công. Sau khi Disable rule và Reset States: ping 8.8.8.8 mới thất bại (timeout) | Kiểm thử ping/DNS/HTTPS từ DC trước và sau khi Disable rule |
| TH1 – Chặn ICMP, cho phép DNS/Web | ping 8.8.8.8 từ DC thất bại; Resolve-DnsName example.com -Server 8.8.8.8 và curl.exe -4 https://example.com thành công | Rule Block ICMP phải nằm trên 2 rule Pass DNS/Web |
| TH2 – Chỉ một host ra Internet (10.0.0.2) | 10.0.0.2 (DC) ping 8.8.8.8 thành công; 10.0.0.3 (LAN-Test) ping 8.8.8.8 thất bại | Cần VM LAN-Test (10.0.0.3) để đối chứng |
| TH3 – Cô lập DMZ khỏi LAN | Baseline (chỉ có Pass DMZ net → Any): DMZ-Web ping 10.0.0.2 thành công (có reply). Sau khi thêm Block DMZ net → LAN net (đặt trên rule Pass): DMZ-Web ping 10.0.0.2 thất bại, nhưng ping 8.8.8.8 từ DMZ-Web vẫn thành công | So sánh baseline (Pass DMZ→Any) và sau khi thêm Block DMZ→LAN |
| TH4 – Port Forward WAN → DMZ (IIS) | Từ máy thật (hoặc WAN-Client), truy cập http://<IP-WAN-pfSense>:8080 hiển thị đúng trang mặc định của IIS chạy trên DMZ-Web (172.16.0.2:80) | Bỏ tick Block private/bogon networks trên WAN |
| TH5 – Logging firewall rule | Sau khi tạo lưu lượng bị chặn (ví dụ ping khi đang Block ICMP ở TH1), vào Status → System Logs → Firewall thấy dòng log ghi rõ thời điểm, IP nguồn/đích, và tên rule đã chặn | Bật Log trên rule Block, kiểm tra Status → System Logs → Firewall |

## Lỗi gặp phải và cách khắc phục

- **File zip đã tải chỉ chứa README/hướng dẫn, không có ISO cài đặt** → tự tải pfSense CE 2.7.2 theo link chính thức của Netgate, kiểm tra SHA-256 trước khi dùng.
- **IP máy thật trùng dải DMZ (172.16.0.2) do Cloudflare WARP đang bật** → tắt WARP trong suốt quá trình làm lab để tránh định tuyến sai khi kiểm thử từ máy thật.
- **`pfSense installer` báo lỗi `vm_fault: pager read error` sau khi cài** → do tháo ISO trước khi Reboot thay vì sau; khắc phục bằng cách kiểm tra lại Boot Order (Hard Disk trước Optical) và đảm bảo ổ quang trống trước khi khởi động lại.
- **Chuyển máy ảo Windows Server từ VMware sang VirtualBox (`VBoxManage clonemedium`) bị màn hình đen, `screenshotpng` báo lỗi `Unsupported resolution 0x0`** → đọc `VBox.log`, phát hiện firmware EFI kẹt ở giai đoạn `PciHostBridgeDxe` / `DXE_AP`; thử đổi chipset (`piix3` → `ich9`), đổi controller ổ đĩa (NVMe → SATA) không cải thiện; cuối cùng giảm từ 2 vCPU xuống 1 vCPU thì máy boot được (nghi do xung đột với kiến trúc CPU lai P-core/E-core của máy chủ).
- **Quên mật khẩu Administrator của máy Windows Server lấy từ VM VMware có sẵn, máy lại là Domain Controller của domain `COMPANY` khác** → quyết định không reset mật khẩu offline (phức tạp với Domain Controller), chuyển sang cài mới Domain Controller từ ISO Windows Server 2022 Evaluation tải trực tiếp từ Microsoft Evaluation Center.
- **Chuột không đồng bộ giữa máy ảo và máy thật khi chia đôi màn hình** → do dùng chuột kiểu PS/2 (tương đối); khắc phục bằng cách bật USB xHCI và đổi `--mouse usbtablet` để chuột dùng tọa độ tuyệt đối.
- **Rule firewall "Default allow LAN to any" tưởng đã tắt nhưng vẫn còn dấu tích xanh** → phân biệt giữa thao tác "tick chọn dòng" (chỉ để chọn) và "Toggle" (đổi trạng thái bật/tắt thật sự); xác nhận lại bằng trạng thái mờ màu và biểu tượng Actions đổi thành ô tích.
