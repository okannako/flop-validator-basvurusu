<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/de1353b7-040e-4b68-9091-48d7940c2689" />


# FLOP Testnet Validator Başvuru Kılavuzu

FLOP Network'ün Devnet ağına **keyless RPC replica** node kurma rehberi. Yani: cuzdansız, imzasız, oy vermeyen, sadece block'ları doğrulayan ve relay eden bir node. Validator başvurusu için **sync & restart evidence** kanıtı üretmeye yarar.

**Ne İçin lazım:**

- Testnet validator başvurusu yaptığını varsayıyoruz (`github.com/flop-labs/network/issues/new?template=operator-application.yml`).
- Form "Sync and restart evidence" alanı istiyor: chain spec hash, finalized heights, peer ID, restart continuity vs.
- Bu kanıtları üretmek için Devnet'te çalışan bir keyless RPC node'a ihtiyacın var.
- Kanıt üretildikten sonra GitHub issue'na yorum olarak eklenir → ekip inceler → validator promote sıran güçlenir.

**Ne değildir:**

- Bu bir **validator** kurulumu değil. Validator kurulumu için imzalı operator kit beklenmekte.
- Bu bir **miner** kurulumu değil. GPU yok, inference yok.
- Bu bir **ödüllü node** değil. Devnet'te doğrudan ödül yok.

**Kime uygun:**

- FLOP testnet veya devnet validator adayları
- Linux + Docker'a hakim, SSH ile sunucu yönetebilen
- 6-12 saat sync sürecine sabredebilen
- Kanıt üretene kadar sunucuyu ayakta tutabilen (ucuz VPS yeter)

**Zaman beklentisi:**

- Kurulum: 20-40 dakika
- Peer bulma: 1-5 dakika
- Sync (genesis'ten tip'e): **6-12 saat** (archive, \~57 GB)
- Observation komutları: 5 dakika
- GitHub yorumu: 2 dakika

## Sunucu ve Sağlayıcı Gereksinimleri

### Donanım (Minimum)

| Bileşen | Minimum | Önerilen |
| --- | --- | --- |
| CPU | 4 vCPU | 8 vCPU |
| RAM | 16 GB | 32 GB |
| Disk | 100 GB SSD | 300 GB NVMe (archive büyüyor) |
| Ağ | 1 Gbps | 1 Gbps |
| OS | Ubuntu 22.04 veya 24.04 | Ubuntu 24.04 |
| GPU | Gerekmez | Gerekmez |

### Kritik: **NAT Tuzağı**

Sunucunun **gerekçe public IP'si interface'e direkt atanmış** olması şart. NAT arkasındaki VPS'lerde (QEMU SLIRP, "shared IP" tarzı planlar) libp2p peer handshake tamamlanmıyor, node 0 peer'da kalıyor.

Sunucuyu almadan önce ya da almadığında hemen kontrol et:

```sh
ip -4 addr show
PUB_IP=$(curl -s https://ipinfo.io/ip)
echo "Public IP: $PUB_IP"
ip -4 addr show | grep -q "$PUB_IP" && echo "✅ DIRECT PUBLIC IP (devam et)" || echo "❌ NAT (bu VPS uygun değil)"
```

`❌ NAT` görürsen VPS'i değiştir. Para kaybetmemek için satın almadan sağlayıcıya "direct public IP on interface" soracaksın.

### Hetzner Kullanırken Dikkat

Hetzner'da kripto node'larıyla ilgili geçmişte sıkıntılar yaşandı — 2022'de Solana validator'ları toplu olarak kapatılmıştı. Günümüzde politikaları değişmiş olabilir, kullanmak istiyorsan sağlayıcının güncel kullanım şartlarını kontrol et. Aynı titizlik diğer büyük bulut sağlayıcıları (AWS, GCP, Azure, DigitalOcean) için de geçerli.

### NAT'lı Planlara Dikkat

Contabo'nun bile bazı eski "VPS SSD" planları NAT'lı veriliyor (QEMU SLIRP). Mutlaka **Cloud VPS** veya **Dedicated Server** iste. Hostname'inde `10.0.2.15` veya `192.168.x.x` gibi IP görürsen NAT demektir.

