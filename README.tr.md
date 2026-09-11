Bu dokümanı farklı dilde oku: [English](README.md)

# zapret2

**zapret2**, otonom, yüksek performanslı, çok platformlu bir Derin Paket İnceleme (DPI) atlatma aracı ve programlanabilir ağ paketi manipülatörüdür. Harici üçüncü taraf VPN veya proxy sunucularına ihtiyaç duymaksızın internet sansürlerini, hız kısıtlamalarını (throttling) ve trafik imza analizlerini aşmayı sağlar.

---

## Temel Özellikler

- **Otonom ve Sunucusuz**: Doğrudan yönlendiricinizde (router) veya yerel bilgisayarınızda çalışır; uzak proxy, VPN sunucusu veya ücretli abonelik gerektirmez.
- **Hibrit Yüksek Performanslı Mimari**:
  - **C Çekirdeği (`nfqws2` / `winws2`)**: İşletim sistemi seviyesinde paket yakalama (NFQUEUE, WinDivert, divert soketi), durum bilgili (stateful) bağlantı takibi, protokol ayrıştırma ve paket birleştirmeyi neredeyse sıfır ek yükle gerçekleştirir.
  - **Lua Desenkronizasyon Motoru (`zapret-lib.lua`, `zapret-antidpi.lua`)**: Ayrıştırılmış paket ağaçları (dissects) üzerinde esnek ve betiklenebilir DPI atlatma stratejilerini çalıştırır.
- **Geniş Kapsamlı Desenkronizasyon Stratejileri**:
  - TCP çoklu bölme (multisplit), çoklu sıra bozma (multidisorder), çakışan diziler (`seqovl`) ve bant dışı (`oob`) manipülasyon.
  - Özel TTL/autottl, hatalı sağlama toplamı (bad checksum), MD5 imzası ve TLS SNI/Kyber yük modifikasyonları içeren sahte (fake) paket enjeksiyonu.
  - TCP pencere boyutu ölçekleme (`wssize`), SYN verisi (`syndata`) ve zaman damgası yeniden sıralaması (`tcp_ts_up`).
  - QUIC / HTTP/3 ilk paket desenkronizasyonu ve sahte paket gönderimi.
  - Dinamik IP parçalama (`ipfrag`).
- **Programlanabilir Protokol Gizleme (Obfuscation)**: İsteğe bağlı protokolleri gizleme desteği (örneğin WireGuard UDP trafiğini ICMP yankı pingleri üzerinden tünelleme).
- **Çoklu Platform Desteği**:
  - **Linux**: `nftables` veya `iptables` (NFQUEUE) kullanan geleneksel dağıtımlar.
  - **OpenWrt**: Düşük kaynaklı gömülü yönlendiriciler için optimize edilmiştir.
  - **FreeBSD ve pfSense**: `ipfw` veya `pf` üzerinden çekirdek divert yönlendirmesi.
  - **OpenBSD**: `pf` üzerinden paket yakalama.
  - **Windows**: WinDivert çekirdek sürücüsü kullanan `winws2` ile yerel destek.

---

## zapret2'deki Yenilikler (zapret1 ile Karşılaştırma)

*zapret1* (`nfqws1`) sürümünde desenkronizasyon stratejileri doğrudan C koduna gömülmüştü; bu da yüzlerce katı komut satırı parametresine yol açıyor ve hızla değişen DPI filtreleme tekniklerine uyum sağlamayı zorlaştırıyordu.

**zapret2 bu paradigmayı değiştirir:**
1. **Lua Destekli Stratejiler**: Desenkronizasyon saldırıları Lua betikleriyle yazılır. Ağ protokollerine hakim olan herkes, C kaynak koduna dokunmadan veya yeniden derleme yapmadan stratejileri yazabilir, özelleştirebilir ve zincirleyebilir.
2. **Yük Tipi (Payload-Type) Farkındalığı**: Bağlantı protokolleri ile spesifik paket veri tiplerini (örn. `tls_client_hello`, `http_req`, `quic_initial`) birbirinden ayırarak filtreler.
3. **Otomatik TCP Segmentasyonu**: `zapret-lib.lua`, bağlantının MSS değerini takip eder ve izin verilen boyuttan büyük paketleri otomatik olarak parçalar (örn. kuantum sonrası Kyber el sıkışmalı TLS paketleri veya büyük `seqovl` tamponları).
4. **Esnek Filtre Aralıkları (Ranges)**: Paket sayısı, veri içeren paketler veya bayt ofsetlerine göre çalışan yönlü aralık denetimleri (`--in-range` ve `--out-range`) sayesinde Lua motorunun gereksiz çalışması engellenir.
5. **Evrensel Veri Blokları (Blobs)**: Sabit kodlanmış sahte yük parametreleri yerine hex dizgilerinden veya dosyalardan yüklenen esnek ikili bloklar (blob) kullanılır.

