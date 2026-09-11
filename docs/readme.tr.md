Bu dokümanı farklı dilde oku: [English](../README.md) | [Русский](readme.md)

## İlgili Kılavuzlar

- [Tam Kılavuz (İngilizce)](manual.en.md)
- [Tam Kılavuz (Rusça)](manual.md)
- [Ana Proje Sayfası (Türkçe)](../README.tr.md)

---

## Bu Projeye Neden İhtiyaç Var?

Herhangi bir üçüncü taraf sunucuya bağlanmayı gerektirmeyen, tamamen yerel çalışan bağımsız bir DPI (Derin Paket İnceleme) atlatma aracıdır. HTTP(S) sitelerinin engellenmesini veya yavaşlatılmasını (throttling), VPN gibi servisleri engellemeye yönelik TCP ve UDP protokollerinin imza analizlerini aşmaya yardımcı olur. Ayrıca protokollerin kısmi şeffaf biçimde gizlenmesi (obfuscation) amacıyla da kullanılabilir.

Proje öncelikli olarak OpenWrt çalıştıran düşük donanımlı gömülü cihazları (yönlendiriciler / router) hedefler. Geleneksel Linux sistemleri, FreeBSD, OpenBSD ve Windows desteklenmektedir. MacOS teknik nedenlerden ötürü desteklenmemektedir ve desteklenmesi planlanmamaktadır. Bazı özel durumlarda çözümü farklı firmware sistemlerine manuel olarak entegre etmek mümkündür.

---

## Geliştiriciye Destek (Bağışlar)

Projeyi faydalı buluyorsanız ve geliştirmeyi desteklemek isterseniz, kripto cüzdan adresleri aşağıdadır:

- **USDT ERC**: `0x3d52Ce15B7Be734c53fc9526ECbAB8267b63d66E`
- **USDT TRC**: `TEzAAtn4VhndqEaAyuCM78xh5W2gCjwWEo`
- **BTC**: `bc1qhqew3mrvp47uk2vevt5sctp7p2x9m7m5kkchve`
- **ETH**: `0x3d52Ce15B7Be734c53fc9526ECbAB8267b63d66E`

---

## zapret1'den Farkı Nedir?

`zapret2`, zapret projesinin yeni nesil gelişmiş sürümüdür.
Eski sürümün ana bileşeni olan *nfqws1*'in en büyük problemi, aşırı sayıda komut satırı seçeneğiyle yüklenmiş olması ve sansür mekanizmaları ile kullanıcılar arasındaki artan mücadelede trafiğe müdahale konusunda yeterli esnekliği sağlayamamasıydı.
DPI engellerini aşmak zamanla değişen, daha hassas ve özgün müdahaleler gerektirmekte; eski yöntemler ise zamanla işlevini yitirebilmektedir.

Stratejiler; DPI sistemine yönelik saldırı senaryosunu yöneten programlardır. *nfqws1*'de bu stratejiler doğrudan C kaynak koduna gömülmekteydi. C diliyle kod yazmak; yeterli geliştirici yetkinliği ve ciddi zaman gerektiren zorlu bir süreçtir.

*nfqws2*'nin temel amacı; ağ alanında bilgi sahibi, DPI zafiyetlerini anlayan ve temel programlama becerisine sahip herkesin kendi strateji programlarını yazabilmesini sağlamaktır.

*nfqws2*, protokol tanıma, yeniden birleştirme (reassembly), şifre çözme, profil yönetimi, hostlist (alan adı listeleri), ipset ve temel filtreleme gibi temel yetenekleri C seviyesinde korur. Ancak trafiğe bizzat doğrudan müdahale etme mantığını C içinden tamamen çıkarır. DPI aldatma ("desync") mantığı tamamen **Lua** betik diline aktarılmıştır.

