# Bybit — Phân tích sự cố $1.5B (02/2025)

> **Loại tấn công:** Supply Chain — Safe{Wallet} Frontend Compromise  
> **Thiệt hại:** ~$1.5B (~401,347 ETH + stETH + mETH)  
> **Tác nhân:** Lazarus Group / TraderTraitor (DPRK) — xác nhận bởi FBI, Mandiant, và on-chain analysis  
> **Chuỗi sự kiện:** 04/02/2025 (initial compromise) → 21/02/2025 (drain)

---

## 1. Tổng quan

Vụ hack Bybit (2025) là vụ trộm crypto lớn nhất trong lịch sử. Điểm đặc biệt: **Bybit không bị tấn công trực tiếp**. Lazarus Group xâm nhập vào **Safe{Wallet}** — third-party multi-sig wallet provider mà Bybit sử dụng để quản lý cold wallet — sau đó inject malicious JavaScript để thao túng giao diện signing.

Cuộc tấn công kéo dài 17 ngày từ khi compromise máy developer đến khi drain. Trong 17 ngày đó, attacker âm thầm reconnaissance và chuẩn bị payload trong khi Bybit và Safe{Wallet} không biết gì.

---

## 2. Timeline & Kill Chain

### 04/02/2025 — Compromise Developer Machine

Một developer của Safe{Wallet} bị tấn công qua **job scam trên LinkedIn/GitHub/Telegram**. Vector giống hệt các vụ tấn công CEX trước của Lazarus:

> *"Tải project này về để làm technical interview"*

Project: **`MC-Based-Stock-Invest-Simulator-main.zip`**

Payload thực thi RCE thông qua PyYAML unsafe deserialization:

```python
# Vulnerable — PyYAML <5.1 (hoặc dùng Loader=yaml.Loader explicitly)
import yaml
data = yaml.load(stream, Loader=yaml.Loader)   # UNSAFE: full deserialization, RCE
data = yaml.load(stream)                        # UNSAFE: PyYAML <5.1 default

# Safe
data = yaml.safe_load(stream)                  # Safe: subset of YAML, no Python object
data = yaml.load(stream, Loader=yaml.SafeLoader)  # Safe: explicit
```

Sau khi thực thi: loader Python download và chạy **Poseidon backdoor** (Mythic framework) trong RAM, kết nối C2 tại `getstockprice[.]com`, exfil credentials.

**Credentials bị đánh cắp:**
- `~/.aws/credentials` — AWS access key và secret
- `~/.ssh/` — SSH private keys
- Browser saved passwords / cookies
- Any cloud provider config files

### 05–19/02/2025 — Reconnaissance & Preparation (15 ngày)

Attacker dùng stolen AWS credentials đăng nhập AWS infrastructure của Safe{Wallet}:

```bash
# AWS session token có thời hạn 12 giờ — attacker phải refresh liên tục
aws sts get-session-token \
  --serial-number "arn:aws:iam::ACCOUNT:mfa/developer" \
  --token-code "MFA_CODE" \
  --duration-seconds 43200

# Reconnaissance
aws iam list-users
aws iam list-roles
aws s3 ls
aws s3 sync s3://production-frontend ./local-copy/
```

Login được thực hiện qua **ExpressVPN IP + Kali Linux user-agent** để blend in. Thử thêm MFA device mới vào tài khoản → thất bại.

Sau khi download frontend codebase (Next.js), attacker phân tích và viết payload chỉ kích hoạt khi phát hiện địa chỉ Bybit cold wallet trong transaction payload.

### 19/02/2025 — Inject Malicious JavaScript

Attacker ghi đè file JavaScript production trên S3:

```bash
# Attacker's command
aws s3 cp ./malicious/_app-52c9031bfa03da47.js \
  s3://safe-wallet-production/static/_app-52c9031bfa03da47.js
```

**Logic của malicious code:**

```javascript
// Simplified pseudocode of injected payload
function interceptTransaction(txPayload) {
  const BYBIT_SIGNERS = ['0x...', '0x...', '0x...'];
  
  if (!isSignerInList(txPayload.signer, BYBIT_SIGNERS)) {
    return txPayload; // Normal flow for non-Bybit transactions
  }
  
  // Store original payload for UI display
  const originalPayload = deepCopy(txPayload);
  
  // Replace recipient address with attacker's address
  txPayload.to = ATTACKER_ADDRESS;
  txPayload.data = encodeAttackerTransferCall();
  
  // Submit modified payload for signing
  const signedTx = await requestSignature(txPayload);
  
  // After signing: restore original payload in UI
  // Next signer sees "correct" transaction details
  signedTx.displayPayload = originalPayload;
  
  return signedTx;
}
```

