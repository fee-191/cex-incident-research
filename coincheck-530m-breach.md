# Coincheck — Phân tích sự cố $530M NEM (01/2018)

> **Loại tấn công:** Hot Wallet Drain — Private Key / Access Compromise  
> **Thiệt hại:** ~530M NEM (~$530M tại thời điểm; largest CEX hack theo số lượng token)  
> **Tác nhân:** Unknown (suspected Lazarus Group dựa trên laundering pattern)  
> **Thời điểm:** 26/01/2018

---

## 1. Tổng quan

Coincheck là sàn giao dịch crypto lớn của Nhật Bản. Vụ hack tháng 1/2018 là vụ lớn nhất trong lịch sử tại thời điểm đó — vượt cả Mt. Gox. Điểm đặc biệt: **toàn bộ 523 triệu NEM bị lưu trong một hot wallet duy nhất**, không có multisig, không có cold storage.

Vụ này là điển hình về **failure to implement basic security practices** — không phải tấn công tinh vi.

---

## 2. Kill Chain

### Initial Access (thời điểm chưa xác định)

Các nhà điều tra Nhật Bản xác định có **malware trên máy tính nội bộ** của Coincheck. Nhiều khả năng qua:
- Spearphishing email với attachment hoặc link độc hại
- Hoặc compromised third-party tool trong infrastructure

### Persistence & Reconnaissance

Attacker duy trì access trong thời gian dài, nghiên cứu:
- Cấu trúc ví NEM của Coincheck
- Quy trình rút tiền và signing
- Thời điểm thích hợp để thực hiện drain

### 26/01/2018, 02:57 JST — Drain

Attacker sử dụng credentials đã thu giữ để authorize transfer:
- 523,000,000 NEM rút khỏi hot wallet
- Transferred đến: `NC4PBAO5TPCAVQKBVJV4CUJUP2TBRM4CLBZ...`
- Giao dịch hoàn tất trước khi Coincheck phát hiện (~9 giờ sau)

### 11:25 JST — Detection

Coincheck phát hiện số dư NEM bất thường. Đình chỉ tất cả giao dịch NEM + phần lớn trading hoạt động của sàn.

---

## 3. Root Causes

**1. Không có cold storage cho NEM**
Toàn bộ 523M NEM lưu trong hot wallet. Coincheck không có cold/warm wallet architecture cho NEM — khác hoàn toàn với cách họ quản lý Bitcoin và các assets khác.

**2. Không có multisig**
Hot wallet NEM là single-signature — một key bị compromise là đủ để drain toàn bộ.

**3. Network không được segment**
Máy chủ chứa private key kết nối với internet trực tiếp và không được isolate đúng cách.

**4. Thiếu real-time monitoring**
9 giờ trôi qua từ lúc drain đến khi phát hiện — không có alert nào kích hoạt.

**5. Compliance gaps**
Japan FSA sau đó phát hiện Coincheck thiếu nhiều yêu cầu bảo mật cơ bản theo quy định crypto exchanges.

---

## 4. Aftermath — Industry Impact

Vụ Coincheck 2018 có tác động lớn đến ngành:
- **Japan FSA** ban hành hướng dẫn bảo mật bắt buộc cho tất cả sàn đăng ký tại Nhật
- **NEM Foundation** triển khai tag system để đánh dấu địa chỉ liên quan đến hack, kêu gọi sàn khác từ chối nhận
- **Monex Group** mua lại Coincheck tháng 4/2018, đầu tư vào security infrastructure
- Bồi thường: Coincheck hoàn trả ~¥46.6 tỷ (khoảng 88% giá trị) cho 260,000 user bị ảnh hưởng

---

## 5. Controls

- **Cold storage bắt buộc** cho tất cả assets có giá trị cao, không chỉ BTC/ETH
- **Multisig cho mọi hot wallet** — không có single key authorization cho large balance
- **Network segmentation**: private key server không được connect internet trực tiếp
- **Real-time balance monitoring**: alert ngay khi balance drop > threshold

---

*Nguồn: Japan Financial Services Agency (FSA) investigation report, Coincheck official announcement, NEM Foundation response, Reuters/Bloomberg financial reporting.*
