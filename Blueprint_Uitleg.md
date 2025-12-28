# Uitgebreide Uitleg van de Home Assistant Blueprint

## Introductie
Deze blueprint is ontworpen om een **thuisbatterij slim aan te sturen** op basis van **dynamische energieprijzen**, **prijsvoorspellingen**, **SOC (State of Charge)**, en extra condities zoals **verbruikersblokkering** en **anti-rondpompen**. Het doel is om **kosten te minimaliseren**, **efficiënt te laden/ontladen**, en **conflicten met grote verbruikers (zoals EV-laders)** te voorkomen.

---

## Hoofdfunctionaliteiten
- **Moduswissel op basis van prijsdrempel**
- **Categoriegestuurde ontlading**
- **Dynamische reserve-SOC op basis van forecast**
- **Anti-rondpompen (optioneel)**
- **Verbruik-blokker (optioneel)**

---

## Details per functionaliteit
### 1. Moduswissel op basis van prijsdrempel
- **Prijs < drempel (standaard €0,20/kWh)**: dynamic_charging → ontladen uit, laden op ingesteld vermogen (standaard 8.000 W)
- **Prijs ≥ drempel**: home_optimization → ontladen volgens categorie en SOC-regels

### 2. Categoriegestuurde ontlading
- **HIGH**: Hoog vermogen (standaard 6.000 W)
- **MEDIUM**: Matig vermogen (standaard 1.500 W)
- **LOW**: Alleen ontladen als SOC ≥ drempel (standaard 80%), beperkt vermogen (1.000 W)

### 3. Dynamische reserve-SOC op basis van forecast
- Voorspellingen voor komende 8 uur verhogen SOC-reserve afhankelijk van HIGH/MEDIUM in forecast:
  - HIGH binnen 0–2 uur → reserve naar 55%
  - HIGH binnen 2–4 uur → reserve naar 45%
  - MEDIUM binnen 0–2 uur → reserve naar 40%
  - HIGH binnen 4–8 uur → reserve naar 40%
  - MEDIUM binnen 4–8 uur → reserve naar 35%

### 4. Anti-rondpompen
- Ontladen alleen toegestaan als net-/huislast ≥ ingestelde drempel (standaard 300 W)

### 5. Verbruik-blokker
- Als een opgegeven sensor (bijv. EV-laadpaal) boven ingestelde drempel komt, wordt ontladen geblokkeerd.

---

## Instelbare parameters
- **Prijsdrempel**: standaard €0,20/kWh
- **Vermogens**:
  - HIGH: 6.000 W
  - MEDIUM: 1.500 W
  - LOW: 1.000 W
  - Laden bij goedkoop: 8.000 W
- **SOC-drempels**:
  - Minimale reserve: 35%
  - LOW-categorie ontlading: vanaf 80%
- **Forecast-reserves**: instelbaar per tijdsvenster
- **Anti-rondpompen**: drempel standaard 300 W
- **Verbruik-blokker**: sensor + drempel (bijv. EV-laadpaal)

---

## Triggers
- Wijzigingen in:
  - Huidige prijs
  - Prijscategorie
  - SOC
  - Forecast-sensoren (+1 t/m +8 uur)
  - Anti-rondpompen sensor
  - EV-blokker sensor
- Plus een **periodieke trigger** elke 5 minuten

---

## Acties
- **Selecteer modus** (`dynamic_charging` of `home_optimization`)
- **Stel laad-/ontlaadvermogen in** via `number.*` entiteiten
- Houd rekening met:
  - SOC-reserve
  - Forecast
  - Anti-rondpompen
  - EV-blokker

---

## Waarom deze blueprint interessant is
- Combineert **prijsoptimalisatie**, **voorspellende logica**, en **praktische beperkingen**
- Volledig **instelbaar via UI** (geen code nodig)
- **Modulair**: functies zoals anti-rondpompen en EV-blokker zijn optioneel
- **Uitbreidbaar**: makkelijk om extra condities toe te voegen

---

## Suggesties voor toekomstige uitbreidingen
- PV-integratie: laad extra bij overschot zonne-energie
- Meerdere blokkers: bijvoorbeeld warmtepomp + EV tegelijk
- Slimme profielen: weekendmodus, nachtmodus
- Notificaties: meldingen bij moduswissel of blokkering
- Maximale cycli per dag: voor batterijlevensduur

Er zijn meerdere condities waarin ontladen wordt geblokkeerd:

Prijs < prijsdrempel
→ Ontladen uit (zie hierboven).

SOC ≤ dynamische reserve-SOC
→ Als de batterij onder de berekende reserve zit (bijv. 35%, of hoger bij forecast), wordt niet ontladen.

EV-blokker actief
→ Als de opgegeven sensor (bijv.  sensor.charge_laadstation_kwh_charge_point_power ) ≥ ingestelde drempel (bijv. 2.000 W), wordt ontladen geblokkeerd.


Anti-rondpompen
→ Als de net-/huislastsensor ( sensor.connect_energiemeter_electricity_consumption ) lager is dan de ingestelde drempel (standaard 300 W), wordt ontladen niet toegestaan.

LOW-categorie en SOC < LOW-drempel
→ Bij prijscategorie LOW mag alleen ontladen als SOC ≥ LOW-floor (standaard 80%). Anders ontladen = 0.

## Toevoeging real time situatie


