# SPEC.md — Teknik Sözleşme

> Bu dosya `docs/plan/Titresim_Izleme_Node_Proje_Plani.pdf` Bölüm 7'den birebir aktarılmıştır.
> Buradaki değerler (DBC, pinler, clock, CAN bit timing, flash haritası) **kullanıcının onayı olmadan değiştirilmez.**
> Değişiklik gerekirse önce bu dosya güncellenir, sonra koda yansıtılır.

## 1. Genel CAN parametreleri

| Parametre | Değer |
|---|---|
| Bit hızı ve format | 500 kbit/s, 11 bit standart ID, klasik CAN (CAN FD yok) |
| Bayt sırası | Little-endian (Intel), DBC'de tanımlı |
| STM32 bit timing | SYSCLK 168 MHz (HSE 8 MHz), APB1 42 MHz: Prescaler 6, BS1 = 11 tq, BS2 = 2 tq, SJW = 1 → 500 kbit/s, örnekleme noktası %85,7. Automatic Bus-Off Management: Enable. Automatic Retransmission: Enable. |
| ESP32 bit timing | 500 kbit/s. ESP-IDF v5.5+ yeni TWAI sürücüsünde bit hızı doğrudan sayı; eski sürücüde `TWAI_TIMING_CONFIG_500KBITS()`. Kullanılan ESP-IDF sürümünün dokümantasyonuna bak. |
| Heartbeat | Her node 500 ms'de bir; 1,5 s (3 kayıp) gelmezse karşı taraf "offline" kabul edilir |
| Kayıp takibi | 0x100 / 0x101'deki `Seq` sayacı; gateway atlanan değerleri sayar ve raporlar |

## 2. CAN mesaj sözlüğü (DBC)

Tek doğruluk kaynağı: [`dbc/vibnode.dbc`](dbc/vibnode.dbc). Düşük ID = yüksek öncelik; alarm en düşük ID'yi alır.

| ID | Mesaj | Yön | DLC | Periyot | Sinyaller |
|---|---|---|---|---|---|
| 0x080 | VIB_ALARM | Node → GW | 4 | Durum değişince; aktifken 1 s'de bir | B0 Level u8 (0 NORMAL, 1 UYARI, 2 ALARM) · B1 Cause u8 bit maskesi (b0 RMS, b1 1X genlik, b2 crest) · B2–3 ZScore i16 ×0,01 |
| 0x100 | VIB_FEATURES | Node → GW | 8 | Her pencere (~320 ms) | B0–1 Rms u16 mg · B2–3 Peak u16 mg · B4–5 Crest u16 ×0,001 · B6 Seq u8 · B7 Flags u8 (b0 baseline hazır, b1 sensör OK) |
| 0x101 | VIB_SPECTRUM | Node → GW | 8 | Her pencere | B0–1 DomFreq u16 ×0,01 Hz · B2–3 DomAmp u16 mg · B4–5 FanRpm u16 · B6 Seq u8 · B7 ayrılmış |
| 0x200 | NODE_CMD | GW → Node | 4 | İstek anında | B0 CmdId u8 (1 uyarı eşiği, 2 alarm eşiği, 3 fan duty, 4 baseline'ı yeniden öğren, 5 bootloader'a geç) · B1–2 Param u16 (eşik ×0,01 / duty %) · B3 Token u8 |
| 0x201 | NODE_CMD_ACK | Node → GW | 4 | Komuttan sonra | B0 CmdId u8 · B1 Token u8 · B2 Result u8 (0 OK, 1 geçersiz parametre, 2 meşgul) · B3 ayrılmış |
| 0x701 | HB_NODE | Node → GW | 8 | 500 ms | B0 State u8 (0 INIT, 1 LEARNING, 2 RUNNING, 3 FAULT) · B1 FwMajor · B2 FwMinor · B3 TEC · B4 REC · B5 ResetCause u8 (0 POR, 1 IWDG, 2 yazılım, 3 pin) · B6–7 Uptime u16 s |
| 0x702 | HB_GATEWAY | GW → Node | 4 | 500 ms | B0 Flags u8 (b0 WiFi, b1 MQTT) · B1 FwMajor · B2 FwMinor · B3 ayrılmış |
| 0x7E0 | BL_CMD | GW → BL | 8 | Güncelleme sırasında | B0 Op u8 (1 START, 3 END, 4 ABORT, 5 QUERY) · START: B1–3 Size u24, B4–5 Version u16 · END: B1–4 Crc32 u32 |
| 0x7E1 | BL_DATA | GW → BL | 8 | Güncelleme sırasında | B0–1 Seq u16 · B2–7 imajın 6 baytı (ofset = Seq × 6; en fazla 384 KB imaj) |
| 0x7E8 | BL_RESP | BL → GW | 8 | Her cevap; beklemedeyken 500 ms | B0 Status u8 (0 READY, 1 BUSY, 2 ACK, 3 OK, 0x10+ hata) · B1–2 Seq u16 · B3 hata detayı · B4–5 bootloader sürümü |

