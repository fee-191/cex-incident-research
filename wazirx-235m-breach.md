# WazirX — Phân tích sự cố $235M (07/2024)

> **Loại tấn công:** Supply Chain — Custody Provider Compromise  
> **Thiệt hại:** ~$235M (~45% tổng tài sản của sàn)  
> **Tác nhân:** Lazarus Group (DPRK) — xác nhận bởi on-chain analysis  
> **Thời điểm:** 18/07/2024

---

## 1. Tổng quan

WazirX là sàn giao dịch crypto lớn nhất Ấn Độ. Sự cố 07/2024 có nhiều điểm tương đồng với Bybit (02/2025) — cả 2 đều là **supply chain attack thông qua custody provider**, không phải tấn công trực tiếp vào sàn.

WazirX sử dụng **Liminal** — một công ty custody và multi-party computation — để quản lý và ký giao dịch cold wallet. Lazarus Group compromise Liminal's signing interface, thao túng UI để các approver của WazirX ký vào giao dịch malicious mà tưởng là routine.

**Key difference vs Bybit:** WazirX và Liminal đến nay vẫn còn tranh cãi: Liminal khẳng định hệ thống của họ không bị compromise; WazirX cho rằng signing interface của Liminal đã bị thao túng từ bên ngoài.

---

## 2. Kill Chain

### Giai đoạn chuẩn bị

Lazarus Group nghiên cứu workflow ký giao dịch của WazirX thông qua Liminal:
- WazirX sử dụng Safe (multi-sig smart contract) với 4 signers từ WazirX + 1 signer từ Liminal
- Cần 3/4 WazirX signatures + 1 Liminal signature để approve giao dịch

**Compromise vector (theo on-chain evidence):**
- Attacker thay thế hoặc inject vào Liminal's signing interface
- Khi WazirX approvers review và ký, họ thấy giao dịch hợp lệ nhưng thực tế đang ký vào `upgradeTo()` malicious contract

### 18/07/2024 — Execution

Attacker thực hiện `upgradeTo(malicious_contract)` — upgrade Safe proxy contract để thay đổi logic, sau đó:
- Drain toàn bộ SHIB, ETH, MATIC, PEPE và các token khác
- **~$235M** bị rút trong khoảng thời gian ngắn

**On-chain evidence của Lazarus attribution:**
- Pre-funded attacker wallet từ Tornado Cash — pattern Lazarus
- Laundering qua Ethereum mixer + multi-chain swap ngay sau drain
- Matching với Lazarus TTP từ các vụ trước (Ronin, Atomic Wallet)

---

## 3. Root Causes

**1. Blind signing qua custody provider**
WazirX approvers ký giao dịch qua Liminal interface mà không thể verify on-chain payload một cách độc lập. Nếu Liminal bị compromise, signers không có cách phát hiện.

**2. Upgrade proxy không được giám sát**
Safe (Gnosis Safe) cho phép upgrade proxy contract — một tính năng nguy hiểm nếu không có additional controls. Giao dịch `upgradeTo()` không khác gì giao dịch thông thường trên bề mặt.

**3. Không có independent transaction verification**
Thiếu một verification layer độc lập với Liminal — nếu WazirX có thể verify raw calldata trực tiếp trên Etherscan trước khi ký, attack sẽ bị phát hiện.

**4. Tranh cãi giữa hai bên về responsibility**
Dù lỗi thuộc bên nào, việc thiếu SLA rõ ràng về security responsibility giữa WazirX và Liminal đã tạo ra blind spot.

---

## 4. MITRE ATT&CK Mapping

| Tactic | Technique | Hành động |
|--------|-----------|-----------|
| Initial Access | T1195 — Supply Chain Compromise | Compromise Liminal custody platform |
| Persistence | T1078 — Valid Accounts | Sử dụng legitimate signing flow |
| Defense Evasion | T1036 — Masquerading | Hiển thị giao dịch hợp lệ trên UI |
| Impact | T1565.001 — Stored Data Manipulation | Thao túng calldata trong signing interface |
| Impact | T1657 — Financial Theft | Drain $235M qua manipulated multisig |

---

## 5. So sánh WazirX vs Bybit

| | WazirX (07/2024) | Bybit (02/2025) |
|---|---|---|
| **Supply chain target** | Liminal (custody) | Safe{Wallet} (frontend) |
| **Manipulation method** | Signing UI tampering | JavaScript injection |
| **Victim action** | Ký routine TX | Ký routine TX |
| **Detection** | Sau khi drain | Sau khi drain |
| **Damage** | $235M | $1.5B |
| **Attribution** | Lazarus | Lazarus |

Bybit học được bài học từ WazirX và các vụ tương tự, nhưng vẫn không ngăn được tấn công tương tự 7 tháng sau.

---

## 6. Controls

- **Independent calldata verification:** Approver phải verify raw `calldata` trực tiếp trên blockchain explorer — không chỉ nhìn vào UI của custody provider
- **Disable upgrade proxy cho cold wallet:** Safe contract không nên có upgrade capability cho production wallets
- **Multi-vendor signing verification:** Sử dụng ít nhất 2 signing tool khác nhau; nếu chúng disagree về nội dung TX → từ chối
- **Security SLA với custody provider:** Rõ ràng về responsibility boundary, audit quyền, và incident response

---

*Nguồn: WazirX official statement, Liminal response, ZachXBT on-chain analysis, Elliptic blockchain intelligence report (July 2024).*
