# SEC_Lab
## Repository Structure & Architecture Mapping
**Repository này được tổ chức theo hướng platform-centric, trong đó mỗi nhóm file ánh xạ trực tiếp tới một thành phần trong kiến trúc runtime security của hệ thống.**
### falco/ – Container & Kubernetes Runtime Security

Thư mục falco/ chứa toàn bộ cấu hình liên quan đến Falco Runtime Security Control Plane và các agent chạy trong Kubernetes cluster.

|File|	Mô tả|
|----|------|
|falco-values.yaml|	File Helm values chính để triển khai Falco DaemonSet, cấu hình driver (modern eBPF), quyền privileged và tích hợp Kubernetes metadata|
|falco-custom-values.yaml|	Các rule runtime tự xây dựng để phát hiện hành vi bất thường trong container (spawn shell, network bất thường, truy cập file nhạy cảm, …)|
|falco-addon-sidekick-email.yaml|	Manifest triển khai Falcosidekick độc lập để routing alert từ Falco tới email (có thể mở rộng sang webhook/SIEM)

### wazuh/ – Node VM Runtime Telemetry (Sensor Layer)

Thư mục wazuh/ chứa các cấu hình phục vụ giám sát runtime ở cấp độ hệ điều hành của node VM.
Trong thiết kế này, Wazuh không đóng vai trò control plane, mà chỉ là node security sensor.

|File|	Mô tả|
|----|------|
|ossec.conf|	Cấu hình Wazuh Agent, bao gồm đọc audit log và File Integrity Monitoring|
### DEMO/
#### Demo 1 – Container Runtime (Falco)

- Bên trong container, việc cài và chạy nc để kết nối mạng là hành vi bất thường.

- Falco DaemonSet (chạy privileged, dùng eBPF) phát hiện:
 
  - Thực thi công cụ mạng (nc) → rule Suspicious Network Tool Executed in Container

  - Ghi file dưới /etc trong container → rule Write Under /etc in Container
    
    (Đây là rule custom)
 
- Falco sinh cảnh báo kèm Pod, Namespace, Image,...
#### Demo 2 – Node VM Runtime (Wazuh)

- Hành vi này tạo rồi xoá file trong /etc/sudoers.d/, là thư mục nhạy cảm liên quan đến privilege escalation.
- Wazuh Agent chạy trên Node VM với quyền root phát hiện ngay thông qua File Integrity Monitoring (FIM) realtime, sinh cảnh báo khi file được tạo và bị xoá.


## 1. Kiến trúc Tổng thể (Architecture Overview)
**Hệ thống được thiết kế theo mô hình Defense-in-Depth, tách biệt giám sát thành 2 tầng lớp rõ ràng:**
 ``` 
1.	Tầng Host/Node Security (Wazuh Agent): Chịu trách nhiệm bảo vệ hệ điều hành nền tảng (OS), giám sát các tiến trình hệ thống, đăng nhập SSH và toàn vẹn file hệ thống (FIM),...
2.	Tầng Workload/Container Security (Falco): Chịu trách nhiệm giám sát runtime của các container, phát hiện các hành vi bất thường bên trong Pod,... và audit cấu hình Kubernetes.
 ```
### Architecture – Current Implementation
<img width="624" height="711" alt="image" src="https://github.com/user-attachments/assets/3ca60ae3-2f83-4819-88b6-2deb7c308599" />

### Sơ đồ vị trí Agent:
**Wazuh Agent:**
 ``` 
  •	Vị trí: Cài trực tiếp trên Kubernetes Node (VM).
  •	Loại hình: Systemd Service (Process nền của Linux).
  •	Quyền hạn: Chạy với quyền root để có thể đọc auditd logs và quét toàn bộ filesystem.
  •	Vai trò chính: Agent này đóng vai trò là node-level security sensor, chịu trách nhiệm giám sát các hành vi runtime của hệ điều hành nền bên dưới Kubernetes.
```
**Falco Agent:**
```
  •	Vị trí: Triển khai dạng DaemonSet trên Kubernetes Cluster (mỗi Node một Pod).  
  •	Loại hình: Pod/Container.  
  •	Quyền hạn: Chạy privileged: true; để load kernel module/eBPF probe nhằm bắt syscalls của các container khác.  
  •	Vai trò chính: Bắt syscalls container qua kernel module/eBPF, tích hợp K8s Metadata.
```
## 2. Chi tiết Triển khai & Cấu hình (Implementation Details)
### A. Giám sát Node VM (Wazuh)
**Giám sát truy cập và thay đổi file nhạy cảm (File Integrity Monitoring – FIM)**
```
Theo dõi realtime các thư mục và file quan trọng như:
- /etc
- /etc/containerd
- /var/lib/kubelet/pki,...
Qua đó phát hiện các thay đổi cấu hình ảnh hưởng trực tiếp đến bảo mật node, runtime.
```
**Thu thập telemetry runtime của hệ điều hành**
```
Đọc log từ Linux Auditd và các system log để ghi nhận:
- Process execution
- Network activity
- Các sự kiện hệ thống quan trọng
```

**Cung cấp System Context**
```
Thu thập inventory của node VM bao gồm hệ điều hành, tiến trình, cổng mạng và gói cài đặt, giúp hiểu rõ trạng thái runtime của node tại thời điểm xảy ra sự kiện.
```