## 3. STM32 firmware mimarisi (FreeRTOS)

| Task | Öncelik | Tetikleyici | Görevi |
|---|---|---|---|
| AccelTask | Yüksek | İvmeölçer veri hazır kesmesi (INT1) | Örneği SPI'dan okur, ping-pong tampona yazar; pencere dolunca DspTask'a bildirim gönderir. |
| DspTask | Normal | AccelTask bildirimi | DC çıkarma, Hann penceresi, FFT, özellikler, anomali; CAN çerçevelerini CanTx kuyruğuna koyar. |
| CanTxTask | Normal+1 | Kuyruk | `HAL_CAN_AddTxMessage`'ı çağıran tek yer; mailbox doluysa bekler. |
| CanRxTask | Normal+1 | RX FIFO0 kesmesi → kuyruk | Komutları çözer ve uygular, ACK gönderir; gateway heartbeat'ini izler. |
| FanTask | Düşük | 100 ms | PWM duty'yi uygular; tach periyodundan RPM hesaplar (filtreli). |
| HealthTask | Düşük | 500 ms | Heartbeat ve LED'ler. Tüm task'lar "yaşıyorum" bildirdiyse IWDG'yi besler. |

Kurallar:
- Kesme içinde sadece `...FromISR` API'leri kullanılır. FreeRTOS çağıran kesmelerin NVIC önceliği sayısal olarak ≥ 5 olmalı (`configMAX_SYSCALL_INTERRUPT_PRIORITY`).
- FreeRTOS SysTick'i kullandığı için HAL timebase **TIM6**'ya taşınır.
- bxCAN'da en az bir filtre bankası yapılandırılmadan hiçbir mesaj alınmaz; başlangıçta "hepsini kabul et" filtresi kurulur.
- UART logları tek bir sahip üzerinden (kuyrukla) yazılır; task'lar doğrudan `printf` çağırmaz.
- IWDG zaman aşımı ≈ 2 s. Reset sebebi açılışta RCC bayraklarından okunup heartbeat'e konur.

## 4. Sinyal işleme

| Parametre | Değer (LIS3DSH, kart üstü) |
|---|---|
| Örnekleme hızı (ODR) | 1600 Hz |
| FFT boyu | 1024 nokta (CMSIS-DSP `arm_rfft_fast_f32`) |
| Pencere süresi / çözünürlük | 0,64 s / 1,5625 Hz |
| Nyquist frekansı | 800 Hz |
| Güncelleme aralığı | ~320 ms (%50 örtüşme) |
| Ölçek | ±2 g |

- **RMS:** pencerenin DC'si çıkarılmış sinyalinin karekök ortalaması (mg).
- **Tepe (peak):** penceredeki mutlak en büyük değer (mg). **Crest factor:** tepe / RMS.
- **Baskın frekans:** DC hariç en büyük FFT tepesi; komşu bin'lerle parabolik interpolasyon yapılır.
- **1X genlik:** tach'tan bulunan dönme frekansındaki FFT genliği. Dönme frekansı = RPM / 60 = tach frekansı / 2.
- **Eksen seçimi:** öğrenme başında en yüksek RMS'li eksen seçilip sabitlenir.
- Arctic P8 PWM PST 200–3000 RPM aralığında döner; 3000 RPM'de 1X = 50 Hz.
- DSP süresi DWT cycle counter ile ölçülür ve README'ye yazılır.

