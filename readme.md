readme.md

# 🛫 Flight24 Airline Card

A custom Home Assistant dashboard card that displays real-time flight information using the Flightradar24 integration.
The card shows either:

* The **closest aircraft above 10,000 ft** near your home, or
* The **most tracked flight worldwide**

It includes airline logos, callsign, airline name, route, altitude, and ground speed.
All assets are local, and no JavaScript or HACS repository is required.

---

## ✈️ Features

* Displays nearest high-altitude aircraft or worldwide most-tracked flight
* Shows airline logo, callsign, flight number, and origin/destination
* Detects private aircraft using N-number logic
* Works in standard dashboards and kiosk/fullscreen displays
* Fully local asset loading (PNG/SVG logos)
* Clean, responsive UI

---

## 📦 Required Components

### 1. **Flightradar24 Integration**

This card depends on the Flightradar24 integration by **Alexandr Erohin**:

🔗 [https://github.com/AlexandrErohin/home-assistant-flightradar24](https://github.com/AlexandrErohin/home-assistant-flightradar24)

This integration must provide the following sensors:

* `sensor.flightradar24_current_in_area`
* `sensor.flightradar24_most_tracked`

---

### 2. **HTML Jinja2 Template Card**

Powered by Piotr Machowski’s HTML Jinja2 Template Card:

🔗 [https://github.com/PiotrMachowski/Home-Assistant-Lovelace-HTML-Jinja2-Template-card](https://github.com/PiotrMachowski/Home-Assistant-Lovelace-HTML-Jinja2-Template-card)

Install via HACS (custom repo) or manually.
Card type:

```yaml
type: custom:html-template-card
```

---

## 📁 Airline Logo Assets

This card uses local airline logos stored in:

```
/config/www/assets/
```

You can download a complete collection of airline icons here:

🔗 **Airline Logo Pack:**
[https://github.com/anhthang/soaring-symbols/tree/main/assets](https://github.com/anhthang/soaring-symbols/tree/main/assets)

After downloading, copy the airline folders into:

```
/config/www/assets/
```

Example structure:

```
www/
└── assets/
    ├── fallback-airline/
    ├── private-plane/
    ├── aeromexico/
    ├── american-airlines/
    ├── delta-air-lines/
    ├── united-airlines/
    ├── qatar-airways/
    └── (additional airlines)
```

Home Assistant exposes these as:

```
/local/assets/<airline-folder>/icon.svg
```

PNG or SVG logos are supported.

---

## 📋 Dashboard Card (Standard Version)

Add a **Manual card** to your dashboard and paste:

```yaml
type: custom:html-template-card
card_mod:
  style: |
    ha-card {
      background: #020617;
      padding: 16px 20px;
      border-radius: 16px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
    }
ignore_line_breaks: true
entities:
  - sensor.flightradar24_current_in_area
  - sensor.flightradar24_most_tracked
content: |
  {% set fallback_icon = '/local/assets/fallback-airline/icon.png' %}
  {% set private_icon  = '/local/assets/private-plane/private-plane.png' %}

  {% set local = state_attr('sensor.flightradar24_current_in_area', 'flights') or [] %}
  {% set local_filtered = local | selectattr('altitude', '>', 10000) | list %}

  {% if local_filtered | length > 0 %}
    {% set mode = 'local' %}
    {% set f = local_filtered | sort(attribute='distance') | first %}
  {% else %}
    {% set mode = 'most' %}
    {% set most = state_attr('sensor.flightradar24_most_tracked', 'flights') or [] %}
    {% set f = most[0] if most | length > 0 else None %}
  {% endif %}

  <div style="
    width:100%;
    display:flex;
    flex-direction:column;
    gap:12px;
    font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;
    color:#e5e7eb;
  ">

  {% if not f %}
    <div style="font-size:16px;opacity:0.85;">No flights available ✈️</div>
  {% else %}

    {% set fn        = (f.flight_number or '') | string %}
    {% set cs        = (f.callsign or '') | string %}
    {% set airline   = (f.airline_short or 'Unknown') | string %}
    {% set airline_l = airline | lower %}
    {% set origin    = f.airport_origin_city or 'Unknown' %}
    {% set dest      = f.airport_destination_city or 'Unknown' %}
    {% set alt       = f.altitude or 0 %}
    {% set gs        = f.ground_speed or 0 %}

    {% if mode == 'local' %}
      {% set dist     = (f.distance | float | round(1)) %}
      {% set header   = 'Closest flight • ' ~ dist ~ ' km from home' %}
      {% set sublabel = 'Nearby flight' %}
    {% else %}
      {% set header   = 'Most tracked flight on Flightradar24' %}
      {% set sublabel = 'Most tracked flight' %}
    {% endif %}

    {% set logo_url = fallback_icon %}
    {% if 'unknown' in airline_l %}
      {% set airline_label = sublabel %}
    {% else %}
      {% set airline_label = airline %}
    {% endif %}

    {% if fn.startswith('N') or cs.startswith('N') %}
      {% set logo_url = private_icon %}
      {% set airline_label = 'Private aircraft' %}
    {% endif %}

    <div style="
      display:grid;
      grid-template-columns:minmax(0, 1.5fr) auto;
      gap:16px;
      align-items:center;
    ">

      <div style="display:flex;flex-direction:column;gap:6px;">
        <div style="font-size:11px;opacity:0.7;letter-spacing:0.15em;text-transform:uppercase;">
          {{ header }}
        </div>

        <div style="font-size:26px;font-weight:600;">
          {{ fn if fn | length > 0 else cs }}
        </div>

        <div style="font-size:15px;opacity:0.8;">
          {{ cs }}
        </div>

        <div style="font-size:17px;opacity:0.9;">
          {{ origin }} → {{ dest }}
        </div>

        <div style="margin-top:8px;display:flex;gap:24px;font-size:13px;">
          <div>
            <div style="font-size:10px;opacity:0.6;text-transform:uppercase;letter-spacing:0.15em;">Altitude</div>
            <div style="font-size:20px;font-weight:600;">{{ alt | int }} ft</div>
          </div>
          <div>
            <div style="font-size:10px;opacity:0.6;text-transform:uppercase;letter-spacing:0.15em;">Ground Speed</div>
            <div style="font-size:20px;font-weight:600;">{{ gs | int }} kts</div>
          </div>
        </div>
      </div>

      <div style="text-align:right;">
        <img
          src="{{ logo_url }}"
          style="width:90px;height:auto;filter:drop-shadow(0 0 2px #0008);"
        />
        <div style="
          font-size:12px;
          opacity:0.7;
          text-transform:uppercase;
          margin-top:4px;
        ">
          {{ airline_label }}
        </div>
      </div>

    </div>

  {% endif %}
  </div>
```

---

## 🖥️ Kiosk Version

A fullscreen version for wall-mounted tablets can be added as `cards/kiosk.yaml` using the same template logic with:

```
min-height: 100vh;
border-radius: 0;
padding: 0;
```

---

## 👥 Contributing

Feel free to add more airlines, tweak layout, or create alternate versions.
Pull requests and forks are welcome.

---

## 📜 License

MIT License
© 2025 Colter Wilson (Potseeslc)
