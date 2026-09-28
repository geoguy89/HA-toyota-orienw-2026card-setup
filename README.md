# Toyota Remote Kit for Home Assistant

A dashboard card plus a Home Assistant package for Toyota (and Lexus) vehicles in North America. It was built and tested on a **2026 RAV4 Hybrid (generation 24MM)** and should work on anything the integration supports.

- **Remote control:** lock, unlock, start, stop, hazards, find vehicle and refresh.
- **Climate control:** cabin temperature, front and rear defrost, heated seats and steering wheel, and longer runtime. These settings are applied when you remote start.
- **Scheduled remote start:**
  - Recurring (pick a time and weekdays) or one-time (pick a date and time).
  - An optional *weather gate* that only starts the car when it's colder or hotter than your limits.
  - Phone notifications when a scheduled start is skipped or fails.
- **Tire pressure** on a top-down car diagram, color-coded against your recommended PSI. Hover over or long-press a tire for details.
- **Trip stats:** average MPG, trip MPG, trip A distance and trip count.
- **Oil-change tracker:** "next service" is counted from *your last oil change* (miles or months, whichever comes first), not Toyota's generic maintenance countdown.
- **Status at a glance:** locked or unlocked, and every open door, window, hatch or hood.
- Fuel gauge and range, an optional lease-mileage tracker, and a map.

Everything is plain Home Assistant YAML. The only third-party frontend dependency is `button-card`.

---

## 1. Prerequisites

