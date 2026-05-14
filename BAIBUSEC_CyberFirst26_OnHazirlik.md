# BAIBUSEC — CyberFirst'26
## Purple Team: Uçtan Uca Saldırı ve Savunma
### Ön Hazırlık ve Gereksinimler Rehberi

> **Bu dokümanı eğitimden en az 2-3 gün önce okuyun ve tüm adımları tamamlayın.**
> Eğitim günü kurulum yapılmayacaktır; hazır gelmeyen katılımcılar savunma tarafında yapılacak uygulamaları takip edemez.

---

## İçindekiler

1. [Donanım Gereksinimleri](#1-donanım-gereksinimleri)
2. [Yazılım Gereksinimleri ve İndirmeler](#2-yazılım-gereksinimleri-ve-i̇ndirmeler)
3. [Sanallaştırma Ortamı Kurulumu](#3-sanallaştırma-ortamı-kurulumu)
4. [Ağ Yapılandırması](#4-ağ-yapılandırması)
5. [Ubuntu Server Sanal Makine Hazırlığı](#5-ubuntu-server-sanal-makine-hazırlığı)
6. [Windows 10 Sanal Makine Hazırlığı](#6-windows-10-sanal-makine-hazırlığı)
7. [Docker ve Wazuh + Shuffle Kurulumu](#7-docker-ve-wazuh--shuffle-kurulumu)
8. [Webhook.site Hazırlığı](#8-webhooksite-hazırlığı)
9. [Hazırlık Kontrol Listesi](#9-hazırlık-kontrol-listesi)

---

## 1. Donanım Gereksinimleri

Eğitimde aynı anda çalışacak bileşenler şunlardır:

| Bileşen | Çalışacağı Ortam | Tahmini RAM |
|---|---|---|
| Wazuh Manager + Indexer + Dashboard | Ubuntu 22.04 VM | ~4 GB |
| Shuffle SOAR | Ubuntu 22.04 VM (aynı) | ~4 GB |
| Windows 10 (Kurban Makine) | Windows 10 VM | ~2 GB |
| Host OS (Windows/Linux/macOS) | Fiziksel bilgisayar | ~16 GB |

### Minimum Gereksinimler

| Bileşen | Minimum | Önerilen |
|---|---|---|
| **RAM** | 16 GB | 32 GB |
| **CPU** | 4 çekirdek / 8 iş parçacığı | 6+ çekirdek |
| **Disk** | 80 GB boş alan (SSD) | 120 GB boş alan (SSD) |
| **Sanallaştırma** | VT-x / AMD-V aktif (BIOS) | — |

> ⚠️ **Kritik Uyarı:** 16 GB RAM ile sistem çalışır ancak çok sıkışır. Mümkünse 32 GB kullanın. RAM'iniz 16 GB ise Wazuh'u `docker compose` ile kurarken Elasticsearch heap boyutunu düşürmeniz gerekebilir (bu rehberde açıklanmaktadır).

> ⚠️ **BIOS Kontrolü:** Bilgisayarınızın BIOS/UEFI ayarlarında **Intel Virtualization Technology (VT-x)** veya **AMD-V** seçeneğinin **aktif (Enabled)** olduğundan emin olun. Bu ayar kapalıysa sanal makine açılmaz.

---

## 2. Yazılım Gereksinimleri ve İndirmeler

Aşağıdaki dosyaları eğitimden önce indirip hazır bulundurun. İndirmeler toplamda ~10 GB yer kaplar.

### 2.1 Sanallaştırma Yazılımı (Birini seçin)

| Yazılım | Link | Notlar |
|---|---|---|
| **VMware Workstation Pro 17+** | [vmware.com](https://www.vmware.com/products/workstation-pro.html) | Önerilen; daha stabil |
| **VirtualBox 7.x** | [virtualbox.org](https://www.virtualbox.org/wiki/Downloads) | Ücretsiz alternatif |

> VMware Workstation artık kişisel kullanım için ücretsizdir (lisans gerekmez).

### 2.2 İşletim Sistemi ISO Dosyaları

| ISO | İndirme Linki | Boyut |
|---|---|---|
| **Ubuntu 22.04.4 LTS Server** | [ubuntu.com/download/server](https://ubuntu.com/download/server) | ~2 GB |
| **Windows 10 22H2 (64-bit)** | [microsoft.com/software-download/windows10](https://www.microsoft.com/software-download/windows10) | ~5.8 GB |

> Windows ISO indirmek için **Media Creation Tool**'u kullanın: İndirdiğiniz `.exe` dosyasını çalıştırın → "Create installation media for another PC" seçin → ISO dosyası olarak kaydedin.

### 2.3 Ek Araçlar (Host Bilgisayarınıza veya Telefonunuza)

| Araç | Link | Amaç |
|---|---|---|
| **Telegram** (mobil veya masaüstü) | telegram.org | SOAR bildirimleri için |
| **webhook.site** | Hesap gerekmez, tarayıcıda açılır | Exfiltration takibi |

---

## 3. Sanallaştırma Ortamı Kurulumu

### 3.1 Ubuntu 22.04 Server VM Oluşturma

**VMware Workstation:**
1. `File → New Virtual Machine → Typical`
2. ISO dosyasını seçin
3. Aşağıdaki kaynakları ayarlayın:

| Ayar | Değer |
|---|---|
| RAM | **8192 MB (8 GB veya 10 GB)** |
| CPU | **4 çekirdek** |
| Disk | **60 GB** (thin provisioned) |
| Network | **Şimdilik NAT** (sonra değiştireceğiz) |

4. VM oluşturulduktan sonra kurulumu başlatın:
   - Language: **English**
   - Keyboard: **Turkish** veya **English US**
   - Storage: **Use entire disk**
   - Hostname: `wazuh-server`
   - Username: `labuser`
   - Şifre: `CyberFirst2026!` (veya aklınızda kalacak bir şey)
   - **OpenSSH Server kurulumunu işaretleyin** ✓

5. Kurulum tamamlandıktan sonra yeniden başlatın.

**VirtualBox:**
1. `New → Expert Mode`
2. RAM: **8192 MB veya 10 GB**, Type: **Linux**, Version: **Ubuntu (64-bit)**
3. Disk: **60 GB, VDI, Dynamically allocated**
4. VM ayarlarından → Storage → ISO'yu ekleyin
5. Network → Adapter 1: **NAT** olarak bırakın

---

### 3.2 Windows 10 VM Oluşturma

| Ayar | Değer |
|---|---|
| RAM | **4096 MB (4 GB)** |
| CPU | **2 çekirdek** |
| Disk | **50 GB** |
| Network | **Şimdilik NAT** |

Windows 10 kurulumu sırasında:
- Ürün anahtarı sorulursa: **"I don't have a product key"** seçin
- Sürüm: **Windows 10 Pro** seçin
- Kullanıcı adı: `Victim`
- Şifre: `Password123` (eğitim ortamı, basit tutun)
- Gizlilik ayarları: hepsini kapatabilirsiniz

---

## 4. Ağ Yapılandırması

Sanal makinelerin birbirleriyle ve host bilgisayarla iletişim kurabilmesi için doğru ağ modunu seçmek kritiktir.

### 4.1 Kullanılacak Ağ Topolojisi

```
[Host Bilgisayar]
       |
  [VMnet (Host-Only veya NAT)]
       |
  ┌────┴────────────────┐
  │                     │
[Ubuntu VM]       [Windows 10 VM]
Wazuh + Shuffle    Kurban Makine
192.168.100.10    192.168.100.20
```

### 4.2 VMware Workstation ile Ağ Ayarı

1. `Edit → Virtual Network Editor` açın
2. **VMnet2** (veya kullanılmayan bir VMnet) seçin
3. **Host-only** modunu seçin
4. Subnet IP: `192.168.100.0`, Mask: `255.255.255.0`
5. DHCP'yi aktif bırakın

Her iki VM için de:
- VM ayarları → Network Adapter → **Custom: VMnet2** seçin

### 4.3 VirtualBox ile Ağ Ayarı

1. `File → Host Network Manager → Create`
2. IPv4 Address: `192.168.100.1`, Mask: `255.255.255.0`
3. DHCP Server → Enable → Lower: `192.168.100.10`, Upper: `192.168.100.50`

Her iki VM için de:
- Settings → Network → Adapter 1 → **Host-only Adapter** seçin

### 4.4 IP Adreslerini Kontrol Etme

Ubuntu VM içinde:
```bash
ip a
# 192.168.100.x görmelisiniz
```

Windows 10 VM içinde (CMD):
```cmd
ipconfig
# 192.168.100.x görmelisiniz
```

İki makine birbirini ping'leyebilmeli:
```bash
# Ubuntu'dan Windows'a:
ping 192.168.100.20

# Windows'tan Ubuntu'ya:
ping 192.168.100.10
```

> ⚠️ Windows 10 Firewall, ICMP'yi (ping) bloke edebilir. Yanıt gelmiyorsa bu normaldir; asıl önemli olan TCP bağlantısıdır.

---

## 5. Ubuntu Server Sanal Makine Hazırlığı

Ubuntu VM'e SSH veya doğrudan konsol üzerinden bağlanın.

### 5.1 Sistem Güncellemesi

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git net-tools
```

### 5.2 Docker Kurulumu

```bash
# Eski Docker sürümlerini kaldır
sudo apt remove -y docker docker-engine docker.io containerd runc

# Gerekli bağımlılıkları kur
sudo apt install -y ca-certificates curl gnupg lsb-release

# Docker GPG anahtarını ekle
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker deposunu ekle
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker'ı kur
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Kullanıcıyı docker grubuna ekle (sudo gerektirmeden çalıştırmak için)
sudo usermod -aG docker $USER
newgrp docker

# Test et
docker --version
docker compose version
```

### 5.3 vm.max_map_count Ayarı (Wazuh Indexer için Zorunlu)

Wazuh'un Elasticsearch/OpenSearch tabanlı indexer bileşeni bu ayar olmadan başlamaz:

```bash
# Kalıcı olarak ayarla
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Kontrol et
sysctl vm.max_map_count
# Çıktı: vm.max_map_count = 262144
```

### 5.4 Güvenlik Duvarı Ayarları

```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 443/tcp     # Wazuh Dashboard
sudo ufw allow 1514/tcp    # Wazuh Agent iletişimi
sudo ufw allow 1515/tcp    # Wazuh Agent kaydı
sudo ufw allow 55000/tcp   # Wazuh Manager API
sudo ufw allow 3001/tcp    # Shuffle SOAR UI
sudo ufw enable
sudo ufw status
```

---

## 6. Windows 10 Sanal Makine Hazırlığı

### 6.1 Windows Update'i Devre Dışı Bırakma (Lab ortamı için)

> Eğitim sırasında ani güncellemeler performansı olumsuz etkileyebilir.

1. `Win + R` → `services.msc`
2. **Windows Update** servisini bulun
3. Sağ tık → Properties → Startup type: **Disabled**
4. Service status: **Stop**

### 6.2 Windows Defender'ı Kısıtlama

Eğitim sırasında `spy.ps1` scripti Defender tarafından engellenebilir. Bunu önlemek için:

1. `Windows Security → Virus & threat protection → Manage settings`
2. **Real-time protection → Off** yapın

**VEYA** PowerShell exclusion ekleyin (daha güvenli yaklaşım):
```powershell
# Yönetici PowerShell ile çalıştırın
Set-MpPreference -ExclusionPath "C:\Users\Public"
Set-MpPreference -ExclusionExtension ".ps1"
```

### 6.3 PowerShell Execution Policy Ayarı

```powershell
# Yönetici PowerShell ile çalıştırın
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope LocalMachine -Force
Get-ExecutionPolicy
# Çıktı: Bypass
```

### 6.4 Uzak Masaüstü (RDP) Aktifleştirme (Opsiyonel ama kullanışlı)

1. `System Properties → Remote → Allow remote connections`
2. Firewall rule'unu otomatik eklemeyi kabul edin

---

## 7. Docker ve Wazuh + Shuffle Kurulumu

> Bu bölüm Ubuntu VM üzerinde yapılır.

### 7.1 Wazuh Docker Kurulumu

```bash
# Wazuh Docker deposunu klonla
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.2
cd wazuh-docker/single-node

# Sertifikaları oluştur
docker compose -f generate-indexer-certs.yml run --rm generator

# Wazuh'u başlat
docker compose up -d

# Servislerin ayağa kalkmasını bekle (~3-5 dakika)
docker compose ps

# Wazuh üzerinde bulunan logların hepsini silin ve temiz bir başlangıç yapın.
curl -u admin:SecretPassword -k -X DELETE "https://172.17.0.1:9200/wazuh-alerts-4.x-*"


> **16 GB RAM kullananlar için:** Wazuh Indexer heap boyutunu düşürmek gerekebilir. `docker-compose.yml` içindeki `wazuh.indexer` servisinde:
> ```yaml
> environment:
>   - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
> ```
> Varsayılan 1g yerine 512m kullanın.
```

**Wazuh Dashboard Erişimi:**
- URL: `https://192.168.100.10` (veya Ubuntu VM IP'si)
- Kullanıcı: `admin`
- Şifre: `SecretPassword` (ilk girişte değiştirin)

> ⚠️ Tarayıcı sertifika uyarısı verecektir — "Advanced → Accept risk" ile geçin (self-signed sertifika).

### 7.2 Shuffle SOAR Kurulumu

```bash
# Wazuh klasöründen çıkın
cd ~

# Shuffle deposunu klonla
git clone https://github.com/Shuffle/Shuffle
cd Shuffle

# docker-compose.yml dosyasını düzenleyin.
sudo nano docker-compose.yml

# Wazuh ile portların çakışmaması için opensearch: yazan kısmın altındaki ports: 9200:9200 bulun ve 9201:9200 olarak değiştirin. SHUFFLE_SWARM_CONFIG: değerini de SHUFFLE_SWARM_CONFIG:false olarak düzenleyin

> **16 GB RAM kullananlar için:** Wazuh Indexer heap boyutunu düşürmek gerekebilir. `docker-compose.yml` içindeki `wazuh.indexer` servisinde:
> ```yaml
> environment:
>   - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
> ```
> opensearch:
    image: opensearchproject/opensearch:3.2.0
    hostname: shuffle-opensearch
    container_name: shuffle-opensearch
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Bu satırın zaten doğru
      - bootstrap.memory_lock=true
      - DISABLE_PERFORMANCE_ANALYZER_AGENT_CLI=true
      - cluster.initial_master_nodes=shuffle-opensearch
      - cluster.routing.allocation.disk.threshold_enabled=false
      - cluster.name=shuffle-cluster
      - node.name=shuffle-opensearch
      - node.store.allow_mmap=false
      - discovery.seed_hosts=shuffle-opensearch
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=${SHUFFLE_OPENSEARCH_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - shuffle-database:/usr/share/opensearch/data:z
    ports:
      - 9201:9200
    networks:
      - shuffle
    restart: unless-stopped
    # BURADAN İTİBAREN EKLİYORUZ (Hard Limit):
    deploy:
      resources:
        limits:
          memory: 2G
> Varsayılan 1g yerine 512m kullanın ve maksimum kullanabileceği belleği 2G ye sabitleyin

# Shuffle'ı başlat
docker compose up -d

# Durumu kontrol et
docker compose ps
```

**Shuffle UI Erişimi:**
- URL: `http://192.168.100.10:3001`
- İlk girişte admin hesabı oluşturun:
  - Email: `admin@lab.local`
  - Şifre: `CyberFirst2026!`

### 7.3 Servislerin Çalışır Durumda Olduğunu Doğrulama

```bash
# Tüm container'ları listele
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Aşağıdaki container'ların `Up` durumunda olması beklenir:

| Container | Port |
|---|---|
| `single-node-wazuh.manager-1` | 1514, 1515, 55000 |
| `single-node-wazuh.indexer-1` | 9200 |
| `single-node-wazuh.dashboard-1` | 443 |
| `shuffle-frontend-1` | 3001 |
| `shuffle-backend-1` | 5001 |
| `shuffle-orborus-1` | — |
| `shuffle-opensearch-1` | 9200 |

---

## 8. Webhook.site Hazırlığı

1. [https://webhook.site](https://webhook.site) adresine gidin
2. Sayfa açılır açılmaz size özel bir URL atanır:
   - Örnek: `https://webhook.site/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
3. Bu URL'i not alın — eğitim sırasında `spy.ps1` scriptine gireceksiniz
4. Sayfayı açık tutun; gelen istekleri burada göreceksiniz

> Bu URL eğitim boyunca sizin "C2 sunucunuz" gibi davranacaktır.

---

## 9. Hazırlık Kontrol Listesi

Eğitime gelmeden önce aşağıdakilerin **tamamını** işaretleyin:

```
[ ] Ubuntu 22.04 Server VM kuruldu ve çalışıyor
[ ] Windows 10 VM kuruldu ve çalışıyor
[ ] İki VM aynı ağda birbirini görebiliyor
[ ] Ubuntu'da Docker ve Docker Compose kurulu
[ ] vm.max_map_count = 262144 ayarlandı
[ ] Wazuh tüm container'ları "Up" durumunda
[ ] Wazuh Dashboard'a https üzerinden giriş yapılabiliyor
[ ] Shuffle UI'a http:// üzerinden giriş yapılabiliyor
[ ] Windows 10'da PowerShell Execution Policy = Bypass
[ ] Windows 10'da Defender exclusion veya devre dışı
[ ] webhook.site URL'i not alındı
[ ] Telegram hesabı var ve giriş yapılmış durumda
```

---

> **Sorun mu yaşıyorsunuz?**
> Eğitim öncesinde BAIBUSEC Discord/WhatsApp kanalından yardım alabilirsiniz.
> Dokümanı hazırlayan: Janberk BEŞGÜL | CyberFirst'26
