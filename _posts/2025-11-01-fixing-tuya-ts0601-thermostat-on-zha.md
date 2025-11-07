---
layout: post
title: "Fixing the Moes TS0601 Thermostat on Home Assistant (ZHA)"
date: 2025-11-01
categories: homeassistant zigbee tuya thermostat debugging
tags: [homeassistant, zha, tuya, zigbee, thermostat, moes]
---

## Introduction

This post documents my journey debugging and fixing a **Moes ZHT-002 / TS0601 thermostat** (manufacturer ID `_TZE204_xalsoe3m`) when connected to Home Assistant using **ZHA**.  
Out of the box, ZHA recognized it as a `TS0601` Tuya device, but exposed almost no useful controls — just a generic “Router” entity and a few disabled diagnostics.

It became clear that I needed a **custom ZHA quirk**.

---

## The Problem

ZHA loaded the device using the generic Tuya TS0601 handler, but no climate controls worked:

- **No temperature control** or feedback  
- “Unknown” HVAC mode and “Firmware: unknown”  
- Logs showed it registering as a plain `zigpy.device.Device`  
- No quirk was being matched, even though one existed for other TS0601 models

Home Assistant logs were silent about quirk application:

```
Device: _TZE204_xalsoe3m TS0601  
Quirk: zigpy.device.Device
```

It was time to make my own.

---

## The Journey

### Step 1: Discovering the Device Signature

Using ZHA’s device info dump:

```json
{
  "manufacturer": "_TZE204_xalsoe3m",
  "model": "TS0601",
  "endpoints": {
    "1": {
      "profile_id": "0x0104",
      "device_type": "0x0051",
      "input_clusters": ["0x0000","0x0004","0x0005","0xef00"],
      "output_clusters": ["0x000a","0x0019"]
    }
  }
}
```

This confirmed it was a Tuya MCU device using the **EF00** cluster.

---

### Step 2: Creating a Custom Quirk

Custom quirks live under:

```
/config/custom_zha_quirks/tuya/
```

I started with a minimal working version of `tuya_thermostat.py` that matched the fingerprint and confirmed it loaded:

```
Loaded custom quirks. Please contribute them to https://github.com/zigpy/zha-device-handlers
```

That first milestone meant: ✅ my code was being loaded.

---

## Debugging the Datapoints (DPs)

Tuya thermostats expose data via *datapoints (DPs)* rather than standard attributes.

To debug, I instrumented the MCU cluster:

```python
def _dp_2_attr_update(self, dp_value):
    import logging
    logger = logging.getLogger("tuya.tuya_thermostat")
    dp_id = getattr(dp_value, "dp", None)
    tuya_data = getattr(dp_value, "data", None)
    raw_bytes = getattr(tuya_data, "data", None)
    decoded = getattr(tuya_data, "as_value", raw_bytes)
    logger.warning(f"[DP→Attr] {self.endpoint.device.manufacturer} dp={dp_id} decoded={decoded}")
```

This produced live logs of all Tuya datapoints:

```
[DP→Attr] _TZE204_xalsoe3m dp=18 decoded=2100  
[DP→Attr] _TZE204_xalsoe3m dp=50 decoded=2000
```

From this, I discovered:

| DP  | Meaning                         | Notes                    |
|-----|---------------------------------|--------------------------|
| 16  | Actual temperature (°C × 10)    |                          |
| 18  | Room temperature sensor value    |                          |
| 40  | Child lock (bool)               |                          |
| 50  | Setpoint temperature (°C × 100 scaling confirmed) |    |

That meant the scaling was **10× off** in my first attempt!

---

## Step 3: Fixing Scaling and Mapping

After trial and error, and comparing with **Zigbee2MQTT’s Moes ZHT-002 mapping**, I confirmed:

```python
.tuya_dp(
    dp_id=16,
    ep_attribute=TuyaThermostat.ep_attribute,
    attribute_name=TuyaThermostat.AttributeDefs.local_temperature.name,
    converter=lambda x: x * 10,
)
.tuya_dp(
    dp_id=50,
    ep_attribute=TuyaThermostat.ep_attribute,
    attribute_name=TuyaThermostat.AttributeDefs.occupied_heating_setpoint.name,
    converter=lambda x: x // 10,
    dp_converter=lambda x: x * 10,
)
```

This finally produced working temperature reporting and adjustable setpoints within Home Assistant.

---

## Step 4: Adding More Features

Borrowing ideas from `ts0601_trv.py` (the ZHA TRV quirk) and Zigbee2MQTT’s Tuya converter, I added:

- **Min/max temperature limits**  
- **Temperature calibration offset**  
- **Sensor source mode** (internal, external, or both)  
- **Workday mode selection**  
- **Child lock and frost protection**

These are declared using `.tuya_switch()`, `.tuya_enum()`, and `.tuya_number()` calls in the builder.

---

## Step 5: The Final Working Quirk

Here’s the simplified working core for `_TZE204_xalsoe3m`:

```python
(
    TuyaQuirkBuilder("_TZE204_xalsoe3m", "TS0601")
    .tuya_dp(16,
        ep_attribute=TuyaThermostat.ep_attribute,
        attribute_name=TuyaThermostat.AttributeDefs.local_temperature.name,
        converter=lambda x: x * 10,
    )
    .tuya_dp(50,
        ep_attribute=TuyaThermostat.ep_attribute,
        attribute_name=TuyaThermostat.AttributeDefs.occupied_heating_setpoint.name,
        converter=lambda x: x // 10,
        dp_converter=lambda x: x * 10,
    )
    .tuya_switch(
        dp_id=40,
        attribute_name="child_lock",
        translation_key="child_lock",
        fallback_name="Child lock",
    )
    .tuya_enum(
        dp_id=23,
        attribute_name="working_day",
        enum_class=WorkingDayV02,
        translation_key="working_day",
        fallback_name="Working day mode",
    )
    .tuya_number(
        dp_id=19,
        attribute_name=TuyaThermostat.AttributeDefs.local_temperature_calibration.name,
        type=t.int16s,
        min_value=-9,
        max_value=9,
        step=1,
        unit=UnitOfTemperature.CELSIUS,
        translation_key="local_temperature_calibration",
        fallback_name="Temperature calibration",
    )
    .adds(TuyaThermostat)
    .skip_configuration()
    .add_to_registry()
)
```

---

## Step 6: Success 🎉

After restarting Home Assistant, ZHA loaded:

```
Quirk: zhaquirks.tuya.tuya_thermostat
```

The **climate entity** now exposes:

- Current and target temperature  
- Mode switching (`heat` / `off`)  
- Child lock toggle  
- Calibration offset  
- Workday mode  
- Correct scaling for both actual and setpoint readings  

And yes — `[DP→Attr]` logs still print decoded DP updates for debugging!

---

## Lessons Learned

1. **Tuya devices don’t speak Zigbee directly** — everything is custom MCU DPs.  
2. **Start simple** — confirm your quirk loads, then map one DP at a time.  
3. **Always decode TuyaData properly** — `.as_value` is your friend.  
4. **Borrow from Zigbee2MQTT and ZHA TRV quirks** — they’re excellent references.  
5. **Keep logging while you experiment** — it’s the only visibility you get.

---

## Final Thoughts

The TS0601 thermostats are not hopeless — they just need a translator.  
With a working custom quirk, ZHA can handle them as cleanly as Zigbee2MQTT, no extra gateway needed.

If you’re wrestling with a Tuya TS0601 device, start by identifying your **DP map** — everything else falls into place from there.

---

*Repository:* [leocrawford.github.io](https://github.com/leocrawford/leocrawford.github.io)  
*Device:* `_TZE204_xalsoe3m` (Moes ZHT-002)  
*Home Assistant Core:* 2025.10.4  
*Integration:* ZHA (Texas Instruments CC2531 coordinator)
