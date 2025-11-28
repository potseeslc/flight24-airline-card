# Flight24 Airline Card

A custom [Home Assistant](https://www.home-assistant.io/) card that shows the **closest flight near your home** or the **most tracked flight on Flightradar24**, with airline logo, route, altitude, speed, and aircraft info.

Built using [`html-template-card`](https://github.com/iansimard/html-template-card) and the [Flightradar24 Home Assistant integration](https://github.com/AlexxIT/FlightRadar24).

---

## Features

- Shows **closest local flight above 10,000 ft** (distance from `zone.home`)
- Falls back to **most tracked flight on Flightradar24**
- Uses **ICAO code** from callsign/flight number to load airline logos
- Handles:
  - Unknown airlines gracefully (fallback logo + label)
  - Private aircraft (US tail numbers like `N12345`) with a special icon
- Two layouts:
  - **Regular dashboard card**
  - **Fullscreen kiosk card** (for a wall tablet / big display)

---

## Requirements

1. **Flightradar24 integration** with at least these entities:

   - `sensor.flightradar24_current_in_area`
   - `sensor.flightradar24_most_tracked`

2. **Custom cards / addons**

   - https://github.com/PiotrMachowski/Home-Assistant-Lovelace-HTML-Jinja2-Template-card (Html Template Card)
   - [`card-mod`](https://github.com/thomasloven/lovelace-card-mod) (for styling)

3. **A defined `zone.home`**
   - Used by `distance('zone.home', (lat, lon))` to find the closest local flight.

---

## Assets / Logos

Place your assets in `config/www/assets/` in Home Assistant:

```text
/config/www/assets/
  flightaware_logos/
    AAL.png
    DAL.png
    UAL.png
    KLM.png
    ...
  fallback-airline/
    icon.png
  private-plane/
    private-plane.png
Airline logos are loaded from:

/local/assets/flightaware_logos/<ICAO>.png


Examples:

Delta → DAL.png

United → UAL.png

KLM → KLM.png

Fallback logo:

/local/assets/fallback-airline/icon.png


Private plane logo:

/local/assets/private-plane/private-plane.png


💡 The card derives the ICAO prefix from the callsign first, then the flight number (first 3 characters, uppercased).

Regular Dashboard Card

Use this on your normal Lovelace dashboard.

type: custom:html-template-card
card_mod:
  style: |
    ha-card {
      background: #020617;
      border-radius: 16px;
      padding: 20px;
      box-shadow: 0 2px 6px rgba(0,0,0,0.25);
    }
title: ""
ignore_line_breaks: true
entities:
  - sensor.flightradar24_current_in_area
  - sensor.flightradar24_most_tracked
content: >
  {# --- Asset paths --- #}
  {% set fallback_icon = '/local/assets/fallback-airline/icon.png' %}
  {% set private_icon  = '/local/assets/private-plane/private-plane.png' %}

  {# --- Get flight lists --- #}
  {% set local = state_attr('sensor.flightradar24_current_in_area', 'flights') or [] %}
  {% set most  = state_attr('sensor.flightradar24_most_tracked', 'flights') or [] %}

  {# --- Pick closest local flight above 10,000 ft --- #}
  {% set ns = namespace(best=None, best_dist=999999999) %}
  {% for fl in local %}
    {% if fl is mapping
          and 'latitude' in fl and 'longitude' in fl
          and 'altitude' in fl and (fl['altitude'] or 0) > 10000 %}
      {% set d = distance('zone.home', (fl['latitude'], fl['longitude'])) %}
      {% if d is number and d < ns.best_dist %}
        {% set ns.best = fl %}
        {% set ns.best_dist = d %}
      {% endif %}
    {% endif %}
  {% endfor %}

  {% if ns.best %}
    {% set mode = 'local' %}
    {% set f = ns.best %}
    {% set dist_km = (ns.best_dist / 1000) | round(1) %}
  {% else %}
    {% set mode = 'most' %}
    {% set f = most[0] if most|length > 0 else None %}
  {% endif %}

  <div style="
    width:100%;
    display:flex;
    align-items:center;
    justify-content:center;
    font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;
    color:#e5e7eb;
  ">

  {% if not f %}
    <div style="font-size:16px;opacity:0.85;">
      No flights available ✈️
    </div>
  {% else %}

    {# --- Extract values safely --- #}
    {% set fn       = (f.get('flight_number','')       | string) %}
    {% set cs       = (f.get('callsign','')            | string) %}
    {% set airline  = (f.get('airline_short','Unknown')| string) %}
    {% set airline_l= airline | lower %}
    {% set origin   = f.get('airport_origin_city','Unknown') %}
    {% set dest     = f.get('airport_destination_city','Unknown') %}
    {% set alt      = f.get('altitude', 0) or 0 %}
    {% set gs       = f.get('ground_speed', 0) or 0 %}
    {% set aircraft = f.get('aircraft_type')
                       or f.get('aircraft_model')
                       or f.get('aircraft_code')
                       or 'Unknown' %}

    {# --- Header text --- #}
    {% if mode == 'local' %}
      {% set header   = 'Closest flight • ' ~ dist_km ~ ' km from home' %}
      {% set sublabel = 'Nearby flight' %}
    {% else %}
      {% set header   = 'Most tracked flight on Flightradar24' %}
      {% set sublabel = 'Most tracked flight' %}
    {% endif %}

    {# --- ICAO logo selection --- #}
    {% set logo_url = fallback_icon %}
    {% set icao = '' %}
    {% if cs|length >= 3 %}
      {% set icao = cs[:3] | upper %}
    {% elif fn|length >= 3 %}
      {% set icao = fn[:3] | upper %}
    {% endif %}
    {% if icao %}
      {% set logo_url = '/local/assets/flightaware_logos/' ~ icao ~ '.png' %}
    {% endif %}

    {# --- Label under logo (with private/unknown handling) --- #}
    {% if fn.startswith('N') or cs.startswith('N') %}
      {% set logo_url = private_icon %}
      {% set airline_label = 'Private aircraft' %}
    {% else %}
      {% if 'unknown' in airline_l %}
        {% set airline_label = icao if icao else sublabel %}
      {% else %}
        {% set airline_label = airline %}
      {% endif %}
    {% endif %}

    <div style="
      display:flex;
      flex-direction:column;
      width:100%;
      padding:8px;
      max-width:1000px;
    ">

      <div style="
        width:100%;
        display:grid;
        grid-template-columns:1fr auto;
        gap:16px;
        align-items:center;
      ">

        <div style="display:flex;flex-direction:column;gap:6px;">
          <div style="
            font-size:11px;
            opacity:0.7;
            letter-spacing:0.15em;
            text-transform:uppercase;
          ">
            {{ header }}
          </div>

          <div style="
            font-size:26px;
            font-weight:600;
          ">
            {{ fn if fn|length > 0 else cs }}
          </div>

          <div style="
            font-size:15px;
            opacity:0.8;
          ">
            {{ cs }}
          </div>

          <div style="
            font-size:17px;
            opacity:0.9;
            margin-top:2px;
          ">
            {{ origin }} → {{ dest }}
          </div>

          <div style="
            margin-top:8px;
            font-size:13px;
            display:flex;
            gap:24px;
            flex-wrap:wrap;
          ">

            <div>
              <div style="
                font-size:10px;
                opacity:0.6;
                text-transform:uppercase;
                letter-spacing:0.15em;
              ">
                Altitude
              </div>
              <div style="font-size:20px;font-weight:600;">
                {{ alt|int }} ft
              </div>
            </div>

            <div>
              <div style="
                font-size:10px;
                opacity:0.6;
                text-transform:uppercase;
                letter-spacing:0.15em;
              ">
                Ground Speed
              </div>
              <div style="font-size:20px;font-weight:600;">
                {{ gs|int }} kts
              </div>
            </div>

            <div>
              <div style="
                font-size:10px;
                opacity:0.6;
                text-transform:uppercase;
                letter-spacing:0.15em;
              ">
                Aircraft
              </div>
              <div style="font-size:20px;font-weight:600;">
                {{ aircraft }}
              </div>
            </div>

          </div>
        </div>

        <div style="text-align:right;">
          <div style="
            width:90px;
            height:90px;
            display:flex;
            align-items:center;
            justify-content:center;
            overflow:hidden;
          ">
            <img src="{{ logo_url }}" style="
              max-width:80%;
              max-height:80%;
              object-fit:contain;
            " />
          </div>

          <div style="
            font-size:12px;
            opacity:0.7;
            text-transform:uppercase;
            margin-top:4px;
            text-align:center;
          ">
            {{ airline_label }}
          </div>

          {% if mode == 'most' %}
          <div style="
            font-size:10px;
            opacity:0.6;
            text-transform:uppercase;
            margin-top:3px;
            text-align:center;
            letter-spacing:0.15em;
          ">
            MOST TRACKED FLIGHT
          </div>
          {% endif %}

        </div>

      </div>
    </div>

  {% endif %}
  </div>

Fullscreen Kiosk Card

This version is meant for a dedicated tablet / wall screen with kiosk mode.

type: custom:html-template-card
card_mod:
  style: |
    ha-card {
      background: #020617;
      border-radius: 0;
      padding: 0;
      box-shadow: none;
      height: 100vh;
    }
title: ""
ignore_line_breaks: true
entities:
  - sensor.flightradar24_current_in_area
  - sensor.flightradar24_most_tracked
content: >
  {# --- Asset paths --- #}
  {% set fallback_icon = '/local/assets/fallback-airline/icon.png' %}
  {% set private_icon  = '/local/assets/private-plane/private-plane.png' %}

  {# --- Get flight lists --- #}
  {% set local = state_attr('sensor.flightradar24_current_in_area', 'flights') or [] %}
  {% set most  = state_attr('sensor.flightradar24_most_tracked', 'flights') or [] %}

  {# --- Pick closest local flight above 10,000 ft --- #}
  {% set ns = namespace(best=None, best_dist=999999999) %}
  {% for fl in local %}
    {% if fl is mapping
          and 'latitude' in fl and 'longitude' in fl
          and 'altitude' in fl and (fl['altitude'] or 0) > 10000 %}
      {% set d = distance('zone.home', (fl['latitude'], fl['longitude'])) %}
      {% if d is number and d < ns.best_dist %}
        {% set ns.best = fl %}
        {% set ns.best_dist = d %}
      {% endif %}
    {% endif %}
  {% endfor %}

  {% if ns.best %}
    {% set mode = 'local' %}
    {% set f = ns.best %}
    {% set dist_km = (ns.best_dist / 1000) | round(1) %}
  {% else %}
    {% set mode = 'most' %}
    {% set f = most[0] if most|length > 0 else None %}
  {% endif %}

  <div style="
    min-height:100vh;
    width:100%;
    display:flex;
    align-items:center;
    justify-content:center;
    font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;
    color:#e5e7eb;
  ">

  {% if not f %}
    <div style="font-size:clamp(18px,3vw,28px);opacity:0.85;">
      No flights available ✈️
    </div>
  {% else %}

    {# --- Extract values safely --- #}
    {% set fn       = (f.get('flight_number','')       | string) %}
    {% set cs       = (f.get('callsign','')            | string) %}
    {% set airline  = (f.get('airline_short','Unknown')| string) %}
    {% set airline_l= airline | lower %}
    {% set origin   = f.get('airport_origin_city','Unknown') %}
    {% set dest     = f.get('airport_destination_city','Unknown') %}
    {% set alt      = f.get('altitude', 0) or 0 %}
    {% set gs       = f.get('ground_speed', 0) or 0 %}
    {% set aircraft = f.get('aircraft_type')
                       or f.get('aircraft_model')
                       or f.get('aircraft_code')
                       or 'Unknown' %}

    {# --- Header text --- #}
    {% if mode == 'local' %}
      {% set header   = 'Closest flight • ' ~ dist_km ~ ' km from home' %}
      {% set sublabel = 'Nearby flight' %}
    {% else %}
      {% set header   = 'Most tracked flight on Flightradar24' %}
      {% set sublabel = 'Most tracked flight' %}
    {% endif %}

    {# --- ICAO logo selection --- #}
    {% set logo_url = fallback_icon %}
    {% set icao = '' %}
    {% if cs|length >= 3 %}
      {% set icao = cs[:3] | upper %}
    {% elif fn|length >= 3 %}
      {% set icao = fn[:3] | upper %}
    {% endif %}
    {% if icao %}
      {% set logo_url = '/local/assets/flightaware_logos/' ~ icao ~ '.png' %}
    {% endif %}

    {# --- Label under logo --- #}
    {% if fn.startswith('N') or cs.startswith('N') %}
      {% set logo_url = private_icon %}
      {% set airline_label = 'Private aircraft' %}
    {% else %}
      {% if 'unknown' in airline_l %}
        {% set airline_label = icao if icao else sublabel %}
      {% else %}
        {% set airline_label = airline %}
      {% endif %}
    {% endif %}

    <div style="
      display:flex;
      flex-direction:column;
      align-items:center;
      justify-content:center;
      width:100%;
      max-width:1100px;
      padding:24px;
    ">

      <div style="
        width:100%;
        display:grid;
        grid-template-columns:1fr auto;
        gap:18px;
        align-items:center;
      ">

        <div style="display:flex;flex-direction:column;gap:8px;">
          <div style="
            font-size:clamp(10px,1.4vw,14px);
            opacity:0.7;
            letter-spacing:0.15em;
            text-transform:uppercase;
          ">
            {{ header }}
          </div>

          <div style="
            font-size:clamp(28px,5vw,56px);
            font-weight:600;
          ">
            {{ fn if fn|length > 0 else cs }}
          </div>

          <div style="
            font-size:clamp(16px,2.5vw,28px);
            opacity:0.8;
          ">
            {{ cs }}
          </div>

          <div style="
            font-size:clamp(18px,3vw,32px);
            opacity:0.9;
            margin-top:2px;
          ">
            {{ origin }} → {{ dest }}
          </div>

          <div style="
            margin-top:10px;
            font-size:clamp(12px,1.6vw,16px);
            display:flex;
            gap:36px;
            flex-wrap:wrap;
          ">

            <div>
              <div style="
                font-size:clamp(9px,1.2vw,12px);
                opacity:0.6;
                text-transform:uppercase;
                letter-spacing:0.15em;
              ">
                Altitude
              </div>
              <div style="
                font-size:clamp(20px,3vw,34px);
                font-weight:600;
              ">
                {{ alt|int }} ft
              </div>
            </div>

            <div>
              <div style="
                font-size:clamp(9px,1.2vw,12px);
                opacity:0.6;
                text-transform:uppercase;
                letter-spacing:0.15em;
              ">
                Ground Speed
              </div>
              <div style="
                font-size:clamp(20px,3vw,34px);
                font-weight:600;
              ">
                {{ gs|int }} kts
              </div>
            </div>

            <div>
              <div style="
                font-size:clamp(9px,1.2vw,12px);
                opacity:0.6;
                text-transform:uppercase;
                letter-spacing:0.15em;
              ">
                Aircraft
              </div>
              <div style="
                font-size:clamp(20px,3vw,28px);
                font-weight:600;
              ">
                {{ aircraft }}
              </div>
            </div>

          </div>
        </div>

        <div style="text-align:right;">
          <div style="
            width:clamp(80px,14vw,150px);
            height:clamp(80px,14vw,150px);
            background:transparent;
            display:flex;
            align-items:center;
            justify-content:center;
            overflow:hidden;
          ">
            <img src="{{ logo_url }}" style="
              max-width:80%;
              max-height:80%;
              object-fit:contain;
            " />
          </div>

          <div style="
            font-size:clamp(10px,1.5vw,16px);
            opacity:0.8;
            text-transform:uppercase;
            margin-top:8px;
            text-align:center;
          ">
            {{ airline_label }}
          </div>

          {% if mode == 'most' %}
          <div style="
            font-size:clamp(9px,1.2vw,12px);
            opacity:0.6;
            text-transform:uppercase;
            margin-top:4px;
            text-align:center;
            letter-spacing:0.15em;
          ">
            MOST TRACKED FLIGHT
          </div>
          {% endif %}

        </div>

      </div>
    </div>

  {% endif %}
  </div>
