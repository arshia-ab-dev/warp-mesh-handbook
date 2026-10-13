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
Windows Server Core provides a Windows-based gateway environment without the overhead of the Desktop Experience (GUI). It consumes around **800 MB to 1 GB of RAM** and can be entirely scripted via PowerShell and `netsh`.

---

### Virtual Machine Configuration in VMware
* **OS Version:** Windows Server 2022 / 2025 (Choose **Server Core Installation**).
* **Hardware Specs:** 1 or 2 CPU Cores | **1024 to 1536 MB RAM** | 20 GB Thin Provisioned disk.
* **Network Adapter 1:** Bridged to physical Wi-Fi (Raw internet access).
* **Network Adapter 2:** Bridged to `Microsoft KM-TEST Loopback Adapter` (Internal host link).

---

### Implementation Steps

#### 1. Install Routing Role & Enable Kernel Forwarding
Open PowerShell in Server Core and execute:
```powershell
# Install RRAS Routing Role
Install-WindowsFeature -Name Routing -IncludeManagementTools

# Enable IP Routing in Registry
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' -Name 'IPEnableRouter' -Value 1

# Start the Remote Access service
Set-Service -Name RemoteAccess -StartupType Automatic
Start-Service -Name RemoteAccess
```

#### 2. Run Xray with Wintun
Place `xray.exe`, `wintun.dll`, and `xray-windows-tun.json` into a folder (e.g., `C:\xray`) and run:
```powershell
cd C:\xray
.\xray.exe run -c xray-windows-tun.json
```
*(The virtual network interface `CloudflareMesh` will be established).*

#### 3. Configure NAT in 3 Netsh Commands
Open a separate Command Prompt / PowerShell window:
```cmd
:: 1. Install the NAT routing protocol
netsh routing ip nat install

:: 2. Add CloudflareMesh as the external public NAT interface
netsh routing ip nat add interface name="CloudflareMesh" mode=full

:: 3. Add the internal loopback-connected interface as private
netsh routing ip nat add interface name="Ethernet0" mode=private
```
*(Replace `Ethernet0` with the actual name of your internal network card shown in `netsh interface show interface`).*

#### 4. Configure Windows Host
* On the host's **Microsoft Loopback Adapter**: Set Default Gateway to Server Core's IP (e.g., `10.10.10.2`).
* On the host's **Physical Wi-Fi Adapter**: Leave Default Gateway **BLANK**.

---

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### معرفی و کلیات
نسخه Windows Server Core یک محیط مبتنی بر ویندوز را بدون سنگینی و بار اضافه محیط گرافیکی (Desktop Experience) فراهم می‌کند. این سیستم با **حدود ۸۰۰ مگابایت تا ۱ گیگابایت رم** اجرا شده و تمام تنظیمات روتینگ آن با پاورشل و `netsh` قابل اسکریپت‌نویسی است.

---

### تنظیمات ماشین مجازی در VMware
* **نسخه:** Windows Server 2022 یا 2025 (انتخاب گزینه **Server Core Installation**).
* **سخت‌افزار:** ۱ یا ۲ هسته پردازنده | **۱۰۲۴ تا ۱۵۳۶ مگابایت رم** | دیسک ۲۰ گیگابایت Thin Provisioned.
* **کارت شبکه ۱:** Bridged به کارت وای‌فای فیزیکی (اینترنت خام).
* **کارت شبکه ۲:** Bridged به کارت `Microsoft KM-TEST Loopback Adapter` (ارتباط داخلی به ویندوز اصلی).

---

### مراحل گام‌به‌گام پیاده‌سازی

#### ۱. نصب رول روتینگ و فعال‌سازی فورواردینگ
ترمینال PowerShell سرور Core را باز کرده و اجرا کنید:
```powershell
# نصب رول روتینگ (RRAS)
Install-WindowsFeature -Name Routing -IncludeManagementTools

# فعال‌سازی فورواردینگ پکت‌ها در رجیستری
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' -Name 'IPEnableRouter' -Value 1

# استارت خودکار سرویس ریموت اکسس
Set-Service -Name RemoteAccess -StartupType Automatic
Start-Service -Name RemoteAccess
```

#### ۲. اجرای Xray با درایور Wintun
فایل‌های `xray.exe`، `wintun.dll` و `xray-windows-tun.json` را در یک پوشه (مثلاً `C:\xray`) قرار داده و اجرا کنید:
```powershell
cd C:\xray
.\xray.exe run -c xray-windows-tun.json
```
*(کارت شبکه مجازی `CloudflareMesh` ساخته می‌شود).*

#### ۳. راه‌اندازی NAT با ۳ دستور خط فرمانی Netsh
یک پنجره ترمینال دیگر باز کنید و دستورات زیر را بزنید:
```cmd
:: ۱. نصب ماژول NAT در پروتکل روتینگ
netsh routing ip nat install

:: ۲. معرفی کارت شبکه وارپ به عنوان اینترفیس عمومی اینترنت
netsh routing ip nat add interface name="CloudflareMesh" mode=full

:: ۳. معرفی کارت متصل به Loopback به عنوان اینترفیس محلی
netsh routing ip nat add interface name="Ethernet0" mode=private
```
*(به جای `Ethernet0` نام کارت شبکه متصل به Loopback را قرار دهید که با دستور `netsh interface show interface` قابل مشاهده است).*

#### ۴. تنظیم ویندوز اصلی لپ‌تاپ
* در ویندوز اصلی، وارد تنظیمات IPv4 کارت **Microsoft Loopback** شوید و **Default Gateway** را آی‌پی ویندوز سرور Core (مثلاً `10.10.10.2`) بگذارید.
* گیت‌وی کارت وای‌فای فیزیکی را **خالی** بگذارید.
