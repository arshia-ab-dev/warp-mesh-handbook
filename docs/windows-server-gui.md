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
This method uses Windows Server with the Desktop Experience (GUI). It is ideal for administrators who prefer graphical configuration consoles over terminal commands, but it requires around **2 to 3 GB of RAM** inside VMware.

---

### Virtual Machine Configuration in VMware
* **Hardware Specs:** 2 CPU Cores | **2048 to 3072 MB RAM** | 35 GB Thin Provisioned disk.
* **Network Adapter 1:** Bridged to physical Wi-Fi (for upstream internet).
* **Network Adapter 2:** Bridged to `Microsoft KM-TEST Loopback Adapter` (internal link to Host).

---

### Step-by-Step Implementation

#### 1. Running Xray on Windows Server
1. Place `xray.exe`, `wintun.dll`, and `xray-windows-tun.json` inside the VM.
2. Run PowerShell as Administrator:
   ```powershell
   .\xray.exe run -c xray-windows-tun.json
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
2. با دسترسی Administrator اجرا کنید تا کارت شبکه `CloudflareMesh` ساخته شود.

#### ۲. نصب رول روتینگ از طریق Server Manager
1. برنامه **Server Manager** را باز کرده و روی **Add roles and features** کلیک کنید.
2. رول **Remote Access** را انتخاب کنید و در بخش Role Services، تیک **Routing** را بزنید و نصب را تکمیل کنید.

#### ۳. فعال‌سازی NAT در کنسول RRAS
1. کنسول **Routing and Remote Access** را باز کنید (`rrasmgmt.msc`).
2. روی نام سرور کلیک‌راست کرده > **Configure and Enable Routing and Remote Access**.
3. گزینه **Custom configuration** را انتخاب کنید > تیک‌های **NAT** و **LAN routing** را بزنید و استارت کنید.
4. در پنل سمت چپ، زیرشاخه **IPv4 > NAT** را باز کنید:
   * روی **NAT** کلیک‌راست کنید > **New Interface...** > کارت شبکه **`CloudflareMesh`** را انتخاب کنید.
   * گزینه *"Public interface connected to the Internet"* را انتخاب کرده و تیک *"Enable NAT on this interface"* را بزنید > OK.
   * مجدداً روی **NAT** کلیک‌راست کنید > **New Interface...** > کارت شبکه داخلی متصل به Loopback را انتخاب کنید.
   * گزینه *"Private interface connected to private network"* را انتخاب کنید > OK.

#### ۴. تنظیم ویندوز اصلی لپ‌تاپ
* روی کارت **Microsoft Loopback**: مقدار Default Gateway را آی‌پی ویندوز سرور (`10.10.10.2`) قرار دهید.
* روی کارت **Wi-Fi فیزیکی**: فیلد Default Gateway را **کاملاً خالی بگذارید**.
```

---

### ۴. فایل `docs/troubleshooting.md`