## Adım 1: Sunucuyu Hazırla

SSH ile sunucuna root olarak bağlan. Docker + Compose + yardımcı araclar kur:

```sh
# Docker varsa atla, yoksa kur
docker --version && docker compose version || {
  curl -fsSL https://get.docker.com | sh
  systemctl enable --now docker
}

# Ek paketler
apt update
apt install -y jq curl openssl ufw

# Çalışma dizini
mkdir -p ~/flop-devnet-rpc && cd ~/flop-devnet-rpc
```

Son komuttan sonra `/root/flop-devnet-rpc` içinde olmalısın (`pwd` ile kontrol).

## Adım 2: Paketi İndir ve Doğrula

FLOP ekibi setup dosyalarını bir gist'te pinli olarak yayınlıyor. Belirli bir revision'a kilitli:

```sh
cd ~/flop-devnet-rpc

files=https://gist.githubusercontent.com/sv/7c8b93f3868bc15e3b6eabe0578057e3/raw/21adaa99c601003278d1682bf937696f21d31432
for file in docker-compose.rpc.yml node-entrypoint.sh rpc_ready.py rpc-pool.env.example rpc-origin.Caddyfile; do
  curl --fail --location --output "$file" "$files/$file"
done
```

**SHA-256 doğrulaması** (güvenlik kritik):

```sh
sha256sum --check <<'SUMS'
6487cd84d5a257659297ad7b8fdd55978e6d8327bfe029d9460645da2591345d  docker-compose.rpc.yml
7279e81c0d6de4b6c59733b64132f929b8b7dd5825b8f37433d09940a93f983d  node-entrypoint.sh
6a066b7540628c7db4f455d7bfd2540db54355a4c0a2959d97032b76bcf641cd  rpc_ready.py
4f7295735f75b5a7b4c0babe3be4b0eef687decab99ba1612f91ee6f3b04d1bb  rpc-pool.env.example
e184d4f38f054dbe7947efde4d85e6a8964f02e594666ad42b9216dd98a4b2d5  rpc-origin.Caddyfile
SUMS
```

5 satır `OK` görmelisin. Herhangi bir satır `FAILED` dersen durdur, tekrar indir.

## Adım 3: Chain Spec'i İndir

Devnet'in chain spec dosyası ayrı, resmi explorer sayfasından:

```sh
cd ~/flop-devnet-rpc

curl --fail --location --output chain.raw.json \
  https://explorer.devnet.flop.network/validator-downloads/flop_devnet_ts-99c172c095340ace.raw.json

printf '%s  %s\n' \
  3b213c38e7678285bc38fbdfded5dbe3d6acbd16e836b14aaaeef2f6ae1cd16e \
  chain.raw.json | sha256sum --check
```

`OK` görmelisin.

**Genesis hash doğrulaması** — chain spec'inin canlı Devnet ile aynı chain'de olduğunu te'yit:

```sh
curl -s https://rpc.devnet.flop.network -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"chain_getBlockHash","params":[0],"id":1}' | jq -r .result
```

Beklenen: `0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631`. Bu değer **kesinlikle eşleşmeli**. Eger eslesmezse chain reset olmuş demektir, yeni spec indirmen gerek.

## Adım 4: Env Dosyasını Doldur

Example'dan kopyala, public IP ve bind IP'yi doldur:

```sh
cd ~/flop-devnet-rpc
cp -n rpc-pool.env.example rpc-pool.env

# Public IP otomatik al
PUB_IP=$(curl -s https://ipinfo.io/ip)
echo "Public IP: $PUB_IP"

# HOST_IP=0.0.0.0 (tüm interface'leri dinle), PUBLIC_ADDR=/ip4/PUB_IP/tcp/30333
sed -i "s|^HOST_IP=.*|HOST_IP=0.0.0.0|" rpc-pool.env
sed -i "s|^PUBLIC_ADDR=.*|PUBLIC_ADDR=/ip4/$PUB_IP/tcp/30333|" rpc-pool.env

# Doğrula
grep -E '^(HOST_IP|PUBLIC_ADDR|CHAIN|FLOP_NODE_IMAGE)=' rpc-pool.env
```

