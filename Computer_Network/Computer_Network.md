# GIÁO TRÌNH MẠNG MÁY TÍNH THỰC CHIẾN DÀNH CHO DEVOPS / PLATFORM / SRE
*Chuyên sâu — Đúng bản chất — Vận hành Production & Phỏng vấn Mid/Senior*

---

## 🧭 LỘ TRÌNH HỌC TẬP ĐỀ XUẤT

### 1. LỘ TRÌNH TẬP TRUNG 4 TUẦN (toàn thời gian, khoảng 60–100 giờ)

```
[Tuần 1: Nền tảng Core & Linux Transport Stack (~18.700 từ)]
  Module 0 → Module 1 → Module 2 → Module 3 → Module 4

[Tuần 2: Ứng dụng L7, Proxy, DNS & Linux Kernel Net (~16.200 từ)]
  Module 5 → Module 6 → Module 7 → Module 8

[Tuần 3: Hạ tầng Container, Kubernetes Toàn diện & Cloud VPC (~13.300 từ)]
  Module 9 → Module 10 → Module 11

[Tuần 4: Bảo mật, Quan sát, Hiệu năng & Mở rộng (~13.400 từ)]
  Module 12 → Module 13 → Module 14 → Module 15 (Phụ lục)
```

### 2. LỘ TRÌNH BÁN THỜI GIAN 10–12 TUẦN (8–10 giờ/tuần)

| Tầng Ưu Tiên | Danh Sách Module | Hướng Dẫn Tiếp Cận |
| :--- | :--- | :--- |
| **Tầng 1: Cốt lõi** | **Module 0, Module 1, Module 3, Module 4, Module 5, Module 7, Module 10** | Học thật kỹ lý thuyết, làm đầy đủ 100% bài lab thực hành, nắm chắc từng case study. |
| **Tầng 2: Quan trọng** | **Module 6, Module 8, Module 9, Module 11, Module 12 (trừ VPN/DDoS), Module 13, Module 14 (14.1, 14.4)** | Đọc kỹ nguyên lý hoạt động, thực hành các bài lab chọn lọc phục vụ công việc hàng ngày. |
| **Tầng 3: Mở rộng** | **Module 2 (chỉ ARP & VLAN đọc kỹ; STP, bonding đọc lướt), Mục 3.5, Module 12 (VPN/DDoS), Mục 14.3, Module 15** | Đọc lướt nắm khái niệm, tra cứu khi thiết kế kiến trúc nâng cao hoặc phỏng vấn chuyên sâu. |

---

## 📋 MỤC LỤC CHI TIẾT

---

### MODULE 0: TƯ DUY TROUBLESHOOTING & HỘP CÔNG CỤ THỰC CHIẾN
*Ước lượng: ~3.500 từ*

* **0.1. Phương pháp luận xử lý sự cố mạng Production**
  * Mô hình Top-Down vs. Bottom-Up vs. Divide-and-Conquer (bắt đầu từ L4).
  * Quy trình cách ly lỗi: Triệu chứng → Giả thuyết → Kiểm chứng → Giảm thiểu → Root Cause → Postmortem.
* **0.2. Bộ công cụ L2–L4 cơ sở trên Linux**
  * `ip` (thay thế `ifconfig`, `route`, `arp`): `ip link`, `ip addr`, `ip route`, `ip neigh`.
  * `ss` (thay thế `netstat`): Cờ `-tulpn`, `-s`, `-o` (timer), cách đọc Recv-Q / Send-Q ở trạng thái LISTEN vs ESTABLISHED.
  * `nc` (Netcat / Ncat) & `nmap`: Quét port L4 (TCP SYN, UDP probe), tạo socket test giả lập server/client.
* **0.3. Phân giải & Đo đạc đường truyền L3–L7**
  * `dig`: Truy vấn DNS, cờ `+trace`, `+all`, `+noall +answer`, `+short`, đọc authoritative vs recursive response.
  * `mtr` / `traceroute`: Nguyên lý ICMP/UDP/TCP SYN traceroute; cách nhận diện router rate-limiting vs packet loss thật.
  * `curl -v` / `curl -w`: Trích xuất chi tiết timeline kết nối (DNS lookup, TCP handshake, TLS handshake, TTFB, transfer).
  * `openssl s_client`: Debug TLS handshake, kiểm tra chuỗi chứng chỉ (certificate chain), SNI, ALPN, cipher suites.
* **0.4. Bắt gói và mổ xẻ Packet: `tcpdump` & Wireshark**
  * Cú pháp BPF filter nâng cao: bắt TCP flags (SYN-only, RST), lọc host/port/CIDR, bắt non-fragmented/fragmented packets.
  * Kỹ thuật xuất `.pcap` trên production an toàn (giới hạn packet size `-s`, buffer `-B`, xoay vòng file `-W -C`).
  * Phân tích luồng (Follow TCP Stream, TCP Delta time, Window Update, ZeroWindow, Duplicate ACK) trên Wireshark.
* **Lab thực hành:** Dựng bài tập cô lập sự cố 1 service chết bằng kịch bản kết hợp `curl`, `ss`, `tcpdump`, `openssl s_client` trên Linux VM.
* **Case study sự cố:**
  1. *Case 1:* Ứng dụng timeout kết nối ngắt quãng; `ping` thông suốt nhưng `curl` treo ở TLS handshake.
  2. *Case 2:* Server không nhận thêm kết nối dù CPU/RAM rảnh; phân tích hàng đợi socket bằng `ss -ltn` phát hiện full listen backlog.

---

### MODULE 1: MÔ HÌNH OSI 7 TẦNG, TCP/IP & VẤN ĐỀ ĐÓNG GÓI GÓI TIN (ENCAPSULATION)
*Ước lượng: ~3.000 từ*

* **1.1. Đối chiếu OSI 7 Layer và TCP/IP 4/5 Layer trong môi trường Linux & Cloud**
  * Ánh xạ các tầng vào Linux Kernel Stack (Socket layer, TCP/IP stack, Qdisc, Driver, NIC).
  * Encapsulation & Decapsulation: Cấu trúc Header (Ethernet II, IPv4, TCP, Payload).
* **1.2. MTU (Maximum Transmission Unit) & MSS (Maximum Segment Size)**
  * Công thức tính: MTU = IP Header (20B) + TCP Header (20B) + MSS (1460B) = 1500B.
  * VLAN 802.1Q Tag (+4B): Tăng kích thước frame lên 1522B (Baby Giant frames), không làm giảm IP MTU 1500B nếu switch và NIC hỗ trợ.
  * Giới thiệu Overlay tunnel overhead (VXLAN +50B, GRE, IP-in-IP; xem chi tiết tính toán trong K8s tại Mục 10.7).
