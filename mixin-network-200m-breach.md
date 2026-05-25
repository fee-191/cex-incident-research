# Mixin Network — Phân tích sự cố $200M (09/2023)

> **Loại tấn công:** Supply Chain — Cloud Infrastructure Compromise  
> **Thiệt hại:** ~$200M (400k+ ETH, BTC, và các ERC-20 assets)  
> **Tác nhân:** Lazarus Group (DPRK) — xác nhận bởi on-chain analysis và FBI  
> **Thời điểm:** ~04:00 UTC+8, ngày 23/09/2023

---

## 1. Tổng quan

Mixin Network là giao thức Layer-2 phi tập trung, nhưng sử dụng **database Cloud tập trung** (Google Cloud Platform) để lưu trạng thái tài sản người dùng (User Ledger). Đây là điểm mâu thuẫn cốt lõi dẫn đến sự cố.

Thay vì tấn công thuật toán đồng thuận hay smart contract, Lazarus nhắm vào **Control Plane** — hệ thống Cloud quản lý logic phê duyệt và trạng thái ví. Sau khi compromise được máy trạm của một kỹ sư DevOps, attacker dành 17+ ngày reconnaissance trước khi thực hiện drain thực sự.

---

## 2. Kill Chain

### 04:00 UTC+8, 23/09/2023 — Point of no return

Nhưng để hiểu vụ này, cần xem chuỗi sự kiện bắt đầu từ trước đó nhiều tuần.

### Giai đoạn chuẩn bị — Initial Access

Một kỹ sư vận hành hệ thống (DevOps/SRE) có quyền truy cập cao vào GCP infrastructure của Mixin bị tiếp cận qua **LinkedIn job scam** — pattern điển hình của Lazarus Group từ 2019.

Kỹ sư được mời làm bài "technical assessment" bằng cách chạy một Python project:  
**`MC-Based-Stock-Invest-Simulator-main.zip`**

Project này chứa mã độc khai thác **PyYAML RCE** thông qua `yaml.load()` không dùng SafeLoader:

```python
# Code trong project lure (vulnerable)
import yaml
config = yaml.load(open("config.yaml"))  # RCE vector

# Safe version
config = yaml.safe_load(open("config.yaml"))
```

Payload tải Poseidon malware (framework MythicAgents) vào memory, kết nối C2, và bắt đầu thu thập credentials.

### Credential Harvesting

- Quét toàn bộ filesystem: GCP credentials file (`~/.config/gcloud/`), SSH keys, AWS/cloud config
- **Session Token hijacking:** thu giữ session token có thời hạn dài → bypass MFA (phiên đã xác thực từ trước)
- Duy trì persistence qua legitimate cloud credentials — không cần malware trên máy chủ GCP

### GCP Reconnaissance (17+ ngày)

Attacker dùng stolen credentials đăng nhập GCP qua legitimate gcloud CLI. Trong thời gian này:

- Liệt kê IAM roles, policies, service accounts
- Khám phá cấu trúc Cloud Storage buckets
- **Download toàn bộ frontend codebase** (`app.mixin.one`)
- Phân tích logic phê duyệt giao dịch trong JavaScript
- Viết và test malicious payload JavaScript chỉ nhắm vào địa chỉ ví Mixin

### Data Injection — 23/09/2023

Hai attack vector được thực hiện đồng thời:

**Vector 1 — JavaScript Manipulation:**  
Ghi đè `_app-.js` trên Cloud Storage bucket. Payload inject thay đổi `recipient_address` trong giao dịch nhưng giữ nguyên hiển thị UI. Kết quả: người vận hành thấy địa chỉ đúng trên màn hình nhưng ký vào payload đã bị thay đổi (**Blind Signing**).

**Vector 2 — Database Manipulation:**  
Can thiệp trực tiếp vào Centralized User Ledger trên Cloud DB. Inject các bản ghi giao dịch giả, thay đổi logic xác thực balance và withdrawal approval.

### Execution & Drain

Khi nhân viên vận hành thực hiện lệnh nạp/rút định kỳ, hệ thống MPC của Mixin tự động ký vào payload đã bị thay đổi địa chỉ. Tài sản chuyển thẳng đến ví attacker.

### Detection & Response

Mixin phát hiện sự cố khi số dư ví nóng sụt giảm đột ngột trong khi Ledger nội bộ không ghi nhận lệnh tương ứng. Đóng băng cổng rút được kích hoạt — nhưng 200M USD đã tẩu tán qua Solana và Ethereum trong **vài phút**.

---

## 3. MITRE ATT&CK Mapping

