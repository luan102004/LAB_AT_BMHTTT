LAB 4 - Khảo sát và đánh giá bề mặt mạng bằng Nmap
Họ tên: Lưu Gia Luân
Lớp: 11_ĐH_CNPM1
MSSV: 1150080064
Môn: An toàn hệ thống thông tin
Lab: LAB 4 - Nmap
Môi trường
Thành phần	Phiên bản
Máy thật	Windows 11
Ảo hóa	Oracle VirtualBox [ĐIỀN phiên bản]
Máy quét	Kali Linux 2026.2, Nmap 7.99
Máy đích	Metasploitable 2
Mạng	VirtualBox Host-Only 192.168.56.0/24
Máy	IP
Windows host	192.168.56.1
Kali	192.168.56.101
Metasploitable 2	192.168.56.102
Cách dựng môi trường
VirtualBox: dùng mạng Host-Only 192.168.56.0/24, bật DHCP.
Kali: Adapter 1 = Host-only, tạo snapshot Before-LAB4.
Metasploitable 2: gắn Metasploitable.vmdk vào Controller IDE, Adapter 1 = Host-only (không Bridged), đăng nhập msfadmin/msfadmin.
Kiểm tra: ping -c 4 192.168.56.102 từ Kali.
Tình huống và kết quả

Đích quét: 192.168.56.102.

#	Tình huống	Lệnh	Kết quả	PASS/FAIL
1	Ping	ping -c 4 192.168.56.102	4/4 gói, 0% loss	PASS
2	Host discovery	sudo nmap -sn 192.168.56.0/24	4 host up (.1, .100, .101, .102)	PASS
3	TCP Connect	nmap -sT	23 open, 977 closed	PASS
4	SYN	sudo nmap -sS	23 open, 977 closed	PASS
5	FIN	sudo nmap -sF	23 open|filtered, 977 closed	PASS
6	Xmas	sudo nmap -sX	23 open|filtered, 977 closed	PASS
7	NULL	sudo nmap -sN	Chưa có ảnh	CHƯA
8	ACK	sudo nmap -sA	Chưa có ảnh	CHƯA
9	UDP top 20	sudo nmap -sU --top-ports 20	2 open, 3 open|filtered, 15 closed	PASS
10	Version	sudo nmap -sV	vsftpd 2.3.4, OpenSSH 4.7p1, Apache 2.2.8, Samba 3.x, MySQL 5.0.51a	PASS
11	OS	sudo nmap -O	Linux 2.6.9 - 2.6.33	PASS
12	Aggressive	sudo nmap -A	Version + OS + traceroute + scripts	PASS
13	NSE smb-os-discovery	--script smb-os-discovery -p 445	Unix (Samba 3.0.20-Debian)	PASS
14	NSE MS17-010	--script smb-vuln-ms17-010 -p 445	Không báo VULNERABLE, kết luận: chưa xác định	PASS
15	Xuất kết quả	-oN, -oX, -oG, xsltproc	Có ket_qua.txt, ket_qua.xml, smb.txt, bao_cao.html	PASS
16	Trước/sau hardening	-sV trước và sau	Chưa thực hiện	CHƯA