**Dikkat:**

- `HOST_IP=0.0.0.0` — Docker port'u tüm interface'lerde dinleyecek. NAT'lı provider'da gerekli, direkt IP'de de güvenli.
- `PUBLIC_ADDR=/ip4/<public_ip>/tcp/30333` — diğer peer'lar sana bağlanmak için bu adresi kullanacak.
- **`PUBLIC_ADDR`'da hostname değil IP kullan.** DNS sorun çıkartır.
- `FLOP_NODE_IMAGE` pinli sha'da bırak, `latest` kullanma.

## Adım 5: BOOTNODES'u Genişlet (Kritik)

Bu kurulumun en kritik adımı. Default env'de **sadece bir** bootnode var ve o tek başına keyless RPC'leri kabul etmekte zorlanıyor. 0 peers'da sıkılır kalırsın.

Çözüm: Devnet'teki başka public node'ları BOOTNODES listesine eklemek. Birkaç operötörün acıkça publish ettiği P2P adreslerini aklatıp ekleyebilirsin.

```sh
cd ~/flop-devnet-rpc

# Resmi bootnode + 3 ek public peer
sed -i 's|^BOOTNODES=.*|BOOTNODES=/ip4/139.180.223.117/tcp/30333/p2p/12D3KooWACzHKN2YkLtSwjTYXYtMHcVNWnJhG3hRdUkqa5DzBu9X /ip4/136.244.77.30/tcp/30333/p2p/12D3KooWJczvBNuzXNkD4AuiD4dwrKqVwXYzK2bYeDe4vBMLM2yT /ip4/43.143.199.81/tcp/30333/p2p/12D3KooWMhfWgpxzRo5XSb3nfe7yjJok8NxruqF1u1CqFGHaCqjb /ip4/65.109.36.231/tcp/30333/p2p/12D3KooWJZBres7eDNVSLH9HPxd1mLUpMiqHiBRAvDyxiAe98DHs|' rpc-pool.env

grep ^BOOTNODES rpc-pool.env
```

**Önemli kurallar:**

- Her adres `/ip4/<IP>/tcp/<port>/p2p/<peer_id>` formatında olmalı. **Peer ID eksik olan** adresler substrate tarafından reddedilir, node başlayamaz.
- Adresler boşluk ile ayrılır (comma değil).
- Bu listedeki adresler zamanla eskiyebilir — sunucu kapatan operatörler olabilir. En az 2-3 tanesinin canlı olması yeter.

**Güncel public peer bulma:**

GitHub'da başka operatör başvurularına bakarak güncel peer adresleri bulabilirsin: `github.com/flop-labs/network/issues?q=is%3Aissue+operator+devnet+validator` → son issue'ları aç → `Advertised public P2P multiaddress` alanındaki adresleri kopyala.

## Adım 6: Firewall ve Port 30333

P2P portu (30333/TCP) dinlırlik ve dinlemenin yanı sıra dışarıdan erişilebilir olmalı.

**UFW varsa:**

```sh
ufw status
# Aktifse:
ufw allow 22/tcp    # SSH'ı kapatma
ufw allow 30333/tcp
ufw status
```

**Provider firewall paneli:**

Contabo/Vultr/OVH gibi sağlayıcıların kendi panellerinden ek firewall olabilir. Oradan da 30333/TCP inbound izin ver.

**Dolayılı test:**

Kurulum bitince `https://portchecker.co` adresinden `<public_ip>:30333` kontrolü yap. `Open` olmalı.

## Adım 7: Compose ile Başlat ve Peer Kontrolü

Config kontrolü + ilk çalıştırma:

```sh
cd ~/flop-devnet-rpc
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml config --quiet && echo "config OK"

docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d

sleep 20
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml ps
```

`ps` çıktısında **iki container da `Up`** görmeli:

- `flop-devnet-rpc-pool-rpc-1`
- `flop-devnet-rpc-pool-ready-1`

Eğer rpc container'ı `Restarting` veya `Created` durumundaysa sorun var:

```sh
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml logs --tail=30 rpc
```

Logda sorunu görürsün (genelde env dosyası veya BOOTNODES hatası).

