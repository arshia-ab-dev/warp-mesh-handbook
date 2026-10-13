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

### 🏗️ Gateway Deployment Methodologies

Choose the method that best matches your workflow and hardware:

```text
┌────────────────────────────────────────────────────────────────────────┐
│  Method 1: Windows Server VM Gateway (GUI-driven via RRAS & NAT)       │
│  Method 2: Standalone Ubuntu Home Server (Dedicated physical appliance)│
│  Method 3: VMware + Ubuntu Minimal + Loopback (Ultra-lightweight CLI)  │
└────────────────────────────────────────────────────────────────────────┘
```

---

#### 🥇 Method 1: Windows Server in VMware as a Gateway (GUI-Driven via RRAS)
If you prefer a familiar graphical interface (GUI) over Linux terminal commands and have enough RAM (requires 2–3 GB for the VM):

```text
[ Windows Host Laptop ] ──── (Microsoft Loopback: 10.10.10.1) ────┐
                                                                   ▼
[ Physical Wi-Fi Router ] ◄── (Bridged Adapter) ── [ Windows Server VM (10.10.10.2) ]
                                                            │
                                                   [ Xray TUN (CloudflareMesh) ]
                                                            │
                                                   [ RRAS IPv4 NAT Service ]
                                                            │
                                                            ▼
                                                   [ Cloudflare Edge ]
```

##### Step 1: Virtual Hardware Setup
In VMware Workstation, create a Windows Server VM (2019, 2022, or 2025 Desktop Experience) with:
* **2 CPU Cores**, **2048 to 3072 MB RAM**, and a **35 GB Thin Provisioned disk**.
* **Network Adapter 1:** Bridged to physical Wi-Fi (for raw internet access).
* **Network Adapter 2:** Bridged to `Microsoft KM-TEST Loopback Adapter` (internal link to Host).

