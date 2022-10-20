
## Davis AirLink API examples

Here are a couple HOWTOs for how to get and use data from the AirLink API.

### Query the local REST API

Here's how to query the local API and format the result nicely.  Substitute in your AirLink's IP address.

```
$ wget -q -O - http://192.168.2.60/v1/current_conditions | jq .
{
  "data": {
    "did": "0123456789AB",
    "name": "your name here",
    "ts": 1666301599,
    "conditions": [
      {
        "lsid": 556237,
        "data_structure_type": 6,
        "temp": 73.1,
        "hum": 46.9,
        "dew_point": 51.6,
        "wet_bulb": 57.5,
        "heat_index": 72.3,
        "pm_1_last": 10,
        "pm_2p5_last": 20,
        "pm_10_last": 20,
        "pm_1": 9.94,
        "pm_2p5": 19.81,
        "pm_2p5_last_1_hour": 20.19,
        "pm_2p5_last_3_hours": 21.28,
        "pm_2p5_last_24_hours": 24.87,
        "pm_2p5_nowcast": 21.46,
        "pm_10": 21.13,
        "pm_10_last_1_hour": 22.06,
        "pm_10_last_3_hours": 23.26,
        "pm_10_last_24_hours": 27.74,
        "pm_10_nowcast": 23.52,
        "last_report_time": 1666301599,
        "pct_pm_data_last_1_hour": 100,
        "pct_pm_data_last_3_hours": 100,
        "pct_pm_data_nowcast": 100,
        "pct_pm_data_last_24_hours": 100
      }
    ]
  },
  "error": null
}
```


### Home Assistant

Add this to your configuration.yaml file (substituting in your AirLink IP address) in order to define two sensors, one with a calculated PM2.5 AQI, the other with the raw value from the sensor

```
    #-------------------------------------------------
    # items from rest query of Davis AirLink JSON
    # ref: https://community.home-assistant.io/t/davis-weatherlink-integration/134737
    #-------------------------------------------------

    - platform: rest
      name: 'AirLink'
      resource: http://192.168.2.60/v1/current_conditions
      value_template: 'Davis AirLink'
      json_attributes:
        - data

    #
    # calculate AQI pm2.5 with a little macro wizardry courtesy of:
    #    https://www.reddit.com/r/homeautomation/comments/umrs3f/comment/i9zvmoa/?utm_source=share&utm_medium=web2x&context=3
    #
    # 2018 breakpoints from https://www.airnow.gov/sites/default/files/2020-05/aqi-technical-assistance-document-sept2018.pdf
    #

    - platform: template
      sensors:
        airlink_aqi:
          friendly_name: "Airlink AQI"
          value_template: >
            {% macro aqi(val, val_l, val_h, aqi_l, aqi_h) -%}
               {{ (((aqi_h-aqi_l)/(val_h - val_l) * (val - val_l)) + aqi_l)|round(0) }}
            {%- endmacro %}
            {% set v = state_attr("sensor.airlink","data")["conditions"][0]["pm_2p5"] %}
            {% if v <= 12.0 %}               {{ aqi(v,     0,  12.0,   0,  50) }}
            {% elif    12.0 < v <=  35.4  %} {{ aqi(v,  12.1,  35.4,  51, 100) }}
            {% elif    35.4 < v <=  55.4  %} {{ aqi(v,  35.5,  55.4, 101, 150) }}
            {% elif    55.4 < v <= 150.5  %} {{ aqi(v,  55.5, 150.4, 151, 200) }}
            {% elif   150.4 < v <= 250.4  %} {{ aqi(v, 150.4, 250.4, 201, 300) }}
            {% elif   250.5 < v <= 500.4  %} {{ aqi(v, 250.5, 500.4, 301, 500) }}
            {% else %} {{ 500 }}
            {% endif %}
          unit_of_measurement: "AQI"

    # this just grabs the raw data so we can view it in 'all' in HA
    - platform: template
      sensors:
        airlink_aqi_raw:
          friendly_name: "Airlink Raw pm2.5"
          value_template: '{{ state_attr("sensor.airlink","data")["conditions"][0]["pm_2p5"] }}'
          unit_of_measurement: "pm2.5"
```

### Integrate with weewx

A nice [weewx](https://weewx.com)  extension is [HERE](https://github.com/chaunceygardiner/weewx-airlink)
