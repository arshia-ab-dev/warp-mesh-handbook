<div align="center">

# ⚡ Windows Server Core Gateway Deployment Guide
### Lightweight CLI-Driven Gateway via PowerShell & Netsh NAT

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
Windows Server Core delivers a Windows routing environment without the resource consumption of the Desktop Experience GUI. It operates on **800 MB to 1 GB of RAM** and is configured entirely via PowerShell and `netsh`.

---

### Virtual Machine Configuration in VMware
* **OS Edition:** Windows Server 2022 / 2025 (Select **Server Core Installation**).
* **Hardware Specs:** 1 or 2 CPU Cores | **1024 to 1536 MB RAM** | 20 GB Thin Provisioned disk.
* **Network Adapter 1:** Bridged to physical Wi-Fi (Upstream raw internet).
* **Network Adapter 2:** Bridged to `Microsoft KM-TEST Loopback Adapter` (Internal host link).

---

### Step-by-Step Implementation

#### 1. Setup Xray on Server Core
In Server Core PowerShell, create a working directory and extract Xray:
```powershell
New-Item -ItemType Directory -Path C:\xray -Force
Set-Location C:\xray

# Expand Xray archive (assuming archive is copied or downloaded)
Expand-Archive -Path .\Xray-windows-64.zip -DestinationPath C:\xray -Force
```
*(Ensure `wintun.dll` and `xray-windows-tun.json` reside inside `C:\xray`).*

#### 2. Install Routing Role & Enable Kernel Routing
In Server Core PowerShell:
```powershell
# Install RRAS Routing Role
Install-WindowsFeature -Name Routing -IncludeManagementTools

# Enable IP routing in Windows registry
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' -Name 'IPEnableRouter' -Value 1

# Configure and start Remote Access service
Set-Service -Name RemoteAccess -StartupType Automatic
Start-Service -Name RemoteAccess
```

#### 3. Run Xray with Wintun
Start the Xray tunnel:
```powershell
cd C:\xray
.\xray.exe run -c .\xray-windows-tun.json
```
*(The virtual network interface `CloudflareMesh` will be created).*

#### 4. Configure NAT via Netsh
Open a separate PowerShell window and execute:
```powershell
# 1. Install NAT routing protocol
netsh routing ip nat install

# 2. Add CloudflareMesh as external public NAT interface
netsh routing ip nat add interface name="CloudflareMesh" mode=full

# 3. Add internal loopback adapter as private interface
netsh routing ip nat add interface name="Ethernet0" mode=private
```
*(Note: Replace `Ethernet0` with the actual name of your internal loopback adapter, visible via `Get-NetAdapter`).*

#### 5. Configure Windows Host Network
On your primary Windows laptop:
* On **Microsoft Loopback Adapter**: Set Default Gateway to Server Core's IP (e.g., `10.10.10.2`).
* On **Physical Wi-Fi Adapter**: Leave Default Gateway **BLANK**.

---

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### معرفی و کلیات
نسخه Windows Server Core محیط روتینگ ویندوز را بدون هدررفت رم برای محیط دسکتاپ فراهم می‌کند. این سیستم با **حدود ۸۰۰ مگابایت تا ۱ گیگابایت رم** اجرا شده و تمام تنظیمات آن از طریق PowerShell و دستورات استاندارد `netsh` انجام می‌شود.

---

### تنظیمات ماشین مجازی در VMware
* **نسخه:** Windows Server 2022 یا 2025 (انتخاب گزینه **Server Core Installation**).
* **سخت‌افزار:** ۱ یا ۲ هسته پردازنده | **۱۰۲۴ تا ۱۵۳۶ مگابایت رم** | دیسک ۲۰ گیگابایت Thin Provisioned.
* **کارت شبکه ۱:** Bridged به کارت وای‌فای فیزیکی (اینترنت خام).
* **کارت شبکه ۲:** Bridged به کارت `Microsoft KM-TEST Loopback Adapter` (لینک داخلی به هاست).

---

### مراحل گام‌به‌گام پیاده‌سازی

#### ۱. راه‌اندازی فایل‌های Xray در Server Core
ترمینال PowerShell سرور Core را باز کرده و پوشه کار را بسازید:
```powershell
New-Item -ItemType Directory -Path C:\xray -Force
Set-Location C:\xray

# استخراج فایل زیپ Xray با دستور بومی پاورشل
Expand-Archive -Path .\Xray-windows-64.zip -DestinationPath C:\xray -Force
```
*(اطمینان حاصل کنید فایل‌های `wintun.dll` و `xray-windows-tun.json` داخل `C:\xray` قرار دارند).*

#### ۲. نصب رول روتینگ و فعال‌سازی فورواردینگ
در PowerShell سرور Core:
```powershell
# نصب رول روتینگ (RRAS)
Install-WindowsFeature -Name Routing -IncludeManagementTools

# فعال‌سازی فورواردینگ پکت‌ها در رجیستری
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' -Name 'IPEnableRouter' -Value 1

# استارت خودکار سرویس ریموت اکسس
Set-Service -Name RemoteAccess -StartupType Automatic
Start-Service -Name RemoteAccess
```

#### ۳. اجرای Xray با درایور Wintun
تونل Xray را استارت کنید:
```powershell
cd C:\xray
.\xray.exe run -c .\xray-windows-tun.json
```
*(کارت شبکه مجازی `CloudflareMesh` ایجاد می‌شود).*

#### ۴. راه‌اندازی NAT با دستورات Netsh در PowerShell
یک پنجره PowerShell دیگر باز کرده و دستورات زیر را اجرا کنید:
```powershell
# ۱. نصب پروتکل NAT
netsh routing ip nat install

# ۲. معرفی کارت وارپ به عنوان اینترفیس عمومی اینترنت
netsh routing ip nat add interface name="CloudflareMesh" mode=full

# ۳. معرفی کارت داخلی متصل به Loopback به عنوان اینترفیس خصوصی
netsh routing ip nat add interface name="Ethernet0" mode=private
```
*(توجه: به جای `Ethernet0` نام کارت شبکه متصل به Loopback را بگذارید که با دستور `Get-NetAdapter` دیده می‌شود).*

#### ۵. تنظیم ویندوز اصلی لپ‌تاپ
* در ویندوز اصلی، وارد تنظیمات IPv4 کارت **Microsoft Loopback** شوید و **Default Gateway** را آی‌پی ویندوز سرور Core (مثلاً `10.10.10.2`) بگذارید.
* گیت‌وی کارت وای‌فای فیزیکی را **کاملاً خالی** بگذارید.
