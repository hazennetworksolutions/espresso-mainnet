# 🟤 Espresso Network Mainnet 1 — Validator Node Kurulum Rehberi (Türkçe)

**Espresso Network validator node'u çalıştırmak ve Validator Bootstrap Program'a başvurmak için eksiksiz rehber**  
_Docker kurulumu, key üretimi, validator kaydı, systemd servis kurulumu — adım adım._

![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat&logo=ubuntu&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-29.x-2496ED?style=flat&logo=docker&logoColor=white)
![Espresso](https://img.shields.io/badge/Espresso-Mainnet%201-6B4226?style=flat)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

[hazennetworksolutions.com](https://www.hazennetworksolutions.com)

---

> **Yazar:** HazenNetworkSolutions  
> **Ağ:** Espresso Mainnet 1  
> **Image Tag:** 20260512  
> **Son Güncelleme:** 19 Mayıs 2026

---

## İçindekiler

- [Donanım Gereksinimleri](#donanım-gereksinimleri)
- [Adım 1 — Docker Kurulumu](#adım-1--docker-kurulumu)
- [Adım 2 — Dizin Yapısı Oluşturma](#adım-2--dizin-yapısı-oluşturma)
- [Adım 3 — Docker Image'larını İndirme](#adım-3--docker-imagelarını-i̇ndirme)
- [Adım 4 — Validator Key'lerini Üretme](#adım-4--validator-keylerini-üretme)
- [Adım 5 — Metadata JSON Dosyası Oluşturma](#adım-5--metadata-json-dosyası-oluşturma)
- [Adım 6 — Ethereum Cüzdanı Hazırlama](#adım-6--ethereum-cüzdanı-hazırlama)
- [Adım 7 — Ethereum RPC Endpoint Alma](#adım-7--ethereum-rpc-endpoint-alma)
- [Adım 8 — Validator'ı Ethereum'a Kayıt Etme](#adım-8--validatörü-ethereuma-kayıt-etme)
- [Adım 9 — Environment Dosyası Oluşturma](#adım-9--environment-dosyası-oluşturma)
- [Adım 10 — Systemd Servis Dosyası Oluşturma](#adım-10--systemd-servis-dosyası-oluşturma)
- [Adım 11 — Node'u Başlatma](#adım-11--nodeu-başlatma)
- [Adım 12 — Node Sağlığını Doğrulama](#adım-12--node-sağlığını-doğrulama)
- [Adım 13 — Bootstrap Program'a Başvurma](#adım-13--bootstrap-programa-başvurma)
- [Node İzleme](#node-i̇zleme)
- [Node Güncelleme](#node-güncelleme)

---

## Donanım Gereksinimleri

| Bileşen | Minimum |
| ------- | ------- |
| İşletim Sistemi | Ubuntu 22.04+ |
| CPU | 1 Core (herhangi bir modern işlemci) |
| RAM | 8 GB |
| Depolama | Yok denecek kadar az (kilobyte seviyesi) |
| Ağ | Stabil internet, public IP zorunlu |
| Açık Portlar | 8585/TCP (API), 9000/UDP (LibP2P) |

> ✅ Bu rehber **Regular Node** (DA olmayan) kurulumunu anlatmaktadır. DA node'ları yalnızca DA komitesindekiler için gereklidir — çoğu operatör buna ihtiyaç duymaz.

---

## Adım 1 — Docker Kurulumu

Docker'ı resmi repository'den kurun (`snap` veya `apt install docker.io` kullanmayın):

```bash
apt-get update && apt-get install -y ca-certificates curl gnupg && \
install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
chmod a+r /etc/apt/keyrings/docker.gpg && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
apt-get update && \
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Kurulumu doğrulayın:

```bash
docker --version && systemctl status docker
```

Beklenen çıktı: `Docker version 29.x.x` ve `active (running)`.

---

## Adım 2 — Dizin Yapısı Oluşturma

```bash
mkdir -p /opt/espresso/keys /opt/espresso/store
```

| Dizin | Amacı |
| ----- | ----- |
| `/opt/espresso/keys` | BLS ve Schnorr private key'lerini saklar |
| `/opt/espresso/store` | Node'un SQLite veritabanını saklar |

---

## Adım 3 — Docker Image'larını İndirme

Espresso sequencer image'ını indirin (Mainnet 1):

```bash
docker pull ghcr.io/espressosystems/espresso-sequencer/sequencer:20260512
```

Staking CLI image'ını indirin:

```bash
docker pull ghcr.io/espressosystems/espresso-sequencer/staking-cli:main
```

> 💡 Sequencer image'ı ~500MB'dır, indirilmesi birkaç dakika sürebilir.
>
> ⚠️ **Not:** Resmi Espresso dökümanlarında image path'i `espresso-network/sequencer` olarak gösteriliyor, ancak doğru registry path'i yukarıda kullanılan `espresso-sequencer/sequencer`'dır. Bu rehberdeki komutları kullanın.

---

## Adım 4 — Validator Key'lerini Üretme

BLS (staking) ve Schnorr (state) key'lerinizi oluşturmak için key generator'ı çalıştırın:

```bash
docker run -v /opt/espresso/keys:/keys \
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260512 \
  keygen -o /keys
```

Komut terminal çıktısı vermez — bu normaldir, key'ler sessizce dosyaya kaydedilir. Dosyanın oluşturulduğunu doğrulayın:

```bash
cat /opt/espresso/keys/0.env
```

Beklenen çıktı:

```
# Seed: <rastgele_hex>
ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY=BLS_VER_KEY~...
ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY=BLS_SIGNING_KEY~...
ESPRESSO_SEQUENCER_PUBLIC_STATE_KEY=SCHNORR_VER_KEY~...
ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY=SCHNORR_SIGNING_KEY~...
```

Ardından **x25519 key** oluşturun (yaklaşan Cliquenet protokol yükseltmesi için gerekli):

```bash
docker run --rm \
  -v /opt/espresso/keys:/keys \
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260512 \
  keygen --scheme x25519 -o /keys
```

Bu komut `0.env` dosyasını yalnızca x25519 key ile overwrite eder. Tüm key'leri birleştirerek dosyayı manuel olarak yeniden oluşturmanız gerekir:

```bash
cat > /opt/espresso/keys/0.env << 'EOF'
# Seed: BURAYA_ORIJINAL_SEED_YAZIN
ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY=BLS_VER_KEY~BLS_PUBLIC_KEY
ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY=BLS_SIGNING_KEY~BLS_PRIVATE_KEY
ESPRESSO_SEQUENCER_PUBLIC_STATE_KEY=SCHNORR_VER_KEY~SCHNORR_PUBLIC_KEY
ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY=SCHNORR_SIGNING_KEY~SCHNORR_PRIVATE_KEY
ESPRESSO_SEQUENCER_PUBLIC_X25519_KEY=X25519_PUBLIC_KEY
ESPRESSO_SEQUENCER_PRIVATE_X25519_KEY=X25519_SK~X25519_PRIVATE_KEY
EOF
```

3 public key'in mevcut olduğunu doğrulayın:

```bash
grep "PUBLIC" /opt/espresso/keys/0.env
```

3 satır görmelisiniz: `PUBLIC_STAKING_KEY`, `PUBLIC_STATE_KEY`, `PUBLIC_X25519_KEY`.

> 🔐 **KRİTİK: Aşağıdaki bilgileri hemen güvenli bir yere yedekleyin (şifre yöneticisi, harici disk):**
>
> | Key | Ön Ek | Açıklama |
> | --- | ----- | -------- |
> | `ESPRESSO_SEQUENCER_PRIVATE_STAKING_KEY` | `BLS_SIGNING_KEY~...` | Consensus mesajlarını imzalar |
> | `ESPRESSO_SEQUENCER_PRIVATE_STATE_KEY` | `SCHNORR_SIGNING_KEY~...` | Finalize edilmiş state'leri imzalar |
> | `ESPRESSO_SEQUENCER_PRIVATE_X25519_KEY` | `X25519_SK~...` | Cliquenet protokol yükseltmesi için gerekli |
> | `Seed` | hex string | Key'leri yeniden üretmek için kullanılır |
>
> Yerel bilgisayarınıza indirin:
> ```bash
> scp root@SUNUCU_IP:/opt/espresso/keys/0.env ./espresso-keys-yedek.env
> ```
>
> ⚠️ **Private key'lerinizi veya seed'inizi asla kimseyle paylaşmayın.**  
> ⚠️ Her BLS key yalnızca bir kez Ethereum mainnet'e kayıt edilebilir.

---

## Adım 5 — Metadata JSON Dosyası Oluşturma

Validator metadata'nız herkese açık olarak yayınlanır ve Espresso staking dashboard'unda görüntülenir. Sunucunuzda nginx kurulu ve çalışıyor olmalıdır.

Nginx kurulu değilse önce kurun:

```bash
apt-get install -y nginx && systemctl enable nginx && systemctl start nginx
```

Sunucunuzun public IPv4 adresini öğrenin:

```bash
curl -s -4 ifconfig.me
```

Metadata dosyasını oluşturun (değerleri kendi bilgilerinizle değiştirin):

```bash
mkdir -p /var/www/html/espresso && cat > /var/www/html/espresso/metadata.json << 'EOF'
{
  "pub_key": "BLS_VER_KEY~BURAYA_PUBLIC_STAKING_KEY_YAZIN",
  "name": "ValidatorAdiniz",
  "description": "Açıklamanız",
  "company_name": "Şirket Adınız",
  "company_website": "https://websiteniz.com"
}
EOF
```

`BLS_VER_KEY~BURAYA_PUBLIC_STAKING_KEY_YAZIN` yerine Adım 4'teki `ESPRESSO_SEQUENCER_PUBLIC_STAKING_KEY` değerini yazın.

Erişilebilir olduğunu test edin:

```bash
curl -s http://SUNUCU_IP/espresso/metadata.json
```

Beklenen çıktı: Az önce oluşturduğunuz JSON içeriği.

> 💡 `pub_key` alanı kayıtlı BLS key'inizle eşleşmelidir — bu, dashboard'da taklit edilmenizi engeller.

---

## Adım 6 — Ethereum Cüzdanı Hazırlama

Bu validator için **özel bir Ethereum cüzdanı** oluşturmanız gerekir. Mevcut cüzdanlarınızı kullanmayın.

1. MetaMask'ta (veya başka bir Ethereum cüzdanında) yeni bir cüzdan oluşturun
2. **12 kelimelik seed phrase'i** güvenli bir yere kaydedin
3. **Ethereum adresini** not edin (`0x` ile başlar)
4. Bu adrese **0.01–0.02 ETH** (Ethereum Mainnet) gönderin (gas ücreti için)

> ⚠️ Kayıt işlemi yaklaşık 300.000 gas harcar (~güncel fiyatlarla $1–2).  
> ⚠️ Her Ethereum adresi yalnızca **bir** validator kaydedebilir.  
> ⚠️ ETH'yi yalnızca Ethereum Mainnet üzerinden gönderin (BSC, Polygon vb. zincirlerde değil).

---

## Adım 7 — Ethereum RPC Endpoint Alma

Validator'ın L1'i okuyabilmesi için bir Ethereum Mainnet RPC endpoint'ine ihtiyacınız var.

**Tavsiye edilen: Infura (ücretsiz plan — günde 3M kredi, fazlasıyla yeterli)**

1. [infura.io](https://infura.io) adresine gidin ve ücretsiz hesap oluşturun
2. Yeni proje oluşturun → **Ethereum** seçin
3. API key'inizi kopyalayın

Endpoint'leriniz şu şekilde olacak:
- **HTTP:** `https://mainnet.infura.io/v3/API_KEY`
- **WebSocket:** `wss://mainnet.infura.io/ws/v3/API_KEY`

> 💡 Normal bir Espresso node'u günde yaklaşık 15.000–20.000 API isteği yapar — ücretsiz plan limitinin çok altında.

---

## Adım 8 — Validator'ü Ethereum'a Kayıt Etme

Bu adım, validator'ünüzü Ethereum mainnet'teki Espresso stake table contract'ına kayıt eder. Bu işlemi yalnızca **bir kez** yapmanız gerekir.

Kayıt komutunu çalıştırın (tüm yer tutucu değerleri kendinizinkilerle değiştirin):

```bash
docker run --rm \
  -e MNEMONIC="on iki kelimelik seed phrase buraya yazilir" \
  -e ACCOUNT_INDEX=0 \
  -e L1_PROVIDER="https://mainnet.infura.io/v3/API_KEY" \
  -e STAKE_TABLE_ADDRESS="0xCeF474D372B5b09dEfe2aF187bf17338Dc704451" \
  -e CONSENSUS_PRIVATE_KEY="BLS_SIGNING_KEY~PRIVATE_STAKING_KEY" \
  -e STATE_PRIVATE_KEY="SCHNORR_SIGNING_KEY~PRIVATE_STATE_KEY" \
  ghcr.io/espressosystems/espresso-sequencer/staking-cli:main \
  staking-cli register-validator \
  --commission 10.00 \
  --metadata-uri "http://SUNUCU_IP/espresso/metadata.json"
```

| Parametre | Açıklama |
| --------- | -------- |
| `MNEMONIC` | 12 kelimelik seed phrase'iniz (tırnak içinde, boşlukla ayrılmış) |
| `ACCOUNT_INDEX` | Türetme indeksi, ilk hesap için `0` kullanın |
| `L1_PROVIDER` | Infura HTTP RPC URL'niz |
| `CONSENSUS_PRIVATE_KEY` | Adım 4'teki `BLS_SIGNING_KEY~...` değeri |
| `STATE_PRIVATE_KEY` | Adım 4'teki `SCHNORR_SIGNING_KEY~...` değeri |
| `--commission` | Komisyon oranınız (0.00–100.00) |
| `--metadata-uri` | Adım 5'teki metadata.json URL'niz |

Başarılı çıktı:

```
Success! transaction hash: 0x...
event: ValidatorRegisteredV2 { account: 0xADRESINIZ, blsVK: BLS_VER_KEY~..., ... }
```

**Komutu çalıştırdıktan sonra terminal geçmişini hemen temizleyin:**

```bash
history -c && history -w
```

> ⚠️ Çıktıdaki **Ethereum hesap adresinizi** not edin — Bootstrap Program formunda buna ihtiyacınız olacak.

---

## Adım 9 — Environment Dosyası Oluşturma

Node için environment konfigürasyon dosyasını oluşturun:

```bash
cat > /opt/espresso/espresso.env << 'EOF'
ESPRESSO_SEQUENCER_CDN_ENDPOINT=cdn.main.net.espresso.network:1737
ESPRESSO_STATE_RELAY_SERVER_URL=https://state-relay.main.net.espresso.network
ESPRESSO_SEQUENCER_GENESIS_FILE=/genesis/mainnet.toml
ESPRESSO_SEQUENCER_EMBEDDED_DB=true
RUST_LOG=warn,libp2p=off
RUST_LOG_FORMAT=json
ESPRESSO_SEQUENCER_STATE_PEERS=https://query.main.net.espresso.network
ESPRESSO_SEQUENCER_CONFIG_PEERS=https://cache.main.net.espresso.network
ESPRESSO_SEQUENCER_L1_PROVIDER=https://mainnet.infura.io/v3/API_KEY
ESPRESSO_SEQUENCER_L1_WS_PROVIDER=wss://mainnet.infura.io/ws/v3/API_KEY
ESPRESSO_SEQUENCER_API_PORT=8585
ESPRESSO_SEQUENCER_STORAGE_PATH=/store
ESPRESSO_SEQUENCER_KEY_FILE=/keys/0.env
ESPRESSO_SEQUENCER_LIBP2P_BIND_ADDRESS=0.0.0.0:9000
ESPRESSO_SEQUENCER_LIBP2P_ADVERTISE_ADDRESS=SUNUCU_IP:9000
EOF
```

Değiştirin:
- `API_KEY` → Infura API key'iniz (hem HTTP hem WS satırlarında)
- `SUNUCU_IP` → Sunucunuzun public IPv4 adresi

Dosyayı doğrulayın:

```bash
cat /opt/espresso/espresso.env
```

---

## Adım 10 — Systemd Servis Dosyası Oluşturma

Docker container'ını yöneten systemd servis dosyasını oluşturun:

```bash
cat > /etc/systemd/system/espresso.service << 'EOF'
[Unit]
Description=Espresso Sequencer Node
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
RestartSec=10
ExecStartPre=-/usr/bin/docker stop espresso
ExecStartPre=-/usr/bin/docker rm espresso
ExecStart=/usr/bin/docker run --name espresso \
  --env-file /opt/espresso/espresso.env \
  -v /opt/espresso/keys:/keys \
  -v /opt/espresso/store:/store \
  -p 8585:8585 \
  -p 9000:9000/udp \
  ghcr.io/espressosystems/espresso-sequencer/sequencer:20260512 \
  sequencer -- http -- catchup -- status
ExecStop=/usr/bin/docker stop espresso

[Install]
WantedBy=multi-user.target
EOF
```

---

## Adım 11 — Node'u Başlatma

Servisi etkinleştirin ve başlatın:

```bash
systemctl daemon-reload && \
systemctl enable espresso && \
systemctl start espresso
```

Servis durumunu kontrol edin:

```bash
systemctl status espresso
```

Beklenen çıktı: `active (running)`.

Canlı logları görüntüleyin:

```bash
journalctl -u espresso -n 50 --no-pager
```

> 💡 İlk başlatmada node, peer'lardan network config'ini yükler. Bu 10–30 saniye sürebilir. Logda `loaded config` ve her saniye artan view numaraları göreceksiniz — bu normal ve beklenen bir davranıştır.

---

## Adım 12 — Node Sağlığını Doğrulama

**Healthcheck (tüm modüller 200 dönmeli):**

```bash
curl -s http://localhost:8585/healthcheck
```

Beklenen çıktı:

```json
{"status":"available","modules":{"catchup":{"0":200,"1":200},"state-signature":{"0":200,"1":200},"status":{"0":200,"1":200}}}
```

**Validator'ınızın dashboard'da göründüğünü doğrulayın:**

1. [stake.espresso.network](https://stake.espresso.network) adresine gidin
2. Ethereum adresinizi arayın: `0xADRESINIZ`
3. Validator'ünüz `INACTIVE` durumunda görünmeli (bu doğrudur — delegation aldıktan sonra `ACTIVE` olur)

> ✅ **Normal olan durumlar:**
> - `WARN` seviyesinde loglar — beklenen davranış
> - `Vote sending timed out` — node katılmaya çalışıyor ancak henüz delegation yok
> - `LCV3 signature posted by nodes not on the stake table` — delegation gelene kadar beklenen
> - Her ~1 saniyede artan view numaraları — node'un zinciri takip ettiğini gösterir
>
> ❌ **Normal olmayan durumlar:**
> - Servisin `failed` veya `inactive (dead)` göstermesi
> - Healthcheck'in 200 dönmemesi
> - View numaralarının 5 dakikadan fazla sabit kalması

---

## Adım 13 — Bootstrap Program'a Başvurma

Espresso Foundation Validator Bootstrap Program, seçilen 30'a kadar validator'a başlangıç olarak **1.000.000 ESP** delegation sunmaktadır.

Başvuru formu: [Bootstrap Program Başvuru Formu](https://docs.google.com/forms/d/e/1FAIpQLScC5jk-jGE4d4hFRSuQNoP9MrmRB6oy3uRDagS-K5xc-OH64A/viewform)

| Alan | Değer |
| ---- | ----- |
| Operator name | Operatör/şirket adınız |
| Operator website | Website URL'niz |
| Primary contact name | Adınız |
| Primary contact email | E-posta adresiniz (opsiyonel) |
| Telegram / Discord handle | Handle'ınız |
| **Ethereum account** | Adım 8'de kullanılan adres (örn. `0xCaA51...`) |
| Region | Sunucunuzun bölgesi (örn. EU - Finlandiya) |
| Validator experience | Altyapı deneyiminizi açıklayın |

> 💡 Foundation, operatörleri uptime, kaçırılan slotlar, yanıt süresi ve yazılım güncelleme hızına göre değerlendirir. Her 6 ayda bir, en iyi performans gösterenler **ana delegasyon programına** geçme hakkı kazanır (daha büyük delegation).

---

## Yedekleme ve Kurtarma

### Yedeklenmesi Gerekenler

| Dosya / Bilgi | Kritik | Notlar |
|---|---|---|
| `/opt/espresso/keys/0.env` | 🔴 EVET | 3 private key'i içerir — **kaybolursa kurtarılamaz** |
| Ethereum wallet mnemonic | 🔴 EVET | Tüm L1 işlemleri için gerekli (ödüller, komisyon, kayıt silme) |
| `/opt/espresso/espresso.env` | 🟡 Önerilir | Infura key + sunucu IP — kolayca yeniden oluşturulabilir ama yedeklemek iyi |

### Key Yedekleme

Herhangi bir `keygen` komutu çalıştırmadan önce mutlaka yedek alın:

```bash
cp /opt/espresso/keys/0.env /opt/espresso/keys/0.env.backup
```

`0.env` dosyasını yerel bilgisayarınıza indirin:

```bash
scp root@SUNUCU_IP:/opt/espresso/keys/0.env ~/espresso-keys-yedek.env
```

İçeriği bir şifre yöneticisine kaydedin (Bitwarden, 1Password vb.).

> ⚠️ **Uyarı:** `keygen --scheme x25519` komutu `0.env` dosyasını **tamamen overwrite eder**. Herhangi bir keygen komutu çalıştırmadan önce mutlaka yedek alın.

### Kurtarma: Yeni Sunucuya Geçiş

Sunucuya erişiminizi kaybetseniz bile `0.env` yedeğiniz varsa:

1. Yeni sunucuya bu rehberin 1. adımından itibaren kurulumu yapın
2. Key dosyasını geri yükleyin:
   ```bash
   scp ~/espresso-keys-yedek.env root@YENI_SUNUCU_IP:/opt/espresso/keys/0.env
   ```
3. `espresso.env` dosyasındaki `ESPRESSO_SEQUENCER_LIBP2P_ADVERTISE_ADDRESS` değerini yeni sunucu IP'siyle güncelleyin
4. Yeni sunucu IP'sini `metadata.json` dosyasına da güncelleyin (IP tabanlı URL kullanıyorsanız)
5. Node'u başlatın — **Ethereum'a yeniden kayıt gerekmez**, validator kimliğiniz korunur

---

## Node İzleme

**Servis durumunu kontrol etme:**

```bash
systemctl status espresso
```

**Son 50 log satırını görüntüleme:**

```bash
journalctl -u espresso -n 50 --no-pager
```

**Healthcheck:**

```bash
curl -s http://localhost:8585/healthcheck
```

**Bağlı peer sayısını kontrol etme:**

```bash
curl -s http://localhost:8585/status/metrics | grep libp2p_num_connected_peers
```

**Node'u yeniden başlatma:**

```bash
systemctl restart espresso
```

**Node'u durdurma:**

```bash
systemctl stop espresso
```

---

## Node Güncelleme

Yeni bir image versiyonu yayınlandığında:

1. Yeni image'ı indirin:

```bash
docker pull ghcr.io/espressosystems/espresso-sequencer/sequencer:YENI_TAG
```

2. Systemd servis dosyasındaki tag'i güncelleyin (`ESKI_TAG` yerine o an çalışan tag'i yazın):

```bash
sed -i 's/sequencer:ESKI_TAG/sequencer:YENI_TAG/g' /etc/systemd/system/espresso.service
```

3. Yeniden yükleyin ve başlatın:

```bash
systemctl daemon-reload && systemctl restart espresso
```

4. Node'un çalıştığını doğrulayın:

```bash
systemctl status espresso && curl -s http://localhost:8585/healthcheck
```

> 💡 Güncellemelerden haberdar olmak için [Espresso Network GitHub Releases](https://github.com/EspressoSystems/espresso-network/releases) sayfasını takip edin.

---

## Yazar Hakkında

Bu rehber **HazenNetworkSolutions** tarafından hazırlanmıştır.  
🌐 [hazennetworksolutions.com](https://www.hazennetworksolutions.com)
