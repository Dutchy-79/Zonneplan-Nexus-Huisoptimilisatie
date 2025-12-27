# Home Assistant Blueprint: Slimme Thuisbatterij Optimalisatie

## Introductie
Deze repository bevat een Home Assistant blueprint voor slimme aansturing van een thuisbatterij op basis van dynamische energieprijzen, forecast, SOC, en optionele functies zoals anti-rondpompen en verbruik-blokker (EV).

## Installatie
1. Kopieer `blueprints/zonneplan_battery_optimization_switching.yaml` naar `config/blueprints/automation/<jouw-map>/`.
2. Herlaad Blueprints in Home Assistant (Instellingen → Automatiseringen & Scènes → Blueprints → Herladen).
3. Maak een nieuwe automatisering op basis van de blueprint en koppel de entiteiten.

## Vereiste entiteiten of vergelijkbare sensoren
- `sensor.zonneplan_current_electricity_tariff`
- `sensor.zonneplan_current_tariff_group`
- `sensor.nexus_solarman_battery`
- `sensor.zonneplan_forecast_tariff_group_hour_1` … `_hour_8`
- `select.thuisbatterij_battery_control_mode`
- `number.thuisbatterij_max_charge_power_home_optimization`
- `number.thuisbatterij_max_discharge_power_home_optimization`

### Optioneel
- Anti-rondpompen: `sensor.connect_energiemeter_electricity_consumption`
- EV-blokker: `sensor.charge_laadstation_kwh_charge_point_power`

## Visuele Flowchart (Mermaid)
> GitHub ondersteunt Mermaid; de flow wordt direct gerenderd in de README.

```mermaid
flowchart TD
  A[Start / Trigger] --> B{Prijs < drempel?}
  B -- Ja --> C[Zet modus = dynamic_charging]
  C --> C1[Discharge = 0 W]
  C1 --> C2[Charge = power_charge_when_cheap]
  B -- Nee --> D[Zet modus = home_optimization]
  D --> E{SOC ≤ dyn_reserve?}
  E -- Ja --> E1[Discharge = 0 W]
  E -- Nee --> F{EV-blokker actief?}
  F -- Ja --> F1[Discharge = 0 W]
  F -- Nee --> G{Anti-rond OK (netlast > min)?}
  G -- Nee --> G1[Discharge = 0 W]
  G -- Ja --> H{Categorie}
  H -- HIGH --> H1[Discharge = power_high]
  H -- MEDIUM --> H2[Discharge = power_medium]
  H -- LOW --> I{SOC ≥ low_floor?}
  I -- Nee --> I1[Discharge = 0 W]
  I -- Ja --> I2[Discharge = power_low_cap]
```

## Voorbeeldconfiguratie
Zie [`example_config.yaml`](example_config.yaml).

## Uitgebreide uitleg
Zie [`Blueprint_Uitleg.md`](Blueprint_Uitleg.md).

Wanneer gaat de ontlaadwaarde naar 0 W?

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