| Tactic | Technique | Hành động |
|--------|-----------|-----------|
| Initial Access | T1566.002 — Spearphishing Link | LinkedIn job scam → project độc hại |
| Execution | T1059.006 — Python | Khai thác PyYAML RCE qua `yaml.load()` |
| Persistence | T1078.004 — Cloud Accounts | Dùng stolen GCP session token |
| Credential Access | T1528 — Steal Application Access Token | Thu giữ GCP credentials và session token |
| Discovery | T1087 — Account Discovery | Enum GCP IAM roles và service accounts |
| Discovery | T1083 — File and Directory Discovery | Khám phá Cloud Storage structure |
| Collection | T1530 — Data from Cloud Storage | Download frontend JS codebase |
| C2 | T1071 — Application Layer Protocol | HTTPS C2 callback |
| Impact | T1565.001 — Stored Data Manipulation | Inject JS payload + DB manipulation |
| Impact | T1657 — Financial Theft | Drain ~$200M qua manipulated transactions |

---

## 4. Root Causes

**1. Centralized Ledger trong kiến trúc "phi tập trung"**  
Mixin xử lý giao dịch phi tập trung nhưng lưu User Ledger trên Cloud DB tập trung → Single Point of Failure. Attacker không cần bẻ khóa giao thức, chỉ cần compromise database.

**2. Developer có quyền direct-write vào Production**  
Kỹ sư bị compromise có thể ghi trực tiếp lên GCP Cloud Storage từ máy cá nhân, không qua CI/CD pipeline được kiểm soát. Một máy bị phishing = toàn bộ production bị kiểm soát.

**3. Không có Integrity Check cho static assets**  
JavaScript trên Cloud Storage không được ký số hay hash-verified trước khi serve. Thay đổi file JS không trigger bất kỳ alert nào.

**4. MPC không bảo vệ được Blind Signing**  
MPC chống mất mát key vật lý nhưng không bảo vệ được khi payload giao dịch bị manipulate trước khi submit cho signing nodes. Các node MPC ký vào payload đã bị thay đổi mà không biết.

**5. Không có Real-time Reconciliation**  
Mixin không tự động so sánh on-chain balance với internal Ledger. Phát hiện chỉ xảy ra sau khi sự chênh lệch đủ lớn để nhân viên nhận ra — quá trễ.

---

## 5. Controls

### Static Asset Protection
```yaml
# CI/CD pipeline — enforce SRI
build:
  - compute_sha256_all_js_files
  - sign_with_code_signing_key
  - upload_to_s3_with_integrity_metadata

serve:
  - verify_integrity_before_response
  - block_if_hash_mismatch
  - alert_on_unexpected_change
```

**Subresource Integrity (SRI):**
```html
<script src="/static/_app.js"
  integrity="sha256-abc123..."
  crossorigin="anonymous"></script>
```

**S3/GCS Object Lock** — prevent overwrite:
```bash
aws s3api put-object-lock-configuration \
  --bucket production-frontend \
  --object-lock-configuration '{"ObjectLockEnabled":"Enabled"}'
```

### IAM — Zero Direct Access
```json
{
  "Effect": "Deny",
  "Action": ["s3:PutObject", "s3:DeleteObject"],
  "Resource": "arn:aws:s3:::production-frontend/*",
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalArn": "arn:aws:iam::ACCOUNT:role/cicd-deploy-role"
    }
  }
}
```

### Architecture Fix — Distributed Ledger
Thay thế Centralized Cloud DB bằng mô hình FROST/MPC với state được lưu ở nhiều vùng hạ tầng độc lập:
```
Mixin Node A (AWS Singapore)
    + Mixin Node B (Azure Hong Kong)
    + Mixin Node C (On-premise Seoul)
    ──────────────────────────────
    → Consensus required để update state
    → Không có single cloud provider có thể compromise toàn bộ
```

### Real-time Reconciliation
```python
async def reconcile_ledger(interval_minutes=5):
    while True:
        onchain_balance = await get_onchain_balance(hot_wallet)
        ledger_balance = await get_internal_ledger_balance()
        
        if abs(onchain_balance - ledger_balance) > THRESHOLD:
            await trigger_safe_mode()
            await alert_security_team(
                f"CRITICAL: Balance mismatch detected. "
                f"On-chain: {onchain_balance}, Ledger: {ledger_balance}"
            )
        
        await asyncio.sleep(interval_minutes * 60)
```

### Kill-Switch — < 60 seconds
```
Detection → Automated Safe Mode trigger → Pause all withdrawals
         └→ Alert on-call security team
         └→ Isolate affected wallet
         └→ Begin forensic log collection
```

---

## 6. Lessons Learned

1. **MPC là cần thiết nhưng không đủ** — Mixin có MPC, nhưng vẫn mất 200M USD. MPC bảo vệ key, không bảo vệ payload.
2. **"Decentralized" protocol với centralized state là contradiction** — nếu có một Database có thể bị manipulate, coi như tập trung.
3. **Privileged workstation là high-value target** — DevOps machine có production access cần được isolate và monitor ngang ngửa production server.
4. **Static assets phải được treat như code** — JavaScript trên CDN/S3 CÓ THỂ bị thay đổi. Integrity checking là bắt buộc, không phải optional.

---

*Nguồn: Mixin Network official statement, SlowMist incident analysis, Chainalysis report, on-chain data từ Etherscan/Solscan, CertiK security analysis.*
