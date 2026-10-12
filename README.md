<div align="center">

# 🌐 Cloudflare Zero Trust & WARP Mesh Handbook
### Comprehensive Guide for Anti-Censorship, Local Gateways, and Bidirectional Mesh Networking

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
This repository provides a battle-tested network architecture designed to bypass advanced Deep Packet Inspection (DPI), eliminate regional restrictions, and establish a true **bidirectional virtual network (Mesh / VLAN)** between devices (e.g., Laptop $\leftrightarrow$ Phone) using **Cloudflare Zero Trust (WARP Connector)** without requiring a third-party VPS.

---

### 🧠 Core Concepts & Technical Discoveries

1. **Why Standard WireGuard Fails (DPI Detection):**  
   National firewalls inspect the initial WireGuard Handshake Initiation packet (fixed byte `0x01` and length of 148 bytes) and drop the connection immediately (`connect timeout`).
2. **Why UDP Noise Works While Fragmentation Fails:**  
   * **Fragmentation:** Slices TCP/TLS records (ClientHello) to obscure the SNI. Since WireGuard operates purely over **UDP** and encapsulates inner web traffic, the firewall never sees the inner TLS packets; thus, fragmentation is completely ineffective.
   * **UDP Noise:** Injects pseudo-random dummy UDP packets *prior* to the handshake initiation. This desynchronizes the stateful DPI inspection and allows the real handshake to pass.
3. **The MASQUE Protocol Trap:**  
   Cloudflare Zero Trust defaults to MASQUE (QUIC / HTTP/3 over port 443 with long ECDSA keys). Third-party cores (Xray/Sing-box) expect 32-byte Curve25519 keys, causing immediate `Connection was reset` errors unless explicitly configured as WireGuard in the dashboard.
4. **The Inbound Limitation of Proxy Cores:**  
   Xray and Sing-box are outbound proxy clients. When acting as a WireGuard peer, they drop unsolicited inbound packets. This prevents peer-to-peer applications (RDP, local web servers) from working unless a true OS-level network interface (TUN/Loopback Gateway) is created.

---

### 🏗️ Master Network Architecture

To circumvent Wi-Fi 802.11 3-address frame limitations (which prevent bridge routing over wireless cards) and avoid routing deadlocks on a single machine, we decouple the gateway from the host using a **Microsoft Loopback Adapter** and an isolated virtual machine:

```text
[ Mobile / Other Devices ]
         │ (Wi-Fi / ICS: 192.168.137.x)
         ▼
[ Windows Host Laptop ] ──── (Microsoft Loopback: 10.10.10.1) ────┐
                                                                   ▼
[ Physical Wi-Fi Router ] ◄── (Bridged Interface) ── [ Ubuntu Server VM (10.10.10.2) ]
                                                                   │
                                                        [ Xray TUN (tun0) with Noise ]
                                                                   │
                                                                   ▼
                                                        [ Cloudflare Zero Trust ]
```

---

### 🚀 Step-by-Step Deployment

#### Step 1: Cloudflare Zero Trust Setup
1. Navigate to **Settings** > **WARP Client** > **Device profiles**.
2. Edit **Default profile** and switch **Device tunnel protocol** from **MASQUE** to **WireGuard**.
3. Go to **Networks** > **Mesh** (or WARP Connector), click **Add node** (e.g., `laptop`), and copy the generated token (`eyJhIj...`).