Kết quả: người ký thấy giao dịch hợp lệ trên màn hình, ký qua Ledger hardware wallet, nhưng thực tế ký vào payload đã bị thay đổi địa chỉ — đây là **Blind Signing**: hardware wallet hiển thị transaction hash thô, không decode nội dung ngữ nghĩa (địa chỉ nhận, số tiền).

### 21/02/2025, 14:13 UTC — Drain

Nhân viên Bybit thực hiện routine transfer từ cold sang hot wallet. Malicious JS kích hoạt:

1. Detect Bybit signer address ✓
2. Replace `masterCopy` upgrade call payload
3. Nhân viên ký qua Ledger (thấy UI bình thường)
4. `upgradeTo(attacker_contract)` được execute on-chain
5. Attacker's contract gains full control of Safe multisig
6. **~401,347 ETH + stETH + mETH bị drain trong vài phút**

### 21/02/2025, 14:15 UTC — Cover Tracks

Chỉ **2 phút sau** khi drain hoàn tất, attacker xóa malicious JS file khỏi S3 bucket. Clean crime scene.

---

## 3. MITRE ATT&CK Mapping

| Tactic | Technique | Chi tiết |
|--------|-----------|----------|
| Reconnaissance | T1595.002 — Active Scanning | Monitor Bybit wallet transactions on Etherscan |
| Resource Development | T1583.001 — Domains | Đăng ký domain C2: `getstockprice[.]com` |
| Resource Development | T1587.001 — Malware | Develop Poseidon backdoor (Mythic framework) |
| Resource Development | T1608.001 — Stage Capabilities | Fork legitimate GitHub project để tạo lure ZIP |
| Initial Access | T1566.002 — Spearphishing Link | Job scam → malicious project download |
| Execution | T1059.006 — Python | Khai thác PyYAML RCE |
| Execution | T1610 — Deploy Container | Docker privileged container exploit |
| Persistence | T1078.004 — Cloud Accounts | Stolen AWS session token |
| Credential Access | T1003 — OS Credential Dumping | Steal `~/.aws/credentials` |
| Discovery | T1087.004 — Cloud Account Discovery | Enum IAM users, roles, policies |
| Discovery | T1619 — Cloud Storage Object Discovery | Enumerate S3 buckets |
| Collection | T1530 — Data from Cloud Storage | Download frontend JS codebase |
| C2 | T1071.001 — Web Protocols | HTTPS callback to `getstockprice[.]com` |
| C2 | T1573.001 — Symmetric Cryptography | XOR-encrypted exfil data |
| Exfiltration | T1041 — Exfiltration Over C2 Channel | Credentials → C2 |
| Impact | T1565.001 — Stored Data Manipulation | Overwrite JS on S3 → manipulate TX payload |
| Impact | T1657 — Financial Theft | Drain ~$1.5B via manipulated multisig |

---

## 4. Root Causes

**1. Direct production S3 write access từ developer workstation**  
```
# What Lazarus exploited
Developer machine → (stolen AWS creds) → Direct s3:PutObject → Production bucket

# What should exist
Developer machine → Git commit → PR review → CI/CD pipeline → S3 (deploy role only)
```

**2. Không có Static Asset Integrity**  
Safe{Wallet} không ký số hay hash-verify JavaScript trước khi serve. File thay đổi không trigger bất kỳ alert nào. Từ khi inject đến khi Bybit phát hiện: **2 ngày** (nếu phát hiện trước drain).

**3. Blind Signing trên hardware wallet**  
Ledger/Trezor hiển thị raw transaction hash, không decode và hiển thị semantic content. Người dùng ký vào hash mà không biết nội dung thật sự là gì.

**4. AWS session token TTL quá dài**  
`--duration-seconds 43200` = 12 giờ cho human workstation. Window đủ để attacker thực hiện extensive reconnaissance trong mỗi session.

**5. Third-party trust without verification**  
Bybit tin tưởng hoàn toàn vào Safe{Wallet} frontend mà không có independent verification layer cho giao dịch cold wallet.

---

## 5. Controls

### Protect Static Assets — Defense-in-depth