**Peer kontrolü (60-90 saniye sonra):**

```sh
sleep 90
curl -s http://127.0.0.1:9619/ready | jq
```

Beklenen çıktı:

```json
{
  "rpc_ok": true,
  "best": <artan sayı>,
  "finalized": <artan sayı>,
  "highest": 4627000+,
  "peers": 3+,
  "ready": false,
  "reasons": ["sync lag ..."]
}
```

**En kritik alanlar:**

- `peers` **1+** olmalı — 2 dakika sonra hâlâ 0 ise BOOTNODES'u yeniden gözden geçir
- `best` sabit kalmamalı — her çağırışta büyük büyük artıyor olmalı
- `ready: false` + `reasons: ["sync lag ..."]` NORMAL — sync devam ediyor demek
- Docker'da `(unhealthy)` görebilirsin, sorun değil — sync bitince `healthy` olacak

## Adım 8: Sync'in Bitmesini Bekle

Archive sync **6-12 saat** sürebilir. Devnet'in boyutu \~57 GB, 4.6M+ block. Bu süre:

- Sunucu hızına
- Peer sayısı ve kalitesine (daha fazla peer = daha hızlı)
- Ağ bant genişliğine

bağlı.

**Ara ara kontrol:**

```sh
curl -s http://127.0.0.1:9619/ready | jq '{best, finalized, highest, peers, best_frozen_s, reasons}'
```

**Beklentiler:**

- `peers` istikrarlı veya artan (8+ ideal)
- `best` sürekli artan
- `highest - best` küçülen
- `ready` sonunda `true` olacak, `reasons` boşalacak

**Tahmini zaman hesabı:**

```sh
# İki ölçüm arası ne kadar block gelmiş
FIRST=$(curl -s http://127.0.0.1:9619/ready | jq .best)
sleep 300
SECOND=$(curl -s http://127.0.0.1:9619/ready | jq .best)
echo "5 dk'da gelen block: $((SECOND - FIRST))"
echo "Kalan tahmini: $(curl -s http://127.0.0.1:9619/ready | jq '(.highest - .best) / 1000')"K block
```

**Çalışma sırasında yapmaman gerekenler:**

- `docker compose down -v` — volume'u siler, chain veritabanı gider
- Sunucuyu reboot etmek (sync kesintiye uğrar ama devam eder)
- Image'ı değiştirmek
- `rpc-pool.env`'i editlemek (yeniden başlatma gerekir)

**Güvenli işlemler:**

- `docker compose logs -f rpc` — log takibi
- `docker compose ps` — durum kontrolü
- Ready endpoint sorguları

**Sync bittiğinde:**

```json
{
  "rpc_ok": true,
  "best": 4700000+,
  "finalized": 4700000+,
  "highest": 4700000+,
  "peers": 8+,
  "ready": true,
  "reasons": []
}
```

`ready: true` görünce evidence adımına geç.

## Adım 9: Observation ve Restart Evidence

Sync tamamlandığında (ready: true) kanıt üretmeye hazırsın. Bu komutlar resmi gist'in 2. bölümünden geliyor, birebir kullanılır.

**Yardımcı fonksiyon + sabitler:**

```sh
cd ~/flop-devnet-rpc

rpc() {
  curl --fail --silent --show-error --max-time 15 "$1" \
    -H 'Content-Type: application/json' \
    --data "$(jq -cn --arg method "$2" --argjson params "$3" \
      '{jsonrpc:"2.0",id:1,method:$method,params:$params}')" | jq -er '.result'
}

LOCAL=http://127.0.0.1:9944/
FLOP=https://rpc.devnet.flop.network/
GENESIS=0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631
```

**Finalized height okuma:**

```sh
finalized_height() {
  local head params hex
  head=$(rpc "$1" chain_getFinalizedHead '[]')
  params=$(jq -cn --arg head "$head" '[$head]')
  hex=$(rpc "$1" chain_getHeader "$params" | jq -er .number)
  printf '%d\n' "$hex"
}
```

**Tek observation fonksiyonu:**

