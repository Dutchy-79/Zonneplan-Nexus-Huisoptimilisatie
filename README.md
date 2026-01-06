# Home Assistant Blueprint: Slimme Thuisbatterij Optimalisatie

## Introductie
Deze repository bevat een Home Assistant blueprint voor slimme aansturing van een thuisbatterij op basis van dynamische energieprijzen, forecast, SOC, en optionele functies zoals anti-rondpompen en verbruik-blokker (EV).

## Installatie
1. Kopieer `blueprints/zonneplan_battery_optimization_switching.yaml` naar `config/blueprints/automation/<jouw-map>/`.
2. Herlaad Blueprints in Home Assistant (Instellingen → Automatiseringen & Scènes → Blueprints → Herladen).
3. Maak een nieuwe automatisering op basis van de blueprint en koppel de entiteiten.

Variant toegevoegd waarbij de forecast states zelf in te vullen zijn voor de nederlandse versie.
Zie [`blueprint/Zonneplan_Thuisbatterij_Optimalisatie_sensor-defaults_leeg_custom_state_labels`](blueprint/Zonneplan_Thuisbatterij_Optimalisatie_sensor-defaults_leeg_custom_state_labels).

Betere versie waarbij er minder naar de zonneplan server wordt gestuurd en deze je eruit gooit.
blueprint/Nexus_huisoptimalisatie_V3.yaml

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

## Voorbeeldconfiguratie
Zie [`example_config.yaml`](example_config.yaml).

## Uitgebreide uitleg
Zie [`Blueprint_Uitleg.md`](Blueprint_Uitleg.md).

## Flow chart
![flow_chart](https://github.com/user-attachments/assets/cf3cfa2a-9980-402e-8fc9-16353a824f11)


# Preset Zonneplan Thuisbatterij Optimalisatie — README (PATCHES)

**Versie:** 2026-01-06 • **Scope:** Patches voor anti‑rondpompen (kW→W + export neutraliseren), HIGH‑export toelaten, en uitgebreide debug‑logging.

Deze README beschrijft de aangebrachte patches en het ontwerp‑rationale in de blueprint:
> *Preset Zonneplan Thuisbatterij Optimalisatie (idempotent, geen dynamic_charging)*.

---

## Inhoudsopgave
- [Doel en context](#doel-en-context)
- [Kernpatches](#kernpatches)
- [Waarom deze patches?](#waarom-deze-patches)
- [Hoe werken de patches?](#hoe-werken-de-patches)
  - [Anti‑rondpompen: kW→W normalisatie + export→0](#anti-rondpompen-kww-normalisatie--export0)
  - [HIGH mag exporteren](#high-mag-exporteren)
  - [Idempotente wijzigingen](#idempotente-wijzigingen)
  - [Prijsdrempel (price_floor)](#prijsdrempel-price_floor)
- [Configuratie en parameters](#configuratie-en-parameters)
- [Debug‑logging en observability](#debug-logging-en-observability)
- [Validatie en test‑checklist](#validatie-en-test-checklist)
- [Veelvoorkomende scenario’s](#veelvoorkomende-scenarios)
- [Bekende beperkingen](#bekende-beperkingen)
- [Changelog](#changelog)

---

## Doel en context
De blueprint stuurt de thuisbatterij aan op basis van:
- **Huidige prijs** en **prijscategorie** (*low/normal/high*),
- **Forecast categorieën** voor de komende 8 uur,
- **SOC‑reserve** (dynamisch),
- **Anti‑rondpompen** (alleen ontladen bij voldoende net-/huislast),
- **EV‑verbruik‑blokker** (ontladen blokkeren bij actief laden boven drempel).

De automatisering is **idempotent**: eerst worden doelwaarden (targets) berekend; alleen als targets afwijken van huidige waarden worden service‑calls uitgevoerd. Hierdoor voorkom je onnodige wijzigingen en chattering.

---

## Kernpatches
1. **Anti‑rondpompen: kW→W normalisatie + export→0**  
   Waarden van de verbruikssensor worden automatisch omgerekend naar **Watt**. Export (negatieve waarden) wordt naar **0** gezet zodat dit de ontlaad‑toets niet onterecht blokkeert.
2. **HIGH mag exporteren**  
   Bij prijscategorie **HIGH** wordt ontladen toegestaan **zonder** toets op `house_pwr_ok` (anti‑rondpompen). Alleen **EV‑blokker** en **SOC‑reserve** kunnen ontladen nog tegenhouden.
3. **Debug‑logging**  
   Twee `system_log.write`‑regels geven inzicht in inputwaarden, afgeleide variabelen en targets t.o.v. current. Dit versnelt troubleshooting.

---

## Waarom deze patches?
Tijdens een trace bleek ontladen **0** te blijven bij een hoge SOC, omdat `house_pwr_ok` **false** werd door een **eenheden‑mismatch** (sensor leverde waarden als *kW* / kleine decimalen, die vergeleken werden met een drempel in **W**).  
Daarnaast is het **bewust** wenselijk om bij **HIGH** prijs export toe te staan (ontladen richting net), terwijl bij **NORMAL/LOW** anti‑rondpompen actief blijft om onnodige rondpompen te voorkomen.

---

## Hoe werken de patches?

### Anti‑rondpompen: kW→W normalisatie + export→0
**Blueprint‑fragment:**
```jinja
house_pwr_ok: >-
  {% set raw = states(v_house_pwr) %}
  {% set p = raw | float(0) %}
  {% set unit = (state_attr(v_house_pwr, 'unit_of_measurement') | default('', true) | lower) %}
  {% if unit in ['kw'] %}
    {% set pW = p * 1000 %}
  {% elif p < 50 %}
    {% set pW = p * 1000 %}
  {% else %}
    {% set pW = p %}
  {% endif %}
  {% set pW = [pW, 0] | max %}          {# export (negatief) wordt 0 #}
  {% set minp = v_min_net_load | float(300) %}
  {{ pW >= minp }}
