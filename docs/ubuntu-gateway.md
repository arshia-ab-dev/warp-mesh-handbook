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
This guide covers deploying Ubuntu Server 24.04 LTS (Minimal) as an anti-censorship gateway. It consumes **under 200 MB of RAM** and almost 0% CPU at idle, utilizing Linux's native `netfilter` subsystem for seamless dual-stack (IPv4/IPv6) routing.

---

### Deployment Scenarios

#### Scenario A: Standalone Hardware (Physical Mini-PC, Old Laptop, or Raspberry Pi)
* Connect the Ubuntu machine directly to your home router (via Ethernet or Wi-Fi).
* Its IP (e.g., `192.168.1.100`) acts as the Default Gateway for your home network.

#### Scenario B: Single Laptop via VMware & Loopback Adapter
* Eliminates 802.11 3-address wireless frame bridging issues by routing traffic through an isolated virtual connection:
  * **Adapter 1:** Bridged to physical Wi-Fi (for upstream internet).
  * **Adapter 2:** Bridged to the host's `Microsoft KM-TEST Loopback Adapter` (internal link to Windows).

---

### Step-by-Step Installation & Setup

#### 1. Hardware Allocation (for VMware)
* **CPU:** 1 Core | **RAM:** 1024 MB | **Disk:** 15 GB Thin Provisioned.
* In the OS installer, select **Ubuntu Server (Minimised)**.
* Install required tools:
  ```bash
  sudo apt update && sudo apt install -y open-vm-tools curl unzip iptables net-tools
  ```
* Ensure time synchronization (critical for WireGuard handshakes):
  ```bash
  sudo timedatectl set-ntp on
  ```

#### 2. Setting Up Windows Loopback & VMware (Scenario B Only)
1. **Install Microsoft Loopback Adapter on Windows:**
   * Press `Win + R` > run `hdwwiz.exe`.
   * Choose *"Install hardware manually"* > **Network adapters** > **Microsoft** > **`Microsoft KM-TEST Loopback Adapter`**.
2. **Configure VMware Virtual Network Editor:**
   * Open **Virtual Network Editor** as Administrator.
   * Add network `VMnet2` > select **Bridged** > Bridge to **`Microsoft KM-TEST Loopback Adapter`**.
   * Assign Ubuntu's second network adapter to `VMnet2`.

#### 3. Transferring & Extracting Xray
From your Windows PowerShell, copy the Xray archive over SCP:
```powershell
scp .\Xray-linux-64.zip user@10.10.10.2:/home/user/
```
In Ubuntu, extract it:
```bash
unzip Xray-linux-64.zip -d ~/xray && chmod +x ~/xray/xray
```

#### 4. Configuring Static IP via Netplan
Edit your Netplan file:
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

#### 5. Windows Host Network Settings (Scenario B)
* **Microsoft Loopback Adapter:**
  * IP: `10.10.10.1` | Subnet: `255.255.255.0` | Gateway: `10.10.10.2` | DNS: `1.1.1.1`
  * IPv6: `fd00:10:10::1` | Prefix: `64` | IPv6 Gateway: `fd00:10:10::2`
* **Physical Wi-Fi Adapter:**
  * Static IP on home network (e.g., `192.168.1.50`).
  * **Default Gateway: Leave BLANK!**

---

### Running the Gateway

#### Option A: Manual Session Run (Temporary Testing)
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

#### Option B: Permanent Auto-Start (Systemd)
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

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### معرفی و کلیات
این راهنما نحوه راه‌اندازی Ubuntu Server 24.04 LTS (Minimal) را به عنوان یک گیت‌وی ضد فیلتر آموزش می‌دهد. این روش با مصرف **کمتر از ۲۰۰ مگابایت رم** و فشار پردازشی نزدیک به صفر، از زیرسیستم `netfilter` لینوکس برای روتینگ شفاف و دوگانه (IPv4/IPv6) استفاده می‌کند.

---

### سناریوهای استقرار

#### سناریوی اول: سخت‌افزار مستقل (مینی‌کیس، رزبری‌پای یا لپ‌تاپ قدیمی)
* دستگاه اوبونتو با کابل شبکه یا وای‌فای مستقیماً به مودم خانه متصل است.
* آی‌پی آن (مثلاً `192.168.1.100`) به عنوان Default Gateway سایر دستگاه‌های شبکه خانگی تنظیم می‌شود.

#### سناریوی دوم: پیاده‌سازی روی یک تک‌لپ‌تاپ با VMware و کارت Loopback
* برای دور زدن باگ فریم‌های ۳ آدرسی وای‌فای، ارتباط ویندوز اصلی با اوبونتو از طریق کارت شبکه مجازی Loopback ایزوله می‌شود:
  * **کارت ۱:** Bridged به وای‌فای فیزیکی (اینترنت خام).
  * **کارت ۲:** Bridged به کارت `Microsoft KM-TEST Loopback Adapter` (لینک داخلی به ویندوز).

---

### مراحل گام‌به‌گام راه‌اندازی

#### ۱. تنظیمات سخت‌افزار ماشین مجازی (برای سناریوی دوم)
* **پردازنده:** ۱ هسته | **رم:** ۱۰۲۴ مگابایت | **دیسک:** ۱۵ گیگابایت Thin Provisioned.
* در مراحل نصب، گزینه **Ubuntu Server (Minimised)** را انتخاب کنید.
* نصب پکیج‌های پایه:
  ```bash
  sudo apt update && sudo apt install -y open-vm-tools curl unzip iptables net-tools
  ```
* فعال‌سازی همگام‌سازی ساعت (حیاتی برای هندشیک وایرگارد):
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
   * کارت شبکه دوم ماشین مجازی اوبونتو را روی این `VMnet2` بگذارید.

#### ۳. انتقال فایل‌های Xray به اوبونتو
در PowerShell ویندوز، فایل زیپ Xray را با SCP ارسال کنید:
```powershell
scp .\Xray-linux-64.zip user@10.10.10.2:/home/user/
```
در اوبونتو آن را اکسترکت کنید:
```bash
unzip Xray-linux-64.zip -d ~/xray && chmod +x ~/xray/xray
```

#### ۴. تنظیم آی‌پی ثابت با Netplan در اوبونتو
فایل Netplan را ویرایش کنید:
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```
کارت شبکه متصل به Loopback (مثلاً `ens37`) را تنظیم کنید:
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
  * آی‌پی دستی بگذارید (مثلاً `192.168.1.50`).
  * **Default Gateway: کاملاً خالی بگذارید!**

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

# ۳. سرویس Systemd
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
