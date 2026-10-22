# Gree "I Feel" hőmérséklet-küldő (ESP32-C3 + ESPHome)

[English version](README.en.md)

Egy kis ESP32-C3 alapú eszköz, ami a Home Assistantban lévő szobahőmérséklet-szenzor értékét infravörösen elküldi egy Gree klímának, **pontosan úgy, ahogy a távirányító "I Feel" funkciója teszi**. Így a klíma nem a magasan, a készülékházban lévő saját érzékelője alapján szabályoz, hanem egy általad választott (például alacsonyan, a szoba közepén elhelyezett) szenzor alapján.

Miért jó ez?
- Az inverteres klíma nem kapcsolgat ki-be a készülék melletti, kifújt levegő által torzított mérés miatt.
- Kisebb a hőingadozás, jobb a komfortérzet, és kevésbé "pattog" a klíma műanyag burkolata.
- A szabályozáshoz több szoba átlaga is használható (lásd lent).

> **Fontos:** a projekt a Gree **YBEF1FB6** távirányítóval és két Gree klímával (Amber Royale 3,5 kW, Smart One 5,3 kW) lett kipróbálva. A protokollt a távirányító kereteinek rögzítésével derítettem ki, nem hivatalos dokumentációból. Más típusnál eltérhet. Saját felelősségre használd.

## Hogyan működik

A Gree távirányító "I Feel" üzemmódban minden gombnyomás után, és enélkül is 10 percenként kiküld egy rövid, **külön IR keretet**, amiben a mért hőmérséklet van. A klíma ezt a keretet önmagában is elfogadja, és a saját érzékelője helyett ezt használja.

A keret felépítése:

| Rész | Érték |
|---|---|
| Fejléc | ~6000 µs jel / ~3000 µs szünet |
| Adat | 16 bit, **LSB először**: `T` `0xA5` |
| `T` | a hőmérséklet egész °C-ban (pl. 24 → `0x18`) |
| `0xA5` | állandó bájt |
| Bitek | jel 630 µs; 0: szünet 575 µs, 1: szünet 1675 µs |
| Záró jel | 630 µs |
| Vivő | 38 kHz |

Az ESP az átvett hőmérséklet és az offset összegét egész számra kerekíti, majd 2 percenként elküldi. A klíma csak egész fokot fogad, a tizedeket a kerekítés elveszti.

## Hardver

- ESP32-C3 Super Mini (a YAML `esp32c3` variánsra van írva)
- IR LED (pl. TSAL6400 vagy bármilyen 940 nm-es adó LED)
- NPN tranzisztor a LED meghajtásához (pl. BC337)
- Ellenállások: 1 kΩ (GPIO → tranzisztor bázis), 33 Ω (LED soros ellenállás)

Bekötés:

```
GPIO5 ──[1 kΩ]── bázis (BC337)
                 kollektor ── LED katód
3V3 ──[33 Ω]── LED anód
                 emitter ── GND
```

(Az ellenállásokat a saját LED-edhez méretezd.)

A LED működését a telefon kamerájával ellenőrizheted: az IR fény a kamerán lilás fehér villanásként látszik. Ha nem villan, mérd át a LED-et közvetlenül a 3V3 és GND között egy soros ellenállással, mert az olcsó LED-ek között hibás darab is előfordul.

A LED a klíma felé nézzen. Két klímához két külön eszköz kell, egyenként a saját klímájával szemben, hacsak egy LED nem lát mindkettőt.

## Telepítés

