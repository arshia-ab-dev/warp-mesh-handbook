<div align="center">

# 🌐 Cloudflare Zero Trust & Anti-Censorship Gateway Handbook
### Complete Architectural Guide for DPI Bypass, Gateway Deployment, and Mesh Networking

[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20Android-lightgrey.svg)]()
[![Core](https://img.shields.io/badge/cores-Xray%20%7C%20Sing--box-orange.svg)]()

---

### 🌐 Select Language / انتخاب زبان

<a href="#english">
  <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1fa_1f1f8.svg" width="22" valign="middle" alt="English" /> <b>English Version</b>
</a>
&nbsp;&nbsp;&nbsp;•&nbsp;&nbsp;&nbsp;
<a href="#persian">
  <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="22" valign="middle" alt="فارسی" /> <b>نسخه فارسی</b>
</a>

---

</div>

<a name="english"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1fa_1f1f8.svg" width="26" valign="middle" alt="US Flag" /> English

### Overview
This handbook provides battle-tested architectures to bypass aggressive Deep Packet Inspection (DPI), turn your machines into transparent anti-censorship gateways, and establish private bidirectional mesh networks (Cloudflare Zero Trust / WARP Connector) without purchasing external VPS servers.

---

### 🧠 Core Concepts & Technical Discoveries

1. **Why Standard WireGuard Fails (DPI Detection):**  
   Firewalls inspect the initial WireGuard Handshake Initiation packet (fixed byte `0x01` and length of 148 bytes) and drop it immediately (`connect timeout`).
2. **Why UDP Noise Works While Fragmentation Fails:**  
   * **Fragmentation:** Slices TCP/TLS records (ClientHello) to hide the SNI. Because WireGuard operates over **UDP** and encapsulates inner web traffic, the firewall never sees the inner TLS packets; thus, fragmentation is completely ineffective.
   * **UDP Noise:** Injects pseudo-random dummy UDP packets *prior* to the handshake initiation. This confuses stateful DPI inspection and allows the real handshake to pass.
3. **The MASQUE Protocol Trap:**  
   Cloudflare Zero Trust defaults to MASQUE (QUIC / HTTP/3 over port 443 with long ECDSA keys). Third-party cores (Xray/Sing-box) expect 32-byte Curve25519 keys, causing immediate `Connection was reset` errors unless explicitly set to WireGuard in the dashboard.
4. **The Inbound Limitation of Proxy Cores:**  
   Xray and Sing-box are outbound proxy clients. When acting as a WireGuard peer, they drop unsolicited inbound packets. To enable true peer-to-peer services (like RDP or local servers), a dedicated gateway architecture must be established.

---

### 🏗️ Gateway Deployment Comparison

| Gateway Method | Resource Usage | Management Interface | Recommended Guide |
| :--- | :--- | :--- | :--- |
| **🥇 Ubuntu Server (Minimal)** | **RAM: ~180 MB** \| CPU: ~0.1% | Terminal / Netfilter CLI | [📖 Ubuntu Gateway Guide](docs/ubuntu-gateway.md) |
| **🥈 Windows Server Core** | **RAM: ~900 MB** \| CPU: ~0.5% | PowerShell / `netsh` CLI | [📖 Windows Server Core Guide](docs/windows-server-core.md) |
| **🥉 Windows Server GUI** | **RAM: ~2.5 GB** \| CPU: ~2% | Windows Server Manager (GUI) | [📖 Windows Server GUI Guide](docs/windows-server-gui.md) |

---

### 📂 Quick Links & Documentation

* **Configurations:** Find all ready-to-use JSON templates in the [`configs/`](configs/) folder.
* **Detailed Guides:**
  * [Dedicated & VMware Ubuntu Gateway Guide](docs/ubuntu-gateway.md)
  * [Windows Server Core (CLI) Gateway Guide](docs/windows-server-core.md)
  * [Windows Server Desktop Experience (GUI) Gateway Guide](docs/windows-server-gui.md)
  * [Troubleshooting & Network Matrix](docs/troubleshooting.md)

---

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### معرفی پروژه
این کتابچه راهنما شامل معماریهای مهندسیشده و آزمودهشده برای عبور از فیلترینگ پیشرفته (DPI)، تبدیل سیستمها به گیتویهای ضد فیلتر و برقراری ارتباطات دوطرفه مش (Cloudflare Zero Trust / WARP Connector) بدون نیاز به خرید سرور مجازی (VPS) خارجی است.

---

### 🧠 مبانی فنی و کشفیات کلیدی

1. **علت مسدود شدن وایرگارد استاندارد (DPI):**  
   فایروالهای مخابرات بسته اولیه هندشیک وایرگارد (بایت ثابت `0x01` و طول ۱۴۸ بایت) را بلافاصله شناسایی کرده و ارتباط را قطع میکنند (`connect timeout`).
2. **چرا نویز (Noise) پیروز شد اما فرگمنت (Fragment) شکست خورد؟**  
   * **فرگمنت:** مخصوص ترافیک TCP/TLS وب است تا SNI را تکهتکه کند. ترافیک وایرگارد بر بستر **UDP** است و بستههای وب درون کپسول رمزنگاریشده آن قرار دارند؛ فیلترچی اصلاً محتوای وب را نمیبیند، بنابراین فرگمنت روی وایرگارد بیاثر است.
   * **نویز:** قبل از بسته اصلی هندشیک، چند بسته تصادفی و بیمعنی به عنوان طعمه پرتاب میکند. فایروال با دیدن این بستهها وضعیت بازرسی جریان را گم کرده و ترافیک را عبور میدهد.
3. **تله پروتکل MASQUE کلودفلر:**  
   کلودفلر اخیراً پروتکل پیشفرض دستگاهها را به MASQUE تغییر داده است که از کلیدهای طویل ECDSA استفاده میکند. هستههای پروکسی این کلیدها را نمیشناسند و ارور `Connection was reset` میدهند. راهکار، بازگرداندن دستی پروتکل به WireGuard در پنل است.
4. **محدودیت ذاتی هستههای پروکسی در ترافیک ورودی مش:**  
   نرمافزارهای Xray و Sing-box ذاتاً «پروکسی خروجی یکطرفه» هستند. بستههای ورودی ناخواسته مش (ارتباط گوشی به لپتاپ برای RDP یا وبسرور) توسط این هستهها دور انداخته میشوند؛ بنابراین برای یک شبکه دوطرفه پایدار، تفکیک گیتوی الزامی است.

---

### 🏗️ مقایسه روشهای راهاندازی گیتوی (Gateway)

| روش پیادهسازی گیتوی | مصرف منابع سیستم | شیوه مدیریت | لینک آموزش اختصاصی |
| :--- | :--- | :--- | :--- |
| **🥇 اوبونتو سرور (پیشنهادی)** | **رم: ~۱۸۰ مگابایت** \| سیپییو: ۰.۱٪ | ترمینال لینوکس / iptables | [📖 راهنمای اوبونتو سرور](docs/ubuntu-gateway.md) |
| **🥈 ویندوز سرور Core** | **رم: ~۹۰۰ مگابایت** \| سیپییو: ۰.۵٪ | پاورشل ویندوز / netsh | [📖 راهنمای ویندوز سرور Core](docs/windows-server-core.md) |
| **🥉 ویندوز سرور گرافیکی (GUI)** | **رم: ~۲.۵ گیگابایت** \| سیپییو: ۲٪ | کنسول گرافیکی Server Manager | [📖 راهنمای ویندوز سرور GUI](docs/windows-server-gui.md) |

---

### 📂 دسترسی سریع به فایلها و مستندات

* **کانفیگها:** تمام فایلهای JSON آماده در پوشه [`configs/`](configs/) قرار دارند.
* **آموزشهای تفکیکشده در پوشه `docs/`:**
  * [راهنمای جامع گیتوی اوبونتو (فیزیکی یا VMware)](docs/ubuntu-gateway.md)
  * [راهنمای گیتوی ویندوز سرور Core با خط فرمان](docs/windows-server-core.md)
  * [راهنمای گیتوی ویندوز سرور گرافیکی با RRAS](docs/windows-server-gui.md)
  * [چکلیست عیبیابی و حل خطاهای شبکه](docs/troubleshooting.md)

---

### 📄 پروانه انتشار (License)
این پروژه تحت پروانه [MIT License](LICENSE) منتشر شده است.
