# VEVOR Weather Station 7-in-1 — Adaptation OpenMQTTGateway

## Objectif

Rendre le gateway M5Stack Core2 + module CC1101 (M5Stack M146, 855‑925 MHz)
capable de réceptionner la station météo **VEVOR 7-in-1 (YT60231, version UE 868 MHz)**,
puis de publier les mesures vers le broker MQTT (Home Assistant, etc.).

## Principe retenu

La station VEVOR émet en **2-FSK à ~868.31 / 868.38 MHz** (écart ~868.35 MHz).
Elle n'est donc PAS décodable par le gateway RF classique (RCSwitch/OOK).
On utilise la gateway **RTL_433** de OpenMQTTGateway, portée par la librairie
`rtl_433_ESP` (décodeur `Vevor-7in1`, protocole 263), en mode FSK sur CC1101.

## Modifications effectuées

### 1. Nouvel environnement PlatformIO (`environments.ini`)

Environnement ajouté (conservé tel quel l'ancien `esp32-m5stack-core2-cc1101` pour l'OOK) :

`[env:esp32-m5stack-core2-cc1101-vevor]`

- `<board>m5stack-core2</board>`
- `board_build.partitions = default_16MB.csv` — l'app image (RTL_433 agglutine tous les
  décodeurs) dépasse 2 Mo : la partition `min_spiffs.csv` est trop petite.
- `lib_deps` :
  - `m5unified`
  - `rtl_433_ESP` (librairie officielle, contient `src/rtl_433/devices/vevor_7in1.c`)
  - `smartrc-cc1101-driver-lib` (requis par le code OMG `commonRF.cpp` qui appelle
    `initCC1101()` avant la mise en route du récepteur RTL_433)

### 2. Flags de build

```ini
build_flags =
  ${com-esp32.build_flags}
  '-DvalueAsATopic=true'           ; topic MQTT avec modèle + id
  '-DGateway_Name="OMG_M5CORE2_CC1101_VEVOR"'
  '-DZboardM5STACK="M5Stack"'
  '-DZgatewayRTL_433="rtl_433"'    ; gateway RTL_433 (au lieu de ZgatewayRF)
  '-DZradioCC1101="CC1101"'
  '-DOOK_MODULATION=false'         ; mode FSK (obligatoire pour VEVOR)
  '-DRF_FREQUENCY=868.35f'         ; entre les deux porteuses ~868.31/868.38 MHz
  '-DRF_CC1101="CC1101"'
  '-DRF_MODULE_CS=25'
  '-DRF_MODULE_GDO0=13'            ; data RX
  '-DRF_MODULE_GDO2=35'
  '-DRF_MODULE_SCK=18'
  '-DRF_MODULE_MISO=38'
  '-DRF_MODULE_MOSI=23'
  '-DSIGNAL_RSSI=true'
  '-DNO_DEAF_WORKAROUND=true'
```

> `-DRF_FREQUENCY` est bien la fréquence utilisée par RTL_433 :
> `RFConfiguration.cpp` (reInit) -> `gatewayRTL_433.cpp` `enableRTLreceive()` ->
> `rtl_433.initReceiver(RF_MODULE_RECEIVER_GPIO, fréquence)`.
> Pour CC1101, `RF_MODULE_RECEIVER_GPIO` est `RF_MODULE_GDO0` (rtl_433_ESP.h).

### 3. Environnement par défaut (`platformio.ini`)

```ini
default_envs = esp32-m5stack-core2-cc1101-vevor
```

## Câblage / DIP switches (module M5Stack M146)

Le module ne choisi que les connexions **CSN / GDO0 / GDO2** (pas la fréquence) :

```
CSN  = GPIO 25
GDO2 = GPIO 35
GDO0 = GPIO 13
```

DIP switches (ON vers l'inscription "ON") :

```
Bloc CSN (1..4)        : 1 = ON, 2-4 = OFF   -> GPIO25
Bloc GDO2/GDO0 (1..6)  : 1 = ON, 5 = ON, sinon OFF -> GDO2=35, GDO0=13
```

SPI sur le M-Bus du Core2 : MOSI=23, MISO=38, SCK=18.
Antenne : 868 MHz (module 855‑925 MHz).

## Compilation / flash

```bash
pio run -e esp32-m5stack-core2-cc1101-vevor             # compiler
pio run -e esp32-m5stack-core2-cc1101-vevor -t upload    # flasher
pio device monitor                                       # console 115200 bauds
```

## Sortie MQTT attendue

Topic (grâce à `valueAsATopic=true`) :

```
home/OpenMQTTGateway/RTL_433toMQTT/Vevor-7in1/0/<id>
```

Message JSON :

```json
{
  "model": "Vevor-7in1",
  "id": 48399,
  "channel": 0,
  "battery_ok": 1,
  "temperature_C": 34.4,
  "humidity": 24,
  "wind_avg_km_h": 4.5,
  "wind_max_km_h": 9.3,
  "wind_dir_deg": 9,
  "rain_mm": 7.7,
  "uv": 6,
  "light_lux": 91780,
  "mic": "CHECKSUM",
  "protocol": "Vevor Wireless Weather Station 7-in-1",
  "rssi": -33
}
```

La station émet environ toutes les 20 secondes.

## Réglage fin si besoin

- Paquets manqués : essayer `-DRF_FREQUENCY=868.300f` (certains utilisateurs
  rapportent 868.307/868.376 MHz).
- Débug du signal : ajouter `-DDEMOD_DEBUG=true` (et/ou `-DRTL_DEBUG=4`,
  `-DAVERAGE_RSSI=5000`) pour voir la réception dans le monitor série.
- Changement de fréquence à chaud (sans recompiler), via MQTT :
  topic `home/OpenMQTTGateway/commands/MQTTtoRF/config`, payload `{"frequency":868.35}`.

## Informations / limites (issues communautaires)

- Le CC1101 est moins sensible qu'un RTL-SDR : on ne capte en général qu'une
  partie des paquets (~40 %), d'où des trous dans les relevés ; normal.
- Filtrage recommandé côté Home Assistant (template sensors) pour écarter les
  valeurs aberrantes (T > 60 °C, humidité > 100 %, etc.).
- rtl_433_ESP v0.6.2 (= celle référencée dans `platformio.ini`) inclut le décodeur
  `Vevor-7in1`. Ne pas la laisser être rétrogradée à une version antérieure.

## Références

- Allure de référence OMG : `[env:esp32dev-rtl_433-fsk]` (même mode FSK/CC1101).
- Décodeur rtl_433 : `rtl_433/src/devices/vevor_7in1.c`.
- Discussion d'aide à l'intégration : chatgpt.com/share/6aafea32-d1a4-83eb-b5e9-b13aba6a3aa3