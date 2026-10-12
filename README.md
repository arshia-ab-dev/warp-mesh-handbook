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
   Xray and Sing-box are outbound proxy clients. When acting as a WireGuard peer, they drop unsolicited inbound packets. To enable peer-to-peer services (like RDP), a dedicated gateway architecture must be established.

---

### 🏗️ Gateway Architecture

#### 🌟 Primary Method: Dedicated Ubuntu Server / Mini-PC on LAN
The standard and recommended deployment is using a standalone Ubuntu machine (Mini-PC, old laptop, or Raspberry Pi) connected directly to your home router.

```text
[ Clients: Phone / Laptop / TV ]
         │ (DHCP Gateway: 192.168.1.100)
         ▼
[ Dedicated Ubuntu Gateway (192.168.1.100) ]
         │ ── Xray TUN (tun0) with UDP Noise ──► [ Cloudflare Zero Trust ]
         ▼
[ Home Router (192.168.1.1) ] ──► Internet
```

#### 🔄 Alternative Method: Single-Laptop Setup via VMware & Loopback
If you only have a single Windows laptop without a secondary device:
To prevent wireless 802.11 3-address frame bridging issues on Wi-Fi, host and Ubuntu VM communicate over an isolated **Microsoft KM-TEST Loopback Adapter**:
* **Loopback Link:** Windows (`10.10.10.1`) $\longleftrightarrow$ Ubuntu VM (`10.10.10.2`).
* **Windows Wi-Fi Adapter:** Gateway left **BLANK** (all traffic forced into Loopback).
* **ICS:** Windows shares the Loopback connection to other devices on `192.168.137.x`.

---

### ⚙️ Ubuntu Static IP via Netplan (For VMware or LAN)

To assign a permanent static IP in Ubuntu:
1. Edit the Netplan configuration file:
   ```bash
   sudo nano /etc/netplan/01-netcfg.yaml
   ```
2. Configure your interface (e.g., `ens37` for loopback or `eth0` for LAN):
   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       ens37:
         addresses:
           - 10.10.10.2/24
           - fd00:10:10::2/64
   ```
3. Apply changes:
   ```bash
   sudo netplan apply
   ```

---

### 🚀 Running the Ubuntu Gateway

#### Option A: Manual Session Run (Only for temporary testing)
Execute these commands in your active session. They will reset upon reboot:
```bash
# 1. Enable Kernel Forwarding & relaxed Reverse Path filtering
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=2

# 2. Allow Firewall Forwarding and activate Dual-Stack NAT
sudo ufw disable
sudo iptables -P FORWARD ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -o tun0 -j MASQUERADE

# 3. Start Xray
sudo ./xray run -c config.json
```

#### Option B: Make It Permanent (Optional Auto-Start on Boot)
If you want the gateway to run automatically on system boot:

1. **Persistent Sysctl Settings:**
   ```bash
   sudo bash -c 'cat << EOF >> /etc/sysctl.conf
   net.ipv4.ip_forward = 1
   net.ipv6.conf.all.forwarding = 1
   net.ipv4.conf.all.rp_filter = 2
   EOF'
   sudo sysctl -p
   ```
2. **Persistent Firewall Rules:**
   ```bash
   sudo apt install -y iptables-persistent
   sudo netfilter-persistent save
   ```
3. **Auto-Start Xray Service via Systemd:**
   Create `/etc/systemd/system/xray.service`:
   ```ini
   [Unit]
   Description=Xray Anti-Censorship Gateway Service
   After=network.target

   [Service]
   Type=simple
   User=root
   WorkingDirectory=/home/user/xray
   ExecStart=/home/user/xray/xray run -c /home/user/xray/config.json
   Restart=always
   RestartSec=3

   [Install]
   WantedBy=multi-user.target
   ```
   Enable and start the service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now xray
   ```

---

### 🛠️ Optional Features & Customizations

