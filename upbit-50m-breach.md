# Upbit — Phân tích sự cố $50M (11/2019)

> **Loại tấn công:** Hot Wallet Drain — Private Key / Signing Key Compromise  
> **Thiệt hại:** 342,000 ETH (~$41.5M tại thời điểm; ~$1.1B tại giá ETH 2024)  
> **Tác nhân:** Lazarus / Andariel Group (DPRK) — xác nhận bởi FBI và KISA (2020)  
> **Thời điểm:** 13:06 KST, ngày 27/11/2019

---

## 1. Tổng quan

Upbit là sàn giao dịch crypto lớn nhất Hàn Quốc, vận hành bởi Dunamu Inc., đạt chuẩn ISO/ISMS. Toàn bộ 342,000 ETH bị rút ra từ hot wallet trong **một giao dịch duy nhất** — điều này cho thấy kẻ tấn công đã nắm quyền kiểm soát hoàn toàn signing key của hot wallet trước khi thực hiện drain.

Giao dịch drain có thể xem công khai trên Etherscan:  
- **Từ:** `0x6cC5F688a315f3dC28A7781717a9A798a59fDA7b` (Upbit hot wallet)  
- **Đến:** `0xa09871AEadF4994Ca12f5c0b6056BBd1d343c029` (ví attacker)  
- **Giá trị:** 342,000 ETH trong 1 transaction

Upbit phát hiện sự cố khoảng 2 giờ sau khi giao dịch được thực hiện. Tất cả hoạt động nạp/rút bị đình chỉ trong 2 tuần. Lazarus Group sau đó phân phối ETH qua nhiều ví trung gian và laundering qua các DEX và privacy services.

---

## 2. Kill Chain

> **Lưu ý:** Upbit không công bố chi tiết kỹ thuật về vector tấn công ban đầu. Kill chain dưới đây được tái tạo từ các pattern đã biết của Lazarus Group và phân tích on-chain của Chainalysis, FBI, và các nhà nghiên cứu độc lập.

### Bước 1 — Initial Access (Phishing / Malware)

Dựa trên TTP nhất quán của Lazarus Group trong các vụ tấn công CEX cùng giai đoạn:

- **Spearphishing** nhắm vào nhân viên vận hành ví hoặc admin hệ thống, thường qua LinkedIn job scam hoặc email giả mạo đối tác
- Hoặc **malicious document** khai thác Office macro / PDF exploit được gửi qua email
- Mục tiêu: đặt foothold trên máy trạm của người có quyền ký giao dịch hoặc truy cập credentials hệ thống ví

### Bước 2 — Credential & Key Harvesting

Sau khi compromise được máy trạm:

- Dump AWS credentials / cloud config files
- Thu giữ private key, keystore file, hoặc thông tin đăng nhập HSM của người vận hành ví
- Hoặc đánh cắp session token / API key có quyền khởi tạo giao dịch

### Bước 3 — Lateral Movement & Reconnaissance

- Di chuyển trong mạng nội bộ để tiếp cận "control plane" của ví
- Xác định hot wallet address và số dư
- Nghiên cứu quy trình phê duyệt: cần bao nhiêu chữ ký, ai ký, qua interface nào

### Bước 4 — Execution

- Dùng key / credentials đã thu giữ để ký và broadcast giao dịch drain 342,000 ETH
- Giao dịch hợp lệ về mặt kỹ thuật (ký bởi key thật) → hệ thống không có cơ chế chặn

### Bước 5 — Laundering

- Ngay sau drain, 342,000 ETH được chia nhỏ và chuyển qua nhiều ví trung gian
- Một phần được convert sang BTC qua atomic swap
- Laundering qua Tornado Cash (trước khi bị OFAC sanction), Ren Protocol, và các CEX nhỏ ở Đông Nam Á không có KYC chặt
- FBI và DOJ thu hồi được ~$8.3M qua phối hợp quốc tế (2022-2023)

---

## 3. MITRE ATT&CK Mapping

| Tactic | Technique | Hành động |
|--------|-----------|-----------|
| Initial Access | T1566 — Phishing | Spearphishing nhắm nhân viên vận hành ví |
| Initial Access | T1195 — Supply Chain Compromise | (Có thể) compromise qua third-party tool/vendor |
| Execution | T1059 — Command & Scripting | Chạy malware/loader trên máy trạm bị compromise |
| Credential Access | T1555 — Credentials from Password Stores | Thu giữ stored credentials, keystore files |
| Credential Access | T1528 — Steal Application Access Token | Lấy API key / session token có quyền ký giao dịch |
| Discovery | T1087 — Account Discovery | Xác định người có quyền ký và quy trình phê duyệt |
| Discovery | T1083 — File and Directory Discovery | Tìm private key files, wallet config |
| Lateral Movement | T1021 — Remote Services | Di chuyển qua SSH / RDP trong mạng nội bộ |
| Collection | T1560 — Archive Collected Data | Thu thập và nén credentials trước khi exfil |
| Exfiltration | T1041 — Exfiltration Over C2 Channel | Gửi key/credentials về C2 infrastructure |
| Impact | T1657 — Financial Theft | Drain 342,000 ETH từ hot wallet |
| Impact | T1565.001 — Stored Data Manipulation | Có thể thay đổi whitelist address trước khi thực hiện |