# Uitleg: Wat doet de Zonneplan Thuisbatterij Blueprint?  
Deze uitleg hoort bij de blueprint en beschrijft exact wat je kunt verwachten op basis van de variabelen die door de automation worden berekend.

---

## Inkomende waarden (voorbeeld)
Uit de automation-trace kwamen de volgende waarden:

```
price: 0.2465149
cat: low
soc: 45
fc:
  - normal
  - high
  - high
  - high
  - high
  - normal
  - normal
  - low
```

Deze waarden bepalen hoe de blueprint verder rekent.

---

## 1) Prijsbeslissing
De blueprint vergelijkt de huidige prijs (`price`) met de ingestelde drempel `price_floor` (standaard **0.20 €/kWh**):

- **Als price < 0.20** → *goedkope uren* → modus **dynamic_charging**
- **Als price ≥ 0.20** → *duurdere uren* → modus **home_optimization**

In het voorbeeld:  
**0.2465 ≥ 0.20 → home_optimization-route.**

---

## 2) Dynamische reserve (dyn_reserve)
De blueprint bepaalt altijd eerst wat de minimale SOC moet zijn op basis van de **komende 8 uur**.

De standaard-minimumreserve is:  
**`soc_reserve_min = 35%`**

Daarbovenop komen extra verhogingen afhankelijk van de forecast `fc`:

| Periode | Wat checkt de blueprint? | Effect |
|--------|---------------------------|--------|
| 0–2 uur | Zit er een `high`? | dyn_reserve wordt **55%** |
| 2–4 uur | Zit er een `high`? | dyn_reserve wordt **45%** |
| 0–2 uur | Zit er een `normal`? | dyn_reserve wordt **40%** |
| 4–8 uur | Zit er `high`? | dyn_reserve wordt **40%** |
| 4–8 uur | Zit er `normal`? | dyn_reserve wordt **35%** (blijft basis) |

In het voorbeeld:
- In **0–2 uur** zit `high` (fc[1] = high) → dyn_reserve = **55%**.

**Resultaat:** de batterij moet minimaal 55% hebben om te mogen ontladen.

---

## 3) Blokkerlogica
De blueprint heeft twee optionele blokkers:

### 3.1 EV-blokker
- Als deze actief is én het EV-verbruik boven drempel ligt → **ontladen = 0 W**.

### 3.2 Anti-rondpompen
- Als de huishoudlast < ingestelde drempel → **ontladen = 0 W**.

In het voorbeeld waren deze niet actief.

---

## 4) Ontlaadbeslissing
De blueprint beslist nu of ontladen **mag**, en zo ja, hoeveel vermogen.

### 4.1 Eerste harde stop
```
Als soc ≤ dyn_reserve → ontladen = 0 W
```

Voorbeeld:  
`soc = 45%` en `dyn_reserve = 55%` → **45 ≤ 55 → ontladen = 0 W**.

### 4.2 Categoriegestuurde ontlading
Deze stap wordt alleen uitgevoerd **als ontladen überhaupt is toegestaan**.  
Dus in het voorbeeld **wordt deze stap overgeslagen**.

Normale logica is:
- `cat == high`  → power_high (6000 W)
- `cat == normal` → power_medium (1500 W)
- `cat == low`    → alleen bij **SOC ≥ soc_low_floor** (80%), anders 0 W

Omdat je SOC slechts 45% is, zou `low` ook **0 W** geven.

---

## 5) Laadbeslissing
Laadgedrag hangt af van de prijsroute:
- In **dynamic_charging** (goedkope uren): laadvermogen wordt gezet naar `power_charge_when_cheap` (8000 W standaard).
- In **home_optimization** (duurdere uren): er wordt **geen laadvermogen gezet**, de batterij blijft zoals hij is.

In dit voorbeeld ben je in **home_optimization**, dus:
- **niet laden** (tenzij je eerdere instelling zo stond)
- **niet ontladen** (door SOC ≤ dyn_reserve)

---

## 6) Eindresultaat voor jouw voorbeeld
Met de waarden die je gaf doet de blueprint het volgende:

- **select_mode → home_optimization**
- **discharge_power → 0 W** (SOC te laag, dyn_reserve 55%)
- **charge_power → geen wijziging** (home_optimization verandert dit niet)

De batterij blijft dus **stil**, totdat:
- prijs weer goedkoop wordt (dynamic_charging), of
- SOC genoeg stijgt om **boven dyn_reserve** te komen, of
- forecast verandert waardoor dyn_reserve daalt.

---

## 7) Wanneer gaat de batterij wél ontladen?
Ontladen gebeurt alleen als **alle** voorwaarden waar zijn:
1. prijs ≥ drempel (duurdere uren), én
2. geen EV-blokker actief, én
3. huislast ≥ anti-rondpompen-drempel, én
4. SOC > dyn_reserve, én
5. afhankelijk van categorie:
   - high → power_high
   - normal → power_medium
   - low → **alleen** bij SOC ≥ soc_low_floor

---

## 8) Wanneer gaat de batterij laden?
**Alleen** bij goedkope uren:
- prijs < drempel → dynamic_charging
- ontladen = 0 W
- laden = `power_charge_when_cheap`

---

# Kortom
Deze blueprint is ontworpen om:
- slim te laden bij goedkope prijs,
- extra reserve op te bouwen bij dure forecast,
- ontladen toe te staan wanneer er geld te besparen valt,
- en tegelijk EV-laden & rondpompen te voorkomen.