##### Step 2: Running Xray on Windows Server
1. Download `xray.exe`, [`wintun.dll`](https://www.wintun.net), and `xray-windows-tun.json`.
2. Run PowerShell as Administrator:
   ```powershell
   .\xray.exe run -c xray-windows-tun.json
   ```
   The virtual network interface `CloudflareMesh` will be created with unblocked internet.

##### Step 3: Enabling the RRAS / NAT Role (GUI)
1. Open **Server Manager** > click **Add roles and features**.
2. Select **Remote Access** > under Role Services, check **Routing** (DirectAccess and VPN will auto-select).
3. Complete the installation wizard.
4. Open the **Routing and Remote Access** console (`rrasmgmt.msc`).
5. Right-click your server name > **Configure and Enable Routing and Remote Access**.
6. Choose **Custom configuration** > check **NAT** and **LAN routing** > Finish > Start service.
7. In the left panel, expand **IPv4 > NAT**:
   * Right-click **NAT** > **New Interface...** > select **`CloudflareMesh`**.
   * Choose *"Public interface connected to the Internet"* and check *"Enable NAT on this interface"* > OK.
   * Right-click **NAT** again > **New Interface...** > select your internal interface (connected to Loopback).
   * Choose *"Private interface connected to private network"* > OK.

##### Step 4: Configure Windows Host
On your main Windows laptop:
* On the **Microsoft Loopback Adapter**: Set IPv4 Default Gateway to the Windows Server VM's IP (`10.10.10.2`).
* On the **Physical Wi-Fi Adapter**: Assign a static IP and **leave Default Gateway BLANK**.

---

#### 🥈 Method 2: Standalone Ubuntu Home Server (Dedicated Hardware Appliance)
Ideal if you have an old laptop, mini-PC, or Raspberry Pi running Ubuntu connected to your home router:

```text
[ Clients: Phone / Laptop / TV ]
         │ (Set Default Gateway: 192.168.1.100)
         ▼
[ Dedicated Ubuntu Gateway (192.168.1.100) ]
         │ ── Xray TUN (tun0) with UDP Noise ──► [ Cloudflare Edge ]
         ▼
[ Home Router (192.168.1.1) ] ──► Internet
```

1. Deploy `xray-linux.json` on the Ubuntu machine.
2. Enable IP forwarding and NAT:
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
   ```
3. On other devices, set their **Default Gateway** to the Ubuntu IP (`192.168.1.100`).

---

#### 🥉 Method 3: Single-Laptop Setup via VMware & Ubuntu Minimal (Ultra-Light CLI)
Designed to run an isolated gateway inside a single laptop with virtually zero CPU footprint and under 200 MB of RAM usage:

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
                                                        [ Cloudflare Edge ]
```

---

### 🔧 Step-by-Step Implementation for Method 3

#### 1. Installing the Microsoft Loopback Adapter on Windows
1. Press `Win + R`, type **`hdwwiz.exe`**, and press Enter.
2. Choose *"Install the hardware that I manually select from a list (Advanced)"* > Next.
3. Select **Network adapters** > Next.
4. Under Manufacturer, choose **Microsoft**; under Model, select **`Microsoft KM-TEST Loopback Adapter`** > Next > Finish.

#### 2. Configuring VMware Virtual Network Editor
1. In VMware Workstation, go to **Edit > Virtual Network Editor**.
2. Click **Change Settings** (Admin privileges).
3. Click **Add Network** (e.g., `VMnet2`).
4. Select **Bridged**, and in the *"Bridged to"* dropdown, select **`Microsoft KM-TEST Loopback Adapter`**.
5. Assign your Ubuntu VM's second network adapter to this custom `VMnet2`.

#### 3. Installing Ubuntu Server 24.04 LTS (Minimal)
* Hardware specs: **1 CPU core**, **1024 MB RAM**, and a **15 GB Thin Provisioned disk**.
* In the installer, select **Ubuntu Server (Minimised)**.
* Install guest utilities and required packages:
  ```bash
  sudo apt update && sudo apt install -y open-vm-tools curl unzip iptables net-tools
  ```
* Ensure clock synchronization (essential for WireGuard/TLS handshakes):
  ```bash
  sudo timedatectl set-ntp on
  ```

#### 4. Transferring Xray to Ubuntu
From your Windows PowerShell, securely copy the Xray archive to your VM using built-in SCP:
```powershell
scp .\Xray-linux-64.zip user@10.10.10.2:/home/user/
```
Inside Ubuntu, extract it:
```bash
unzip Xray-linux-64.zip -d ~/xray && chmod +x ~/xray/xray
```

#### 5. Configuring Static IP via Netplan in Ubuntu
Edit the Netplan configuration:
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```
Configure your loopback interface (e.g., `ens37`):
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
Apply the configuration:
```bash
sudo netplan apply
```

#### 6. Windows Host Network Settings
* **Microsoft Loopback Adapter:**
  * IP: `10.10.10.1` | Subnet: `255.255.255.0` | Gateway: `10.10.10.2` | DNS: `1.1.1.1`
  * IPv6: `fd00:10:10::1` | Prefix: `64` | IPv6 Gateway: `fd00:10:10::2`
* **Physical Wi-Fi Adapter:**
  * Static IP on home network (e.g., `192.168.1.50`).
  * **Leave Default Gateway BLANK!** *(Forces all traffic through Loopback).*
* **Internet Connection Sharing (ICS) to Other Devices:**
  * In `ncpa.cpl`, right-click Microsoft Loopback Adapter > Properties > Sharing tab > check *"Allow other network users..."* and select your Wi-Fi or Hotspot adapter to broadcast clean internet on `192.168.137.x`.
* **Run VM Headless (Background):**
  In VMware, select **VM > Power > Run in Background** (or close the window and choose *"Run in background"*).

---

### 🚀 Running the Ubuntu Gateway (Method 3)

#### Option A: Manual Session Run (Temporary Testing)
Execute these commands in your active terminal session (resets on reboot):
```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=2

sudo ufw disable
sudo iptables -P FORWARD ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -o tun0 -j MASQUERADE

sudo ~/xray/xray run -c ~/xray/config.json
```

#### Option B: Make It Permanent (Optional Auto-Start on Boot)
```bash
# 1. Persistent Sysctl
sudo bash -c 'cat << EOF >> /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.rp_filter = 2
EOF'
sudo sysctl -p

# 2. Persistent Firewall
sudo apt install -y iptables-persistent
sudo netfilter-persistent save

# 3. Systemd Service
sudo bash -c 'cat << EOF > /etc/systemd/system/xray.service
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
EOF'

sudo systemctl daemon-reload
sudo systemctl enable --now xray
```

---

### 🛠️ Optional Features & Customizations

#### 1. Turn Ubuntu or Windows into a Local DNS Server (Port 53)
To have Ubuntu or Windows act as a local DNS resolver (like `1.1.1.1`) for all devices on your LAN:
Add a `dokodemo-door` inbound to `"inbounds"` in your `config.json`:
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
Add `"dns-in"` to your fallback routing rule targeting `"warp-out"`. Devices can now set their DNS to the Gateway machine's IP!

#### 2. Removing Iran Bypass Rules (Route 100% Traffic through WARP)
To force all traffic through the encrypted tunnel (e.g., when outside Iran or requiring full anonymity), simply **delete** these two rules from the `"routing"` section:
```json
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

#### 3. Windows Firewall Configuration for Inbound RDP
To allow incoming Remote Desktop connections from other mesh devices:
In Windows PowerShell (Admin), open port 3389:
```powershell
New-NetFirewallRule -DisplayName "Allow RDP Mesh" -Direction Inbound -LocalPort 3389 -Protocol TCP -Action Allow
```

---

### 📂 Configuration Catalog

All production configurations reside in the [`configs/`](configs/) folder:

| Configuration File | Target Platform | Description |
| :--- | :--- | :--- |
| [`xray-linux.json`](configs/xray-linux.json) | Ubuntu Server | Gateway router mode, `tun0`, LAN SOCKS/HTTP, Dual-Stack IPv6 & Smart Iran routing. |
| [`xray-windows-tun.json`](configs/xray-windows-tun.json) | Windows 10/11 & Server | Standalone full-system tunnel via `wintun.dll` with NCSI-compatible DNS routing. |
| [`xray-windows-proxy.json`](configs/xray-windows-proxy.json) | Windows 10/11 | Lightweight local SOCKS5 (`2080`) / HTTP (`2081`) proxy without admin rights. |
| [`xray-android.json`](configs/xray-android.json) | Android | Optimized for v2rayNG / NekoBox using Android's native VpnService on port `10808`. |
| [`singbox-1.15.json`](configs/singbox-1.15.json) | Multi-platform | Modern Sing-box 1.15+ schema using root `endpoints` and rule-actions. |

---

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### معرفی پروژه
این کتابچه راهنما شامل معماری‌های مهندسی‌شده و آزموده‌شده برای عبور از فیلترینگ پیشرفته (DPI)، تبدیل سیستم‌ها به گیت‌وی‌های ضد فیلتر و برقراری ارتباطات دوطرفه مش (Cloudflare Zero Trust / WARP Connector) بدون نیاز به خرید سرور مجازی (VPS) خارجی است.

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

### 🏗️ روش‌های پیاده‌سازی گیت‌وی (Gateway)

بسته به شرایط و سلیقه خود یکی از ۳ روش زیر را انتخاب کنید:

```text
┌────────────────────────────────────────────────────────────────────────┐
│  روش ۱: ویندوز سرور در VMware به عنوان گیت‌وی (محیط کاملاً گرافیکی RRAS)│
│  روش ۲: سرور اوبونتو فیزیکی مستقل در شبکه خانگی (دستگاه اختصاصی LAN)   │
│  روش ۳: ترکیب VMware + اوبونتو سرور مینیمال + کارت Loopback (بسیار سبک) │
└────────────────────────────────────────────────────────────────────────┘
```

---

#### 🥇 روش اول: ویندوز سرور در VMware به عنوان گیت‌وی (محیط کاملاً گرافیکی با RRAS)
اگر کار با پنجره‌ها و منوهای گرافیکی ویندوز (GUI) را به تایپ دستورات ترمینال لینوکس ترجیح می‌دهید و رم کافی دارید (نیازمند اختصاص ۲ تا ۳ گیگابایت رم به ماشین مجازی):

```text
[ ویندوز اصلی لپ‌تاپ ] ──── (کارت Loopback: 10.10.10.1) ────┐
                                                             ▼
[ مودم خانگی اینترنت ] ◄── (کارت Bridged) ── [ ویندوز سرور در VMware (10.10.10.2) ]
                                                            │
                                                   [ Xray TUN (CloudflareMesh) ]
                                                            │
                                                   [ سرویس NAT در RRAS ]
                                                            │
                                                            ▼
                                                   [ ابر کلودفلر ]
```

##### مرحله ۱: تنظیم سخت‌افزار ماشین مجازی
در نرم‌افزار VMware، یک ماشین مجازی ویندوز سرور (۲۰۱۹، ۲۰۲۲ یا ۲۰۲۵ با قابلیت Desktop Experience) بسازید:
* **۲ هسته پردازنده**، **۲ تا ۳ گیگابایت رم** و **دیسک ۳۵ گیگابایتی Thin Provisioned**.
* **کارت شبکه ۱:** روی **Bridged** به کارت وای‌فای فیزیکی (برای دریافت اینترنت خام مودم).
* **کارت شبکه ۲:** روی **Bridged** به کارت `Microsoft KM-TEST Loopback Adapter` (لینک داخلی به ویندوز اصلی).

##### مرحله ۲: اجرای Xray در ویندوز سرور
1. فایل‌های `xray.exe`، [`wintun.dll`](https://www.wintun.net) و کانفیگ `xray-windows-tun.json` را داخل ماشین مجازی قرار دهید.
2. با دسترسی Administrator اجرا کنید تا کارت شبکه `CloudflareMesh` ساخته شده و اینترنت ویندوز سرور آزاد شود.

##### مرحله ۳: فعال‌سازی روتینگ و NAT در ویندوز سرور (از طریق GUI)
1. برنامه **Server Manager** را باز کرده و روی **Add roles and features** کلیک کنید.
2. رول **Remote Access** را انتخاب کنید و در بخش Role Services، تیک **Routing** را بزنید و نصب کنید.
3. کنسول **Routing and Remote Access** را باز کنید (`rrasmgmt.msc`).
4. روی نام سرور کلیک‌راست کرده > **Configure and Enable Routing and Remote Access**.
5. گزینه **Custom configuration** را انتخاب کنید > تیک‌های **NAT** و **LAN routing** را بزنید و سرویس را استارت کنید.
6. در پنل سمت چپ، زیرشاخه **IPv4 > NAT** را باز کنید:
   * روی **NAT** کلیک‌راست کنید > **New Interface...** > کارت شبکه **`CloudflareMesh`** را انتخاب کنید.
   * گزینه *"Public interface connected to the Internet"* را انتخاب کرده و تیک *"Enable NAT on this interface"* را بزنید > OK.
   * مجدداً روی **NAT** کلیک‌راست کنید > **New Interface...** > کارت شبکه داخلی (متصل به Loopback) را انتخاب کنید.
   * گزینه *"Private interface connected to private network"* را انتخاب کنید > OK.

##### مرحله ۴: تنظیم ویندوز اصلی لپ‌تاپ
* روی کارت **Microsoft Loopback**: در تنظیمات IPv4، مقدار Default Gateway را آی‌پی ویندوز سرور (`10.10.10.2`) بگذارید.
* روی کارت **Wi-Fi فیزیکی**: آی‌پی دستی بگذارید و فیلد **Default Gateway را کاملاً خالی بگذارید**.

---

#### 🥈 روش دوم: سرور اوبونتو فیزیکی مستقل در شبکه خانگی
ایده‌آل اگر یک مینی‌کیس، رزبری‌پای یا لپ‌تاپ قدیمی با سیستم‌عامل اوبونتو متصل به مودم خانه دارید:

```text
[ کلاینت‌ها: گوشی / لپ‌تاپ / تلویزیون ]
         │ (تنظیم Default Gateway روی 192.168.1.100)
         ▼
[ سرور اوبونتو گیت‌وی (192.168.1.100) ]
         │ ── تونل Xray با نویز روی tun0 ──► [ ابر کلودفلر ]
         ▼
[ مودم خانگی (192.168.1.1) ] ──► اینترنت
```

1. فایل `xray-linux.json` را در اوبونتو اجرا کنید.
2. فوروارد هسته و NAT را روشن کنید:
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
   ```
3. در سایر دستگاه‌ها، تنها کافیست **Default Gateway** را آی‌پی اوبونتو (`192.168.1.100`) بگذارید.

---

#### 🥉 روش سوم: پیاده‌سازی روی یک تک‌لپ‌تاپ با VMware و اوبونتو مینیمال (بسیار سبک)
برای راه‌اندازی گیت‌وی با مصرف رم زیر ۲۰۰ مگابایت و بدون فشار روی پردازنده لپ‌تاپ:

```text
[ گوشی و سایر دستگاه‌ها ]
         │ (Wi-Fi / ICS: 192.168.137.x)
         ▼
[ لپ‌تاپ ویندوزی ] ──── (کارت Loopback: 10.10.10.1) ────┐
                                                         ▼
[ مودم خانگی اینترنت ] ◄── (کارت Bridged) ── [ اوبونتو سرور (10.10.10.2) ]
                                                         │
                                               [ Xray با نویز روی tun0 ]
                                                         │
                                                         ▼
                                               [ ابر کلودفلر ]
```

---

### 🔧 راهنمای پیاده‌سازی گام‌به‌گام روش سوم

#### ۱. نصب کارت شبکه Microsoft Loopback در ویندوز
1. کلیدهای `Win + R` را بزنید، تایپ کنید **`hdwwiz.exe`** و اینتر کنید.
2. گزینه دوم (*Install the hardware that I manually select from a list*) را انتخاب کرده و Next را بزنید.
3. گزینه **Network adapters** را انتخاب کنید.
4. در سمت چپ **Microsoft** و در سمت راست **`Microsoft KM-TEST Loopback Adapter`** را انتخاب کرده و Next را بزنید تا نصب شود.

#### ۲. تنظیم Virtual Network Editor در VMware
1. در نرم‌افزار VMware Workstation به مسیر **Edit > Virtual Network Editor** بروید.
2. دکمه **Change Settings** را برای دسترسی ادمین بزنید.
3. دکمه **Add Network** را بزنید و یک شبکه جدید (مثلاً `VMnet2`) اضافه کنید.
4. آن را روی **Bridged** قرار داده و در منوی کشویی کارت **`Microsoft KM-TEST Loopback Adapter`** را انتخاب کنید.
5. کارت شبکه دوم ماشین مجازی اوبونتو را روی این `VMnet2` قرار دهید.

#### ۳. نصب اصولی Ubuntu Server 24.04 LTS (Minimal)
* سخت‌افزار ماشین مجازی: **۱ هسته پردازنده**، **۱۰۲۴ مگابایت رم** و **دیسک ۱۵ گیگابایتی Thin Provisioned**.
* در مراحل نصب گزینه **Ubuntu Server (Minimised)** را انتخاب کنید.
* نصب ابزارهای مهم سیستم:
  ```bash
  sudo apt update && sudo apt install -y open-vm-tools curl unzip iptables net-tools
  ```
* فعال‌سازی همگام‌سازی ساعت با اینترنت (بسیار حیاتی برای هندشیک وایرگارد):
  ```bash
  sudo timedatectl set-ntp on
  ```

#### ۴. انتقال فایل‌های Xray به اوبونتو از طریق ویندوز
در PowerShell ویندوز با دستور زیر فایل زیپ Xray را مستقیم به ماشین مجازی بفرستید:
```powershell
scp .\Xray-linux-64.zip user@10.10.10.2:/home/user/
```
سپس در اوبونتو آن را اکسترکت کنید:
```bash
unzip Xray-linux-64.zip -d ~/xray && chmod +x ~/xray/xray
```

#### ۵. تنظیم آی‌پی ثابت با Netplan در اوبونتو
فایل Netplan را ویرایش کنید:
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```
تنظیمات اینترفیس لوپ‌بک (مثلاً `ens37`):
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
اعمال تغییرات:
```bash
sudo netplan apply
```

#### ۶. تنظیمات کارت‌های شبکه در ویندوز
* **کارت شبکه Microsoft Loopback:**
  * IP: `10.10.10.1` | Subnet: `255.255.255.0` | Gateway: `10.10.10.2` | DNS: `1.1.1.1`
  * IPv6: `fd00:10:10::1` | Prefix: `64` | IPv6 Gateway: `fd00:10:10::2`
* **کارت شبکه Wi-Fi فیزیکی:**
  * آی‌پی دستی ست کنید (مثلاً `192.168.1.50`).
  * **Default Gateway: کاملاً خالی بگذارید!** *(کلید جلوگیری از نشت و لوپ).*
* **اشتراک‌گذاری اینترنت (ICS) با گوشی:**
  * در `ncpa.cpl`، روی کارت Loopback کلیک‌راست کرده > Properties > تب Sharing > تیک اشتراک‌گذاری را بزنید و کارت هات‌اسپات را انتخاب کنید تا دستگاه‌ها روی رنج `192.168.137.x` اینترنت بگیرند.
* **اجرای ماشین مجازی در پس‌زمینه (Headless):**
  هنگام بستن پنجره VMware، گزینه **Run in background** را بزنید تا هیچ پنجره گرافیکی باز نماند و رم لپ‌تاپ هدر نرود.

---

### 🚀 نحوه اجرای گیت‌وی در اوبونتو (روش سوم)

#### حالت اول: اجرای موقت دستی (فقط برای تست سشن فعلی)
این دستورات با ریستارت شدن لینوکس ریست می‌شوند:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=2

sudo ufw disable
sudo iptables -P FORWARD ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -o tun0 -j MASQUERADE

sudo ~/xray/xray run -c ~/xray/config.json
```

#### حالت دوم: دائمی کردن تنظیمات (اختیاری - اجرای خودکار با بوت)
```bash
# ۱. دائمی کردن تنظیمات هسته
sudo bash -c 'cat << EOF >> /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.rp_filter = 2
EOF'
sudo sysctl -p

# ۲. دائمی کردن قوانین فایروال
sudo apt install -y iptables-persistent
sudo netfilter-persistent save

# ۳. سرویس Systemd برای اجرای پس‌زمینه
sudo bash -c 'cat << EOF > /etc/systemd/system/xray.service
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
EOF'

sudo systemctl daemon-reload
sudo systemctl enable --now xray
```

---

### 🛠️ بخش‌های اختیاری و شخصی‌سازی

#### ۱. تبدیل اوبونتو یا ویندوز به یک سرور DNS محلی (پورت ۵۳)
اگر می‌خواهید سیستم شما مانند `1.1.1.1` برای دستگاه‌های دیگر سرور DNS شود:
در بخش `"inbounds"` فایل `config.json` این بلوک را اضافه کنید:
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
تگ `"dns-in"` را به قانون منتهی به `"warp-out"` در بخش routing اضافه کنید. حالا سایر سیستم‌ها می‌توانند DNS خود را آی‌پی سیستم شما بگذارند!

#### ۲. نحوه حذف قوانین روتینگ ایران (عبور ۱۰۰٪ ترافیک از وارپ)
اگر مایلید تمام ترافیک و حتی سایت‌های ایرانی از داخل تونل وارپ رد شوند:
در بخش `"routing"` هر کانفیگی، **این دو قانون را حذف کنید**:
```json
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

#### ۳. تنظیم فایروال ویندوز برای دسترسی به RDP در شبکه مش
اگر قصد اتصال ریموت دسکتاپ از گوشی به ویندوز را دارید، پورت ۳۳۸۹ را در فایروال باز کنید:
```powershell
New-NetFirewallRule -DisplayName "Allow RDP Mesh" -Direction Inbound -LocalPort 3389 -Protocol TCP -Action Allow
```

---

### 📂 کاتالوگ فایل‌های کانفیگ

تمام فایل‌های آماده و استاندارد در پوشه [`configs/`](configs/) قرار دارند:

| نام فایل | سیستم‌عامل هدف | توضیحات معماری |
| :--- | :--- | :--- |
| [`xray-linux.json`](configs/xray-linux.json) | سرور اوبونتو | حالت روتر گیت‌وی، اینترفیس `tun0`، پروکسی شبکه روی پورت ۲۰۸۰، روتینگ ایران و IPv6. |
| [`xray-windows-tun.json`](configs/xray-windows-tun.json) | ویندوز ۱۰ و ۱۱ و سرور | تونل کامل سیستم با `wintun.dll` و مدیریت خودکار دی‌ان‌اس ویندوز (NCSI). |
| [`xray-windows-proxy.json`](configs/xray-windows-proxy.json) | ویندوز ۱۰ و ۱۱ | حالت پروکسی سبک SOCKS5 (`2080`) و HTTP (`2081`) بدون نیاز به دسترسی ادمین. |
| [`xray-android.json`](configs/xray-android.json) | اندروید | مخصوص v2rayNG و NekoBox با تکیه بر VpnService بومی سیستم‌عامل روی پورت ۱۰۸۰۸. |
| [`singbox-1.15.json`](configs/singbox-1.15.json) | چندسکویی | ساختار مدرن Sing-box 1.15 با ریشه `endpoints` و حذف فیلدهای منسوخ‌شده. |

---

### 📄 پروانه انتشار (License)
این پروژه تحت پروانه [MIT License](LICENSE) منتشر شده است.
