<div align="center">

# 🖥️ Windows Server Desktop Experience (GUI) Gateway Guide
### Graphical Gateway Deployment via Server Manager & RRAS Console

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
This method utilizes Windows Server with Desktop Experience (GUI). It is designed for administrators who prefer graphical configuration consoles over terminal commands, requiring **2 to 3 GB of RAM** allocated in VMware.

---

### Virtual Machine Configuration in VMware
* **Hardware Specs:** 2 CPU Cores | **2048 to 3072 MB RAM** | 35 GB Thin Provisioned disk.
* **Network Adapter 1:** Bridged to physical Wi-Fi (Upstream raw internet).
* **Network Adapter 2:** Bridged to `Microsoft KM-TEST Loopback Adapter` (Internal host link).

---

### Step-by-Step Implementation

#### 1. Running Xray on Windows Server
1. Place `xray.exe`, `wintun.dll`, and `xray-windows-tun.json` inside the VM.
2. Run PowerShell as Administrator:
   ```powershell
   .\xray.exe run -c .\xray-windows-tun.json
   ```
   *(Interface `CloudflareMesh` will be established).*

#### 2. Installing the Routing Role (GUI)
1. Open **Server Manager** > click **Add roles and features**.
2. Select **Remote Access** > under Role Services, check **Routing** (DirectAccess and VPN will auto-select) > Install.

#### 3. Configuring NAT via RRAS Console
1. Open the **Routing and Remote Access** console (`rrasmgmt.msc`).
2. Right-click your server name > **Configure and Enable Routing and Remote Access**.
3. Choose **Custom configuration** > check **NAT** and **LAN routing** > Finish > Start service.
4. Expand **IPv4 > NAT** in the left panel:
   * Right-click **NAT** > **New Interface...** > select **`CloudflareMesh`**.
   * Choose *"Public interface connected to the Internet"* and check *"Enable NAT on this interface"* > OK.
   * Right-click **NAT** again > **New Interface...** > select the internal network adapter (connected to Loopback).
   * Choose *"Private interface connected to private network"* > OK.

#### 4. Configuring Windows Host Network
* On the host's **Microsoft Loopback Adapter**: Set Default Gateway to the Windows Server VM's IP (e.g., `10.10.10.2`).
* On the host's **Physical Wi-Fi Adapter**: Leave Default Gateway **BLANK**.

---

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### معرفی و کلیات
این روش برای کاربرانی مناسب است که مایلند تمام مراحل روتینگ را از طریق پنجره‌ها و کنسول‌های گرافیکی ویندوز (GUI) انجام دهند. این حالت نیازمند اختصاص **۲ تا ۳ گیگابایت رم** در VMware است.

---

### تنظیمات ماشین مجازی در VMware
* **سخت‌افزار:** ۲ هسته پردازنده | **۲ تا ۳ گیگابایت رم** | دیسک ۳۵ گیگابایتی Thin Provisioned.
* **کارت شبکه ۱:** Bridged به کارت وای‌فای فیزیکی (اینترنت خام).
* **کارت شبکه ۲:** Bridged به کارت `Microsoft KM-TEST Loopback Adapter` (لینک داخلی به هاست).

---

### مراحل گام‌به‌گام پیاده‌سازی

#### ۱. اجرای Xray در ویندوز سرور
1. فایل‌های `xray.exe`، `wintun.dll` و کانفیگ `xray-windows-tun.json` را داخل ماشین مجازی قرار دهید.
2. با دسترسی Administrator در PowerShell اجرا کنید تا کارت شبکه `CloudflareMesh` ساخته شود.

#### ۲. نصب رول روتینگ از طریق Server Manager
1. برنامه **Server Manager** را باز کرده و روی **Add roles and features** کلیک کنید.
2. رول **Remote Access** را انتخاب کنید و در بخش Role Services، تیک **Routing** را بزنید و نصب را تکمیل کنید.

#### ۳. فعال‌سازی NAT در کنسول RRAS
1. کنسول **Routing and Remote Access** را باز کنید (`rrasmgmt.msc`).
2. روی نام سرور کلیک‌راست کرده > **Configure and Enable Routing and Remote Access**.
3. گزینه **Custom configuration** را انتخاب کنید > تیک‌های **NAT** و **LAN routing** را بزنید و سرویس را استارت کنید.
4. در پنل سمت چپ، زیرشاخه **IPv4 > NAT** را باز کنید:
   * روی **NAT** کلیک‌راست کنید > **New Interface...** > کارت شبکه **`CloudflareMesh`** را انتخاب کنید.
   * گزینه *"Public interface connected to the Internet"* را انتخاب کرده و تیک *"Enable NAT on this interface"* را بزنید > OK.
   * مجدداً روی **NAT** کلیک‌راست کنید > **New Interface...** > کارت شبکه داخلی متصل به Loopback را انتخاب کنید.
   * گزینه *"Private interface connected to private network"* را انتخاب کنید > OK.

#### ۴. تنظیم ویندوز اصلی لپ‌تاپ
* روی کارت **Microsoft Loopback**: مقدار Default Gateway را آی‌پی ویندوز سرور (`10.10.10.2`) قرار دهید.
* روی کارت **Wi-Fi فیزیکی**: فیلد Default Gateway را **کاملاً خالی بگذارید**.
