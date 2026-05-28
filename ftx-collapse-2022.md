# FTX — Phân tích sự cố $400M+ (11/2022)

> **Loại tấn công:** Insider Threat + Unauthorized Access (concurrent với exchange collapse)  
> **Thiệt hại:** ~$400M trong vụ hack (unauthorized drain); ~$8B trong vụ collapse (fraud by executives)  
> **Tác nhân:** Nội bộ FTX (confirmed) + Unknown hacker (concurrent, may be insider)  
> **Thời điểm:** 11/11/2022 — cùng ngày FTX nộp đơn phá sản

---

## 1. Tổng quan

FTX là trường hợp đặc biệt trong danh sách này vì **hai sự kiện xảy ra đồng thời:**

1. **Collapse do fraud:** Sam Bankman-Fried (SBF) và executives sử dụng tiền khách hàng cho Alameda Research → $8B+ thiệt hại
2. **Hack/unauthorized drain:** Ngay khi FTX nộp đơn phá sản, ~$400M bị rút khỏi ví FTX mà ban quản lý mới (John Ray III) không authorize

Sự kiện #2 — vụ hack — vẫn chưa hoàn toàn được làm rõ về attribution. Có 3 giả thuyết:
- **A) Insider** (former FTX employee với key access) — giả thuyết phổ biến nhất
- **B) External hacker** khai thác window chaos khi FTX đang sụp đổ
- **C) SBF-related** nhằm che giấu assets trước bankruptcy proceedings

DOJ indictment năm 2023 đề cập đến vụ drain nhưng không identify attacker cụ thể trong public charges.

---

## 2. Vụ Hack — Timeline

### 11/11/2022 — FTX nộp đơn Chapter 11 Bankruptcy

Hàng triệu USD bắt đầu chảy ra khỏi FTX wallets:
- ~$400M rút khỏi FTX và FTX US wallets
- Transactions broadcast trong vài giờ sau bankruptcy filing
- Tiền nhanh chóng được swap sang ETH rồi laundered

### Real-time detection

Cộng đồng crypto trên Twitter theo dõi on-chain và cảnh báo trong vài tiếng. FTX Telegram admin ban đầu thông báo đây là hack, yêu cầu user xóa FTX app.

John Ray III (CEO mới) thừa nhận có "unauthorized third-party access" nhưng không identify attacker.

---

## 3. Collapse vs Hack — Phân biệt

| | FTX Collapse (Fraud) | FTX Hack (Unauthorized) |
|---|---|---|
| **Thủ phạm** | SBF, Caroline Ellison + executives | Unknown (suspected insider) |
| **Thiệt hại** | ~$8B+ | ~$400M |
| **Method** | Misuse of customer funds | Unauthorized wallet drain |
| **Pháp lý** | Criminal charges filed | Under investigation |
| **Timeline** | Months–years | Hours |

---

## 4. Root Causes (Hack component)

**1. Key management trong chaos**
Khi bankruptcy xảy ra, quy trình kiểm soát key và access bị gián đoạn. Không rõ ai còn có access và ai không.

**2. Không có automatic key rotation khi leadership change**
Transition sang John Ray III's management không kèm theo immediate key rotation — cựu nhân viên có thể vẫn có access.

**3. Insider threat không được address**
Nhiều cựu nhân viên FTX có thể có keys hoặc credentials từ khi còn làm việc.

---

## 5. Lessons Learned

Vụ FTX có hai bài học riêng biệt:

**Về fraud prevention:**
- **Proof of Reserves** bắt buộc và independent audit — không thể để exchange tự audit
- **Segregation of customer funds** từ trading desk/prop trading
- **Regulatory oversight** rõ ràng cho CEX

**Về key management khi incident:**
- **Emergency key rotation protocol** — khi có major incident (bankruptcy, M&A, key personnel departure) phải rotate keys ngay lập tức
- **Key access audit trail** — biết chính xác ai có key nào, revoke ngay khi không còn cần
- **Hardware security modules** — keys không thể exfil nếu lưu trong HSM

---

## 6. Industry Impact

FTX collapse là catalyst cho nhiều thay đổi:
- Proof of Reserves trở thành industry standard (Binance, Kraken, Coinbase)
- Regulatory pressure tăng mạnh toàn cầu (EU MiCA, SEC enforcement, Hong Kong licensing)
- Ngành crypto mất ~$1 trillion market cap trong aftermath
- Nhiều sàn nhỏ không survive "FTX contagion"

---

*Nguồn: DOJ indictment of Sam Bankman-Fried (December 2022), FTX bankruptcy filings, John Ray III Congressional testimony, Chainalysis blockchain analysis, CoinDesk original reporting on Alameda/FTX balance sheet.*
