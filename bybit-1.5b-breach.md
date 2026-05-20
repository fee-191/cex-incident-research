# Bybit — Phân tích sự cố $1.5B (02/2025)

> **Loại tấn công:** Supply Chain — Safe{Wallet} Frontend Compromise  
> **Thiệt hại:** ~1.5 tỷ USD (~401,347 ETH + các tài sản khác)  
> **Tác nhân:** Lazarus Group / TraderTraitor (DPRK)

---

## 1. Tổng quan

**Thời gian:** 21/02/2025, ~14:13 UTC

Đây là vụ hack lớn nhất lịch sử crypto tính đến thời điểm xảy ra. Bybit không bị tấn công trực tiếp — kẻ tấn công xâm nhập vào **Safe{Wallet}**, provider ví multi-sig mà Bybit sử dụng để quản lý cold wallet, rồi thao túng giao diện signing để nhân viên Bybit ký nhầm giao dịch chuyển quyền kiểm soát ví cho attacker.

Chuỗi tấn công kéo dài từ 04/02 đến 21/02/2025 — 17 ngày chuẩn bị trước khi drain.

---

## 2. Timeline & Kill Chain

### 04/02/2025 — Compromise máy developer Safe{Wallet}

Một developer của Safe{Wallet} bị tấn công qua **job scam trên LinkedIn/GitHub/Telegram**. Attacker mời developer tải về một project kiểm tra kỹ năng: `MC-Based-Stock-Invest-Simulator-main.zip`.

Project này chứa mã độc khai thác **PyYAML RCE** (`yaml.load()` không dùng SafeLoader):
```python
# Vulnerable
data = yaml.load(stream)

# Safe
data = yaml.safe_load(stream)
```

Payload tải Poseidon/MythicAgents về RAM, kết nối C2 tại `getstockprice[.]com`, đánh cắp AWS credentials từ `~/.aws`.

### 05–17/02/2025 — Reconnaissance & Preparation

- Dùng AWS session token (stolen) để login vào AWS infrastructure của Safe{Wallet} qua ExpressVPN + Kali Linux user-agent
- Thử thêm MFA device vào tài khoản → thất bại
- Liệt kê IAM roles, S3 buckets, phân tích codebase Next.js của Safe{Wallet}
- Viết malicious JavaScript payload **chỉ nhắm riêng vào địa chỉ ví cold wallet của Bybit**

Session token AWS hết hạn sau 12 giờ → attacker phải lấy token mới liên tục, dùng:
```bash
aws sts get-session-token --serial-number "$ARN" \
  --token-code "$FINAL_CODE" \
  --duration-seconds 43200
```

### 19/02/2025 — Inject malicious JavaScript

Attacker ghi đè file JavaScript production trên S3 bucket (`_app-52c9031bfa03da47.js`). Code độc chỉ kích hoạt khi phát hiện giao dịch từ ví Bybit, và thực hiện:

1. Lưu bản sao giao dịch gốc
2. **Thay thế địa chỉ nhận** (recipient address) bằng địa chỉ của attacker
3. Gửi payload đã sửa để người ký phê duyệt
4. Khi ký xong → **hoàn trả giao diện về giao dịch gốc** để che giấu với người ký tiếp theo

Người ký thấy giao dịch hợp lệ trên UI, ký qua Ledger/Trezor mà không phát hiện payload đã bị thay đổi.

### 21/02/2025 — Drain & Cover Tracks

- Nhân viên Bybit thực hiện lệnh chuyển ETH định kỳ từ cold sang hot wallet
- Malicious JS kích hoạt → blind signing → overwrite `masterCopy` slot → attacker kiểm soát ví
- ~401,347 ETH + stETH + các tài sản khác bị drain
- **2 phút sau:** attacker xóa malicious JS khỏi S3 để xóa dấu vết

---

## 3. MITRE ATT&CK Mapping

| Tactic | Technique | Hành động |
|--------|-----------|-----------|
| Reconnaissance | T1595 — Active Scanning | Theo dõi Bybit trên Etherscan, dò GitHub |
| Resource Development | T1583 — Acquire Infrastructure | Đăng ký domain C2 `getstockprice.com` |
| Resource Development | T1608 — Stage Capabilities | Fork GitHub project hợp lệ để tạo lure |
| Initial Access | T1566 — Phishing | Job scam qua LinkedIn/Telegram |
| Initial Access | T1190 — Exploit Public-Facing App | RCE PyYAML via `yaml.load()` |
| Execution | T1059 — Command & Scripting | Loader Python + Poseidon/MythicAgents |
| Execution | T1609 — Container Administration | Docker privileged container escape |
| Persistence | T1078 — Valid Accounts | Dùng stolen AWS session token |
| Credential Access | T1003 — OS Credential Dumping | Steal `~/.aws/credentials` |
| Discovery | T1087 — Account Discovery | Enum IAM users/roles/policies |
| Discovery | T1083 — File & Directory Discovery | Tìm S3 bucket có quyền ghi |
| C2 | T1071 — Application Layer Protocol | HTTPS polling đến C2 |
| C2 | T1573 — Encrypted Channel | XOR encrypt exfil data |
| Impact | T1565.001 — Stored Data Manipulation | Inject JS thay đổi recipient address |

---

## 4. Root Causes

**1. Developer có quyền ghi trực tiếp vào Production S3**  
Không cần qua CI/CD pipeline → một máy bị compromise là toàn bộ frontend bị kiểm soát.

**2. Không có Integrity check cho static assets**  
Safe{Wallet} không verify hash/signature của JS trước khi serve → attacker ghi đè tự do.

**3. Blind signing**  
Ledger/Trezor hiển thị raw transaction hash, không decode và hiển thị ý nghĩa giao dịch → người dùng ký mà không hiểu nội dung thật.

**4. AWS session token tồn tại quá lâu**  
`--duration-seconds 43200` (12 tiếng) cho human user workstation → window đủ lớn để attacker thực hiện reconnaissance.

---

## 5. Controls

### Static Asset Protection
- **Subresource Integrity (SRI):** browser verify hash của script trước khi execute
- **S3 Object Lock:** chặn ghi đè file JS ngoài CI/CD pipeline
- **Content-Security-Policy (CSP):** giới hạn script execution source

### Signing Security
- **Out-of-band Verification:** giao dịch lớn yêu cầu xác nhận qua kênh thứ hai độc lập (video call, hardware device riêng)
- **Intent Verification:** người ký xác nhận 3 thông tin trên màn hình riêng: địa chỉ nhận + số lượng + loại tài sản
- **Segregation of Duties:** Maker (Ops) → Checker (Compliance) → Approver (Management) — không ai giữ 2 vai trò

### Infrastructure
- **Zero Direct Access:** IAM policy chặn `s3:PutObject` từ tài khoản cá nhân lên Production bucket
- **Short-lived tokens:** human user workstation không được dùng `--duration-seconds > 3600`
- **Hardened workstations:** máy chuyên dụng cho admin, không có email/web, cấu hình đóng băng

### Active Defense
- **Kill-Switch < 60s:** đóng băng cổng rút khi phát hiện bất thường
- **Real-time Reconciliation:** so sánh Ledger nội bộ với Blockchain mỗi 5-10 phút

---

*Nguồn: Mandiant incident report, Safe{Wallet} post-mortem, Bybit official statement, on-chain analysis từ ZachXBT và Chainalysis.*