1. Telepítsd az [ESPHome](https://esphome.io)-ot (dashboard vagy parancssor).
2. Másold le a `secrets.yaml.example` fájlt `secrets.yaml` néven, és töltsd ki (WiFi, API kulcs, AP jelszó). **A `secrets.yaml` nem kerülhet gitbe**, a `.gitignore` ezt kizárja.
3. Szerkeszd az `esp32-c3-ifeel.yaml` tetején a `substitutions` blokkot:
   - `name` és `friendly_name`: az eszköz neve,
   - `source_sensor`: a HA-beli szenzor entitása, amit a klímának küldesz.
4. Töltsd fel az ESP-re (első alkalommal USB-n, utána OTA-val).
5. A Home Assistantban az ESPHome integráció automatikusan megtalálja az eszközt.

Több klímához másold le az YAML-t, és minden példányban írd át a `substitutions` blokkot. Ha ugyanazt a `secrets.yaml`-t használod, az `api_key` kulcsot eszközönként külön névvel vedd fel (pl. `api_key_1`, `api_key_2`), és a YAML `api:` részében írd át ennek megfelelően.

## Használat

Először kapcsold be az "I Feel" funkciót a **távirányítón**, mert a klíma csak ilyenkor figyel a keretre (a projektet így teszteltem). Utána **vedd ki az elemeket a távirányítóból**, különben az 10 percenként felülírja az ESP értékét a saját hőmérsékletével.

A HA-ban az eszköz az alábbi entitásokat adja:

| Entitás | Mit csinál |
|---|---|
| `I Feel offset` | Hozzáadódik a mért értékhez, −10 és +10 °C között, 0,5-es lépésekben. Pozitív érték: a klíma melegebbnek hiszi a szobát. |
| `I Feel utoljára küldött` | A klímának utoljára küldött (egész) hőmérséklet. |
| `I Feel eltelt idő` | Az utolsó küldés óta eltelt percek. |
| `I Feel azonnali küldés` | Gomb: azonnal elküldi az aktuális értéket. |

Az ESP 90 másodperccel az indulás után küld először, utána 2 percenként. A logban ezt látod: `I Feel keret elküldve: 25 C`.

## Több szoba átlaga

Ha a klíma több összenyitott szobát fűt, érdemes a szenzorok **súlyozott átlagát** küldeni. A `homeassistant-templates.yaml` egy kész példát tartalmaz egy Home Assistant sablon szenzorra, ami:
- a nem elérhető szenzorokat kihagyja,
- a súlyokat a maradékra újraosztja,
- `unavailable` értéket ad, ha egyik szenzor sem ad számot (ilyenkor az ESP nem küld keretet).

Tippek a súlyozáshoz: a klíma saját terének szenzora kapjon nagyobb súlyt, a belső, záródó ajtós helyiségeket (WC, kamra) hagyd ki, a klíma közelében lévő szenzort pedig ne használd, mert ugyanazt a torzítást mutatja, mint a klíma saját érzékelője.

## Hibaelhárítás

- **`Auth Expired` a WiFi-nél:** az `output_power: 8.5dB` beállítás ezt megszüntette az ESP32-C3 Super Mini-nél. Ha a te példányod másképp viselkedik, kísérletezz az értékkel.
- **A klíma nem reagál:** ellenőrizd a LED-et telefonkamerával, és nézd meg, hogy az "I Feel" be van-e kapcsolva a távirányítón.
- **Nem küld keretet, és a logban `érvénytelen` figyelmeztetés van:** a forrásszenzor `unavailable` vagy 5 °C alatt/45 °C felett van.
- **Az ESPHome beépített `climate: gree` komponense nem használható erre:** nem támogatja az "I Feel" keretet, és a nálam lévő klímáknál a kimenete miatt a klíma újraindult.

## Ismert korlátok és nyitott kérdések

- A klíma egész fokot fogad; a távirányító kerekítési módját (kerekít vagy levág) nem mértem ki, az ESP kerekít.
- Nem tudom, mennyi ideig fogadja el a klíma az utolsó értéket, ha elmaradnak a keretek (a távirányító 10 percenként küld, az ESP 2 percenként).
- A 30 °C-ot a klíma nem fogadta el (a távirányítón a legnagyobb látott érték 29 volt), ezért a kód 5–45 °C-ra korlátoz.
- A projekt a Gree YBEF1FB6 távirányító kereteire épül; más modellnél a keretek eltérhetnek.

## Fájlok

| Fájl | Tartalom |
|---|---|
| `esp32-c3-ifeel.yaml` | Az ESPHome konfiguráció |
| `secrets.yaml.example` | A titkos értékek mintája (WiFi, API kulcs, AP jelszó) |
| `homeassistant-templates.yaml` | Példa HA sablon szenzor a súlyozott átlaghoz |
| `.gitignore` | Kizárja a `secrets.yaml` fájlt és a buildfájlokat |
| `README.en.md` | Az angol nyelvű leírás |