```sh
observe() {
  local local_height reference_height common_height local_hash reference_hash
  test "$(rpc "$LOCAL" chain_getBlockHash '[0]')" = "$GENESIS"
  test "$(rpc "$FLOP" chain_getBlockHash '[0]')" = "$GENESIS"
  curl --fail --silent --show-error --max-time 5 http://127.0.0.1:9619/ready >/dev/null
  local_height=$(finalized_height "$LOCAL")
  reference_height=$(finalized_height "$FLOP")
  common_height=$local_height
  if (( reference_height < common_height )); then common_height=$reference_height; fi
  local_hash=$(rpc "$LOCAL" chain_getBlockHash "[$common_height]")
  reference_hash=$(rpc "$FLOP" chain_getBlockHash "[$common_height]")
  test "$local_hash" = "$reference_hash"
  printf '\nUTC=%s local_finalized=%s reference_finalized=%s common_height=%s hash=%s peer=%s\n' \
    "$(date -u +%FT%TZ)" "$local_height" "$reference_height" "$common_height" \
    "$local_hash" "$(rpc "$LOCAL" system_localPeerId '[]')"
}
```

**3 observation + restart:**

```sh
# Peer ID'yi kaydet (restart sonrası aynı olması şart)
peer_before=$(rpc "$LOCAL" system_localPeerId '[]')
echo "Peer ID: $peer_before"

# Observation 1
observe

# 60 saniye bekle
sleep 60

# Observation 2
observe

# Restart
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml restart rpc ready

# Readiness bekle (1-3 dakika)
while ! curl -sf http://127.0.0.1:9619/ready | jq -e '.ready == true' >/dev/null 2>&1; do
  echo "Readiness bekliyor..."
  sleep 10
done

# Observation 3 (restart sonrası)
observe

# Peer ID aynı mı
test "$(rpc "$LOCAL" system_localPeerId '[]')" = "$peer_before" && echo "✅ Peer ID unchanged" || echo "❌ Peer ID changed"
```

**3 observation çıktısını kaydet** — bunları GitHub issue'na yapıştıracaksın.

Beş kritik koncept:

1. **Local ve reference genesis hash eşleşmeli** (test komutu zaten doğruluyor)
2. **Finalized heights artan olmalı** (1→2→3 observation'da artmalı)
3. **Common height hash'leri eşleşmeli** (LOCAL ve FLOP aynı block'u imzalamış olmalı)
4. **Restart sonrası finality ilerlemeye devam** etmeli
5. **Peer ID değişmemeli** (volume ve node key korundu demek)

Herhangi biri fail ise kanıt geçerli değil, sorunu çözmek gerek.

## Adım 10: Operator Application Formunu Aç (Henüz Yapmadıysan)

Daha önce devnet validator candidate olarak başvurmadıysan, önce formu aç. Devnet keyless RPC kurulumu yapıp sync bittiyse evidence'ı doğrudan ilk başvuruda doldurabilirsin.

### İki Strateji: Ne Zaman Başvuru

**A) Önce başvuru, sonra evidence (urgency için)**

- Sunucu henüz hazır değilken bile başvuru aç, `planning a host` seç
- İlk 50 koltuk kuyruğunda yerini hemen al
- Sync bitince Adım 11 ile observation'ları yorum olarak ekle
- **Avantaj:** erken başvuru, erken promote sırası
- **Dezavantaj:** başvuruda evidence boş, sonradan update gerekir

**B) Önce sync, sonra tam başvuru (temiz)**

- Adım 1-9'u tamamla, node sync'li ve observation'lar hazır olsun
- Sonra başvuru aç, evidence alanını doğrudan doldur
- `sync and restart evidence ready` olarak preparation stage seç
- **Avantaj:** tek ve temiz başvuru, ekibe hemen incelemeye hazır görünür
- **Dezavantaj:** sync 6-12 saat sürer, o sürede kuyrukta başkaları öne geçebilir

**Karar:** Acelen yoksa **B** daha derli toplu. Kuyruk kritik (örn. ilk 50 koltuk söz konusu) ise **A** daha güvenli.

### Form URL

```
https://github.com/flop-labs/network/issues/new?template=operator-application.yml&network=devnet
```

GitHub hesabınla giriş yap. Form template otomatik yüklür. Üstte "Validator / RPC operator application" yazsın.

