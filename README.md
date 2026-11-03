# PZEM for Home Assistant core

The `pzem` integration, laid out as it sits in a
[home-assistant/core](https://github.com/home-assistant/core) checkout:

```
homeassistant/components/pzem_014_016/   the integration
tests/components/pzem_014_016/           its tests and snapshots
```

It follows the `sofar` integration: a menu for a serial or a network link, a
reconfigure flow, a `DataUpdateCoordinator`, and its connection taken from the
`modbus` integration with `async_get_unit`, so the link is shared with anything
else on the same bus.

## Before opening the pull request

1. Publish [`pzem-014-016-modbus`](../pzem-014-016-modbus) to PyPI. `manifest.json` requires
   `pzem-014-016-modbus==0.1.0`.
2. Replace `@sambartle` in `manifest.json` with your GitHub handle.
3. Copy both folders into a core checkout and make the repository-wide edits:

   ```bash
   # add to .strict-typing, then change `strict-typing` in quality_scale.yaml to done
   echo "homeassistant.components.pzem_014_016.*" >> .strict-typing   # keep the file sorted
   python3 -m script.gen_requirements_all
   python3 -m script.hassfest
   ```

   `hassfest` writes the `CODEOWNERS` line and `config_flows.py` /
   `integrations.json` entries.
4. Run the tests and type checks:

   ```bash
   pytest tests/components/pzem_014_016 --cov=homeassistant.components.pzem_014_016
   mypy homeassistant/components/pzem_014_016
   ```
5. The documentation page for `home-assistant.io` and the brand images are
   separate pull requests. The `docs-*` and `brands` rules in
   `quality_scale.yaml` are `todo` until they land, and the manifest declares no
   `quality_scale` tier yet.

## What was checked

The tests (35, 100 % coverage of the integration) pass against Home Assistant
2026.10.4 with `pytest-homeassistant-custom-component`, and the integration
type checks under `mypy --strict`. That release predates two APIs the `sofar`
example uses on `dev`: `async_setup_entry` / `async_unload_entry` returning
`None`, and `async_unload_platforms_or_raise`. This folder uses them as `sofar`
does, so it has to be run against a current `dev` checkout; the checks above ran
with those two functions adapted to the older bool-returning forms.

`hassfest` was not run, because it needs a full core checkout.

## Design notes

- The meter has no serial number, so the config entry's unique ID is its link
  endpoint and unit ID, and entity IDs hang off the entry ID. Reconfiguring to
  another link therefore keeps the entities and their history.
- Two coordinators, as in `sofar`: the measurements every 5 seconds, one request,
  and the settings every 60 seconds and after a write. A meter that refuses its
  holding registers still reports its measurements; only the numbers go
  unavailable.
- The **Modbus address** number is disabled by default. Writing it also moves the
  config entry's unit ID and unique ID and reloads, and is refused when another
  entry on the same link already holds that address.
- The meter's reset-energy (0x42) and calibration (0x41) function codes are not
  supported: `ModbusUnit` carries standard function codes only.
- `energy` is exposed in Wh by the meter and displayed in kWh.