---

## 4. Root Causes

**1. Hot wallet chứa quá nhiều tài sản**  
342,000 ETH (~12% tổng ETH lưu thông tại thời điểm đó) trong hot wallet không có hard cap. Upbit sau này công bố chuyển sang mô hình chỉ giữ <5% tài sản trong hot wallet.

**2. Key management yếu**  
Private key hoặc signing credentials của hot wallet không được bảo vệ bởi HSM hoặc multi-party computation. Một điểm compromise duy nhất đủ để drain toàn bộ.

**3. Không có velocity control**  
Một giao dịch chuyển 342,000 ETH (tương đương 100% số dư hot wallet) không bị gắn cờ và yêu cầu xác nhận bổ sung. Không có threshold nào kích hoạt alert trước khi giao dịch được broadcast.

**4. Thiếu Out-of-band verification**  
Không có cơ chế xác nhận giao dịch giá trị lớn qua kênh thứ hai độc lập với hệ thống signing.

**5. Monitoring không đủ nhạy**  
Upbit phát hiện sự cố ~2 giờ sau khi giao dịch on-chain. Không có real-time reconciliation giữa internal ledger và on-chain state.

---

## 5. So sánh với Mixin (2023) và Bybit (2025)

| | Upbit (2019) | Mixin (2023) | Bybit (2025) |
|---|---|---|---|
| **Thiệt hại** | $50M | $200M | $1.5B |
| **Entry point** | Workstation / nội bộ | DevOps workstation | Developer Safe{Wallet} |
| **Vector** | Key theft + hot wallet drain | Cloud DB + JS manipulation | Supply chain JS injection |
| **Root cause chính** | Hot wallet over-exposure + weak key mgmt | Centralized ledger + no asset integrity | No static asset integrity + blind signing |
| **Phát hiện** | ~2 giờ sau | Vài phút sau khi drain | Sau khi drain xong |
| **Attribution** | Lazarus / Andariel | Lazarus | Lazarus |

**Pattern nhất quán:** Lazarus Group không tấn công protocol hay smart contract mà nhắm vào **con người có đặc quyền** (developer, admin, operator). MPC và multi-sig bảo vệ key nhưng không bảo vệ được khi attacker kiểm soát máy của người ký hoặc can thiệp vào UI signing.

---

## 6. Controls

### Wallet Architecture
```
Cold Wallet (>90%)
├── Air-gapped
├── Physical key ceremony (3-of-5 approval)
└── No network connection

Warm Wallet
├── 2-of-3 multisig
├── Requires SoD approval
└── Daily sweep limit

Hot Wallet (<5%)
├── MPC-protected signing
├── Per-transaction velocity limit
└── Auto-sweep to warm when balance > threshold
```

### Velocity Controls
- Hard limit: giao dịch > X ETH → mandatory manual review
- Time-based: tổng withdrawal > Y ETH/hour → automatic pause
- Address-based: giao dịch đến địa chỉ mới chưa trong whitelist → holding period 24-48h

### Key Management
- **HSM** cho private key — key không tồn tại ở dạng plaintext trong RAM hay disk
- **MPC sharding** — chia mảnh key cho nhiều người giữ độc lập, loại bỏ Single Point of Failure
- Key rotation định kỳ (MPC resharing)

### Approval Protocol
```
Giao dịch < $10K    → Tự động (velocity check only)
$10K - $100K        → Maker + Checker (2 người)
$100K - $1M         → Maker + Checker + Security review
> $1M               → Maker + Checker + Approver + Out-of-band verification
```

### Real-time Reconciliation
- So sánh Internal Ledger với on-chain state mỗi 5 phút
- Sai lệch → kích hoạt Safe Mode, pause withdrawals
- Alert đến on-call security team trong < 60 giây

---

## 7. Aftermath

Upbit công bố sau sự cố đã:
- Chuyển sang mô hình ví phân tầng — 70% tài sản trong cold wallet
- Tăng cường bảo mật hot wallet với MPC
- Nâng cấp monitoring và reconciliation system
- Mở rộng đội ngũ security và threat intelligence

Vụ việc là một trong các trigger chính khiến ngành CEX chuyển đổi từ hot wallet đơn giản sang MPC + cold storage architecture.

---

*Nguồn: FBI PSA I-040920-PSA (April 2020), KISA security advisory, Chainalysis 2020 Crypto Crime Report, on-chain data từ Etherscan, ZachXBT independent analysis.*