### Alanlar

| Alan | Devnet için ne yaz |
| --- | --- |
| **Title** | `operator: <team-id> devnet validator candidate` |
| **Team ID** | Kendi public takma adın (örn. `NakoTurk`). Her yerde aynı kullan. |
| **Network** | `devnet` (dropdown) |
| **Requested role** | `validator candidate` (dropdown). RPC pool'a da katkı istersen `both` |
| **Existing application or introduction** | Önceki issue URL'in yoksa boş bırak |
| **Planned capacity and host** | Aşağıdaki template'i doldur |
| **Preparation stage** | Sync bittiyse `sync and restart evidence ready`, değilse `planning a host` |
| **Sync and restart evidence** | Sync bittiyse Adım 9'daki observation çıktıları (Adım 11'deki template ile), değilse `"Will update within 24 hours after keyless RPC sync completes"` |
| **Questions or blockers** | Engel varsa yaz, yoksa `none` |
| **Before submitting** | 3 checkbox — hepsini tikle |

### Planned Capacity and Host (Template)

Bu alana şunu yapıştır, kendi değerlerinle değiştir:

```
Requested validator seats / existing FLOP seats: 1 requested / 0 existing
RPC nodes offered (active / standby): 1 active keyless RPC replica / 0 standby
Provider / region: <Contabo Cloud VPS, Germany (Nurnberg)>
CPU / RAM / SSD capacity / OS: <8 vCPU / 32 GB / 300 GB NVMe / Ubuntu 24.04>
Public host label or P2P address (if prepared): <team-name-rpc-01>, /ip4/<public_ip>/tcp/30333/p2p/<peer_id>
```

### Submit

1. Tüm alanları doldur
2. 3 checkbox'ı tikle
3. "Create" butonuna bas
4. Issue URL'sini kaydet (örn. `github.com/flop-labs/network/issues/XXXX`)
5. Issue'yu "Watch" et — ekibin yorumu için bildirim alsın

### Sync Bittiğinde

Sync devam ederken başvuru açtıysan, bittiğinde Adım 11'i uygula — observation çıktılarını yorum olarak ekleyeceksin.

### Dikkat

- Formda **asla** seed, keystore parolası, session token veya .env içeriği yazma
- `Team ID` sonradan değiştirmek zor, dikkatli seç
- Takip başına, ağ başına **tek issue** — ikinci açma, mevcut olanı güncelle

## Adım 11: Evidence'ı GitHub Issue'na Ekle

Observation'lar başarılı çıktıysa validator başvurusu issue'na yorum olarak ekle. Template:

```markdown
### Sync and restart evidence — <tarih>

**Host and image**
- Public host label: `<kendi label>`
- Advertised P2P: `/ip4/<public_ip>/tcp/30333/p2p/<senin_peer_id>`
- Node image digest: `ghcr.io/flop-labs/flop-core/node-devnet@sha256:84498fd73db2464eb13088ab34d369ddf7cddd22a45c810d140c9ff8ec16c91d`
- Guide revision: `gist sv/7c8b93f3868bc15e3b6eabe0578057e3 @ 21adaa99c601003278d1682bf937696f21d31432`

**Chain identity**
- Chain spec SHA-256: `3b213c38e7678285bc38fbdfded5dbe3d6acbd16e836b14aaaeef2f6ae1cd16e`
- Expected genesis: `0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631`
- Local and reference genesis match: verified

**Observation 1**
UTC=<zaman> local_finalized=<yükseklik> reference_finalized=<yükseklik> common_height=<yükseklik> hash=<hash> peer=<peer_id>

**Observation 2** (after ~60s)
UTC=<zaman> local_finalized=<yükseklik> reference_finalized=<yükseklik> common_height=<yükseklik> hash=<hash> peer=<peer_id>

**Restart**
- `docker compose restart rpc ready` executed; readiness returned to true
- Same Compose project and volume retained
- No Tailscale on this host

**Observation 3** (after restart)
UTC=<zaman> local_finalized=<yükseklik> reference_finalized=<yükseklik> common_height=<yükseklik> hash=<hash> peer=<peer_id>

**Continuity checks**
- Finality advancing: observations 1 → 2 → 3 heights strictly increasing
- Common height hashes match between local and reference at each observation
- Peer ID unchanged after restart: `<peer_id>`
```