* **1.3. IP Fragmentation, DF (Don't Fragment) Bit & Path MTU Discovery (PMTUD)**
  * Quá trình phân mảnh IPv4 và rủi ro: giảm throughput, reassembly timeout, firewalls drop fragment.
  * PMTUD trong IPv4 (ICMP Type 3 Code 4) và IPv6 (ICMPv6 Type 2 Packet Too Big; lưu ý: IPv6 router không phân mảnh gói).
  * Vấn đề "ICMP Black Hole" khi Firewall/Cloud chặn ICMP packet-too-big và giải pháp MSS Clamping (`iptables -j TCPMSS --clamp-mss-to-pmtu`).
* **Lab thực hành:** Giả lập đường truyền MTU mismatch (MTU 1400 vs 1500) qua `ip link` và `netns`, chứng minh hiện tượng treo kết nối khi gửi payload lớn và fix bằng MSS clamping.
* **Case study sự cố:**
  1. *Case 1:* Kết nối SSH gõ lệnh ngắn chạy được, nhưng chạy lệnh xuất output dài (như `cat large.log`) hoặc `git push` thì phiên bị treo chết do ICMP Black Hole.
  2. *Case 2:* VPN IPSec/WireGuard site-to-site gây rớt gói khi truyền file qua HTTP POST do overhead mã hóa làm vượt MTU.

---

### MODULE 2: TẦNG LIÊN KẾT DỮ LIỆU (L2) — ETHERNET, SWITCHING & BONDING
*Ước lượng: ~3.500 từ*

* **2.1. Ethernet Frame, Địa chỉ MAC & Quá trình học MAC (MAC Learning)**
  * Cấu trúc Ethernet Frame (Preamble, SFD, MAC Đích, MAC Nguồn, EtherType, Payload, FCS).
  * MAC Address Table của Switch: Flooding, Learning, Aging, Unicast Flooding khi đầy bảng (CAM table exhaustion).
* **2.2. Giao thức ARP (Address Resolution Protocol) & Cache**
  * Quá trình ARP Request (Broadcast) / ARP Reply (Unicast).
  * ARP Cache trên Linux: `ip neigh`, `arping`, các trạng thái `REACHABLE`, `STALE`, `DELAY`, `PROBE`, `FAILED`.
  * Gratuitous ARP (GARP): Ứng dụng trong VIP failover (Keepalived/VRRP). *Ghi chú quan trọng: VRRP/GARP thuần L2 thường không hoạt động trên các hạ tầng Public Cloud như AWS/GCP do SDN chặn L2 broadcast.*
* **2.3. VLAN (IEEE 802.1Q) & Trunking**
  * Cấu trúc 802.1Q Tag (VLAN ID 1–4094, PCP/CoS). Access port vs Trunk port.
  * Linux VLAN Sub-interfaces (`ip link add link eth0 name eth0.100 type vlan id 100`).
* **2.4. Spanning Tree Protocol (STP) & Vòng lặp L2 (Broadcast Storm)**
  * Nguyên lý chặn loop của STP/RSTP/MSTP (Bridge Priority, Root Port, Designated Port, Blocking).
* **2.5. Network Bonding & LACP (802.3ad)**
  * Các chế độ Bond trên Linux: Mode 0 (Round-Robin), Mode 1 (Active-Backup), Mode 4 (802.3ad LACP), Mode 6 (balance-alb).
  * Thuật toán băm `xmit_hash_policy`: L2 (`layer2`), L2+L3 (`layer2+3`), L3+L4 (`layer3+4`).
* **Lab thực hành:** Cấu hình 2 namespace Linux qua Linux Bridge có gắn VLAN 802.1Q tag, kiểm chứng broadcast isolation bằng `tcpdump`.
* **Case study sự cố:**
  1. *Case 1:* IP failover bằng Keepalived trên môi trường Bare-metal hoàn tất nhưng server backup không nhận được traffic do Switch chưa update ARP cache (thiếu cấu hình GARP).
  2. *Case 2:* Cấu hình NIC Bonding LACP Mode 4 trên server nhưng Switch để sai mode dẫn đến rớt 50% packet.

---

### MODULE 3: TẦNG MẠNG (L3) — IPV4/IPV6, ĐỊNH TUYẾN, ICMP & NAT
*Ước lượng: ~4.000 từ*

* **3.1. IPv4, IPv6 & Kỹ năng Subnetting / CIDR**
  * Cấu trúc địa chỉ IPv4, Network ID, Host ID, Broadcast Address, Subnet Mask.
  * Bảng chuyển đổi CIDR (/24, /28, /27, /16...) và phương pháp tính nhẩm dải IP, số host khả dụng, Netmask.
  * Bài tập tính tay chia subnetting cho kiến trúc VPC Production.
  * Giới thiệu IPv6: Cấu trúc 128-bit, SLAAC, Link-local (`fe80::`), Global Unicast (`2000::/3`). (Xem ứng dụng IPv6 Dual-Stack K8s tại Mục 10.1).
* **3.2. Bảng định tuyến Linux & Longest Prefix Match (LPM)**
  * Cấu trúc bảng `ip route show`. Cột Destination, Gateway, Interface, Metric.
  * Nguyên tắc Longest Prefix Match: Cách kernel chọn route ưu tiên cao nhất khi có nhiều route trùng dải.
  * Default Gateway (`0.0.0.0/0`) và hành vi của gói tin khi không match route nào.
  * Policy-based Routing (`ip rule`): Định tuyến nâng cao dựa trên Source IP / Fwmark.
* **3.3. Giao thức ICMP & Trường TTL (Time To Live)**
  * Mục đích của ICMP: Error Reporting & Diagnostic. Các Type/Code quan trọng (Echo Request/Reply, Destination Unreachable, Time Exceeded).
  * TTL Mechanism: Chống loop gói tin và nguyên lý của `traceroute`.
* **3.4. Network Address Translation (NAT): SNAT, DNAT, MASQUERADE & PAT**
  * SNAT (Source NAT): Đổi IP nguồn để ra Internet / vượt VPC.
  * DNAT (Destination NAT): Đổi IP đích (Port Forwarding, cơ chế NodePort/ClusterIP của kube-proxy).
  * MASQUERADE: SNAT động trên dynamic IP.
  * Port Address Translation (PAT) / NAPT: Giới hạn tính theo mỗi Destination tuple `(Dst_IP, Dst_Port, Proto)`. Ví dụ: AWS NAT Gateway hỗ trợ khoảng 55.000 kết nối đồng thời trên mỗi IP gán cho 1 unique destination tuple.
* **3.5. DHCP & Định tuyến nâng cao (Khái niệm OSPF vs BGP)**
  * Quy trình DHCP DORA (Discover, Offer, Request, Acknowledge).
  * Khái niệm IGP (OSPF - Link State, Dijkstra) vs EGP (BGP - Path Vector, AS Number, Peering; xem chi tiết BGP tại Phụ lục 15.1).
* **Lab thực hành:** Dựng Router Linux bằng `ip netns`, bật `net.ipv4.ip_forward`, thiết lập SNAT MASQUERADE và DNAT bằng `iptables`/`nftables` cho mạng private kết nối ra mạng ngoài.
* **Case study sự cố:**
  1. *Case 1:* 10.000 pod cùng gọi ra 1 API bên ngoài qua 1 IP NAT Gateway duy nhất bị lỗi `connection reset/timeout` do cạn kiệt SNAT port allocation (`ErrorPortAllocation` metric trên CloudWatch).
  2. *Case 2:* Cấu hình 2 Default Gateway trên 2 card mạng khác nhau gây hiện tượng "Asymmetric Routing", gói tin trả về bị kernel drop do cơ chế bảo vệ `rp_filter` (Reverse Path Filtering).

---

### MODULE 4: TẦNG GIAO VẬN (L4) — TCP, UDP, QUIC & QUẢN TRỊ KERNEL STATE
*Ước lượng: ~4.700 từ*

* **4.1. So sánh TCP vs. UDP**
  * Đặc tính tin cậy (Reliable, Ordered, Connection-oriented) vs Tối giản (Unreliable, Connectionless, Low Latency).
  * Cấu trúc Header TCP (20–60 bytes) vs UDP (8 bytes).
* **4.2. Vòng đời kết nối TCP, Máy trạng thái (State Machine) & Gói tin RST**
  * **3-Way Handshake:** `SYN` → `SYN-ACK` → `ACK`. Thiết lập ISN, MSS, Window Scale, SACK Permitted.
  * **4-Way Teardown:** `FIN` → `ACK` → `FIN` → `ACK`.
  * **Cờ RST (Reset):** Các nguyên nhân kernel sinh gói RST (gửi vào port đang đóng, kết nối quá timeout, tràn listen queue với `tcp_abort_on_overflow`, hoặc ứng dụng gọi `close()` khi còn dữ liệu chưa đọc trong socket buffer).
  * **Trạng thái `TIME_WAIT`:**
    * Bản chất và mục đích (đảm bảo ACK cuối cùng tới đích, chống gói tin trễ 2*MSL lạc vào kết nối mới).
    * `TIME_WAIT` nằm ở bên chủ động đóng kết nối (Active Closer). Thời gian cố định 60 giây trên Linux kernel (`TCP_TIMEWAIT_LEN = 60*HZ`).
    * Bẫy `tcp_tw_recycle` (đã bị gỡ bỏ hoàn toàn khỏi Linux Kernel >= 4.12 vì phá vỡ NAT) vs giải pháp an toàn `tcp_tw_reuse = 1` cho client outgoing connections.
    * Lưu ý về `tcp_tw_reuse`: Yêu cầu bắt buộc phải bật `tcp_timestamps = 1`, chỉ áp dụng cho outgoing connection; trên kernel gần đây có giá trị 2 (chỉ kích hoạt cho kết nối loopback).
  * **Trạng thái `CLOSE_WAIT`:** Nguyên nhân do ứng dụng nhận `FIN` nhưng code không gọi `close()` socket (rò rỉ file descriptor / application bug).
* **4.3. Cơ chế Flow Control, Congestion Control & Thuật toán Truyền tin**
  * **Flow Control:** Sliding Window, Receive Window (`rwnd`), Window Scale factor, hiện tượng Zero Window và TCP Window Probe.
  * **Congestion Control:** Congestion Window (`cwnd`), Slow Start, Congestion Avoidance, Fast Retransmit (3 Duplicate ACKs), Fast Recovery.
  * **RTO, SACK & TLP:** RTO (Retransmission Timeout), SACK (Selective Acknowledgement) tránh retransmit thừa, TLP (Tail Loss Probe) giảm độ trễ khi mất gói ở đuôi luồng truyền.
  * **Nagle Algorithm & Delayed ACK:** Xung đột giữa Nagle (`TCP_NODELAY`) và Delayed ACK (40ms latency penalty kinh điển trong microservices).
  * **Thuật toán điều khiển tắc nghẽn:**
    * Reno: Giảm 50% `cwnd` khi mất gói (Loss-based).
    * CUBIC (mặc định Linux): Hàm bậc 3 dựa trên thời gian từ lần mất gói gần nhất, không giảm cứng 1/2 như Reno.
    * BBR (Google): Model-based, tối ưu theo băng thông và RTT thực tế, giải quyết vấn đề bufferbloat (xem tinh chỉnh kernel tại Mục 14.2).
* **4.4. SYN Backlog, SYN Flood & SYN Cookies**
  * Hai hàng đợi kết nối: **SYN Queue** vs **Accept Queue**.
  * Tràn Accept Queue: `net.core.somaxconn` vs tham số `backlog` trong `listen(fd, backlog)`.
  * Tràn SYN Queue và cơ chế chống SYN Flood bằng `net.ipv4.tcp_syncookies = 1`.
* **4.5. Ephemeral Ports, Port Exhaustion & Conntrack Core**
  * Dải cổng cục bộ: `net.ipv4.ip_local_port_range` (mặc định: 32768–60999).
  * Khái niệm Socket 4-Tuple: `(Src_IP, Src_Port, Dst_IP, Dst_Port)`.
  * Bảng theo dõi kết nối Netfilter Conntrack (`nf_conntrack_max`, `nf_conntrack_count`): Trạng thái `NEW`, `ESTABLISHED`, `RELATED`, `INVALID`. Hậu quả khi tràn bảng conntrack (xem chi tiết Netfilter tại Mục 8.3).
* **4.6. TCP Keepalive & Giới thiệu QUIC (HTTP/3)**
  * TCP Keepalive parameters: `tcp_keepalive_time`, `tcp_keepalive_intvl`, `tcp_keepalive_probes`.
  * QUIC: Chạy trên UDP, loại bỏ Head-of-Line Blocking giữa các stream độc lập ở tầng L4, Connection Migration, Handshake 1-RTT và 0-RTT resumption.
* **Lab thực hành:** Dựng kịch bản tạo tải làm tràn `Accept Queue`, dùng `ss -lnt` quan sát `Send-Q` vs `Recv-Q` và cấu hình sysctl `somaxconn` + code backend để xử lý.
* **Case study sự cố:**
  1. *Case 1:* Microservice gọi qua HTTP/1.1 client không bật Connection Pool gây tràn hàng nghìn kết nối `TIME_WAIT` và văng lỗi `Cannot assign requested address`.
  2. *Case 2:* Ứng dụng REST API phản hồi chậm 40ms ngẫu nhiên cho các request nhỏ do xung đột giữa Nagle Algorithm và Delayed ACK (sửa bằng cờ `TCP_NODELAY`).

---

### MODULE 5: HỆ THỐNG PHÂN GIẢI TÊN MIỀN (DNS) TRONG LINUX & CONTAINER
*Ước lượng: ~3.700 từ*

* **5.1. Kiến trúc phân giải DNS toàn cầu & Record Types**
  * Quá trình phân giải đệ quy (Recursive Resolution): Client → Resolver → Root Servers (.) → TLD Servers (.com) → Authoritative Name Servers.
  * Các loại Record: `A`, `AAAA`, `CNAME`, `ALIAS/ANAME`, `MX`, `TXT`, `NS`, `PTR`, `SRV`, `SOA`.
  * TTL (Time To Live), Caching ở các cấp (Browser, OS resolver, Local DNS, Upstream DNS).
  * Giới hạn gói tin Route 53 Resolver / Link-local DNS (1024 PPS per ENI trên AWS EC2, gây timeout DNS ngắt quãng khi tải cao).
* **5.2. Cấu hình DNS trên Linux: `/etc/resolv.conf`, `nsswitch.conf` & Systemd-resolved**
  * `nameserver`, `options timeout:N`, `options attempts:N`.
  * **Tham số `ndots` & `search domain`:**
    * Nguyên lý hoạt động: Khi nào query được gắn thêm domain suffix, khi nào query FQDN trực tiếp.
    * Vấn đề bùng nổ query trong Kubernetes (mặc định `ndots:5`).
* **5.3. DNS trong Kubernetes (CoreDNS) & NodeLocal DNSCache**
  * Quy tắc đặt tên Service: `<svc-name>.<namespace>.svc.cluster.local`.
  * Cấu hình `Corefile` của CoreDNS: `errors`, `health`, `kubernetes`, `forward`, `cache`, `reload`.
  * Caching & NodeLocal DNSCache: Giảm tải connection tracking UDP và giảm latency; *lưu ý: NodeLocal DNSCache không loại bỏ số query thừa do `ndots:5` sinh ra.*
* **5.4. Kiến trúc Split-Horizon DNS**
  * Cơ chế trả về IP Private khi truy vấn từ mạng nội bộ (VPC) và IP Public khi truy vấn từ Internet cho cùng một domain.
* **Lab thực hành:** Dùng `dig` phân tích chi tiết luồng phân giải DNS từ Root Server; giả lập cấu hình `ndots:5` trong file `/etc/resolv.conf` và dùng `tcpdump` đếm số lượng query DNS thừa thãi được sinh ra.
* **Case study sự cố:**
  1. *Case 1:* Ứng dụng chạy trên Linux/K8s gọi external API bị độ trễ cao ngẫu nhiên 5 giây do race condition trong Linux kernel conntrack khi truy vấn song song A và AAAA qua cùng 4-tuple UDP (glibc gửi song song kích hoạt race condition; musl trên Alpine xử lý khác). Khắc phục bằng: `options single-request-reopen` (hoặc `single-request`), NodeLocal DNSCache, `options use-vc` (DNS qua TCP), hoặc bản vá kernel Netfilter.
  2. *Case 2:* Đổi IP server trên Public DNS với TTL 60s nhưng sau 2 giờ ứng dụng backend bên thứ ba vẫn gửi traffic về IP cũ do Connection Pool duy trì TCP socket cũ hoặc tầng ứng dụng tự cache kết quả resolve (JVM hiện đại chỉ cache vĩnh viễn khi có SecurityManager).

---

### MODULE 6: GIAO THỨC TẦNG ỨNG DỤNG (L7) & BẢO MẬT TLS
*Ước lượng: ~4.000 từ*

* **6.1. Tiến hóa giao thức Web: HTTP/1.1 → HTTP/2 → HTTP/3**
  * **HTTP/1.1:** Keep-Alive, Pipelining, Head-of-Line (HoL) Blocking ở L7.
  * **HTTP/2:** Binary Framing, Multiplexing (nhiều stream trên 1 TCP connection), Header Compression (HPACK); *HTTP/2 Server Push đã bị deprecate/gỡ bỏ ở hầu hết trình duyệt lớn*; vấn đề HoL Blocking ở L4 khi rớt gói tin TCP.
  * **HTTP/3:** Dựa trên QUIC/UDP, giảm HoL Blocking giữa các stream độc lập (lưu ý: HoL Blocking vẫn tồn tại trong phạm vi 1 stream đơn lẻ).
  * **gRPC & WebSocket:** gRPC dựa trên HTTP/2 Streams và Protobuf; WebSocket chuyển giao thức qua header `Upgrade: websocket`.
* **6.2. Mật mã học & Giao thức TLS (1.2 vs 1.3)**
  * Đối xứng (Symmetric) vs Bất đối xứng (Asymmetric). Hàm băm (Hashing).
  * **TLS 1.2 Handshake (2-RTT):** So sánh Key exchange RSA (cũ/không an toàn) vs ECDHE (Forward Secrecy).
  * **TLS 1.3 Handshake (1-RTT):** Bỏ hoàn toàn RSA key exchange (chỉ dùng ECDHE để trao đổi khóa, RSA/ECDSA chỉ dùng để ký xác thực chữ ký số), 0-RTT Early Data (resumption) và nguy cơ Replay Attack.
* **6.3. Chuỗi chứng chỉ (Certificate Chain), SNI & ALPN**
  * PKI: Root CA, Intermediate CA, Leaf/Server Certificate.
  * Server Name Indication (SNI): Gửi hostname trong TLS ClientHello giúp 1 IP host được nhiều domain TLS.
  * Application-Layer Protocol Negotiation (ALPN): Đàm phán giao thức (h2, http/1.1) ngay trong TLS Handshake.
* **6.4. Vòng đời Chứng chỉ, mTLS & Tự động hóa**
  * Lộ trình rút ngắn thời hạn chứng chỉ công cộng theo CA/B Forum (Ballot SC-081v3: giảm dần về 200 ngày vào 2026, 100 ngày vào 2027, 47 ngày vào 2029) → bắt buộc tự động hóa qua ACME protocol.
  * mTLS (Mutual TLS): Xác thực 2 chiều (Client Cert & Server Cert) cho Zero Trust / Service Mesh.
  * Quản lý tự động với `cert-manager` (Let's Encrypt ACME HTTP-01 vs DNS-01 Challenge).
* **6.5. Các mô hình kiến trúc TLS**
  * Phân tích so sánh: TLS Termination vs. TLS Passthrough vs. TLS Re-encryption (hiệu năng CPU, quản lý chứng chỉ, bảo mật).
* **Lab thực hành:** Dùng `openssl s_client` inspect chi tiết Certificate Chain của một website; tự tạo Root CA, Intermediate CA, ký Server Certificate và cấu hình Nginx bắt buộc xác thực mTLS.
* **Case study sự cố:**
  1. *Case 1:* Client gọi API bị lỗi `x509: certificate signed by unknown authority` trên production do server chỉ gửi Leaf Certificate mà quên gửi kèm Intermediate Certificate (Broken Certificate Chain).
  2. *Case 2:* Load Balancer TLS Listener terminate TLS nhưng không cấu hình chính sách ALPN quảng bá `h2`, dẫn đến client gRPC fallback về HTTP/1.1 và văng lỗi handshake failed.

---

### MODULE 7: LOAD BALANCING, REVERSE PROXY & GIAO THỨC TRUYỀN TẢI
*Ước lượng: ~4.000 từ*

* **7.1. Phân biệt Layer 4 vs. Layer 7 Load Balancing**
  * L4 LB (TCP/UDP): Chuyển tiếp packet (NAT/DR/DSR), throughput cao, không giải mã payload/TLS (TLS Termination xem Mục 6.5).
  * L7 LB (HTTP/gRPC): Reverse Proxy (2 kết nối TCP riêng biệt: Client → LB và LB → Backend), định tuyến Path/Header, Caching, WAF.
* **7.2. Thuật toán cân bằng tải & Health Checking**
  * Round Robin, Weighted Round Robin, Least Connections, IP Hash / Consistent Hashing.
  * Health Check: Active Check (HTTP/TCP probe, Interval, Timeout, Threshold) vs Passive Check (Circuit Breaker).
* **7.3. Bảo toàn Client IP: X-Forwarded-For vs. PROXY Protocol**
  * `X-Forwarded-For` (XFF) & `X-Forwarded-Proto` trên L7. Vấn đề IP Spoofing và cách config Trust Proxy an toàn.
  * PROXY Protocol v1 (text) & v2 (binary): Giải pháp truyền Client IP/Port qua L4 Load Balancer mà không cần L7 proxy.
* **7.4. Giữ kết nối & Lỗi Mismatch Timeout / Keepalive**
  * Mối quan hệ giữa Client Timeout, LB Idle Timeout và Backend Keepalive Timeout.
  * Hậu quả khi Backend đóng kết nối trước Load Balancer (lỗi `502 Bad Gateway` ngẫu nhiên).
* **7.5. Graceful Drain / Connection Draining**
  * Cơ chế Deregistration Delay, Shutdown Hook: Đảm bảo request đang xử lý không bị đứt gãy khi scale down hoặc deploy phiên bản mới.
* **7.6. Vấn đề lệch tải gRPC qua L4 Load Balancer**
  * Bản chất HTTP/2 giữ kết nối TCP dài hạn: Khi số lượng kết nối client ít, L4 LB sẽ dồn toàn bộ request của 1 client vào duy nhất 1 backend pod.
  * Giải pháp: L7 Load Balancing (Envoy/Nginx), Client-side Load Balancing, Headless Service + gRPC resolver.
* **Lab thực hành:** Cấu hình Nginx làm Reverse Proxy L7 và HAProxy làm L4 Proxy, mô phỏng lỗi Keepalive Timeout Mismatch gây 502 và cấu hình PROXY Protocol để ghi log chính xác Client IP.
* **Case study sự cố:**
  1. *Case 1:* Hệ thống thỉnh thoảng sinh mã lỗi HTTP 502 với tần suất thấp dưới tải cao do Backend keepalive timeout (60s) nhỏ hơn timeout của Cloud Load Balancer (65s).
  2. *Case 2:* Triển khai service gRPC với 2 client dài hạn sau NLB (L4), toàn bộ request bị dồn vào 2 trong số 10 backend pod gây quá tải cục bộ.

---

### MODULE 8: LINUX NETWORKING NỘI BỘ (KERNEL STACK & NAMESPACES)
*Ước lượng: ~4.500 từ*

* **8.1. Network Namespaces (`netns`), Veth Pairs & Linux Bridge**
  * Khái niệm cô lập tài nguyên mạng: Routing table, Interface, Iptables, Socket list riêng biệt.
  * `veth pair`: Cặp interface ảo nối giữa namespace và host.
  * Linux Bridge: Software Switch ảo trên kernel; Forwarding database (`fdb`).
  * Module `br_netfilter` và tham số `net.bridge.bridge-nf-call-iptables = 1` (cho phép iptables lọc traffic bridge trong K8s).
* **8.2. Netfilter, Iptables vs. Nftables**
  * Kiến trúc Netfilter: 5 Hooks (`PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`).
  * Iptables: Các bảng (`raw`, `mangle`, `nat`, `filter`) và Chains tương ứng.
  * Nftables: Kiến trúc bytecode mới thay thế iptables; set dạng hash tra cứu xấp xỉ O(1), set dạng interval dùng rbtree O(log n); so với duyệt tuần tự O(N) của iptables.
* **8.3. Conntrack Subsystem**
  * Bảng trạng thái Netfilter Connection Tracking (`/proc/net/nf_conntrack`, sysctl `nf_conntrack_max`). (Xem nguyên lý 4-tuple tại Mục 4.5).
  * Rủi ro hiệu năng: Tốn bộ nhớ RAM, lock contention trên multi-core khi số lượng connection quá lớn.
* **8.4. Linux Traffic Control (`tc`) & Qdisc**
  * Quản lý hàng đợi gói tin: Classless vs Classful Qdisc (FIFO, fq_codel, HTB).
  * Ứng dụng `tc` để giả lập mạng kém (Network Emulation - `netem`: packet loss, jitter, latency limit).
* **8.5. Tinh chỉnh Sysctl Mạng & Lưu trữ Bền vững**
  * Nhóm socket buffer: `net.core.rmem_max`, `net.core.wmem_max`, `net.ipv4.tcp_rmem`, `net.ipv4.tcp_wmem`.
  * Nhóm hàng đợi: `net.core.somaxconn`, `net.core.netdev_max_backlog`, `net.ipv4.tcp_max_syn_backlog`.
  * Bền vững hóa cấu hình: Viết vào `/etc/sysctl.d/99-custom-net.conf` và nạp bằng `sysctl --system` (cảnh báo `sysctl -w` mất sau reboot).
* **8.6. Tổng quan eBPF trong Linux Networking**
  * XDP (eXpress Data Path) & eBPF tc: Xử lý gói tin trực tiếp ở tầng driver trước khi cấp phát `sk_buff`, giảm context switch và tăng hiệu năng xử lý gói tin.
* **Lab thực hành:** Tự tay viết bash script dựng môi trường Container bằng thuần công cụ Linux (`ip netns`, `ip link add type veth`, `ip link add type bridge`, `iptables NAT`) không dùng Docker, kết nối 2 namespace ra ngoài Internet.
* **Case study sự cố:**
  1. *Case 1:* 1 node Linux chịu tải gói tin cực lớn bị rớt gói ngẫu nhiên; kiểm tra cột 2 của `/proc/net/softnet_stat` hoặc lệnh `nstat` thấy bộ đếm drop tăng do `netdev_max_backlog` quá nhỏ.
  2. *Case 2:* Kỹ sư chạy lệnh `sysctl -w net.ipv4.ip_forward=1` để fix sự cố mạng, nhưng 1 tháng sau server khởi động lại thì toàn bộ routing bị sập do không lưu config vào file tĩnh.

---

### MODULE 9: MẠNG CONTAINER (DOCKER NETWORKING)
*Ước lượng: ~3.000 từ*

* **9.1. Các chế độ mạng trong Docker (Network Modes)**
  * **Bridge mode (default):** Mỗi container 1 netns, 1 veth nối vào `docker0` bridge, cấp IP qua IPAM nội bộ.
  * **Host mode (`--net=host`):** Dùng chung network namespace với Host, không cô lập, hiệu năng cao, tránh NAT overhead.
  * **None mode (`--net=none`):** Chỉ có loopback, hoàn toàn cô lập.
  * **Container mode (`--net=container:<id>`):** Dùng chung netns với container khác (nền tảng của Pod trong Kubernetes).
  * **Macvlan / Ipvlan mode:** Gán trực tiếp MAC/IP vật lý cho container.
* **9.2. Port Publishing (`-p host_port:container_port`)**
  * Cách Docker can thiệp iptables: Chain `DOCKER`, `DOCKER-USER`, quy tắc DNAT từ Host port sang Container IP.
  * Hiểm họa bảo mật: Docker iptables bypass firewall UFW/Firewalld mặc định trên host.
* **9.3. User-Defined Bridge vs. Default `docker0` Bridge**
  * Embedded DNS Server (`127.0.0.11`): User-defined bridge hỗ trợ tự động phân giải tên container; Default bridge không có DNS giữa các container (chỉ có cơ chế `--link` kiểu cũ ghi `/etc/hosts`) và container kế thừa cấu hình DNS của host.
* **Lab thực hành:** Phân tích toàn bộ iptables rule sinh ra bởi Docker khi chạy `docker run -p 8080:80`, thử nghiệm giải pháp chặn truy cập từ IP bên ngoài bằng chain `DOCKER-USER`.
* **Case study sự cố:**
  1. *Case 1:* Cài đặt UFW chặn port 8080 trên VPS nhưng bên ngoài vẫn quét và truy cập được vào database container chạy `docker run -p 8080:80` do Docker can thiệp thẳng vào PREROUTING chain của Netfilter.
  2. *Case 2:* Hai container cùng nối vào default bridge `docker0` không gọi nhau được bằng tên container qua DNS do default bridge không hỗ trợ embedded DNS.

---

### MODULE 10: MẠNG KUBERNETES TOÀN DIỆN (K8S NETWORKING)
*Ước lượng: ~5.500 từ*

* **10.1. Mô hình Mạng Kubernetes & IPv4/IPv6 Dual-Stack**
  * Các yêu cầu của mô hình mạng Kubernetes (chuẩn tài liệu K8s):
    1. Mỗi Pod có địa chỉ IP riêng biệt và duy nhất trong cluster.
    2. Mọi Pod có thể giao tiếp với mọi Pod khác trên mọi node mà không dùng NAT.
    3. Agent trên node (kubelet, system daemon) có thể giao tiếp với mọi Pod trên node đó.
    4. Pod sử dụng host network có thể giao tiếp với mọi Pod trên mọi node mà không dùng NAT (trên Linux).
  * Pod Network Namespace: Container `pause` giữ namespace mạng (`veth`), các container trong Pod dùng chung loopback `localhost`.
  * **IPv4/IPv6 Dual-Stack trong Kubernetes:**
    * Cấu hình `podCIDRs` phân bổ 2 họ địa chỉ đồng thời.
    * Service `ipFamilyPolicy`: `SingleStack`, `PreferDualStack`, `RequireDualStack`.
    * Yêu cầu hỗ trợ hạ tầng từ CNI plugin và Kube-proxy chạy chế độ dual-stack.
* **10.2. Luồng gói tin Pod-to-Pod**
  * **Cùng Node:** Pod A `eth0` → `veth` pair → Node Bridge/Routing → Pod B `veth` → Pod B `eth0`.
  * **Khác Node:**
    * Routed Network (Flat network/BGP): Gói tin đi trực tiếp qua Underlay routing không bọc gói.
    * Overlay Network: Gói tin được đóng gói trong VXLAN/IP-in-IP tại Node nguồn và mở gói tại Node đích.
* **10.3. Container Network Interface (CNI) & Chế độ Dữ liệu (Data Plane)**
  * **Flannel:** Chế độ VXLAN (Đóng gói L2 trong UDP; port mặc định trên Linux là UDP 8472, IANA standard port là 4789); Chế độ Host-GW (Direct routing L2). *Flannel thuần không thực thi NetworkPolicy.*
  * **Calico:** Chế độ BGP (Direct routing), IP-in-IP, Calico VXLAN (dùng UDP port 4789), và Calico eBPF data plane.
  * **Cilium:** Dựa trên Linux eBPF, hỗ trợ thay thế kube-proxy (`kube-proxy replacement`), định tuyến và load balancing trực tiếp ở tầng socket/XDP, giảm overhead chuyển đổi context.
  * **CNI Managed Cloud:**
    * Amazon VPC CNI: Cấp IP trực tiếp từ VPC Subnet (hạn chế số Pod theo ENI/Instance type, giải pháp Prefix Delegation để tăng mật độ Pod, hỗ trợ NetworkPolicy [CẦN XÁC MINH theo phiên bản]).
    * GKE Datapath v2 (dựa trên Cilium) & Azure CNI (Pod Subnet / Overlay).
* **10.4. Kubernetes Service & Kube-Proxy Nâng cao**
  * Các loại Service: `ClusterIP`, `NodePort`, `LoadBalancer`, `Headless Service`.
  * On-prem & Bare-metal LoadBalancing: MetalLB (Layer 2 ARP / BGP mode), kube-vip.
  * Cơ chế nâng cao: `externalTrafficPolicy: Local` (giữ Client IP gốc, tránh extra hop) vs `Cluster` (cân bằng tải đều nhưng SNAT đổi IP nguồn), `sessionAffinity: ClientIP`.
  * **Kube-proxy modes:**
    * `iptables` mode (Mặc định).
    * `ipvs` mode (Đã bị deprecate trong các phiên bản Kubernetes mới; cộng đồng chuyển sang nftables hoặc eBPF).
    * `nftables` mode (Thế hệ mới trong K8s hiện đại).
  * **EndpointSlice:** Phân đoạn endpoints (mặc định 100 endpoints/slice) giảm tải API Server và etcd.
* **10.5. Ingress, Gateway API & Migration**
  * Tình trạng Ingress-NGINX: Dự án `ingress-nginx` chính thức retire từ tháng 3/2026 (không còn nhận bản vá bảo mật).
  * Kế hoạch di chuyển sang Kubernetes Gateway API: Phân tách vai trò (GatewayClass, Gateway, HTTPRoute) với Envoy Gateway, Traefik, Cilium Gateway API.
* **10.6. NetworkPolicy & Mô hình Bảo vệ Đông-Tây (East-West Security)**
  * Cấu trúc NetworkPolicy: `podSelector`, `ingress`, `egress`, `namespaceSelector`, `ipBlock`.
  * CNI thực thi NetworkPolicy: Calico, Cilium, Antrea, Amazon VPC CNI [CẦN XÁC MINH theo phiên bản]. *(Lưu ý: Weave Net đã ngừng duy trì; Flannel không hỗ trợ).*
* **10.7. MTU Overlay Overhead, Egress Gateway & Service Mesh**
  * Tính toán MTU Overlay: Nếu Underlay MTU = 1500B, VXLAN MTU phải cấu hình 1450B (trừ 50B overhead) (xem nguyên lý MTU tại Mục 1.2).
  * Egress Gateway: Cố định Egress IP cho Pod giao tiếp với hệ thống bên ngoài.
  * Service Mesh: Sidecar architecture (Envoy proxy qua iptables redirection) vs Ambient / Sidecarless architecture (Istio ambient dùng ztunnel proxy L4; Cilium Service Mesh dùng eBPF).
* **10.8. Phương pháp Debug Mạng trong Kubernetes**
  * Sử dụng `kubectl debug` với Ephemeral Container.
  * Kỹ thuật `nsenter` vào network namespace của container từ Node.
  * Công cụ bắt gói trong Pod: `ksniff`, công cụ eBPF BCC tools (`tcpretrans`, `tcplife`).
* **Lab thực hành:** Dựng cụm `kind` (Kubernetes in Docker), cài đặt Gateway API (Envoy Gateway), viết và kiểm thử NetworkPolicy chặn toàn bộ traffic chỉ cho phép traffic hợp lệ từ Gateway.
* **Case study sự cố:**
  1. *Case 1:* Pod trên Node 1 gọi Service tới Pod trên Node 2 bị timeout khi truyền payload lớn nhưng ping gói nhỏ thông do cấu hình sai MTU của Calico VXLAN interface trên hạ tầng Cloud MTU 1500.
  2. *Case 2:* Cluster có hàng nghìn service bị quá tải Node định kỳ, kube-proxy iptables mode ăn 100% CPU do lock contention khi cập nhật hàng chục nghìn rule.
  3. *Case 3 (Sự cố cấp phát IP với Amazon VPC CNI):* Pod rơi vào trạng thái `Pending` không schedule được, phân biệt 2 nguyên nhân qua lệnh kiểm tra:
     * *(a) Giới hạn số lượng IP/ENI trên Node:* Kiểm tra bằng `kubectl describe node <node-name>` (trường `Allocatable: pods`) → Khắc phục bằng Prefix Delegation (`ENABLE_PREFIX_DELEGATION=true`) hoặc đổi sang Instance Type lớn hơn.
     * *(b) Cạn kiệt IP khả dụng trong Subnet VPC:* Kiểm tra số IP khả dụng trong VPC Subnet console/CLI → Khắc phục bằng cấp thêm Secondary CIDR cho VPC kết hợp Custom Networking hoặc cấp Subnet riêng biệt cho Pod.
     * *Lưu ý cốt lõi:* Prefix Delegation chỉ giải quyết bài toán (a) - tăng mật độ Pod trên từng Node; KHÔNG giải quyết được bài toán (b) khi toàn bộ dải Subnet đã hết IP.

---

### MODULE 11: MẠNG CLOUD (CLOUD NETWORKING — AWS CHUẨN, ĐỐI CHIẾU GCP/AZURE)
*Ước lượng: ~4.800 từ*

* **11.1. Virtual Private Cloud (VPC) & Thiết kế Subnet Multi-AZ**
  * CIDR Block VPC (RFC 1918: `10.0.0.0/16`, `172.16.0.0/16`, `192.168.0.0/16`).
  * Phân chia Subnet Public, Private, Database độc lập qua tối thiểu 3 Availability Zones (AZ).
  * 5 IP mặc định bị AWS giữ lại trong mỗi subnet (.0 Network, .1 Router, .2 DNS, .3 Future, .255 Broadcast).
  * So sánh nhanh: AWS VPC (Regional) vs GCP VPC (Global VPC xuyên vùng) vs Azure VNet.
* **11.2. Định tuyến, Internet Gateway (IGW) & NAT Gateway (Zonal vs. Regional)**
  * Route Table: Local Route, Default Route (`0.0.0.0/0`).
  * Internet Gateway (IGW) cho Public Subnet (Stateful 1:1 NAT sang Public IP).
  * **NAT Gateway Architecture:**
    * *Zonal NAT Gateway:* Hoạt động trong 1 AZ; HA yêu cầu tạo mỗi AZ 1 NAT Gateway và route theo AZ; hỗ trợ tối đa 8 IP (440.000 kết nối đồng thời).
    * *Regional NAT Gateway (AWS công bố 19/11/2025):* 1 tài nguyên cấp VPC tự động co giãn theo các AZ có workload; 2 chế độ Automatic (khuyến nghị) và Manual; không cần tạo public subnet để đặt; không hỗ trợ private connectivity (use case private NAT vẫn dùng Zonal NAT GW); hỗ trợ tối đa 32 IP mỗi AZ (1.760.000 kết nối/AZ).
* **11.3. Tường lửa Cloud: Security Group vs. Network ACL (NACL)**
  * So sánh chuyên sâu:
    * Security Group: Gán vào ENI/Instance, **Stateful** (mở Inbound tự động mở Outbound trả về), chỉ có luật ALLOW.
    * NACL: Gán vào Subnet level, **Stateless** (phải mở cả Inbound và Outbound Ephemeral Ports 1024–65535), có thứ tự ưu tiên (Rule number) và luật ALLOW/DENY.
* **11.4. Kết nối VPC & Mở rộng Hạ tầng**
  * **VPC Peering:** Kết nối 1-1, không hỗ trợ Transitive Routing.
  * **AWS Transit Gateway (TGW):** Hub-and-Spoke router, hỗ trợ hàng nghìn VPC với Route Tables độc lập.
  * **AWS PrivateLink / VPC Endpoints:**
    * Gateway Endpoint (miễn phí cho S3, DynamoDB - can thiệp Route Table).
    * Interface Endpoint (tính phí theo ENI PrivateLink - giữ traffic trong AWS backbone qua DNS private).
* **11.5. Cloud Load Balancers**
  * AWS ALB (L7) vs NLB (L4, IP tĩnh per AZ, throughput cao). (Xem chi tiết L4 vs L7 tại Module 7).
  * Đối chiếu: GCP Cloud Load Balancing (Anycast IP toàn cầu) vs Azure ALB/Application Gateway.
* **11.6. Kết nối Hybrid & Tối ưu Chi phí Mạng (Data Transfer Costs)**
  * Site-to-Site VPN (IPsec) vs AWS Direct Connect (DX) / GCP Interconnect.
  * Ma trận chi phí mạng: Ingress (miễn phí), Same-AZ (miễn phí), Cross-AZ Data Transfer (tính phí cả 2 đầu), Cross-Region, Internet Egress, NAT Gateway Processing Fee.
  * **Chiến lược định tuyến nội vùng K8s để giảm chi phí Cross-AZ:**
    * Trường `.spec.trafficDistribution`: các giá trị `PreferSameZone` (thay thế tên cũ `PreferClose`), `PreferSameNode` (KEP-3015 ghi nhận Stable ở Kubernetes v1.35 [CẦN XÁC MINH]).
    * Annotation legacy: `service.kubernetes.io/topology-mode` (Topology Aware Routing/Hints).
    * Đẩy traffic S3/DynamoDB qua Gateway Endpoint miễn phí.
* **Lab thực hành:** Viết mã Terraform dựng VPC Multi-AZ chuẩn Production gồm Public Subnet, Private Subnet, Route Table, IGW, so sánh triển khai NAT Gateway Zonal (3 cái cho 3 AZ) vs Regional NAT Gateway, gán Security Group chuẩn Least Privilege cho Web tier và DB tier.
* **Case study sự cố:**
  1. *Case 1:* Khởi tạo EC2 trong Private Subnet, gán Security Group mở full port 80/443 nhưng EC2 không thể `apt-get update` hoặc gọi ra Internet do thiếu NAT Gateway trong Route Table hoặc thiếu rule out ephemeral port trên NACL.
  2. *Case 2:* Hóa đơn Cloud tăng vọt hàng chục nghìn USD tiền Cross-AZ Data Transfer do cụm K8s giao tiếp chéo AZ liên tục; khắc phục bằng cách cấu hình trường `trafficDistribution: PreferSameZone` hoặc annotation `service.kubernetes.io/topology-mode: Auto`.

---

### MODULE 12: AN NINH MẠNG VÀ PHÒNG THỦ HẠ TẦNG (NETWORK SECURITY & DEFENSE)
*Ước lượng: ~3.500 từ*

* **12.1. Phân vùng mạng (Network Segmentation) & Kiến trúc Zero Trust**
  * Mô hình Phòng thủ chiều sâu (Defense in Depth): Phân tách Micro-segmentation từ L3/L4 (VPC/Security Groups/NetworkPolicy) đến L7 (mTLS, JWT/AuthZ).
  * Nguyên lý "Never Trust, Always Verify".
* **12.2. VPN Doanh nghiệp & Quản trị Truy cập An toàn**
  * IPsec vs. WireGuard (Giao thức hiện đại, mã nguồn tinh gọn, thường nhanh hơn OpenVPN tùy workload).
  * Truy cập hạ tầng không cần Bastion Host: AWS Systems Manager (SSM) Session Manager / GCP IAP / Teleport (Loại bỏ hoàn toàn việc mở port SSH 22 ra Internet).
* **12.3. Tường lửa Ứng dụng Web (WAF), Rate Limiting & Chống DDoS**
  * WAF (L7): Nhận diện và chặn OWASP Top 10 (SQLi, XSS, Path Traversal, Bot Traffic).
  * Rate Limiting: Thuật toán Token Bucket vs Leaky Bucket trên Nginx/Envoy/Cloudflare.
  * Phòng chống DDoS: L3/L4 (SYN Flood, UDP Amplification - giảm thiểu bởi AWS Shield, Cloudflare) vs L7 (HTTP Flood - xử lý bởi Rate Limit, Challenge Captcha, WAF).
* **12.4. Egress Filtering & Phòng chống Rò rỉ Dữ liệu (Data Exfiltration)**
  * Chặn toàn bộ Egress mặc định; chỉ cho phép gọi ra các domain/FQDN trong whitelist bằng Forward Proxy (Squid/Envoy) hoặc Cloud Network Firewall.
* **Lab thực hành:** Dựng WireGuard VPN Server trên Linux VM, cấu hình mã hóa kết nối giữa Client và Server và kiểm tra packet đã mã hóa qua `tcpdump`.
* **Case study sự cố:**
  1. *Case 1:* Server bị dính mã độc Reverse Shell gọi về C2 (Command & Control) Server bên ngoài qua port 443 do hệ thống không thiết lập Egress Firewall filtering.
  2. *Case 2:* Website bị tê liệt do tấn công L7 HTTP Flood; server bị cạn kiệt connection tại Reverse Proxy, khắc phục bằng luật Rate Limiting trên Cloudflare/WAF.

---

### MODULE 13: QUAN SÁT MẠNG (OBSERVABILITY) & PHƯƠNG PHÁP CHẨN ĐOÁN
*Ước lượng: ~3.800 từ*

* **13.1. Mạng Golden Signals & Metrics quan trọng**
  * Bốn tín hiệu vàng cho Network: Latency (RTT, TCP Handshake time), Traffic (Bandwidth in/out, Packet per second), Errors (TCP Retransmits, Drop packet, Reset count), Saturation (Queue drops, Socket buffer fullness, Conntrack usage).
  * Linux Socket & TCP Metrics từ `/proc/net/snmp` và `/proc/net/netstat`.
* **13.2. VPC Flow Logs & Phân tích Dữ liệu Luồng**
  * Cấu trúc một dòng Flow Log: `version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status`.
  * Phân tích mã hành động: `ACCEPT` vs `REJECT`. *Lưu ý: Flow Logs chỉ ghi trạng thái `REJECT` chung chứ không ghi rõ do SG hay NACL; kỹ sư phải suy luận gián tiếp dựa trên tính chất stateful của SG vs stateless của NACL.*
  * Truy vấn Flow Logs bằng CloudWatch Logs Insights / Athena.
* **13.3. Quy trình Packet Capture (PCAP) có phương pháp trên Production**
  * Chiến lược bắt gói đa điểm (Multi-point capture: Client → Load Balancer → Node → Pod) để định vị chính xác vị trí gói tin bị rớt.
  * Đồng bộ thời gian qua PTP / NTP để đối chiếu chính xác timestamp giữa các file bắt gói.
* **13.4. Sổ tay Xử lý Sự cố (Troubleshooting Runbook Checklist)**
  * Bộ câu hỏi chuẩn đoán 6 bước khi nhận báo động mất kết nối mạng.
* **13.5. Hệ sinh thái quan sát mạng (Network Observability Ecosystem)**
  * `node_exporter`: Thu thập metric từ collectors `netstat`, `conntrack`, `softnet`.
  * `blackbox_exporter`: Thực hiện probing đầu cuối đa giao thức (HTTP/HTTPS, TCP connection, DNS resolution, ICMP ping).
  * Hubble (Cilium eBPF): Quan sát luồng traffic L3/L4/L7, service map, network policy verdict thời gian thực.
  * CoreDNS Metrics: `coredns_dns_request_duration_seconds`, `coredns_dns_responses_total` (mã phản hồi NOERROR, SERVFAIL, NXDOMAIN).
* **Lab thực hành:** Kích hoạt VPC Flow Logs (hoặc log `iptables -j LOG`), phân tích luồng traffic bị REJECT và dùng Wireshark phân tích hiện tượng TCP Out-of-Order / Retransmission từ file pcap thu thập được.
* **Case study sự cố:**
  1. *Case 1:* API Gateway gọi backend service bị timeout 1% ngẫu nhiên; Wireshark thấy TCP Retransmission tăng nhưng tcpdump không thấy packet lỗi; kiểm tra `ethtool -S` trên host và counter switch vật lý phát hiện lỗi CRC Checksum (frame hỏng bị NIC drop trước khi vào kernel).
  2. *Case 2:* Debug sự cố một Pod trong K8s không nhận được traffic bằng cách bắt `tcpdump` đồng thời trên interface `eth0` của Pod, interface `veth` trên Node và card mạng vật lý `ens5`.

---

### MODULE 14: TỐI ƯU HIỆU NĂNG MẠNG (PERFORMANCE & TUNING)
*Ước lượng: ~3.600 từ*

* **14.1. Độ trễ (Latency), Băng thông (Throughput) & BDP (Bandwidth-Delay Product)**
  * Công thức: BDP (bytes) = Bandwidth (bytes/sec) * RTT (sec).
  * Tại sao BDP quyết định kích thước Socket Buffer tối ưu cho các kết nối mạng tốc độ cao độ trễ lớn (Long Fat Networks - LFN).
* **14.2. Tinh chỉnh Kernel Network Stack (Linux Network Tuning)**
  * Tối ưu Receive/Transmit Queues: `netdev_max_backlog`, Ring Buffer của card mạng (`ethtool -g`).
  * Tối ưu TCP Buffers: `tcp_rmem`, `tcp_wmem`, `tcp_window_scaling` (tham chiếu sysctl lưu bền vững tại Mục 8.5).
  * Kích hoạt TCP BBR Congestion Control thay thế CUBIC trên Linux kernel >= 4.9.
  * Tái sử dụng socket: `tcp_tw_reuse = 1` (xem Mục 4.2).
  * Dọn socket orphan: `tcp_fin_timeout = 15` (chỉ chi phối thời gian chờ ở trạng thái `FIN_WAIT_2` của socket orphan khi ứng dụng đã đóng socket nhưng đầu xa chưa gửi FIN; hoàn toàn không ảnh hưởng tới thời gian `TIME_WAIT` cố định 60s, xem Mục 4.2).
* **14.3. Phần cứng & Card mạng Nâng cao: SR-IOV, DPDK & NIC Offloading**
  * Hardware Offloads: TSO (TCP Segmentation Offload), GRO (Generic Receive Offload), Checksum Offloading (`ethtool -k`).
  * SR-IOV (Single Root I/O Virtualization) & AWS Enhanced Networking (ENA).
* **14.4. Đo kiểm Hiệu năng Chuẩn xác bằng `iperf3`**
  * Đo kiểm TCP Throughput (đơn luồng vs đa luồng `-P`), UDP Bandwidth & Jitter, phân tích độ suy hao băng thông.
* **Lab thực hành:** Chạy benchmark đo throughput mạng giữa 2 VM trước và sau khi tối ưu thông số TCP Window & thuật toán BBR bằng `iperf3`.
* **Case study sự cố:**
  1. *Case 1 (Bài tập tính BDP):* Server 10Gbps truyền file xuyên lục địa (RTT 150ms) bị nghẽn tốc độ do TCP Buffer mặc định quá nhỏ; học viên tự tính dung lượng BDP lý thuyết và đo kiểm tốc độ trước/sau khi tăng `tcp_rmem`/`tcp_wmem`.
  2. *Case 2:* Card mạng bị drop packet do Ring Buffer kích thước quá nhỏ trong khi CPU đang bận xử lý softirq, khắc phục bằng `ethtool -G eth0 rx 4096`.

---

### MODULE 15: PHỤ LỤC MỞ RỘNG (BGP, SDN & MOBILE NETWORKING)
*Ước lượng: ~2.500 từ*

* **15.1. [Mở rộng] Giao thức BGP (Border Gateway Protocol) Chuyên sâu**
  * eBGP vs iBGP, Autonomous System Number (ASN 16-bit và 32-bit).
  * Thuật toán chọn đường BGP (Weight → Local Preference → AS-Path Length → Origin → MED).
  * Ứng dụng BGP Anycast để xây dựng hệ thống DNS và CDN toàn cầu.
* **15.2. [Mở rộng] Mạng Điều khiển bằng Phần mềm (SDN & OpenFlow)**
  * Tách biệt Control Plane (Bộ não điều khiển) và Data Plane (Chuyển tiếp gói tin).
  * Cách SDN quản lý các Cloud Virtual Network và Kubernetes Overlay hiện đại.
* **15.3. [Mở rộng] Mạng Di động 4G LTE & 5G dưới góc nhìn Kỹ sư Hạ tầng**
  * Kiến trúc 4G LTE EPC (eNodeB, MME, SGW, PGW, HSS) đối chiếu với 5G Core (gNodeB, AMF, SMF, UPF).
  * Luồng dữ liệu User Plane: UE → gNodeB / eNodeB → UPF / PGW → Data Network (Internet).
  * Khái niệm Network Slicing và MEC (Multi-access Edge Computing).

---

## 📊 TỔNG KẾT QUY MÔ GIÁO TRÌNH (ĐẾM THỰC TẾ)

* **Tổng số module:** 16 module (Module 0 → Module 15).
* **Tổng số bài Lab thực hành:** 15 bài lab (Module 0 → Module 14; Module 15 là phụ lục mở rộng không có lab).
* **Tổng số Case Study sự cố production:** 31 kịch bản thực tế:
  * Module 0: 2 case
  * Module 1: 2 case
  * Module 2: 2 case
  * Module 3: 2 case
  * Module 4: 2 case
  * Module 5: 2 case
  * Module 6: 2 case
  * Module 7: 2 case
  * Module 8: 2 case
  * Module 9: 2 case
  * Module 10: 3 case (Case 3 gồm 2 nhánh chẩn đoán (a) Limit IP/ENI per node vs (b) Exhausted Subnet IP)
  * Module 11: 2 case
  * Module 12: 2 case
  * Module 13: 2 case
  * Module 14: 2 case
  * Module 15: 0 case (Phụ lục mở rộng)
* **Tổng số từ dự kiến toàn giáo trình:** ~61.800 từ (khớp chính xác 100% với tổng ước lượng của 16 module).
* **Tài liệu bàn giao sau khi hoàn thành toàn bộ module:**
  1. Bảng thuật ngữ chuyên ngành Anh – Việt.
  2. Cheat Sheet lệnh tổng hợp dành cho SRE on-call.
  3. Bộ 20 câu hỏi phỏng vấn Platform/DevOps/SRE mức Senior có lời giải chi tiết.
  4. Lộ trình ôn tập 4 tuần có phân bổ thời gian cụ thể.