- C kodu, gelen paketlerin yapılandırılmış bir ağaç gösterimini (Wireshark'taki analizlere benzeyen "dissect" yapısını) Lua koduna iletir.
- Bazı protokollerin (TLS, QUIC) parça birleştirme ve şifre çözme sonuçları da buraya aktarılır.
- C kodu paket gönderme, ikili veri işleme, TLS ayrıştırma ve işaretçi konumu arama gibi yardımcı fonksiyonları (helpers) sağlar.
- Lua ile yazılmış bir yardımcılar kütüphanesi (`zapret-lib.lua`) ile birlikte, *nfqws1*'in yeteneklerini daha geniş ve esnek bir biçimde uygulayan hazır bir DPI saldırı programları kütüphanesi (`zapret-antidpi.lua`) mevcuttur.

Böylece paket yapısından anlayan herkes DPI ile mücadele edebilir, fikirlerini test edebilir ve ardından arkadaşlarıyla "tek tıkla" çalışan çözümlerini paylaşabilir. zapret2 bu tür meraklılar için tasarlanmış bir araçtır. Ancak "acemiler için hazır, tek tuşluk sihirli bir çözüm" değildir; projenin amacı nesnel nedenlerle her şeyi herkes için aşırı basitleştirmek değildir.

zapret2'nin yetenekleri yalnızca DPI atlatmakla sınırlı değildir. Özünde çok platformlu, programlanabilir bir paket manipülatörüdür. İsteğe bağlı IPv4/IPv6 trafiği üretmek, mevcut trafiği gizlemek/şifrelemek, standart dışı IP iletişim protokolleri kurmak ve özel güvenlik duvarı ya da DPI sistemleri geliştirmek için kullanılabilir.

---

## Nereden Başlamalı?

Başlangıçta karmaşık el kitabına boğulmamak adına, *nfqws2*'nin nasıl çalıştırılacağı ve *nfqws1* stratejilerinin *nfqws2*'ye nasıl aktarılacağı ile başlayalım. Çalışma mantığını kavradıktan sonra, motorun arkasındaki Lua kodlarını inceleyebilir ve [Tam Kılavuz](manual.en.md) dokümanını referans alarak kendi çözümlerinizi üretebilirsiniz.

### Trafik İşleme Mekaniği

Her işletim sisteminde ağ trafiği ilk olarak çekirdeğe (kernel) ulaşır. İlk görev, trafiği oradan çekip *nfqws2* sürecine yönlendirmektir:
- **Linux**: `iptables` veya `nftables`
- **BSD**: `ipfw` veya `pf`
- **Windows**: `windivert`

Trafiği çekirdekten kullanıcı alanına yönlendirmek sistem kaynağı tüketir; bu nedenle mümkün olduğunca fazla filtrelemeyi doğrudan çekirdek seviyesinde yapmak en doğrusudur.

Linux üzerinde test yapmak için bağlantıların ilk paketlerini (TCP 80, 443 ve UDP 443 portları) 200 numaralı NFQUEUE kuyruğuna yönlendiren örnek `nftables` kuralları:

```nft
nft delete table inet ztest
nft create table inet ztest
nft add chain inet ztest post "{type filter hook postrouting priority 101;}"
nft add rule inet ztest post meta mark and 0x40000000 == 0 tcp dport "{80,443}" ct original packets 1-12 queue num 200 bypass
nft add rule inet ztest post meta mark and 0x40000000 == 0 udp dport "{443}" ct original packets 1-12 queue num 200 bypass

sysctl net.netfilter.nf_conntrack_tcp_be_liberal=1 
nft add chain inet ztest pre "{type filter hook prerouting priority -101;}"
nft add rule inet ztest pre meta mark and 0x40000000 == 0 tcp sport "{80,443}" ct reply packets 1-12 queue num 200 bypass
nft add rule inet ztest pre meta mark and 0x40000000 == 0 udp sport "{443}" ct reply packets 1-12 queue num 200 bypass

nft add chain inet ztest predefrag "{type filter hook output priority -401;}"
nft add rule inet ztest predefrag "mark & 0x40000000 != 0x00000000 notrack"
```

Windows'ta paket yakalama mekanizması doğrudan Windows motoru olan `winws2` içerisine entegre edilmiştir ve WinDivert sürücüsünü kullanır.
Portları topluca yakalamak için `--wf-tcp-in`, `--wf-tcp-out`, `--wf-udp-in`, `--wf-udp-out` parametreleri kullanılır (örn. `--wf-tcp-out=80,443`).
Daha hassas yakalama için WinDivert filtre dili kullanılır (`--wf-raw-part`). WinDivert filtre dili `tcpdump` veya `wireshark` sözdizimine benzer.

> [!WARNING]
> WinDivert'in (ve BSD ipfw/pf sistemlerinin) en belirgin kısıtlaması, bağlantıdaki paket sayısına göre filtreleme (iptables'taki `connbytes`, nftables'taki `ct packets`) sınırlandırıcısına sahip olmamasıdır.
> Bir portu tamamen yakalamaya ayarlarsanız, o yöndeki tüm veri akışı (örneğin megabaytlarca dosya indirme/yükleme) kullanıcı alanına iletilir ve işlemciyi aşırı yükleyebilir. Bu nedenle Windows üzerinde WinDivert filtrelerinde payload tipini kontrol eden kurallar yazılmalı veya `--wf-tcp-empty=0` kullanılmalıdır.

Ardından root yetkileriyle *nfqws2* başlatılır:

```bash
nfqws2 --qnum=200 --debug --lua-init=@zapret-lib.lua --lua-init=@zapret-antidpi.lua \
  --filter-tcp=80,443 --filter-l7=tls,http \
  --payload=tls_client_hello --lua-desync=fake:blob=fake_default_tls:tcp_md5:tls_mod=rnd,rndsni,dupsid \
  --payload=http_req --lua-desync=fake:blob=fake_default_http:tcp_md5 \
  --payload=tls_client_hello,http_req --lua-desync=multisplit:pos=1:seqovl=5:seqovl_pattern=0x1603030000
```

Bu örnekte:
- `--lua-init`: Başlangıçta 1 kez çalıştırılacak Lua kodunu veya `@` ile belirtilen dosyayı yükler.
- `--lua-desync`: Profil içinden geçen her pakette çağrılacak Lua fonksiyonunu ve parametrelerini (`param[=değer]`) tanımlar.

*nfqws1*'deki karşılığı:
```bash
nfqws --qnum=200 --debug \
  --filter-tcp=80,443 --filter-l7=tls,http \
  --dpi-desync=fake,multisplit --dpi-desync-fooling=md5sig --dpi-desync-split-pos=1,midsld \
  --dpi-desync-split-seqovl=5 --dpi-desync-split-seqovl-pattern=0x1603030000 \
  --dpi-desync-fake-tls-mod=rnd,rndsni,dupsid
```

### Önemli Yenilikler ve Kavramlar

1. **Yük Tipi (Payload)**: *nfqws1*'de yalnızca bağlantı protokolleri filtrelenebiliyordu. *nfqws2*'de ise her paketin anlık içeriği (payload türü) denetlenir (örn. `tls_client_hello`, `tls_server_hello`, `http_req`, `unknown`).
2. **Bağımsız Desenkronizasyon Fazları**: Eski sürümdeki katı `fake,multisplit` sıralaması yerine, ardışık olarak istenen sayıda ve farklı parametrelerle Lua fonksiyonları çağrılabilir.
3. **Otomatik TCP Segmentasyonu**: `zapret-lib.lua` bağlantının MSS değerini izler. Paket MSS sınırını aşıyorsa otomatik segmentasyon uygulanır. Büyük `seqovl` veya Kyber TLS el sıkışmalarında hata oluşmaz.
4. **Veri Blokları (Blobs)**: Sabit kodlanmış `--dpi-desync-fake-tls` gibi parametreler yerine `blob` değişkenleri kullanılır. Bloblar hex dizgi olarak veya `--blob=name:@dosya` ile parametre olarak aktarılabilir.
5. **Yönlü Aralıklar (Ranges)**: `--in-range` ve `--out-range` ile gelen ve giden yönde paket sınırları belirlenir (`nX`: paket numarası, `dX`: veri paketi numarası, `bX`: iletilen bayt miktarı, `sX`: TCP göreceli sequence). Varsayılan: `--in-range=x --out-range=a --payload=all`.

### Karar Mekanizması (Verdicts)

Her Lua fonksiyonu paket üzerinde bir karar bildirir:
- `VERDICT_PASS`: Paketi olduğu gibi geçir.
- `VERDICT_MODIFY`: Değiştirilmiş paketi ilet.
- `VERDICT_DROP`: Paketi düşür (gönderme).

Nihai karar kombinasyona bağlıdır: Herhangi bir fonksiyon `VERDICT_DROP` döndürürse paket düşürülür; düşürülmezse ve herhangi biri `VERDICT_MODIFY` verdiyse değiştirilmiş sürüm gönderilir; hepsi `VERDICT_PASS` verdiyse orijinal paket geçer.

---

## nfqws1 Stratejilerini nfqws2'ye Uyarlama Örnekleri

### 1. HTTP Fake ve TTL Ayarı
*nfqws1*:
```bash
nfqws \
  --filter-l7=http --dpi-desync=fake --dpi-desync-fake-http=0x00000000 --dpi-desync-ttl=6 \
  --orig-ttl=1 --orig-mod-start=s1 --orig-mod-cutoff=d1
```
*nfqws2*:
```bash
nfqws2 \
  --filter-l7=http \
  --payload=http_req --lua-desync=fake:blob=0x00000000:ip_ttl=6:ip6_ttl=6 \
  --payload=empty --out-range="s1<d1" --lua-desync=pktmod:ip_ttl=1:ip6_ttl=1
```

### 2. Fake Bölme (fakedsplit) ve Badseq / TCP Timestamp
*nfqws1*:
```bash
nfqws \
  --filter-l7=http \
  --dpi-desync=fakedsplit --dpi-desync-fooling=badseq --dpi-desync-badseq-increment=0 --dpi-desync-split-pos=method+2
```
*nfqws2*:
```bash
nfqws2 \
  --filter-l7=http \
  --payload=http_req --lua-desync=fakedsplit:pos=method+2:tcp_ack=-66000:tcp_ts_up
```

### 3. Otomatik TTL (autottl)
*nfqws1*:
```bash
nfqws \
  --filter-l7=tls \
  --dpi-desync=fakedsplit --dpi-desync-fakedsplit-pattern=tls_clienthello_google_com.bin \
  --dpi-desync-ttl=1 --dpi-desync-autottl=-1 --dpi-desync-split-pos=method+2 --dpi-desync-fakedsplit-mod=altorder=1
```
*nfqws2*:
```bash
nfqws2 \
  --blob=tls_google:@tls_clienthello_google_com.bin \
  --filter-l7=tls \
  --payload=tls_client_hello,http_req \
  --lua-desync=fakedsplit:pattern=tls_google:pos=method+2:nofake1:ip_ttl=1:ip6_ttl=1:ip_autottl=-1,3-20:ip6_autottl=-1,3-20
```

### 4. Window Size ve SYN Data
*nfqws1*:
```bash
nfqws --dpi-desync=syndata,multisplit --dpi-desync-split-pos=midsld --wssize 1:6
```
*nfqws2*:
```bash
nfqws2 --lua-desync=wssize:wsize=1:scale=6 --lua-desync=syndata --lua-desync=multisplit:pos=midsld
```

### 5. IP Parçalama (ipfrag)
*nfqws1*:
```bash
nfqws --dpi-desync=ipfrag2 --dpi-desync-ipfrag-pos-udp=8
```
*nfqws2*:
```bash
nfqws2 --lua-desync=send:ipfrag:ipfrag_pos_udp=8 --lua-desync=drop
```

---

## Windows Paketi (zapret-win-bundle) Örneği

`preset_example.cmd` dosyasının `preset2_example.cmd` olarak yeniden yazılmış hali:

```cmd
start "zapret: http,https,quic" /min "%~dp0winws2.exe" ^
  --wf-tcp-out=80,443 ^
  --lua-init=@"%~dp0lua\zapret-lib.lua" --lua-init=@"%~dp0lua\zapret-antidpi.lua" ^
  --lua-init="fake_default_tls = tls_mod(fake_default_tls,'rnd,rndsni')" ^
  --blob=quic_google:@"%~dp0files\quic_initial_www_google_com.bin" ^
  --wf-raw-part=@"%~dp0windivert.filter\windivert_part.discord_media.txt" ^
  --wf-raw-part=@"%~dp0windivert.filter\windivert_part.stun.txt" ^
  --wf-raw-part=@"%~dp0windivert.filter\windivert_part.wireguard.txt" ^
  --wf-raw-part=@"%~dp0windivert.filter\windivert_part.quic_initial_ietf.txt" ^
  --filter-tcp=80 --filter-l7=http ^
    --out-range=-d10 ^
    --payload=http_req ^
     --lua-desync=fake:blob=fake_default_http:ip_autottl=-2,3-20:ip6_autottl=-2,3-20:tcp_md5 ^
     --lua-desync=fakedsplit:ip_autottl=-2,3-20:ip6_autottl=-2,3-20:tcp_md5 ^
    --new ^
  --filter-tcp=443 --filter-l7=tls --hostlist="%~dp0files\list-youtube.txt" ^
    --out-range=-d10 ^
    --payload=tls_client_hello ^
     --lua-desync=fake:blob=fake_default_tls:tcp_md5:repeats=11:tls_mod=rnd,dupsid,sni=www.google.com ^
     --lua-desync=multidisorder:pos=1,midsld ^
    --new ^
  --filter-tcp=443 --filter-l7=tls ^
    --out-range=-d10 ^
    --payload=tls_client_hello ^
     --lua-desync=fake:blob=fake_default_tls:tcp_md5:tcp_seq=-10000:repeats=6 ^
     --lua-desync=multidisorder:pos=midsld ^
    --new ^
  --filter-udp=443 --filter-l7=quic --hostlist="%~dp0files\list-youtube.txt" ^
    --payload=quic_initial ^
     --lua-desync=fake:blob=quic_google:repeats=11 ^
    --new ^
  --filter-udp=443 --filter-l7=quic ^
    --payload=quic_initial ^
     --lua-desync=fake:blob=fake_default_quic:repeats=11 ^
    --new ^
  --filter-l7=wireguard,stun,discord ^
    --payload=wireguard_initiation,wireguard_cookie,stun,discord_ip_discovery ^
     --lua-desync=fake:blob=0x00000000000000000000000000000000:repeats=2
```

---

## Standart Dışı İşlemler ve Dinamik Lua Çalıştırma (`luaexec`)

Örnek: Bilinen yüke sahip orijinal bir isteği; 5 ila 10 karakter arasında rastgele 'a'-'z' harflerinden oluşan bir `seqovl` deseniyle göndermek:

```bash
nfqws2 \
  --lua-desync=luaexec:code='desync.rnd=brandom_az(math.random(5,10))' \
  --lua-desync=tcpseg:pos=0,-1:seqovl=#rnd:seqovl_pattern=rnd \
  --lua-desync=drop:payload=known
```

Burada:
- `luaexec`: Çalışma anında Lua kodunu dinamik olarak yürütür ve üretilen blobu bir sonraki fonksiyona aktarır.
- `#rnd`: `rnd` bloğunun bayt uzunluğunu dinamik olarak parametreye yerleştirir.
- `%rnd`: Bloğun ikili içeriğini parametre olarak çözer.

---

## Önemli Tavsiye: `--debug` Günlüğü

`nfqws2` ve `winws2` araçlarını öğrenirken `--debug` parametresini kullanmayı alışkanlık haline getirin. Hata ayıklama günlükleri olmadan paketlerin hangi kurala takıldığını, neden kesildiğini veya Lua betiğindeki hataları tespit etmek oldukça zordur.

---

## Sadece DPI Aldatması Değil: Protokol Gizleme (Obfuscation)

Örnek: Windows istemci ile VPS sunucusu arasındaki WireGuard UDP trafiğini ICMP ping paketlerine dönüştürerek sansürü aşma:

### 1. Sunucu Tarafı (Linux VPS - `1.2.3.4:5555`)
```nft
table ip ztest {
  chain post {
    type filter hook output priority mangle; policy accept;
    meta mark & 0x40000000 == 0x00000000 udp sport 5555 queue flags bypass to 200
  }
  chain pre {
    type filter hook input priority mangle; policy accept;
    meta mark & 0x40000000 == 0x00000000 icmp type echo-request icmp code 199 queue flags bypass to 200
  }
}
```

```bash
nfqws2 --qnum=200 --server \
  --lua-init=@/opt/zapret2/lua/zapret-lib.lua \
  --lua-init=@/opt/zapret2/lua/zapret-obfs.lua \
  --in-range=a \
  --lua-desync=udp2icmp:ccode=199:scode=199
```

### 2. İstemci Tarafı (Windows)
```cmd
winws2 ^
  --wf-icmp-in=0:199 --wf-udp-out=5555 ^
  --wf-raw-filter="ip.SrcAddr=1.2.3.4 or ip.DstAddr=1.2.3.4" ^
  --lua-init=@lua/zapret-lib.lua ^
  --lua-init=@lua/zapret-obfs.lua ^
  --in-range=a ^
  --lua-desync=udp2icmp:ccode=199:scode=199
```
Bu sayede iki uç arasındaki tüm WireGuard trafiği ağ üzerinde normal ping paketleri (ICMP code 199) gibi görünür ve NAT arkasından sorunsuz geçer.
