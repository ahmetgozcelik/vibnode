# vibnode — Proje bilgisi

## Bu repo nasıl yürütülür
Bu proje **mentor modunda** yürütülür: kodu kullanıcı yazar, Claude öğretir, yönlendirir ve inceler.
Davranış kuralları `.claude/output-styles/mentor.md` dosyasındadır (output style: Mentor).

Her oturumun başında:
1. `PROGRESS.md` dosyasını oku ve kaldığımız yeri iki cümleyle özetle.
2. Sıradaki görevi söyle ve ilgili konu kartını (K kodu) belirt.

## Proje
STM32F407 Discovery (MB997D) üzerinde FreeRTOS tabanlı titreşim izleme node'u, ESP32 CAN–MQTT gateway'i
ve CAN üzerinden firmware güncelleme (bootloader).

- Plan ve öğrenme rehberi: `docs/plan/Titresim_Izleme_Node_Proje_Plani.pdf`
  (Bölüm 7 teknik tasarım, Bölüm 10 haftalık plan ve konu kartları, Ek A cevap anahtarı)
- Teknik sözleşme: `SPEC.md` (plan dokümanının Bölüm 7'sinden oluşturulur)
- CAN mesajları: `dbc/vibnode.dbc` (tek doğruluk kaynağı)
- İlerleme: `PROGRESS.md`
- Öğrenme notları: `docs/learning-notes.md`

## Donanım
- STM32F407 Discovery MB997D, kart üstü LIS3DSH ivmeölçer
- ESP32 geliştirme kartı
- 2 × SN65HVD230 CAN transceiver (3,3 V), hat uçlarında 120 Ω
- Arctic P8 PWM PST fan (200–3000 RPM), 12 V 2 A adaptör
- 24 MHz 8 kanal lojik analizör (PulseView)

## Komutlar (kurulum ilerledikçe güncellenecek)
- Host testleri : `cmake -S tests -B build/tests && ctest --test-dir build/tests`
- STM32 derleme : `cd firmware/app && cmake --preset Debug && cmake --build --preset Debug`
- STM32 flash   : `STM32_Programmer_CLI -c port=SWD -w <elf> -v -rst`
- ESP32         : `idf.py -C gateway build flash monitor`
- DBC'den kod   : `cd common/can && python -m cantools generate_c_source ../../dbc/vibnode.dbc`

## Teknik kurallar
1. DBC, pinler, clock, CAN bit timing ve flash haritası kullanıcının onayı olmadan değişmez.
2. CubeMX dosyalarında kod sadece USER CODE BEGIN/END blokları içine yazılır; uygulama kodu `firmware/app/app/` altındaki ayrı dosyalarda durur.
3. `common/` saf C'dir: HAL veya FreeRTOS başlığı içeremez; her fonksiyonun host testi olur.
4. Kesme içinde sadece ...FromISR API'leri kullanılır. CAN'a sadece CanTxTask yazar.
5. Her değişiklikten sonra ilgili derleme ve testler çalıştırılır; kırmızıysa iş bitmemiştir.
6. Donanım davranışı tahmin edilmez; emin olunmayan her şey için kullanıcıdan ölçüm istenir.
