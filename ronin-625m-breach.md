# Ronin Network (Axie Infinity) — Phân tích sự cố $625M (03/2022)

> **Loại tấn công:** Validator Key Compromise — Multisig Threshold Attack  
> **Thiệt hại:** ~$625M (173,600 ETH + 25.5M USDC)  
> **Tác nhân:** Lazarus Group (DPRK) — xác nhận bởi FBI và US Treasury  
> **Thời điểm:** 23/03/2022 (phát hiện muộn 29/03/2022 — 6 ngày sau)

---

## 1. Tổng quan

Ronin Network là Ethereum sidechain do Sky Mavis xây dựng cho game Axie Infinity. Bridge Ronin cho phép người dùng chuyển tài sản giữa Ethereum mainnet và Ronin chain. Tại thời điểm bị tấn công, bridge đang nắm giữ tài sản của hàng triệu Axie players — trở thành target có giá trị cao nhất trong lịch sử DeFi.

Điểm đặc biệt: **Lazarus không khai thác lỗi smart contract** mà compromise trực tiếp các **validator private keys** để giả mạo giao dịch hợp lệ.

**6 ngày không ai phát hiện** — phát lộ chỉ khi một user report không rút được tiền.

---

## 2. Kill Chain

### Giai đoạn chuẩn bị (tháng 11/2021 → tháng 3/2022)

**Initial Access** — LinkedIn Job Scam:
Một senior engineer của Sky Mavis bị tiếp cận qua LinkedIn với offer lương cực cao từ một "công ty blockchain". Sau nhiều vòng phỏng vấn, engineer nhận được PDF offer letter.

**PDF exploit:** Tài liệu chứa spyware — khi mở, malware lây nhiễm máy engineer, cấp Lazarus quyền truy cập hệ thống nội bộ Sky Mavis.

**Compromise validator keys:**
- Ronin Network lúc đó có **9 validator nodes**
- Để approve transaction bridge: cần **5/9 chữ ký** (threshold multisig)
- Sky Mavis trực tiếp kiểm soát **4 validators**
- **Axie DAO** kiểm soát 1 validator — nhưng đã cấp Sky Mavis quyền ký thay trong 1 chương trình thử nghiệm November 2021 và **QUÊN REVOKE**

Lazarus từ máy bị compromise đã thu giữ:
- 4 private keys của Sky Mavis validators
- Dùng backdoor access qua Axie DAO node → lấy key thứ 5

### 23/03/2022 — Drain

Với 5/9 keys, Lazarus tạo 2 giao dịch rút tiền giả mạo:
- TX 1: 173,600 ETH
- TX 2: 25.5M USDC

Cả 2 transaction được broadcast và confirm on-chain — **không ai phát hiện**.

### 29/03/2022 — Discovery (6 ngày sau)

Một người dùng báo cáo không rút được 5,000 ETH. Team Sky Mavis kiểm tra → phát hiện bridge đã bị drain 6 ngày trước.

---

## 3. Root Causes

**1. Stale permissions không được revoke**
Sky Mavis được cấp quyền ký thay cho Axie DAO node từ tháng 11/2021. Sau khi chương trình kết thúc, permission **không bị revoke** → attacker có thêm 1 key để đạt ngưỡng 5/9.

**2. Low validator diversity**
5/9 validators tập trung ở Sky Mavis → compromise Sky Mavis = compromise majority.

**3. Không có real-time monitoring**
Giao dịch 173,600 ETH (~$590M tại thời điểm đó) không trigger bất kỳ alert nào trong 6 ngày.

**4. PDF phishing không được phòng ngừa**
Engineer mở PDF từ nguồn không tin cậy trên máy có quyền truy cập production credentials.

---

## 4. MITRE ATT&CK Mapping

| Tactic | Technique | Hành động |
|--------|-----------|-----------|
| Initial Access | T1566.001 — Spearphishing Attachment | PDF offer letter chứa spyware |
| Execution | T1204.002 — User Execution: Malicious File | Engineer mở PDF |
| Credential Access | T1003 — Credential Dumping | Thu giữ validator private keys |
| Persistence | T1078 — Valid Accounts | Dùng keys hợp lệ, không cần persistence |
| Impact | T1657 — Financial Theft | Drain $625M qua forged bridge TXs |

---

## 5. Controls

- **Revoke permissions ngay sau khi không cần:** Định kỳ audit tất cả delegated signing rights
- **Geographic + organizational diversity cho validators:** Không để một tổ chức nắm majority
- **Real-time bridge balance monitoring:** Alert ngay khi outflow > X% trong một block
- **Sandbox for external documents:** Tài liệu từ external phải mở trong isolated VM
- **Raise multisig threshold:** 5/9 là thấp; 6/9 hoặc 7/9 giảm attack surface

---

## 6. Aftermath

- Sky Mavis huy động $150M từ Binance + Animoca để bồi thường người dùng
- Nâng số validators lên 21, phân tán sang nhiều tổ chức độc lập
- FBI xác nhận attribution Lazarus Group tháng 4/2022
- Ronin bridge resume sau 2 tháng với security upgrades

---

*Nguồn: FBI statement (April 2022), Sky Mavis post-mortem, Chainalysis blockchain analysis, US Treasury OFAC designation.*
