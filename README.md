# Home Assistant LG washer price scheduler

A Home Assistant package that looks for the cheapest available continuous wash window and sends LG ThinQ the corresponding **delayed-finish** value. It does not start the washer.

The user selects the wash program on the washer and enables Remote Start there. Home Assistant reads the selected program's reported duration, compares hourly finish-delay choices against Nord Pool prices, and asks the washer to finish at the selected time. The package only runs while the washer is in its initial ready state. It deliberately refuses to replace an existing reserved schedule because LG's reported duration may change after reservation.

## Requirements

- Home Assistant with the official LG ThinQ integration.
- A washer exposing current status, total time, Remote Start, delayed-end number and delayed-end timestamp entities.
- A Nord Pool sensor exposing `raw_today` and `raw_tomorrow` price slots in EUR/kWh.
- A configured package directory, or manual merging of the YAML into the corresponding HA sections.

The exact LG entity set varies by model and integration version. Replace the generic entity IDs in the package with your own. The package uses `sensor.washer_current_status` and `binary_sensor.washer_remote_start` as examples.

## Install

1. Back up your HA configuration.
2. Edit the entity IDs in `lg_washer_price_scheduler.yaml` for the washer and Nord Pool sensor.
3. Set `input_number.washer_price_adders` to your own combined per-kWh delivery, tax and supplier-margin additions, in cents/kWh. Leave it at `0` if the Nord Pool sensor already includes them.
4. Place the file in your configured package directory. If packages are not enabled, merge the sections into `configuration.yaml` and the relevant helper/automation files instead.
5. Run Home Assistant's configuration check and reload the affected helpers/automations.
6. On the washer, select the desired program and enable Remote Start. In Home Assistant, set the desired finish deadline and press **Find cheapest wash window**.
7. Confirm the persistent notification says the washer acknowledged the delayed-finish schedule. The card should show the LG delayed-finish timestamp; do not derive a start time from `total_time` after the washer is reserved.

## What it does

- Evaluates integer-hour delayed-finish choices from 1 through 19 hours.
- Requires the complete estimated wash interval to fit before the requested deadline and within available Nord Pool slots.
- Computes a time-weighted average price across the interval and adds the configurable surcharge.
- Sends one delayed-finish value to LG and waits for the matching number, `reserved` state and finish timestamp before reporting success.
- Leaves the current washer schedule untouched when preconditions, price coverage or the LG acknowledgement are missing.
- Does not call a start-program operation.

The price estimate assumes energy use is spread evenly over the reported cycle duration. Actual washer power varies by phase, so this is a scheduling estimate rather than a measured-cost forecast.

## LG duration caveat

In one real 60 °C + Steam schedule, LG reported about 238–258 minutes before reservation, then changed `total_time` to 59 minutes when the status became `reserved`. The dashboard's former start-time calculation consequently showed a false one-hour cycle. This package calculates before requesting the delayed finish, refuses to reschedule while already reserved, and treats LG's delayed-end timestamp as the scheduled finish target. Verify behavior on your own washer and integration version.

## Entity mapping

Replace these generic IDs in the automation:

| Generic ID | Purpose |
| --- | --- |
| `binary_sensor.washer_remote_start` | LG Remote Start enabled |
| `sensor.washer_current_status` | LG washer state (`initial` when ready, `reserved` after scheduling) |
| `sensor.washer_total_time` | Selected program duration in minutes, read before scheduling |
| `number.washer_delayed_end` | LG relative delayed-finish setting in whole hours |
| `sensor.washer_delayed_end` | LG scheduled finish timestamp |
| `sensor.nordpool_price` | Nord Pool sensor with `raw_today` and `raw_tomorrow` |

## Limits

- LG exposes delayed finish in whole hours on the supported entity, so the optimizer cannot select arbitrary start times.
- Program duration comes from LG and can be revised by the washer.
- The schedule is sent to the washer; Home Assistant does not wait until the chosen start time to start it.
- This is a community example, not an official Home Assistant or LG project.

## License

MIT. See `LICENSE`.
