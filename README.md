# CEX Incident Research

Phân tích kỹ thuật các sự cố bảo mật lớn trong ngành crypto exchange (CEX) — attack chain, root cause, và controls để phòng ngừa.

Tài liệu được viết với mục đích học tập và cải thiện bảo mật. Nguồn: thông báo chính thức của các sàn liên quan, on-chain data, báo cáo từ Chainalysis, Elliptic, và các nhà nghiên cứu bảo mật độc lập.

---

## Incidents

| Sự cố | Thiệt hại | Loại tấn công | Tác nhân |
|-------|-----------|---------------|---------|
| [Mixin Network (2023)](./mixin-network-200m-breach.md) | $200M | Cloud Infrastructure Compromise | Lazarus Group |

*Bybit 2025 và Upbit 2019 đang được viết thêm.*

---

## Mục đích

Mỗi bài phân tích gồm:
- **Kill Chain** — từng bước tấn công theo MITRE ATT&CK
- **Root Causes** — nguyên nhân gốc rễ cho phép tấn công xảy ra
- **Controls** — kiểm soát kỹ thuật cụ thể để phòng ngừa

Tài liệu này hỗ trợ thiết kế security controls cho hệ thống CEX, đặc biệt là các Semgrep rules trong [security-template](https://github.com/fee-191/security-template).