**Önemli:**

- `<...>` içindeki her yere gerçek değeri yapıştır (observation çıktılarından)
- Seed, keystore parolası, session token — **asla** yazıp paylasılmaz. Public bilgiler zaten public IP ve peer ID.
- Yorumu issue'nun en altına ekle, varolan description'ı editleme (başkaları gormeli).
- `Comment` butonuna bas, sonra ekibin yanıtı için "Watch" et.

## Sorun Giderme ve Sık Hatalar

| Belirti | Sebep | Çözüm |
| --- | --- | --- |
| `cannot assign requested address` | VPS NAT arkasında | VPS değiştir (direct IP) |
| rpc container `Restarting` + "Peer id is missing" | BOOTNODES'ta peer ID eksik adres var | Sadece tam `/p2p/12D3KooW...` adısını bırak |
| `rpc_ok: false` + URLError | rpc container başlamamış, ready container bunu arayamiyor | `docker compose logs rpc` ile hata bak |
| `peers: 0` 5 dk+ | Pinli bootnode handshake reddi | BOOTNODES'u ek public peer'larla genişlet (Adım 5) |
| `peers: 0` + loglar `Failed to connect using existing socket` | Aynı (handshake reddi) | BOOTNODES'u genişlet |
| `ps` çıktısında `(unhealthy)` | Sync devam ediyor, healthcheck lag'e takılıyor | Normal, görmezden gel. Sync bitince `healthy` |
| portchecker `Closed` | Provider firewall 30333 kapalı | Panel'den inbound aç veya UFW kontrol |
| `genesis hash mismatch` | Chain reset olmuş | Yeni spec indir |
| `no space left on device` | Disk doldu | Bigger VPS'e migrate, yoksa `docker compose down` + delete |
| `Discovered new external address` yanlış IP | `PUBLIC_ADDR` yanlış | env düzelt, restart |

**Debug log açma (tanı için):**

```sh
cd ~/flop-devnet-rpc
if grep -q "^NODE_LOG=" rpc-pool.env; then
  sed -i "s|^NODE_LOG=.*|NODE_LOG=sync=debug,sub-libp2p=debug,libp2p_tcp=debug|" rpc-pool.env
else
  echo "NODE_LOG=sync=debug,sub-libp2p=debug,libp2p_tcp=debug" >> rpc-pool.env
fi
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml down
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d
sleep 45
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml logs --tail=100 rpc 2>&1 | grep -iE "(discover|dial|peer|connect|error|fail)" | tail -30
```

Sorunu buldun mu, debug'ı kapat (disk ve CPU tasarrufu):

```sh
sed -i "/^NODE_LOG=/d" rpc-pool.env
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml down
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d
```

## Günlük İşletim ve Sonraki Adımlar

### Node'u Çalışır Tutmak

Node'u sync bittikten sonra bir süre açık tutmak faydalı:

- Ekip evidence'ı incelerken erişebilir durumda olsun
- Başka kanıt isterlerse hazır
- Testnet operator kit yayımlandığında aynı sunucuyu reuse edebilirsin (ama Testnet için **yeni paket, yeni spec, yeni data dir** gerekir)

**Pratik komutlar:**

```sh
cd ~/flop-devnet-rpc
C="docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml"

# Durum
$C ps
$C logs --tail=50 rpc

# Ready endpoint
curl -s http://127.0.0.1:9619/ready | jq

# Peer sayısı hızlı kontrol
watch -n 10 'curl -s http://127.0.0.1:9619/ready | jq ".peers"'

# Yeniden başlat (gerekirse)
$C restart rpc ready

# Durdur (volume korumalı)
$C down

# Tamamen sil (TEHLİKELİ, chain verisi gider)
$C down -v
```

### Disk Kullanımı Takibi

Archive sürekli büyüyor. Haftada bir kontrol:

```sh
df -h /
du -sh /var/lib/docker/volumes/flop-devnet-rpc-pool_rpc-data
```