#### Step 2: Key Extraction via GitHub Codespaces
*(Downloading the Docker image directly inside restricted networks often hangs due to throttled registries).*
1. Open a blank workspace on [GitHub Codespaces](https://github.com/codespaces).
2. Run the extractor:
   ```bash
   docker run --rm -v $(pwd):/app/output ghcr.io/animmouse/wgcf-connector <YOUR_TOKEN>
   ```
3. Download the generated `.conf` file to obtain your `PrivateKey`, `Address` (`100.96.0.x`), and Cloudflare `PublicKey`.

#### Step 3: Windows Host Network Configuration
1. **Microsoft KM-TEST Loopback Adapter:**
   * **IPv4 Address:** `10.10.10.1` | **Subnet Mask:** `255.255.255.0`
   * **Default Gateway:** `10.10.10.2` (Points to VM) | **DNS:** `1.1.1.1`, `1.1.1.3`
   * **IPv6 Address:** `fd00:10:10::1` | **Prefix Length:** `64`
   * **IPv6 Default Gateway:** `fd00:10:10::2` | **DNS:** `2606:4700:4700::1111`
2. **Physical Wi-Fi Adapter:**
   * Assign static IP (e.g., `192.168.1.50`).
   * **Leave Default Gateway BLANK!** *(Critical to prevent route leakage and loops).*
3. **Internet Connection Sharing (ICS):**
   * Share the Loopback Adapter to Wi-Fi/Hotspot to broadcast clean internet to phones on `192.168.137.x`.

#### Step 4: Ubuntu Gateway Configuration
Enable IP forwarding, disable strict reverse path filtering, and activate dual-stack NAT on Ubuntu:
```bash
# Kernel Routing & Forwarding
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=2

# Firewall & Dual-Stack NAT
sudo ufw disable
sudo iptables -P FORWARD ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
```

---

### 📂 Configuration Catalog

All production configurations reside in the [`configs/`](configs/) folder:

| Configuration File | Target Platform | Description |
| :--- | :--- | :--- |
| [`xray-linux.json`](configs/xray-linux.json) | Ubuntu Server | Gateway router mode, `tun0`, LAN SOCKS/HTTP, Dual-Stack IPv6 & Smart Iran routing. |
| [`xray-windows-tun.json`](configs/xray-windows-tun.json) | Windows 10/11 | Standalone full-system tunnel via `wintun.dll` with NCSI-compatible DNS routing. |
| [`xray-windows-proxy.json`](configs/xray-windows-proxy.json) | Windows 10/11 | Lightweight local SOCKS5 (`2080`) / HTTP (`2081`) proxy without admin rights. |
| [`xray-android.json`](configs/xray-android.json) | Android | Optimized for v2rayNG / NekoBox using Android's native VpnService on port `10808`. |
| [`singbox-1.15.json`](configs/singbox-1.15.json) | Multi-platform | Modern Sing-box 1.15+ schema using root `endpoints` and rule-actions. |

#### Placeholders Reference
Before deploying, replace the placeholders inside the configuration files:
* `YOUR_PRIVATE_KEY_HERE`: Your extracted 44-character Curve25519 private key.
* `YOUR_VIRTUAL_IPV4_HERE/32`: Your assigned overlay IPv4 (e.g., `100.96.0.2/32`).
* `YOUR_VIRTUAL_IPV6_HERE/128`: Your assigned overlay IPv6 (e.g., `2606:4700:...::2/128`).
* `CLEAN_WARP_ENDPOINT_IP:PORT`: Tested unblocked Anycast endpoint (`162.159.193.5:500`).

---

### 🔧 Troubleshooting Reference

| Issue / Symptom | Root Cause | Solution |
| :--- | :--- | :--- |
| `go: connect timeout` | DPI drops WireGuard handshake initiation (`0x01`). | Use port `500`/`4500` with 5-stage random UDP Noise enabled. |
| `Connection was reset` | Public/Private key mismatch or dashboard left on MASQUE. | Switch Zero Trust dashboard profile to WireGuard and re-extract node keys. |
| `CF_HAPPY_EYEBALLS_MITM_FAILURE` | Client detects upstream proxy tampering with TLS pinning. | Reset custom CLI endpoint: `warp-cli tunnel endpoint reset`. |
| Windows "No Internet" (Yellow icon) | NCSI fails to resolve probes over blocked raw UDP 53. | Route DNS through DoH or ensure transparent tunnel routing is uninhibited. |
| VM Forwarding fails | Linux `FORWARD` chain defaults to `DROP`. | Execute `sudo iptables -P FORWARD ACCEPT`. |

---

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### معرفی پروژه
این مخزن شامل یک معماری شبکه‌ای مهندسی‌شده و آزموده‌شده برای دور زدن فیلترینگ پیشرفته (DPI)، رفع کامل تحریم‌ها، و ایجاد یک **شبکه مجازی دوطرفه خصوصی (Mesh / VLAN)** بین دستگاه‌های مختلف (مانند لپ‌تاپ و گوشی) از طریق **Cloudflare Zero Trust (WARP Connector)** بدون نیاز به خرید سرور مجازی (VPS) خارجی است.

---

### 🧠 مبانی فنی و کشفیات کلیدی

1. **علت مسدود شدن وایرگارد استاندارد (DPI):**  
   فایروال‌های مخابرات بسته اولیه هندشیک وایرگارد (بایت ثابت `0x01` و طول ۱۴۸ بایت) را بلافاصله شناسایی کرده و ارتباط را قطع می‌کنند (`connect timeout`).
2. **چرا نویز (Noise) پیروز شد اما فرگمنت (Fragment) شکست خورد؟**  
   * **فرگمنت:** مخصوص ترافیک TCP/TLS وب است تا SNI را تکه‌تکه کند. ترافیک وایرگارد بر بستر **UDP** است و بسته‌های وب درون کپسول رمزنگاری‌شده آن قرار دارند؛ فیلترچی اصلاً محتوای وب را نمی‌بیند، بنابراین فرگمنت روی وایرگارد بی‌اثر است.
   * **نویز:** قبل از بسته اصلی هندشیک، چند بسته تصادفی و بی‌معنی به عنوان طعمه پرتاب می‌کند. فایروال با دیدن این بسته‌ها وضعیت بازرسی جریان را گم کرده و ترافیک را عبور می‌دهد.
3. **تله پروتکل MASQUE کلودفلر:**  
   کلودفلر اخیراً پروتکل پیش‌فرض دستگاه‌ها را به MASQUE تغییر داده است که از کلیدهای طویل ECDSA استفاده می‌کند. هسته‌های پروکسی این کلیدها را نمی‌شناسند و ارور `Connection was reset` می‌دهند. راهکار، بازگرداندن دستی پروتکل به WireGuard در پنل است.
4. **محدودیت ذاتی هسته‌های پروکسی در ترافیک ورودی مش:**  
   نرم‌افزارهای Xray و Sing-box ذاتاً «پروکسی خروجی یک‌طرفه» هستند. بسته‌های ورودی ناخواسته مش (ارتباط گوشی به لپ‌تاپ برای RDP یا وب‌سرور) توسط این هسته‌ها دور انداخته می‌شوند؛ بنابراین برای مش دوطرفه واقعی، تفکیک گیت‌وی از کلاینت الزامی است.

---

### 🏗️ معماری جامع گیت‌وی محلی (Master Architecture)

برای غلبه بر محدودیت فریم‌های ۳ آدرسی وای‌فای (که مانع روتر شدن سیستم روی کارت وای‌فای می‌شود)، ارتباط لپ‌تاپ و ماشین مجازی اوبونتو از طریق یک کارت شبکه مجازی ایزوله به نام **Microsoft KM-TEST Loopback Adapter** پیاده‌سازی شده است:

```text
[ گوشی و سایر دستگاه‌ها ]
         │ (Wi-Fi / ICS: 192.168.137.x)
         ▼
[ لپ‌تاپ ویندوزی ] ──── (کارت Loopback: 10.10.10.1) ────┐
                                                         ▼
[ مودم خانگی اینترنت ] ◄── (Bridge Wi-Fi) ── [ اوبونتو سرور (10.10.10.2) ]
                                                         │
                                               [ Xray با نویز روی tun0 ]
                                                         │
                                                         ▼
                                               [ ابر Cloudflare ]
```

---

### 🚀 مراحل گام‌به‌گام راه‌اندازی

#### مرحله ۱: تنظیم پنل Cloudflare Zero Trust
1. به مسیر **Settings** > **WARP Client** > **Device profiles** بروید.
2. پروفایل **Default** را ویرایش کرده و گزینه **Device tunnel protocol** را از MASQUE به **WireGuard** تغییر داده و ذخیره کنید.
3. به مسیر **Networks** > **Mesh** بروید، روی **Add node** کلیک کنید، نامی مانند `laptop` بدهید و توکن تولیدشده (`eyJhIj...`) را کپی کنید.

#### مرحله ۲: استخراج کلیدها در GitHub Codespaces
*(دانلود ایمیج داکر در اینترنت ایران با افت پکت شدید مواجه می‌شود، اما در سرورهای ابری گیت‌هاب ظرف ۵ ثانیه دانلود می‌شود).*
1. در [github.com/codespaces](https://github.com/codespaces) یک محیط خام (Blank) بسازید.
2. دستور زیر را با توکن خود اجرا کنید:
   ```bash
   docker run --rm -v $(pwd):/app/output ghcr.io/animmouse/wgcf-connector <YOUR_TOKEN>
   ```
3. فایل خروجی `.conf` را دانلود کنید تا مقادیر `PrivateKey`، آدرس‌های `Address` (`100.96.0.x`) و `PublicKey` کلودفلر استخراج شوند.

#### مرحله ۳: تنظیم کارت‌های شبکه در ویندوز
1. **کارت شبکه Microsoft Loopback (مسیر متصل به اوبونتو):**
   * **IPv4 Address:** `10.10.10.1` | **Subnet Mask:** `255.255.255.0`
   * **Default Gateway:** آی‌پی اوبونتو: **`10.10.10.2`** | **DNS:** `1.1.1.1`, `1.1.1.3`
   * **IPv6 Address:** `fd00:10:10::1` | **Prefix Length:** `64`
   * **IPv6 Default Gateway:** `fd00:10:10::2` | **DNS:** `2606:4700:4700::1111`
2. **کارت شبکه Wi-Fi فیزیکی لپ‌تاپ:**
   * آی‌پی دستی ست کنید (مثلاً `192.168.1.50`).
   * **Default Gateway: کاملاً خالی بگذارید!** *(کلید جلوگیری از نشت و لوپ شبکه).*
3. **اشتراک‌گذاری اینترنت (ICS) برای گوشی:**
   * اینترنت کارت Loopback را روی کارت Wi-Fi (یا هات‌اسپات) Share کنید تا گوشی‌ها روی رنج `192.168.137.x` اینترنت آزاد دریافت کنند.

#### مرحله ۴: تنظیمات روتینگ و فایروال در اوبونتو
دستورات زیر را برای باز کردن روتینگ هسته و فعال‌سازی NAT44 و NAT66 در ترمینال اوبونتو اجرا کنید:
```bash
# فعال‌سازی فورواردینگ هسته
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=2

# باز کردن فایروال و فعال‌سازی NAT دوگانه
sudo ufw disable
sudo iptables -P FORWARD ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
```

---

### 📂 کاتالوگ فایل‌های کانفیگ

تمام فایل‌های آماده و استاندارد در پوشه [`configs/`](configs/) قرار دارند:

| نام فایل | سیستم‌عامل هدف | توضیحات معماری |
| :--- | :--- | :--- |
| [`xray-linux.json`](configs/xray-linux.json) | سرور اوبونتو | حالت روتر گیت‌وی، اینترفیس `tun0`، پروکسی شبکه روی پورت ۲۰۸۰، روتینگ ایران و IPv6. |
| [`xray-windows-tun.json`](configs/xray-windows-tun.json) | ویندوز ۱۰ و ۱۱ | تونل کامل سیستم با `wintun.dll` و مدیریت خودکار دی‌ان‌اس ویندوز (NCSI). |
| [`xray-windows-proxy.json`](configs/xray-windows-proxy.json) | ویندوز ۱۰ و ۱۱ | حالت پروکسی سبک SOCKS5 (`2080`) و HTTP (`2081`) بدون نیاز به دسترسی ادمین. |
| [`xray-android.json`](configs/xray-android.json) | اندروید | مخصوص v2rayNG و NekoBox با تکیه بر VpnService بومی سیستم‌عامل روی پورت ۱۰۸۰۸. |
| [`singbox-1.15.json`](configs/singbox-1.15.json) | چندسکویی | ساختار مدرن Sing-box 1.15 با ریشه `endpoints` و حذف فیلدهای منسوخ‌شده. |

#### راهنمای متغیرها (Placeholders)
قبل از استفاده، مقادیر زیر را در فایل‌ها جایگزین کنید:
* `YOUR_PRIVATE_KEY_HERE`: کلید خصوصی ۴۴ حرفی استخراج‌شده.
* `YOUR_VIRTUAL_IPV4_HERE/32`: آی‌پی داخلی مش (مانند `100.96.0.2/32`).
* `YOUR_VIRTUAL_IPV6_HERE/128`: آی‌پی داخلی نسخه ۶ مش (مانند `2606:4700:...::2/128`).
* `CLEAN_WARP_ENDPOINT_IP:PORT`: اندپوینت تست‌شده و باز (`162.159.193.5:500`).

---

### 🔧 چک‌لیست عیب‌یابی سریع

| خطا / نشانه | دلیل فنی | راه‌حل فوری |
| :--- | :--- | :--- |
| `go: connect timeout` | بسته هندشیک وایرگارد توسط DPI مسدود شده است. | از پورت `500` با ۵ لایه نویز UDP تصادفی استفاده کنید. |
| `Connection was reset` | کلید نامعتبر است یا پروتکل روی MASQUE مانده است. | در پنل Zero Trust پروتکل را روی WireGuard بگذارید و نود را مجدد بسازید. |
| `CF_HAPPY_EYEBALLS_MITM_FAILURE` | کلاینت رسمی متوجه پروکسی دستی شده یا گواهی همخوانی ندارد. | دستور ریست اندپوینت را بزنید: `warp-cli tunnel endpoint reset`. |
| علامت زرد وای‌فای (No Internet) | پکت‌های بررسی وضعیت اینترنت ویندوز روی پورت ۵۳ بی‌پاسخ می‌مانند. | ترافیک DNS را از DoH هدایت کنید یا مانع عبور پکت‌های پورت ۵۳ نشوید. |
| عدم عبور بسته‌ها از اوبونتو | زنجیره `FORWARD` فایروال به طور پیش‌فرض روی `DROP` است. | دستور `sudo iptables -P FORWARD ACCEPT` را اجرا کنید. |

---

### 📄 پروانه انتشار (License)
این پروژه تحت پروانه [MIT License](LICENSE) منتشر شده است.
