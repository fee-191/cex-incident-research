# CEX Incident Research

Phân tích kỹ thuật các sự cố bảo mật lớn trong ngành crypto exchange (CEX) — attack chain, root cause, và controls để phòng ngừa.

Tài liệu được viết với mục đích học tập và cải thiện bảo mật. Nguồn: thông báo chính thức của các sàn liên quan, on-chain data, báo cáo từ Chainalysis, Elliptic, Mandiant, và các nhà nghiên cứu bảo mật độc lập.

---

## Incidents

| Sự cố | Thiệt hại | Loại tấn công | Tác nhân |
|-------|-----------|---------------|---------|
| [Bybit (02/2025)](./bybit-1.5b-breach.md) | $1.5B | Supply Chain — Safe{Wallet} JS Injection | Lazarus Group |
| [Mixin Network (09/2023)](./mixin-network-200m-breach.md) | $200M | Cloud Infrastructure Compromise | Lazarus Group |
| [Upbit (11/2019)](./upbit-50m-breach.md) | $50M | Hot Wallet Drain — Key Compromise | Lazarus / Andariel |

**Điểm chung:** Cả 3 vụ đều do Lazarus Group thực hiện. Vector tấn công không phải là giao thức hay smart contract mà là **con người** — developer, DevOps, admin có đặc quyền cao.

---

## Mục đích

Mỗi bài phân tích gồm:
- **Kill Chain** — từng bước tấn công, có timeline cụ thể
- **MITRE ATT&CK Mapping** — technique ID và hành động tương ứng
- **Root Causes** — nguyên nhân gốc rễ cho phép tấn công xảy ra
- **Controls** — kiểm soát kỹ thuật cụ thể để phòng ngừa

Kết quả nghiên cứu được áp dụng vào thiết kế security rules tại [security-template](https://github.com/fee-191/security-template) — bộ Semgrep rules tùy chỉnh cho CEX, phát hiện các anti-pattern dẫn đến các sự cố trên.
