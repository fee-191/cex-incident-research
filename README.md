# CEX Incident Research

Phân tích kỹ thuật các sự cố bảo mật lớn trong ngành crypto exchange (CEX) — attack chain, MITRE ATT&CK mapping, root cause, và controls để phòng ngừa.

Nội dung dựa trên nghiên cứu 31 vụ hack CEX/crypto từ 2014–2025 (bao gồm vụ Bybit 02/2025). Tài liệu phục vụ mục đích học tập và cải thiện bảo mật. Nguồn: thông báo chính thức của các sàn liên quan, FBI advisories, on-chain data từ Chainalysis, ZachXBT, Elliptic, và các nhà nghiên cứu bảo mật độc lập.

---

## Pattern Analysis

**[📊 31 CEX Hacks — Pattern Analysis (2014–2025)](./cex-hacks-pattern-analysis.md)**  
Phân tích tổng hợp 31 vụ: attack vector taxonomy, Lazarus Group deep dive, timeline trends, và defense framework rút ra từ toàn bộ dataset.

---

## Individual Incident Analyses

Phân tích kỹ thuật chi tiết theo kill chain + MITRE ATT&CK:

| Sự cố | Năm | Thiệt hại | Vector | File |
|-------|-----|-----------|--------|------|
| Bybit | 2025 | $1.5B | Supply chain JS injection | [bybit-1.5b-breach.md](./bybit-1.5b-breach.md) |
| WazirX | 2024 | $235M | Custody provider UI tampering | [wazirx-235m-breach.md](./wazirx-235m-breach.md) |
| Mixin Network | 2023 | $200M | Cloud infra + JS manipulation | [mixin-network-200m-breach.md](./mixin-network-200m-breach.md) |
| Ronin/Axie | 2022 | $625M | Validator key compromise | [ronin-625m-breach.md](./ronin-625m-breach.md) |
| FTX | 2022 | $400M+ | Insider + unauthorized drain | [ftx-collapse-2022.md](./ftx-collapse-2022.md) |
| KuCoin | 2020 | $281M | Private key leak | [kucoin-281m-breach.md](./kucoin-281m-breach.md) |
| Coincheck | 2018 | $530M | Hot wallet, no multisig | [coincheck-530m-breach.md](./coincheck-530m-breach.md) |
| Upbit | 2019 | $50M | Hot wallet drain | [upbit-50m-breach.md](./upbit-50m-breach.md) |

---

## Key Insights

**Tổng thiệt hại 31 vụ: ~$6.8 tỷ USD**

**Top attack vectors:**
- 🔑 Private key / hot wallet compromise — 12/31 vụ (~39%)
- 🌉 Bridge / cross-chain exploit — 7/31 vụ (~23%)
- 📦 Supply chain / third-party — 4/31 vụ (~13%)

**Lazarus Group (DPRK) — ~$3.5B+ confirmed:**  
Nhóm tấn công chịu trách nhiệm lớn nhất. Pattern nhất quán: LinkedIn job scam → malicious project → credential theft → months-long recon → drain. Không tấn công protocol/cryptography — nhắm vào **người vận hành** và **UI signing**.

> *"Khi attacker không thể bẻ key, họ tấn công người cầm key hoặc UI mà người cầm key nhìn vào."*

---

## Mục đích

Kết quả nghiên cứu được áp dụng vào thiết kế security rules tại **[security-template](https://github.com/fee-191/security-template)** — bộ Semgrep rules tùy chỉnh cho CEX, phát hiện các anti-pattern dẫn đến các sự cố trên.