### B. Giám sát Workload Container (Falco)
Falco được cấu hình sử dụng kết hợp 2 bộ rule:
- Default Rules: Bộ rule chuẩn của cộng đồng (phát hiện shell, crypto mining, container escape cơ bản).
- Custom Rules (tragtt-runtime-rules.yaml): Các rule tự phát triển tập trung vào các kịch bản cụ thể như:

  •	Spawn interactive shell trong namespace Production.
  
  •	Truy cập Cloud Metadata IP (169.254.169.254).
  
  •	Sử dụng công cụ mạng (netcat, nmap) trong container.

## 3. Luồng dữ liệu End-to-End (Data Flow)

**Hệ thống xử lý hai luồng dữ liệu song song:**
##### Luồng 1: Node VM Monitoring (Wazuh)
```
1.	Thu thập: Wazuh Agent chạy trên Node VM với quyền root thu thập telemetry runtime từ hệ điều hành, bao gồm auditd (process, network) và File Integrity Monitoring (FIM) đối với các file nhạy cảm,...
2.	Chuyển tiếp: Agent mã hóa dữ liệu và gửi qua TCP port 1514 về Wazuh Manager (Control Plane).
3.	Phân tích: Wazuh Manager giải mã (Decoder) và đối chiếu với Rule.
4.	Lưu trữ & Hiển thị: Cảnh báo được lưu vào Indexer (Elasticsearch/OpenSearch) và hiển thị trên Wazuh Dashboard.
```
##### Luồng 2: Workload & Runtime Security (Falco)
```
1.	Thu thập:
  •	Syscall: Falco Driver (eBPF) bắt các lệnh gọi hệ thống từ Container.
  •	K8s Audit: k8s-metacollector thu thập metadata và logs từ K8s API Server.
2.	Phân tích: Falco Engine so sánh sự kiện với file Rule (default + custom).
3.	Xử lý: Nếu vi phạm, Falco đẩy log JSON sang Falcosidekick.
4.	Báo cáo: Falcosidekick format lại bản tin và gửi cảnh báo qua SMTP (Gmail) tới quản trị viên.
```
## 4. Design Decisions
```
•	Tại sao dùng Falco cho Container: Falco thấu hiểu ngữ cảnh Kubernetes (Namespace, Pod Name, Image) tốt hơn Wazuh. Wazuh chỉ nhìn thấy process ID trên host mà không biết process đó thuộc Pod nào.
•	Tại sao dùng Wazuh cho Node: Falco tập trung vào syscall stream, không mạnh về File Integrity Monitoring (FIM) hay log analysis của OS (như SSH login failure) bằng Wazuh. Kết hợp cả hai lấp đầy các điểm mù.
•	Kênh cảnh báo (Alerting): Sử dụng Email cho các cảnh báo runtime từ Falco để đảm bảo tính tức thời (Real-time).
```
## 5. Scalability & Bottlenecks
**Nếu hệ thống mở rộng lên hàng trăm nodes hoặc triển khai Multi-tenant, các vấn đề sau sẽ phát sinh:**
```
1.	Nút thắt cổ chai tại Control Plane (Bottleneck):
  •	Wazuh Manager: Là điểm tập trung duy nhất. Nếu quá nhiều Agent gửi log cùng lúc, Manager sẽ bị quá tải (CPU/Network). -> Giải pháp: Cần triển khai Wazuh Cluster (Master/Worker) phía sau Load Balancer, đồng thời triển khai Indexer theo dạng cluster để đảm bảo khả năng scale và chịu tải. 
  •	K8s Metacollector: Khi cluster quá lớn, lượng event từ K8s API Server rất khủng khiếp, Pod này có thể bị OOM (Out Of Memory).
2.	Vấn đề Multi-tenant (Nhiều team dùng chung):
  •	Hiện tại Falco Rules áp dụng cho toàn bộ Cluster. Khó để team A có bộ luật riêng khác team B.
  •	Cảnh báo Email đang gửi chung vào một hộp thư. Cần định tuyến (Routing) alert dựa trên Namespace để gửi đúng cho team phụ trách.
3.	Hiệu năng Node:
  •	Falco chạy eBPF/Kernel module có thể chiếm 1-3% CPU. Nếu lưu lượng syscall quá lớn (ví dụ Database High-load), Falco có thể làm chậm ứng dụng (drop syscalls).
```
## 6. Future Upgrade

### Data Flow Explanation (Current)
Falco giám sát container runtime và sinh alert độc lập

Wazuh giám sát node VM runtime và có control plane riêng

❌ Hai pipeline song song, không correlation node ↔ container

### Architecture – Target Design
Falco = Runtime Security Control Plane duy nhất

Wazuh = Node-level security sensor

<img width="585" height="713" alt="image" src="https://github.com/user-attachments/assets/7711ab20-6a9e-47d7-960a-40d9e56ecda8" />


### Data Flow Explanation
#### Luồng A – Container Runtime
```
Container

  → Kernel syscall
  
    → Falco DaemonSet
    
      → Falco Control Plane
      
        → Falcosidekick
        
          → Alert / Report
```

- Phát hiện: shell spawn, network bất thường, file access,...
- Có đầy đủ metadata: Pod, Namespace, Image


#### Luồng B – Node VM Runtime

```
OS Process / File / Network

  → Auditd
  
    → Wazuh Agent (chuẩn hoá JSON)
    
      → Falco Control Plane
      
        → Falcosidekick
        
          → Alert / Report
```
- Wazuh không còn là control plane
- Chỉ đóng vai trò node security sensor