## 5. Anomali tespiti

1. **Öğrenme:** ilk 30 saniyede (~90 pencere) RMS ve 1X genliğinin ortalaması (μ) ve standart sapması (σ) çıkarılır. σ'ya bir alt sınır uygulanır (ör. 2 mg).
2. **Çalışma:** her pencerede z = (x − μ) / σ hesaplanır; iki özellikten büyük olan kullanılır.
3. **UYARI:** art arda 3 pencere z ≥ 3. **ALARM:** art arda 3 pencere z ≥ 6.
4. **Normale dönüş:** art arda 5 pencere z < 2 (histerezis).
5. Fan hızı değiştiğinde (komut 3) veya komut 4 geldiğinde baseline yeniden öğrenilir. Eşikler komut 1 ve 2 ile değiştirilebilir.

## 6. ESP32 gateway

ESP-IDF v5.x. Task'lar: CAN alma, MQTT'ye yayın, MQTT komutlarını CAN'a iletme, heartbeat ve izleme, firmware güncelleme (HTTP sunucusu + bootloader durum makinesi). CAN çözme/kodlama, DBC'den cantools ile üretilen ortak C koduyla yapılır; STM32 ile aynı dosyalar kullanılır.

| MQTT konusu | Yön | İçerik (JSON) |
|---|---|---|
| `vibnode/1/features` | GW → broker | rms_mg, peak_mg, crest, seq, ts |
| `vibnode/1/spectrum` | GW → broker | dom_freq_hz, dom_amp_mg, fan_rpm, seq, ts |
| `vibnode/1/alarm` | GW → broker | level, cause, z, ts |
| `vibnode/1/status` | GW → broker (retained) | online, state, fw, tec, rec, reset_cause, uptime_s, lost_frames |
| `gateway/status` | GW → broker (retained, LWT) | online, wifi_rssi, fw, can_state |
| `vibnode/1/cmd` | broker → GW | Ör. `{"cmd": "set_fan_duty", "value": 60}` |
| `vibnode/1/cmd_ack` | GW → broker | cmd, token, result |
| `vibnode/1/fw` | GW → broker | state, progress_pct, error |

- `partitions.csv` içinde yüklenen STM32 firmware'i için 1 MB'lık bir `stm_fw` veri bölümü tanımlanır.
- TWAI hata durumları izlenir; bus-off durumunda sürücünün kurtarma fonksiyonuyla otomatik toparlanır.
- MQTT bağlantısı yokken son ~500 mesaj RAM'deki halka tamponda tutulur.
- WiFi bilgileri ve broker adresi menuconfig/NVS'de tutulur, koda gömülmez. Zaman damgası SNTP ile.

## 7. PC tarafı (Docker)

| Servis | İmaj | Görev |
|---|---|---|
| Mosquitto | `eclipse-mosquitto:2` | MQTT broker. `listener 1883`, `allow_anonymous true` (sadece lab ağında). |
| Telegraf | `telegraf` | `mqtt_consumer` + `json_v2` ayrıştırıcı → InfluxDB |
| InfluxDB | `influxdb:2` | Zaman serisi veritabanı (bucket: `vibnode`) |
| Grafana | `grafana/grafana` | Veri kaynağı ve dashboard JSON'u repo'dan provisioning ile otomatik yüklenir. |

## 8. Bootloader ve uzaktan güncelleme

| Bölge | Adres aralığı | Flash sektörü | Boyut |
|---|---|---|---|
| Bootloader | 0x0800 0000 – 0x0800 7FFF | 0–1 | 32 KB |
| Metadata | 0x0800 8000 – 0x0800 BFFF | 2 | 16 KB (magic, boyut, CRC32, sürüm) |
| Uygulama | 0x0800 C000 – 0x080F FFFF | 3–11 | 976 KB |

