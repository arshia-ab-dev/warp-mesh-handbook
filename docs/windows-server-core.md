# ⚡ راهنمای راه‌اندازی گیت‌وی در Windows Server Core (خط فرمان)

این روش برای کسانی است که می‌خواهند از محیط ویندوز استفاده کنند اما رم و پردازنده لپ‌تاپ برای اجرای محیط گرافیکی هدر نرود.

---

### ۱. مشخصات ماشین مجازی در VMware
* **نسخه:** Windows Server 2022 یا 2025 (گزینه **Server Core Installation**).
* **سخت‌افزار:** ۱ یا ۲ هسته پردازنده | **۱۰۲۴ تا ۱۵۳۶ مگابایت رم** | دیسک ۲۰ گیگابایت Thin Provisioned.
* **کارت شبکه ۱:** Bridged به Wi-Fi فیزیکی (اینترنت خام).
* **کارت شبکه ۲:** Bridged به `Microsoft KM-TEST Loopback Adapter` (لینک داخلی به هاست).

---

### ۲. نصب سرویس روتینگ و اجرای Xray در Server Core

ترمینال PowerShell را در سرور Core باز کرده و دستورات زیر را اجرا کنید:

#### مرحله الف: نصب رول روتینگ (RRAS)
```powershell
Install-WindowsFeature -Name Routing -IncludeManagementTools
```

#### مرحله ب: فعال‌سازی فورواردینگ در هسته ویندوز سرور
```powershell
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' -Name 'IPEnableRouter' -Value 1
```

#### مرحله ج: اجرای Xray با Wintun
فایل‌های `xray.exe`، `wintun.dll` و `xray-windows-tun.json` را در یک پوشه قرار داده و استارت کنید:
```powershell
.\xray.exe run -c xray-windows-tun.json
```
*(کارت شبکه مجازی `CloudflareMesh` ساخته می‌شود).*

---

### ۳. راه‌اندازی NAT با ۳ دستور ساده Netsh (بدون نیاز به GUI)

یک ترمینال جدید باز کنید و این ۳ دستور را بزنید تا ترافیک کارت شبکه داخلی به سمت تونل ماسک شود:

```cmd
:: ۱. نصب ماژول NAT در پروتکل روتینگ
netsh routing ip nat install

:: ۲. معرفی کارت شبکه وارپ به عنوان اینترفیس عمومی اینترنت
netsh routing ip nat add interface name="CloudflareMesh" mode=full

:: ۳. معرفی کارت متصل به Loopback به عنوان اینترفیس محلی
netsh routing ip nat add interface name="Ethernet0" mode=private
```
*(به جای `Ethernet0` نام کارت شبکه متصل به Loopback در ویندوز سرور را قرار دهید که با دستور `netsh interface show interface` قابل مشاهده است).*

---

### ۴. تنظیم ویندوز اصلی لپ‌تاپ
* در ویندوز اصلی، وارد تنظیمات IPv4 کارت **Microsoft Loopback** شوید و **Default Gateway** را آی‌پی ویندوز سرور Core (مثلاً `10.10.10.2`) قرار دهید.
* گیت‌وی کارت وای‌فای فیزیکی را **خالی** بگذارید.
* اینترنت کل ویندوز لپ‌تاپ اکنون از دل ویندوز سرور Core عبور کرده و ضد فیلتر است!
