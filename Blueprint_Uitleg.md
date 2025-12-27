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