---

## Dizin Yapısı

```text
zapret2/
├── binaries/           # Önceden derlenmiş mimari ikilileri (varsa)
├── blockcheck2.sh      # Otomatik DPI atlatma stratejisi tespit ve test aracı
├── blockcheck2.d/      # blockcheck2 için strateji tanımları ve test hedefleri
├── common/             # Yardımcı betikler (işletim sistemi tespiti, güvenlik duvarı, diyaloglar)
├── config.default      # Sistem servisleri için varsayılan yapılandırma şablonu
├── docs/               # Kapsamlı teknik dokümantasyon ve derleme kılavuzları
│   ├── manual.en.md    # Tam zapret2 referans kılavuzu (İngilizce)
│   ├── manual.md       # Tam zapret2 referans kılavuzu (Rusça)
│   ├── readme.md       # Teknik mimari ve başlangıç rehberi (Rusça)
│   └── readme.tr.md    # Teknik mimari ve başlangıç rehberi (Türkçe)
├── files/fake/         # Sahte paketler için ikili yük şablonları (TLS, QUIC vb.)
├── init.d/             # Servis başlatma betikleri (systemd, OpenWrt, OpenRC, SysV vb.)
├── ipset/              # Otomatik engelli listesi indiricileri ve ipset/nftset üreticileri
├── lua/                # Çekirdek Lua kütüphaneleri (zapret-lib, zapret-antidpi, zapret-obfs)
├── nfq2/               # nfqws2, winws2 ve dvtws2 C kaynak kodları
├── install_easy.sh     # Linux / OpenWrt için etkileşimli kolay kurulum betiği
└── uninstall_easy.sh   # Temiz kaldırma betiği
```

---

## Hızlı Başlangıç

### 1. En İyi Stratejiyi Belirleme (`blockcheck2.sh`)
zapret'i arka plan servisi olarak çalıştırmadan önce, servis sağlayıcınızda (ISS) hangi atlatma stratejilerinin çalıştığını bulmak için Linux/BSD üzerinde `blockcheck2.sh` aracını veya Windows üzerinde hazır önayarları çalıştırın:
```bash
sudo ./blockcheck2.sh
```
Etkileşimli yönlendirmeleri izleyerek engellenen alan adlarına yönelik HTTP, HTTPS (TLS 1.2 & 1.3) ve QUIC testlerini gerçekleştirin.

### 2. Otomatik Kurulum (Linux / OpenWrt)
zapret2'yi sistem servisi olarak otomatik kurmak için:
```bash
sudo ./install_easy.sh
```
Betik şu işlemleri yürütür:
- İşletim sisteminizi, init sisteminizi ve güvenlik duvarı altyapınızı (`nftables` veya `iptables`) tespit eder.
- Gerekli ikilileri (`nfqws2`) doğrular veya derler.
- Güvenlik duvarı yönlendirme kurallarını yapılandırır ve servisleri etkinleştirir.

### 3. Windows Üzerinde Kullanım (`winws2`)
Windows'ta zapret2, WinDivert sürücüsü tarafından desteklenen `winws2.exe` ile çalıştırılır:
```cmd
winws2.exe --wf-tcp-out=80,443 ^
  --lua-init=@lua\zapret-lib.lua --lua-init=@lua\zapret-antidpi.lua ^
  --filter-tcp=80,443 --filter-l7=tls,http ^
  --payload=tls_client_hello --lua-desync=fake:blob=fake_default_tls:tcp_md5 ^
  --payload=tls_client_hello,http_req --lua-desync=multisplit:pos=1:seqovl=5
```

---

## Dokümantasyon

- [Kapsamlı Referans Kılavuzu (İngilizce)](docs/manual.en.md)
- [Teknik Mimari ve Başlangıç Rehberi (Türkçe)](docs/readme.tr.md)
- [Teknik Mimari ve Başlangıç Rehberi (Rusça)](docs/readme.md)
- [Derleme Kılavuzları](docs/compile/)

---

## Bağış ve Destek

Projeyi faydalı buluyorsanız ve geliştirmeyi desteklemek isterseniz kripto cüzdan adresleri:
- **USDT ERC20**: `0x3d52Ce15B7Be734c53fc9526ECbAB8267b63d66E`
- **USDT TRC20**: `TEzAAtn4VhndqEaAyuCM78xh5W2gCjwWEo`
- **BTC**: `bc1qhqew3mrvp47uk2vevt5sctp7p2x9m7m5kkchve`
- **ETH**: `0x3d52Ce15B7Be734c53fc9526ECbAB8267b63d66E`

---

## Lisans

Bu proje MIT Lisansı şartları altında lisanslanmış açık kaynaklı bir yazılımdır. Detaylar için [docs/LICENSE.txt](docs/LICENSE.txt) dosyasına bakabilirsiniz.