50 GB altına inânâ dikkat et, disk dolmadan sağlayıcıdan genişletme iste.

### Node Key Yedekleme ve Geri Yükleme

Node key (`secret_ed25519`) sunucunun peer ID'sini belirler. Bu dosya kaybolursa veya değişirse peer ID de değişir ve restart continuity evidence'ın geçersiz olur. Sunucuyu kaybetme ihtimaline karşı mutlaka yedekle.

**Yedekleme:**

```sh
docker compose stop rpc
cp ~/flop-devnet-rpc/data/chains/flop_devnet_ts/network/secret_ed25519 ~/node-key-backup
chmod 600 ~/node-key-backup
docker compose start rpc
```

Yerel makinene indir (SSH'tan çıkıp kendi bilgisayarında çalıştır):

```sh
scp root@<sunucu-ip>:~/node-key-backup ~/flop-node-key-backup
```

Dosyayı şifreli bir yerde (password manager, şifreli USB) sakla. Bu dosya peer ID'nin özel anahtarıdır — **asla paylaşma, repo'ya ya da GitHub issue'ya yapıştırma.**

**Geri yükleme (yeni sunucuda):**

Adım 1-6'yı normal şekilde uygula, ama **Adım 7'den (compose up) önce** node key'i yerleştir:

```sh
mkdir -p ~/flop-devnet-rpc/data/chains/flop_devnet_ts/network/
cp ~/node-key-backup ~/flop-devnet-rpc/data/chains/flop_devnet_ts/network/secret_ed25519
chmod 600 ~/flop-devnet-rpc/data/chains/flop_devnet_ts/network/secret_ed25519
```

Sonra `docker compose up -d` ile başlat. Peer ID'nin aynı kaldığını doğrula:

```sh
docker compose logs rpc 2>&1 | grep "Local node identity"
```

Çıktıdaki `12D3KooW...` değeri eski peer ID'nle birebir aynı olmalı. Farklıysa node key doğru yere kopyalanmamış demektir.

### Node Kapatmak

Çalışırken başka bir sunucuya geçersen eski node'u düzgün kapat:

```sh
cd ~/flop-devnet-rpc
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml down

# Veriyi tutmak istiyorsan (ileride tekrar açabilirsin):
# Hiçbir şey yapma, volume'da kalır

# Tamamen sil:
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml down -v
```

### Testnet'e Geçiş

FLOP ekibi **testnet operator kit** yayımladığında (muhtemelen bu hafta veya haftaya):

1. Bu Devnet node'unu durdurabilirsin (evidence zaten verildi)
2. Yeni paket Testnet için olacak, farklı chain spec ve farklı bootnode'larla
3. **Aynı sunucuda ayrı dizin** ile Testnet node'u kurabilirsin (örn. `~/flop-testnet-rpc/`)
4. Peer ID ayrı, keystore ayrı — karışıklık yok
5. Testnet keyless RPC evidence'ını başvuru issue'na yorum olarak ekle

### Sonraki Milestone

Validator promote edilme adımları:

1. Devnet keyless RPC (bu kılavuz) → kanıt üretildi, başvuruya eklendi ✅
2. Testnet keyless RPC (operator kit cıkınca) → aynı süreç Testnet'te
3. **Signed validator kit** (FLOP ekibi davet gönderdiğinde) → validator anahtarı, kayit, stake
4. Testnet validator aktifleşmesi → 14 gün grace period'da sıralama hesabına girmek

### Resmi Kaynaklar

- **Devnet RPC guide:** `https://gist.github.com/sv/7c8b93f3868bc15e3b6eabe0578057e3`
- **Public Devnet RPC endpoint:** `https://rpc.devnet.flop.network`
- **Testnet durum:** `https://docs.flop.network/networks/testnet`
- **Validator hazırlık:** `https://docs.flop.network/testnet-validator-prep/`
- **Operator başvuru formu:** `https://github.com/flop-labs/network/issues/new?template=operator-application.yml`
- **Resmi doğrulama:** `https://flop.finance/official/`
- **Resmi bot:** `https://ask.flop.finance`
- **GitHub repo:** `https://github.com/flop-labs/network`