#### 1. Turn Ubuntu into a Local DNS Server (Listening on Port 53)
If you want Ubuntu to act as the primary DNS server (like `1.1.1.1`) for all devices on your LAN:
Add a `dokodemo-door` inbound to `config.json` inside the `"inbounds"` array:
```json
{
  "tag": "dns-in",
  "port": 53,
  "listen": "0.0.0.0",
  "protocol": "dokodemo-door",
  "settings": {
    "address": "1.1.1.1",
    "port": 53,
    "network": "udp,tcp"
  }
}
```
And add `"dns-in"` to your fallback routing rule in `routing.rules`. Devices can now set their DNS to the Ubuntu IP!

#### 2. How to Remove Iran Bypass Rules (Route 100% Traffic through WARP)
If you are outside Iran, don't use Iranian banking apps, or want complete privacy with all traffic forced through WARP:
In any `config.json`, open the `"routing"` section and simply **delete** these two rules:
```json
// DELETE THESE TWO RULES:
{
  "type": "field",
  "outboundTag": "direct",
  "ip": ["geoip:private", "geoip:ir"]
},
{
  "type": "field",
  "outboundTag": "direct",
  "domain": ["geosite:ir"]
}
```
After deleting them, 100% of all websites and IPs will be routed through the encrypted Cloudflare tunnel.

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
   نرم‌افزارهای Xray و Sing-box ذاتاً «پروکسی خروجی یک‌طرفه» هستند. بسته‌های ورودی ناخواسته مش (ارتباط گوشی به لپ‌تاپ برای RDP یا وب‌سرور) توسط این هسته‌ها دور انداخته می‌شوند؛ بنابراین برای یک شبکه دوطرفه پایدار، تفکیک گیت‌وی الزامی است.

---

### 🏗️ معماری راه‌اندازی گیت‌وی (Gateway)

#### 🌟 روش اصلی: یک سرور اوبونتو مستقل در شبکه خانگی (توصیه‌شده)
استانداردترین حالت این است که یک کیس، مینی‌کیس، رزبری‌پای یا لپ‌تاپ قدیمی با اوبونتو مستقیماً با کابل شبکه یا وای‌فای به مودم خانه وصل باشد:

```text
[ گوشی، لپ‌تاپ، تلویزیون هوشمند ]
         │ (تنظیم Gateway روی 192.168.1.100)
         ▼
[ سرور اوبونتو گیت‌وی (192.168.1.100) ]
         │ ── تونل Xray با نویز روی tun0 ──► [ ابر کلودفلر ]
         ▼
[ مودم خانگی (192.168.1.1) ] ──► اینترنت
```

#### 🔄 روش فرعی: پیاده‌سازی روی یک تک‌لپ‌تاپ با VMware و کارت Loopback
اگر فقط یک لپ‌تاپ دارید، برای غلبه بر باگ فریم‌های ۳ آدرسی وای‌فای، ارتباط ویندوز با اوبونتوی مجازی از طریق **کارت شبکه Microsoft KM-TEST Loopback** ایزوله می‌شود:
* **لینک Loopback:** ویندوز (`10.10.10.1`) $\longleftrightarrow$ اوبونتو (`10.10.10.2`).
* **کارت وای‌فای ویندوز:** فیلد Gateway **خالی** گذاشته می‌شود (اجبار کل سیستم به عبور از Loopback).
* **قابلیت ICS:** ویندوز اینترنت Loopback را برای گوشی‌ها روی رنج `192.168.137.x` پخش می‌کند.

---

### ⚙️ تنظیم آی‌پی ثابت در اوبونتو با Netplan

برای اینکه آی‌پی کارت شبکه در اوبونتو ثابت بماند:
1. فایل پیکربندی netplan را باز کنید:
   ```bash
   sudo nano /etc/netplan/01-netcfg.yaml
   ```