```markdown
<div align="center">

# 🛠️ Network Troubleshooting & Diagnostics Matrix
### Deep Packet Inspection Bypass, WireGuard Handshakes & Routing Solutions

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

### Diagnostics Matrix

| Error / Symptom | Technical Cause | Immediate Solution |
| :--- | :--- | :--- |
| `go: connect timeout` | Firewall DPI identifies and drops WireGuard's 148-byte `0x01` handshake. | Use tested clean IP and port (`162.159.193.5:500`) with 5-stage UDP Noise enabled. |
| `Connection was reset` (Initial setup) | Protocol mismatch: Cloudflare dashboard profile defaulted to MASQUE instead of WireGuard. | In Cloudflare Zero Trust, switch Device tunnel protocol to WireGuard and recreate the node. |
| `Connection was reset` (on `100.96.0.2`) | Targeted port has no local listening service. | Normal network behavior; start your application (e.g., Python on port 80 or RDP on 3389). |
| `CF_HAPPY_EYEBALLS_MITM_FAILURE` | Client detects a manual proxy intercepting TLS or a forced endpoint overriding pinning. | Turn off system proxies and reset CLI configuration: `warp-cli tunnel endpoint reset`. |
| Windows "No Internet" (Yellow icon) | NCSI fails to resolve probes over blocked raw UDP 53. | Use DoH inside the tunnel or allow transparent tunnel forwarding on port 53. |
| Linux VM drops forwarded traffic | Kernel dropped asymmetric routing or firewall `FORWARD` is blocked. | Run `sudo sysctl -w net.ipv4.conf.all.rp_filter=2` and `sudo iptables -P FORWARD ACCEPT`. |
| Wi-Fi Hotspot breaks WARP tunnel | Windows ICS binds to dynamic Wintun and mutates routing tables. | Use the VMware Gateway architecture or route through a local HTTP/SOCKS proxy. |

---

### Live Diagnostic Commands

#### 1. Live Packet Tracing (on Ubuntu)
To verify if traffic from the host enters Ubuntu and exits through `tun0`:
```bash
sudo tcpdump -i any host 10.10.10.1 -n
```

#### 2. Testing Tunnel Trace
```powershell
curl.exe -x socks5://127.0.0.1:2080 https://cloudflare.com/cdn-cgi/trace
```
*Expected confirmation indicators:* `warp=on` and `gateway=on`.

---

<hr/>

<a name="persian"></a>
## <img src="https://cdn.jsdelivr.net/gh/Arshia-Ab10/noto-emoji@main/third_party/region-flags/waved-svg/emoji_u1f1ee_1f1f7.svg" width="26" valign="middle" alt="Iran Flag" /> فارسی

### جدول جامع عیب‌یابی و حل مشکلات شبکه

| خطا / نشانه | دلیل فنی | راه‌حل فوری |
| :--- | :--- | :--- |
| `go: connect timeout` | بسته هندشیک وایرگارد (`0x01`) توسط فایروال DPI مسدود شده است. | از پورت باز (`162.159.193.5:500`) همراه با ۵ لایه نویز UDP تصادفی استفاده کنید. |
| `Connection was reset` (ابتدای کار) | پروتکل در پنل کلودفلر روی MASQUE مانده است و کلیدهای وایرگارد نامعتبرند. | در پنل Zero Trust پروتکل را روی WireGuard بگذارید و نود را مجدد بسازید. |
| `Connection was reset` (روی `100.96.0.2`) | ترافیک به پورتی فرستاده شده که برنامه‌ای روی آن منتظر نیست. | این نشانه سلامت شبکه است؛ برنامه‌ای مانند پایتون یا RDP را روی آن پورت اجرا کنید. |
| `CF_HAPPY_EYEBALLS_MITM_FAILURE` | کلاینت رسمی متوجه پروکسی دستی شده یا گواهی همخوانی ندارد. | پروکسی‌ها را خاموش کنید و دستور ریست را بزنید: `warp-cli tunnel endpoint reset`. |
| علامت زرد وای‌فای (No Internet) | پکت‌های بررسی وضعیت اینترنت ویندوز روی پورت ۵۳ بی‌پاسخ می‌مانند. | ترافیک DNS را از DoH هدایت کنید یا مانع عبور پکت‌های پورت ۵۳ نشوید. |
| عدم عبور بسته‌ها از اوبونتو | فایروال بسته را انداخته یا مکانیزم `rp_filter` فعال است. | دستورات `sudo sysctl -w net.ipv4.conf.all.rp_filter=2` و `sudo iptables -P FORWARD ACCEPT` را بزنید. |
| قطع شدن وارپ هنگام ساخت هات‌اسپات | سرویس ICS ویندوز به درایور Wintun دست‌درازی کرده و جدول روت را به هم می‌زند. | از معماری ماشین مجازی استفاده کنید یا ترافیک را از طریق پروکسی محلی به اشتراک بگذارید. |

---

### دستورات مانیتورینگ زنده برای عیب‌یابی

#### ۱. ردگیری زنده بسته‌ها در اوبونتو با Tcpdump
برای اینکه ببینید پکت‌های ویندوز وارد ماشین مجازی می‌شوند یا خیر:
```bash
sudo tcpdump -i any host 10.10.10.1 -n
```

#### ۲. تست ارتباط رسمی با کلودفلر
```powershell
curl.exe -x socks5://127.0.0.1:2080 https://cloudflare.com/cdn-cgi/trace
```
*شاخص‌های تایید اتصال موفق:* `warp=on` و `gateway=on`.
