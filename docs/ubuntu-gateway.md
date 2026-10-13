<div align="center">

# 🐧 Ubuntu Server Gateway Deployment Guide
### Complete Setup for Standalone Hardware & VMware Loopback Environments

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
This guide details deploying Ubuntu Server 24.04 LTS (Minimal) as an anti-censorship gateway. It consumes **under 200 MB of RAM** and near-zero CPU at idle, using Linux's native `netfilter` subsystem for dual-stack (IPv4/IPv6) routing.

---

### Deployment Models

* **Model A: Standalone Appliance (Physical Mini-PC, Old Laptop, or Raspberry Pi):**  
  Directly connected to the home router via Ethernet/Wi-Fi. Its IP (e.g., `192.168.1.100`) serves as the Default Gateway for all home devices.
* **Model B: Single Laptop Setup via VMware & Loopback Adapter:**  
  Circumvents 802.11 3-address wireless frame bridging limitations by routing host traffic through an isolated virtual link:
  * **Adapter 1 (VM):** Bridged to physical Wi-Fi (upstream internet).
  * **Adapter 2 (VM):** Bridged to `Microsoft KM-TEST Loopback Adapter` (internal link to Host).

---

### Setup Instructions

#### 1. Virtual Machine Hardware (for Model B in VMware)
* **CPU:** 1 Core | **RAM:** 1024 MB | **Disk:** 15 GB Thin Provisioned.
* Installer selection: **Ubuntu Server (Minimised)**.
* Install required tools inside Ubuntu:
  ```bash
  sudo apt update && sudo apt install -y open-vm-tools curl unzip iptables net-tools
  ```
* Ensure time synchronization (critical for WireGuard handshakes):
  ```bash
  sudo timedatectl set-ntp on
  ```

#### 2. Virtual Network Configuration (for Model B)
1. **Install Microsoft Loopback Adapter on Windows Host:**
   * Press `Win + R` > run `hdwwiz.exe`.
   * Select *"Install hardware manually"* > **Network adapters** > **Microsoft** > **`Microsoft KM-TEST Loopback Adapter`**.
2. **Configure VMware Virtual Network Editor:**
   * Open **Virtual Network Editor** as Administrator.
   * Add network `VMnet2` > select **Bridged** > Bridge to **`Microsoft KM-TEST Loopback Adapter`**.
   * Assign Ubuntu's second network adapter to `VMnet2`.

#### 3. Transferring & Extracting Xray on Ubuntu
From Windows PowerShell, copy the Xray archive over SCP:
```powershell
scp .\Xray-linux-64.zip user@10.10.10.2:/home/user/
```
In Ubuntu terminal, extract and set executable permissions:
```bash
unzip Xray-linux-64.zip -d ~/xray && chmod +x ~/xray/xray
```

#### 4. Setting Static IP via Netplan in Ubuntu
Edit the Netplan configuration:
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```
Configure your interface (e.g., `ens37` for loopback or `eth0` for LAN):
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
Apply:
```bash
sudo netplan apply
```

#### 5. Windows Host Network Settings (Model B Only)
* **Microsoft Loopback Adapter:**
  * IPv4: `10.10.10.1` | Subnet: `255.255.255.0` | Gateway: `10.10.10.2` | DNS: `1.1.1.1`
  * IPv6: `fd00:10:10::1` | Prefix: `64` | IPv6 Gateway: `fd00:10:10::2`
* **Physical Wi-Fi Adapter:**
  * Static IP on home network (e.g., `192.168.1.50`).
  * **Leave Default Gateway BLANK!** *(Prevents traffic leaks and route loops).*

---

### Executing the Gateway on Ubuntu

#### Mode A: Manual Session Run (Temporary Testing)
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
sudo ~/xray/xray run -c ~/xray/config.json
```

