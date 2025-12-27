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

## Voorbeeldconfiguratie
Zie [`example_config.yaml`](example_config.yaml).

## Uitgebreide uitleg
Zie [`Blueprint_Uitleg.md`](Blueprint_Uitleg.md).
