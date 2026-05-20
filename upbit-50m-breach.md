# Upbit — Phân tích sự cố $50M (11/2019)

> **Loại tấn công:** Hot Wallet Drain — Private Key Compromise  
> **Thiệt hại:** 342,000 ETH (~41.5M USD tại thời điểm xảy ra)  
> **Tác nhân:** Lazarus / Andariel Group (DPRK)

---

## 1. Tổng quan

**Thời gian:** Tháng 11/2019

Upbit là sàn giao dịch hàng đầu Hàn Quốc, vận hành bởi Dunamu Inc., đạt chuẩn ISO/ISMS. Tuy nhiên, toàn bộ 342,000 ETH từ ví nóng bị rút trái phép trong một giao dịch duy nhất — do attacker kiểm soát được private key hoặc credentials vận hành ví.

So với Mixin (2023) và Bybit (2025), vụ Upbit 2019 đơn giản hơn về kỹ thuật: không có supply chain attack phức tạp mà là một hot wallet drain cổ điển thông qua key compromise.

---

## 2. Kill Chain

### Bước 1 — Initial Access
Attacker xâm nhập qua phishing hoặc malware nhắm vào máy trạm của nhân viên vận hành ví hoặc quản trị viên hệ thống.

### Bước 2 — Key Compromise
Thu giữ private key, MPC shares, hoặc credentials của người vận hành HSM/ví.

### Bước 3 — Lateral Movement
Thâm nhập sâu vào mạng nội bộ để tiếp cận "control plane" của ví — nơi chứa chính sách phê duyệt và danh sách whitelist.

### Bước 4 — Execution
Dùng key đã chiếm được để ký lệnh rút tiền. Vì được ký bởi key hợp lệ, hệ thống coi đây là giao dịch hợp pháp và cho phép drain hot wallet.

### Bước 5 — Laundering
Tiền chuyển nhanh qua multi-chain, qua DeFi protocols và các sàn khác để làm mờ dấu vết — pattern điển hình của Lazarus Group.

---

## 3. Root Causes

**1. Hot wallet exposure quá lớn**  
Lưu trữ quá nhiều tài sản trong hot wallet mà không có hạn mức (cap) hoặc cơ chế tự động chuyển (sweep) về cold wallet dựa trên ngưỡng số dư.

**2. Key management yếu**  
Private key hoặc thành phần key lưu trong môi trường thiếu bảo vệ bởi HSM. Key tồn tại ở dạng plaintext trong memory hoặc trên disk.

**3. Quy trình phê duyệt lỏng lẻo**  
Thiếu Segregation of Duties và Out-of-band verification cho giao dịch giá trị lớn. Một giao dịch drain 342,000 ETH không bị gắn cờ và yêu cầu phê duyệt đa bên.

**4. Giám sát không đủ nhạy**  
Hệ thống không phát hiện kịp thời hành vi rút tiền bất thường để kích hoạt kill-switch. Khi phát hiện ra thì 342,000 ETH đã chuyển đi.

---

## 4. So sánh với Mixin và Bybit

| | Upbit (2019) | Mixin (2023) | Bybit (2025) |
|---|---|---|---|
| **Entry point** | Workstation nhân viên | DevOps workstation | Developer Safe{Wallet} |
| **Kỹ thuật** | Key theft / hot wallet drain | Cloud DB + JS manipulation | Supply chain JS injection |
| **Root cause** | Key management yếu | Centralized ledger | No static asset integrity |
| **Phát hiện** | Sau khi drain xong | Vài phút sau khi drain | Sau khi drain xong |
| **Attribution** | Lazarus / Andariel | Lazarus | Lazarus |

**Điểm chung của cả 3:** Lazarus Group nhắm vào **người vận hành** (developer, DevOps, admin) — không phải vào giao thức hay smart contract. MPC và multi-sig không phải "viên đạn bạc" nếu kẻ tấn công kiểm soát được người ký hoặc UI signing.

---

## 5. Controls

### Wallet Architecture
| Layer | Tỷ lệ | Yêu cầu |
|-------|--------|---------|
| Hot Wallet | < 5% | MPC, velocity limit, auto-sweep |
| Warm Wallet | Thanh khoản ngày | 2-of-3 multisig, SoD |
| Cold Wallet | > 90% | Air-gapped, 3-of-5 physical approval |

### Key Management
- **HSM:** private key không tồn tại ở dạng plaintext trong memory
- **MPC sharding:** chia mảnh key cho nhiều người giữ độc lập, loại bỏ Single Point of Failure
- **Key rotation** định kỳ

### Velocity Controls
- Hạn mức rút tiền theo phút/giờ/ngày cho từng loại tài sản
- Vượt ngưỡng → tự động tạm dừng, yêu cầu kiểm tra thủ công

### Approval Protocol
- **Maker (Ops) → Checker (Compliance) → Approver (Management)** cho giao dịch lớn
- Out-of-band verification qua kênh thứ hai độc lập
- Không ai giữ đồng thời 2 vai trò

### Active Defense
- **Kill-Switch < 60s:** đóng băng cổng rút khi phát hiện bất thường
- **Real-time Reconciliation:** so sánh Ledger nội bộ với Blockchain mỗi 5-10 phút; sai lệch → Safe Mode

---

*Nguồn: Thông báo chính thức của Upbit/Dunamu, báo cáo từ Chainalysis và các nhà nghiên cứu blockchain.*