#### Mode B: Permanent Auto-Start (Systemd Service)
```bash
# 1. Persistent Sysctl Settings
sudo bash -c 'cat << EOF >> /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.rp_filter = 2
EOF'
sudo sysctl -p

# 2. Persistent Firewall Rules
sudo apt install -y iptables-persistent
sudo netfilter-persistent save

# 3. Systemd Service Unit
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

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### معرفی و کلیات
این راهنما نحوه راه‌اندازی Ubuntu Server 24.04 LTS (Minimal) را به عنوان گیت‌وی ضد فیلتر آموزش می‌دهد. این روش با مصرف **کمتر از ۲۰۰ مگابایت رم** و فشار پردازشی نزدیک به صفر، از زیرسیستم `netfilter` لینوکس برای روتینگ شفاف و دوگانه (IPv4/IPv6) استفاده می‌کند.

---

### سناریوهای استقرار

* **سناریوی اول: سخت‌افزار مستقل (مینی‌کیس، رزبری‌پای یا لپ‌تاپ قدیمی):**  
  مستقیماً با کابل یا وای‌فای به مودم وصل است و آی‌پی آن (مثلاً `192.168.1.100`) به عنوان گیت‌وی دستگاه‌های خانگی تنظیم می‌شود.
* **سناریوی دوم: پیاده‌سازی روی یک تک‌لپ‌تاپ با VMware و کارت Loopback:**  
  برای دور زدن باگ فریم‌های ۳ آدرسی وای‌فای، ترافیک لپ‌تاپ از طریق کارت مجازی Loopback به اوبونتو می‌رسد:
  * **کارت ۱ (ماشین مجازی):** Bridged به وای‌فای فیزیکی (اینترنت خام).
  * **کارت ۲ (ماشین مجازی):** Bridged به کارت `Microsoft KM-TEST Loopback Adapter` (لینک داخلی به ویندوز).

---

### مراحل گام‌به‌گام راه‌اندازی

#### ۱. تنظیمات سخت‌افزار ماشین مجازی (برای سناریوی دوم)
* **پردازنده:** ۱ هسته | **رم:** ۱۰۲۴ مگابایت | **دیسک:** ۱۵ گیگابایت Thin Provisioned.
* در مراحل نصب گزینه **Ubuntu Server (Minimised)** را انتخاب کنید.
* نصب پکیج‌های پایه در اوبونتو:
  ```bash
  sudo apt update && sudo apt install -y open-vm-tools curl unzip iptables net-tools
  ```
* فعال‌سازی همگام‌سازی ساعت با اینترنت:
  ```bash
  sudo timedatectl set-ntp on
  ```

#### ۲. ساخت کارت Loopback و تنظیم VMware (مخصوص سناریوی دوم)
1. **ساخت کارت Loopback در ویندوز:**
   * کلیدهای `Win + R` را بزنید و دستور `hdwwiz.exe` را اجرا کنید.
   * گزینه دوم (*نصب دستی*) > شاخه **Network adapters** > سازنده **Microsoft** > مدل **`Microsoft KM-TEST Loopback Adapter`**.
2. **تنظیم Virtual Network Editor در VMware:**
   * منوی **Edit > Virtual Network Editor** را با دسترسی ادمین باز کنید.
   * یک شبکه جدید (مثلاً `VMnet2`) بسازید > حالت **Bridged** را انتخاب کرده و آن را به **`Microsoft KM-TEST Loopback Adapter`** پل بزنید.
   * کارت شبکه دوم ماشین مجازی اوبونتو را روی این `VMnet2` قرار دهید.

#### ۳. انتقال فایل‌های Xray به اوبونتو
در PowerShell ویندوز اصلی:
```powershell
scp .\Xray-linux-64.zip user@10.10.10.2:/home/user/
```
در ترمینال اوبونتو:
```bash
unzip Xray-linux-64.zip -d ~/xray && chmod +x ~/xray/xray
```

#### ۴. تنظیم آی‌پی ثابت با Netplan در اوبونتو
فایل Netplan را باز کنید:
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

#### ۵. تنظیمات کارت‌های شبکه در ویندوز (سناریوی دوم)
* **کارت شبکه Microsoft Loopback:**
  * IP: `10.10.10.1` | ساب‌نت: `255.255.255.0` | گیت‌وی: `10.10.10.2` | دی‌ان‌اس: `1.1.1.1`
  * IPv6: `fd00:10:10::1` | طول پیشوند: `64` | گیت‌وی نسخه ۶: `fd00:10:10::2`
* **کارت وای‌فای فیزیکی ویندوز:**
  * آی‌پی دستی ست کنید (مثلاً `192.168.1.50`).
  * **Default Gateway: کاملاً خالی بگذارید!** *(کلید جلوگیری از نشت ترافیک).*

---

### نحوه اجرای گیت‌وی در اوبونتو

#### روش اول: اجرای دستی موقت (تست در سشن فعلی)
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

#### روش دوم: دائمی کردن تنظیمات (اجرای خودکار با بوت)
```bash
# ۱. دائمی کردن تنظیمات هسته
sudo bash -c 'cat << EOF >> /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.rp_filter = 2
EOF'
sudo sysctl -p

# ۲. دائمی کردن فایروال
sudo apt install -y iptables-persistent
sudo netfilter-persistent save

# ۳. سرویس خودکار Systemd
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
