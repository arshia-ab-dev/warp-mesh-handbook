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
| Wi-Fi Hotspot breaks WARP tunnel | Windows ICS modifies Wintun bindings and alters routing metrics. | Use the VMware Gateway architecture or share via local SOCKS/HTTP proxy. |

---

### Platform-Specific Diagnostic Commands

#### Linux / Ubuntu Diagnostics
* **Live packet tracing across interfaces:**
  ```bash
  sudo tcpdump -i any host 10.10.10.1 -n
  ```
* **Verify NAT rules:**
  ```bash
  sudo iptables -t nat -L -n -v
  ```
* **Check kernel forwarding status:**
  ```bash
  sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
  ```

#### Windows Diagnostics
* **Test tunnel connection via curl:**
  ```powershell
  curl.exe -x socks5://127.0.0.1:2080 https://cloudflare.com/cdn-cgi/trace
  ```
  *(Look for `warp=on` and `gateway=on`).*
* **Inspect route table:**
  ```powershell
  route print -4
  ```

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
| قطع شدن وارپ هنگام ساخت هات‌اسپات | سرویس ICS ویندوز با بایند Wintun تداخل کرده و جدول روت را به هم می‌زند. | از معماری ماشین مجازی استفاده کنید یا ترافیک را از طریق پروکسی محلی به اشتراک بگذارید. |

---

### دستورات تشخیصی به تفکیک سیستم‌عامل

#### دستورات در لینوکس (اوبونتو)
* **ردگیری زنده بسته‌های عبوری از ویندوز با Tcpdump:**
  ```bash
  sudo tcpdump -i any host 10.10.10.1 -n
  ```
* **بررسی فعال بودن رول‌های NAT:**
  ```bash
  sudo iptables -t nat -L -n -v
  ```
* **بررسی وضعیت فورواردینگ هسته:**
  ```bash
  sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
  ```

#### دستورات در ویندوز
* **تست سلامت تونل با curl:**
  ```powershell
  curl.exe -x socks5://127.0.0.1:2080 https://cloudflare.com/cdn-cgi/trace
  ```
  *(شاخص‌های تایید: `warp=on` و `gateway=on`).*
* **مشاهده جدول روتینگ ویندوز:**
  ```powershell
  route print -4
  ```