**Açılışta karar sırası**
1. RTC backup register'da "güncelleme isteği" işareti varsa → güncelleme modu (işaret temizlenir).
2. Kullanıcı butonu (PA0) basılıysa → güncelleme modu (kurtarma yolu).
3. Metadata geçerli, uygulamanın CRC32'si tutuyor ve ilk kelime (stack pointer) SRAM aralığında (0x2000 0000 – 0x2002 0000) ise → uygulamaya atla.
4. Hiçbiri değilse → güncelleme modu.

**Güncelleme akışı**
1. Gateway `NODE_CMD 5` gönderir → uygulama backup register'a işaret yazar ve yazılımsal reset atar.
2. Bootloader güncelleme moduna girer, 500 ms'de bir `BL_RESP READY` yayınlar.
3. START (boyut, sürüm) → metadata ve gereken uygulama sektörleri silinir (önce BUSY, sonra READY). 128 KB'lık sektör silme 1–2 s sürer; gateway'de START zaman aşımı 10 s olmalı.
4. BL_DATA (seq, 6 bayt) → flash'a yazılır → ACK (seq). Dur-bekle yöntemi: gateway her ACK'i bekler; 100 ms zaman aşımı, 3 tekrar.
5. END (crc32) → flash'taki imajın CRC'si hesaplanır. Tutarsa metadata yazılır, OK döner, reset → yeni uygulama açılır. Tutmazsa hata kodu döner, bootloader'da kalınır.

Metadata START'ta silinir ve sadece doğrulama başarılı olduktan sonra yazılır; akışın herhangi bir yerinde güç kesilirse kart açılışta bootloader'da kalır ve yeniden güncellenebilir.

**Hata kodları:** 0x10 CRC uyuşmazlığı · 0x11 sıra hatası · 0x12 boyut hatası · 0x13 flash silme/yazma hatası · 0x14 zaman aşımı.

**Atlama sırası**
Kullanılan periferikleri kapat (CAN, GPIO, RCC) → SysTick'i durdur → NVIC'te tüm kesmeleri kapat ve bekleyenleri temizle → `SCB→VTOR = 0x0800C000` → MSP'yi uygulamanın stack değerine ayarla → kesmeleri (PRIMASK) geri aç → uygulamanın reset vektörüne atla.

**Uygulama tarafında gerekenler**
- Linker script'te `FLASH ORIGIN = 0x0800C000, LENGTH = 976K`.
- `system_stm32f4xx.c`'de `USER_VECT_TAB_ADDRESS` tanımlanır, `VECT_TAB_OFFSET = 0xC000`.
- Derleme sonrası `.bin` üretilir; `tools/fw_pack.py` boyutu ve CRC32'yi hesaplar.
- **Hafta 1–3 boyunca uygulama normal şekilde 0x08000000'de çalışır.** Taşıma sadece Hafta 4'te yapılır.
- Geliştirme kolaylığı: debug derlemesinde `BL_DEV_ALLOW_UNVERIFIED` bayrağı, metadata geçersiz olsa da stack pointer geçerliyse uygulamaya atlar. Release derlemesinde kapalıdır.
- CRC32 yazılımla (zlib uyumlu) hesaplanır. F4'ün donanım CRC birimi farklı bir varyant kullandığı için ESP32 ve Python ile aynı sonucu vermez.

## 9. Pin haritası (özet — ayrıntı için plan Bölüm 5)

| Sinyal | STM32 pini | Not |
|---|---|---|
| CAN1_RX | PD0 | AF9 |
| CAN1_TX | PD1 | AF9 |
| USART2_TX/RX | PA2/PA3 | 115200 8N1 |
| FAN_PWM | PE9 (TIM1_CH1) | 25 kHz, open-drain |
| FAN_TACH | PB4 (TIM3_CH1) | Input capture, tur başına 2 darbe |
| İvmeölçer | SPI1: PA5/PA6/PA7, CS: PE3, INT1: PE0 | Kart üstü LIS3DSH |
| Kullanıcı butonu | PA0 | Açılışta basılıysa bootloader kurtarma modu |

| Sinyal | ESP32 pini |
|---|---|
| TWAI_TX | GPIO21 |
| TWAI_RX | GPIO22 |