**1. Subresource Integrity (SRI)**
```html
<!-- Browser sẽ từ chối execute nếu hash không khớp -->
<script
  src="https://app.safe.global/static/js/_app.js"
  integrity="sha256-RFljZjaN/bmKGKqdZ7K0tYLKP..."
  crossorigin="anonymous">
</script>
```

**2. S3 Bucket Policy — Deny direct write**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": ["s3:PutObject", "s3:DeleteObject"],
    "Resource": "arn:aws:s3:::safe-wallet-production/*",
    "Condition": {
      "StringNotEquals": {
        "aws:PrincipalArn": [
          "arn:aws:iam::ACCOUNT_ID:role/github-actions-deploy"
        ]
      }
    }
  }]
}
```

**3. File integrity monitoring**
```bash
# Run every 5 minutes in CI
sha256sum dist/_app-*.js > hashes.sha256
git diff hashes.sha256 | \
  grep "^[-+]" | \
  grep -v "^[-+][-+][-+]" | \
  mail -s "ALERT: Frontend file changed" security@company.com
```

### Eliminate Blind Signing

```javascript
// Hardware wallet integration — decode before display
async function requestSignature(txPayload) {
  const decoded = await decodeTx(txPayload);
  
  // Force human-readable display
  const confirmed = await hardwareWallet.displayAndConfirm({
    action: decoded.action,         // "Transfer ETH"
    to: decoded.recipientAddress,   // "0x742d...3F2d"
    amount: decoded.amount,         // "100 ETH"
    contract: decoded.contractName, // "Bybit Cold Wallet"
    rawHash: txPayload.hash         // Also show raw hash
  });
  
  if (!confirmed) throw new Error("User rejected");
  return hardwareWallet.sign(txPayload);
}
```

### IAM Hardening
```bash
# Short-lived tokens for humans (max 1 hour, not 12)
aws sts assume-role \
  --role-arn "arn:aws:iam::ACCOUNT:role/developer-limited" \
  --duration-seconds 3600  # 1h, not 43200 (12h)

# Force MFA for any S3 production operations
aws iam put-role-policy \
  --policy-document '{
    "Condition": {
      "Bool": {"aws:MultiFactorAuthPresent": "false"},
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::production-*"
    }
  }'
```

### Supply Chain Verification
```yaml
# .github/workflows/deploy.yml
deploy:
  steps:
    - name: Build
      run: npm run build

    - name: Generate SRI hashes
      run: |
        find dist -name "*.js" -exec sha256sum {} \; > sri-manifest.json
        
    - name: Sign manifest
      run: |
        cosign sign-blob sri-manifest.json \
          --bundle sri-manifest.bundle \
          --key env://COSIGN_KEY

    - name: Upload to S3 with integrity metadata
      run: |
        aws s3 sync dist/ s3://production-frontend/ \
          --metadata-directive REPLACE \
          --metadata "sri-verified=true,deploy-sha=$GITHUB_SHA"
```

---

## 6. Aftermath & Industry Impact

**Safe{Wallet} response:**
- Immediate suspension of all transactions
- Full security audit by Mandiant
- Implemented file integrity monitoring for S3
- Added deployment verification pipeline

**Bybit response:**
- Resumed operations within 72 hours via emergency fundraising
- Migrated to independent signing infrastructure
- Launched $140M+ bounty for recovery of stolen ETH
- Changed cold wallet management procedures

**Industry impact:**
- Accelerated adoption of SRI for DeFi/CEX frontends
- CISA issued advisory on Lazarus supply chain TTPs
- Multiple exchanges audited their third-party wallet dependencies

---

---

## Nguồn tham khảo

- **FBI — TraderTraitor Advisory (Feb 2025):** IC3 cảnh báo về Lazarus Group / TraderTraitor nhắm vào DeFi/CEX — xác nhận attribution và TTP patterns
- **Mandiant Incident Report (Feb 2025):** Google/Mandiant điều tra kỹ thuật Safe{Wallet} compromise — chi tiết JS injection, timeline
- **Safe{Wallet} Official Statement:** Xác nhận developer machine bị compromise, JS file bị thay đổi trên S3
- **Bybit Official Announcement:** Xác nhận số lượng tài sản bị drain, timeline, và phản ứng khẩn cấp
- **ZachXBT On-chain Analysis:** Tracing 401k ETH qua các ví trung gian và laundering routes (đăng trên X/@zachxbt, Feb 2025)
- **Chainalysis 2025 Crypto Crime Report:** Thống kê tổn thất, Lazarus attribution, industry impact
