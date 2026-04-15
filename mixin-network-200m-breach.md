# Mixin Network — Phân tích sự cố $200M (09/2023)

> **Loại tấn công:** Supply Chain / Cloud Infrastructure Compromise  
> **Thiệt hại:** ~200M USD (400k ETH + BTC + ERC-20 assets)  
> **Tác nhân nghi vấn:** Lazarus Group (Triều Tiên)

---

## 1. Tổng quan

**Thời gian:** ~04:00 UTC+8, ngày 23/09/2023

**Đối tượng bị tấn công:** Cloud Database và Google Cloud Storage (GCS) của Mixin Network — nơi lưu trữ cấu hình hệ thống và trạng thái tài sản người dùng (User Ledger).

**Hậu quả:**
- Toàn bộ tài sản người dùng bị đóng băng trong thời gian dài
- Mixin chỉ có thể hoàn trả tối đa 50% giá trị bằng tiền mặt
- Hệ thống Mixin Kernel phải tạm dừng toàn bộ dịch vụ nạp/rút
- Niềm tin vào mô hình "Light-node" và bảo mật Cloud của Mixin sụp đổ

**Bản chất:** Đây là tấn công Supply Chain cấp độ hạ tầng. Thay vì bẻ khóa thuật toán đồng thuận, Lazarus nhắm vào **Control Plane** — nơi quản lý trạng thái ví và logic phê duyệt rút tiền.

---

## 2. Kill Chain

### Bước 1 — Initial Access (Phishing & Social Engineering)

Một kỹ sư DevOps/SRE có đặc quyền cao bị tiếp cận qua LinkedIn dưới danh nghĩa nhà tuyển dụng kỹ thuật. Kỹ sư này được yêu cầu tải và chạy một project để làm bài kiểm tra kỹ năng.

**Payload:** `MC-Based-Stock-Invest-Simulator-main.zip` — chứa mã độc RCE thông qua thư viện **PyYAML không an toàn** (`yaml.load()` thay vì `yaml.safe_load()`), chiếm quyền điều khiển máy trạm.

### Bước 2 — Credential Harvesting & Persistence

- Mã độc quét máy trạm, thu giữ tệp cấu hình Cloud và khóa SSH
- Hacker sử dụng **Session Token có thời hạn dài** để giả danh kỹ sư
- **MFA bypass:** phiên làm việc đã được xác thực trước đó → token vẫn hợp lệ

### Bước 3 — Database & Cloud Manipulation

- **Reconnaissance:** tải xuống mã nguồn frontend và logic phê duyệt
- **Data Injection:** can thiệp vào Ledger database tập trung — nơi lưu số dư và logic xác thực
- **Blind Signing:** sửa đổi file JavaScript production (`_app-.js`) trên Cloud Storage để thay đổi địa chỉ ví nhận trong payload giao dịch, **nhưng giữ nguyên hiển thị trên UI** của quản trị viên

### Bước 4 — Execution & Exfiltration

- Khi nhân viên thực hiện lệnh nạp/rút định kỳ, hệ thống MPC tự động ký vào payload đã bị thay đổi địa chỉ
- Tài sản chuyển thẳng đến ví của hacker, tẩu tán qua giao thức Multi-chain trong vài phút
- Mixin phát hiện khi số dư ví nóng sụt giảm mạnh và Ledger không giải trình được → đóng băng cổng rút nhưng 200M USD đã tẩu tán qua Solana và Ethereum

---

## 3. MITRE ATT&CK Mapping

| Tactic | Technique | Hành động |
|--------|-----------|-----------|
| Initial Access | T1566.002 (Spearphishing Link) | Gửi dự án mã độc qua LinkedIn |
| Execution | T1059 (Command & Scripting Interpreter) | Chạy loader Python chiếm quyền máy trạm |
| Credential Access | T1528 (Steal Application Access Token) | Ăn cắp Session Token từ tệp credentials Cloud |
| Persistence | T1078.004 (Valid Accounts: Cloud Accounts) | Dùng token hợp lệ truy cập Cloud console nhiều lần |
| Impact | T1565.001 (Stored Data Manipulation) | Thay đổi JS tĩnh để thao túng địa chỉ ví nhận |

---

## 4. Root Causes

**1. Privileged Access không được kiểm soát**  
Kỹ sư bị tấn công có quyền sửa đổi trực tiếp Production Storage từ máy cá nhân, không qua CI/CD được kiểm soát.

**2. Mô hình Hybrid sai lầm**  
Mixin dùng Database Cloud tập trung để quản lý tài sản phi tập trung → Single Point of Failure. Mô hình an toàn hơn cần FROST/MPC với các mảnh khóa lưu ở các vùng hạ tầng độc lập (AWS + Azure + On-premise).

**3. Thiếu Real-time Reconciliation**  
Hệ thống không tự động so sánh số dư Ledger với Blockchain định kỳ. Nếu có, Safe Mode sẽ kích hoạt ngay khi 1M USD đầu tiên bị rút sai lệch.

**4. Thiếu Integrity check cho dữ liệu tĩnh**  
Không có cơ chế kiểm tra tính toàn vẹn của tệp cấu hình và JS trên Cloud Storage → hacker sửa đổi tự do mà không bị phát hiện ngay.

---

## 5. Lessons & Controls

### Active Defense

- **Automated Kill-Switch:** đóng băng toàn bộ cổng rút tiền trong < 60 giây khi phát hiện bất thường
- **Moving Target Defense (MTD):** Resharing định kỳ trong MPC để thay đổi mảnh khóa mà không thay đổi public key — vô hiệu hóa dữ liệu hacker đã thu thập
- **Honey-Nonces:** phát hành giao dịch mồi có nonce bị "lệch bit" để phát hiện Lattice Attack

### Real-time Reconciliation

- **Ledger ↔ Blockchain:** tự động so sánh số dư nội bộ với Blockchain mỗi 5-10 phút; sai lệch → kích hoạt Safe Mode
- **Static Asset Integrity:** kiểm tra hash của tệp tĩnh trên Cloud Storage; thay đổi ngoài CI/CD → chặn ngay

### Infrastructure Hardening

| Layer | Control |
|-------|---------|
| Wallet | Hot (<5%) qua MPC · Warm (2-of-3 multisig) · Cold (>90%, air-gapped, 3-of-5 approval) |
| Workstation | Máy chuyên dụng: không email, không web, cấu hình đóng băng |
| Network | VPN nội bộ + IP Whitelist + FIDO2/Passkey cho quản trị |
| CI/CD | IAM Policy chặn `s3:PutObject` trực tiếp từ tài khoản cá nhân lên Production |

---

## 6. So sánh với các sự cố khác

| | Mixin (2023) | Bybit (2025) | Upbit (2019) |
|---|---|---|---|
| **Vector** | Cloud DB + JS manipulation | Supply chain JS | Hot wallet drain |
| **Entry point** | DevOps workstation | Frontend engineer | Nội bộ không rõ |
| **Root cause** | Centralized ledger + weak IAM | Compromised JS library | Hot wallet exposure |
| **Attribution** | Lazarus Group | Lazarus Group | Lazarus Group |
| **Thiệt hại** | $200M | $1.5B | $50M |
| **Điểm chung** | MPC không bảo vệ được Control Plane khi bị Blind Signing | | |

---

*Phân tích dựa trên các nguồn công khai: thông báo chính thức của Mixin Network, on-chain data, và báo cáo từ Chainalysis, Elliptic.*