2. کارت شبکه مورد نظر (مثلاً `ens37` برای Loopback یا `eth0` برای شبکه خانگی) را تنظیم کنید:
   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       ens37:
         addresses:
           - 10.10.10.2/24
           - fd00:10:10::2/64
   ```
3. ذخیره کرده و اعمال کنید:
   ```bash
   sudo netplan apply
   ```

---

### 🚀 نحوه اجرای گیت‌وی در اوبونتو

#### حالت اول: اجرای موقت دستی (فقط برای تست سشن فعلی)
اگر نمی‌خواهید چیزی در استارتاپ ثبت شود و فقط برای استفاده همان لحظه می‌خواهید، **فقط این فرامین را بزنید** (با ریستارت ریست می‌شوند):
```bash
# ۱. فعال‌سازی فورواردینگ هسته
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=2

# ۲. باز کردن فایروال و فعال‌سازی NAT دوگانه
sudo ufw disable
sudo iptables -P FORWARD ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -o tun0 -j MASQUERADE

# ۳. اجرای Xray
sudo ./xray run -c config.json
```

#### حالت دوم: دائمی کردن تنظیمات (اختیاری - اجرای خودکار با بوت)
اگر می‌خواهید هر بار سیستم روشن شد همه چیز خودکار بالا بیاید:

1. **دائمی کردن فوروارد هسته:**
   ```bash
   sudo bash -c 'cat << EOF >> /etc/sysctl.conf
   net.ipv4.ip_forward = 1
   net.ipv6.conf.all.forwarding = 1
   net.ipv4.conf.all.rp_filter = 2
   EOF'
   sudo sysctl -p
   ```
2. **دائمی کردن قوانین iptables:**
   ```bash
   sudo apt install -y iptables-persistent
   sudo netfilter-persistent save
   ```
3. **اجرای خودکار Xray به عنوان سرویس سیستمی:**
   فایل `/etc/systemd/system/xray.service` را بسازید:
   ```ini
   [Unit]
   Description=Xray Anti-Censorship Gateway Service
   After=network.target

   [Service]
   Type=simple
   User=root
   WorkingDirectory=/home/user/xray
   ExecStart=/home/user/xray/xray run -c /home/user/xray/config.json
   Restart=always
   RestartSec=3

   [Install]
   WantedBy=multi-user.target
   ```
   فعال‌سازی سرویس:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now xray
   ```

---

### 🛠️ بخش‌های اختیاری و شخصی‌سازی

#### ۱. تبدیل اوبونتو به یک سرور DNS محلی (پورت ۵۳)
اگر می‌خواهید اوبونتو مثل `1.1.1.1` برای دستگاه‌های خانه سرور DNS شود:
در فایل `config.json`، اینباند زیر را به آرایه `"inbounds"` اضافه کنید:
```json
{
  "tag": "dns-in",
  "port": 53,
  "listen": "0.0.0.0",
  "protocol": "dokodemo-door",
  "settings": {
    "address": "1.1.1.1",
    "port": 53,
    "network": "udp,tcp"
  }
}
```
و تگ `"dns-in"` را به قانون هدایت به `"warp-out"` اضافه کنید. حالا دستگاه‌های دیگر می‌توانند DNS خود را آی‌پی اوبونتو بگذارند!

#### ۲. نحوه حذف قوانین روتینگ ایران (عبور ۱۰۰٪ ترافیک از وارپ)
اگر مایلید تمام ترافیک و حتی سایت‌های ایرانی بدون استثنا از داخل تونل وارپ رد شوند:
در بخش `"routing"` هر کانفیگی، **این دو قانون را به راحتی حذف کنید**:
```json
// این دو قانون را حذف کنید:
{
  "type": "field",
  "outboundTag": "direct",
  "ip": ["geoip:private", "geoip:ir"]
},
{
  "type": "field",
  "outboundTag": "direct",
  "domain": ["geosite:ir"]
}
```
پس از حذف این دو، تمام سایت‌ها بدون استثنا از تونل رد خواهند شد.

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

---

### 📄 پروانه انتشار (License)
این پروژه تحت پروانه [MIT License](LICENSE) منتشر شده است.