| What | Why |
|---|---|
| Home Assistant 2024.11 or newer | Required by the integration |
| [HACS](https://hacs.xyz) | To install the two items below |
| **Toyota (North America)** integration, **orienw fork**: `https://github.com/orienw/ha-toyota-na` | Supplies the vehicle entities. The fork is needed for 2025+ vehicles (21MM and 24MM) and for the climate controls. |
| **button-card**, from HACS > Frontend | Renders the card |
| An active Toyota **Remote Connect** subscription | Needed for remote commands and door/lock status |

### Install the integration

1. HACS > the three-dot menu > **Custom repositories**. Add `https://github.com/orienw/ha-toyota-na` with type **Integration**.
2. Download the latest release, then restart Home Assistant.
3. **Settings > Devices & services > Add integration > Toyota (North America)**. Sign in with your Toyota app email and password, then enter the verification code.

> **Switching from the original `widewing/ha-toyota-na`?** Both install to the same folder. Remove the widewing entry in HACS *first*, then download the fork, then restart. If you remove widewing *after* downloading the fork, HACS deletes the shared folder. Just re-download the fork in HACS if that happens.

Check that your car's device page shows **Remote Start / Remote Stop buttons** and, under *Configuration*, **Climate Temperature** and the defroster switches. If the climate entities are missing, your vehicle or subscription doesn't expose remote climate. The rest of the kit still works.

---

## 2. Find your entity slug

Every entity for your car shares a prefix, for example `lock.2025_camry_xse` gives the slug `2025_camry_xse`.

1. Open **Developer Tools > Template**.
2. Paste the contents of [`tools/find_my_entities.jinja`](tools/find_my_entities.jinja).
3. Change the first line to your car's lock entity. It prints your slug and every entity on the car.

> If you set the integration up before the fork (on the old widewing version), some of your older entity IDs end in the car name twice, for example `sensor.<slug>_fuel_level_<slug>`. Use the list from the template to fix those few IDs by hand after the find and replace.

---

## 3. Add the package (schedule, scripts, oil tracker)

1. If you haven't already, enable packages in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy [`packages/toyota_remote.yaml`](packages/toyota_remote.yaml) to `<config>/packages/toyota_remote.yaml`.
3. In that file, **find and replace**:
   - `YOUR_CAR`: your slug from step 2.
   - `YOUR_PHONE`: your phone's notify service. Check **Developer Tools > Actions**, then search for `notify.mobile_app_`.
   - `weather.forecast_home`: only if your weather entity has a different name.
4. **Developer Tools > YAML > Check configuration**, then restart Home Assistant.

Set these starting values once. They live in helpers and are all editable from the card:
- *Schedule – Days*, for example `mon,tue,wed,thu,fri`.
- *Schedule – Start Time*.
- *Start If Colder Than* and *Start If Hotter Than* (for example 40 °F and 85 °F).
- *Oil Change Interval*: check your owner's manual. Many 2025+ Toyotas use 10,000 mi or 12 months with synthetic oil.

---

## 4. Add the card

1. Open [`dashboard/car_card.yaml`](dashboard/car_card.yaml) and **find and replace**:
   - `YOUR_CAR`: your slug.
   - `YOUR CAR NAME`: the title, for example "2025 Tundra".
   - `YOUR_CAR_IMAGE_URL`: any image URL. Toyota's own render of your exact car is in the integration diagnostics: **Settings > Devices & services > Toyota > ⋮ > Download diagnostics**, then search for `"image"`.
   - `const REC = 36`: your recommended tire pressure, from the sticker in the driver's door jamb.
2. Go to your dashboard, then **Edit > + Add card > Manual**, and paste the file.

The card is designed for a phone-width column in a dark theme.

---

## Using it

| Where | What it does |
|---|---|
| **Hold** the car photo | Opens *Settings*: log an oil change, oil interval, weather-gate limits, and the lease tracker |
| **CONTROLS** | Every command asks for confirmation. START uses your saved climate settings. |
| **CLIMATE** | − / + changes the cabin temperature. Tap a tile to toggle it. The seat tiles cycle through the options your car supports. **USE** turns the climate settings on or off for remote start. Changes are saved to Toyota's servers, the same as in the Toyota app. |
| **SCHEDULE** | **REPEAT** turns the weekly schedule on or off. Tap **TIME** to set the time. Tap the **S M T W T F S** chips to pick days. **ONE-TIME** plus **WHEN** sets a single start, which turns itself off after it runs. **WEATHER GATE** skips mild days. |
| Tires | Green is OK, amber is slightly off, red is low or has a TPMS warning. Hover over or long-press a tire for details. |
| Oil bar | Hold the photo and tap **Log oil change now** after each oil change. The bar shows miles or months left, whichever runs out first. |

---

## Safety and limits (please read)

- **Never remote start in an enclosed garage.** A hybrid's engine can start at any time while it's running.
- Toyota limits back-to-back remote starts. Many vehicles allow about 2 remote starts between drives. After that the car refuses until it has been driven. The scheduled automation notifies you if the car didn't report running within 3 minutes.
- A remote start is also blocked when a door, the hood or the hatch is open, or when the key is inside.
- Remote start runs for about 10 minutes (about 20 with **Longer Climate Runtime** on). The card's countdown uses those values. Adjust `timer` durations in the package if your vehicle differs.
- Commands go through Toyota's cloud and can take 30 to 90 seconds to confirm.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Card shows "Custom element doesn't exist: button-card" | Install button-card from HACS, then hard-refresh the browser |
| Entities show "Entity not available" in the card | An ID didn't match. Run the template from step 2 and fix it in the card YAML. |
| Doors or locks show "no data yet" | Press **REFRESH**. The car uploads its status when Toyota wakes it. |
| `403 Forbidden` in the logs | Update the integration. Toyota rotates API keys, and the fork tracks these changes. |
| Climate tiles missing | Your car doesn't report remote climate. Check the Toyota app for climate settings. |

## Credits

- Vehicle integration: [orienw/ha-toyota-na](https://github.com/orienw/ha-toyota-na), a fork of [widewing/ha-toyota-na](https://github.com/widewing/ha-toyota-na).
- Card: [custom-cards/button-card](https://github.com/custom-cards/button-card).

This kit is not affiliated with or endorsed by Toyota. Use remote start responsibly.

## License

MIT (see [LICENSE](LICENSE)). This covers the files in this repository only, not the integrations it depends on.
