# Scene and be seen – A very much vibecoded Home Assistant blueprint

A Home Assistant automation blueprint that activates **scenes** (not just lights) from motion and presence sensors, with a strict active time window and an optional late-night scene tier.

## Features

- **Motion and presence sensors:** select one or more `binary_sensor` entities with device class `motion`, `occupancy` or `presence`. Lights stay on while *any* selected sensor is detecting.
- **Scene-based output:** one scene on detection and another when the area clears, so you can set brightness, colour temperature, multiple rooms or any other state you can capture in a scene.
- **Strict active window:** the automation only reacts to motion between a start and end time. Windows that cross midnight (e.g. 17:00 to 07:00) are supported.
- **Configurable no-motion delay:** all sensors must be clear for a set number of seconds before the "no motion" scene is applied.
- **Late-night tier (optional):** from a chosen time until the window ends, use a different on-scene and off-scene (e.g. a dim, warm scene after 23:00).
- **Fallback turn-off:** if you don't want a "no motion" scene, pick entities, areas or devices to simply turn off instead.
- **Handles people already in the room:** if someone is already detected when the window opens or the late period starts, the matching scene is applied immediately rather than waiting for the next motion event.

## Requirements

- Home Assistant **2024.10.0** or newer
- At least one motion, occupancy or presence binary sensor
- At least one scene to activate

## Installation

### Manual

1. Download `motion_scene_lighting.yaml`.
2. Copy it to `<config>/blueprints/automation/` (a subfolder such as `<config>/blueprints/automation/yourname/` is fine).
3. In Home Assistant, go to **Settings → Automations & Scenes → Blueprints** and confirm it appears (reload automations or restart if it doesn't).
4. Create an automation from the blueprint and fill in the inputs.

### Import from URL

Open **Settings → Automations & Scenes → Blueprints → Import Blueprint** and paste the raw GitHub URL of `motion_scene_lighting.yaml`.

## Inputs

### Sensors and timing

| Input | Default | Description |
|---|---|---|
| Motion / presence sensors | *(required)* | One or more binary sensors. The area counts as occupied while any of them is `on`. |
| No-motion delay (seconds) | `120` | How long all sensors must be clear before the "no motion" action runs. |
| Active window start | `17:00:00` | Motion is ignored before this time. |
| Active window end | `07:00:00` | Motion is ignored from this time. If earlier than the start time, the window runs overnight. |

### Scenes (normal period)

| Input | Default | Description |
|---|---|---|
| Scene on motion | *(required)* | Activated when motion is detected inside the window. |
| Scene on no motion | *(empty)* | Activated when the no-motion delay expires. If empty, the fallback targets are turned off instead. |

### Late-night scenes (optional)

| Input | Default | Description |
|---|---|---|
| Late period start | `23:00:00` | From this time until the window end, the late scenes are used. Has no effect unless a late scene is set. |
| Late scene on motion | *(empty)* | Used on motion during the late period. If empty, the normal scene is used. |
| Late scene on no motion | *(empty)* | Used on no motion during the late period. If empty, the normal no-motion behaviour is used. |

### Fallback turn-off targets (optional)

| Input | Default | Description |
|---|---|---|
| Entities to turn off | *(empty)* | Only used when no "no motion" scene applies. Anything selected is turned off with `homeassistant.turn_off`. |

## Example

Hallway lighting that is bright in the evening and dim overnight:

| Input | Value |
|---|---|
| Sensors | `binary_sensor.hallway_motion` |
| No-motion delay | `90` seconds |
| Window | `17:00` to `07:00` |
| Scene on motion | `scene.hallway_evening` |
| Scene on no motion | `scene.hallway_off` |
| Late period start | `23:00` |
| Late scene on motion | `scene.hallway_night_dim` |
| Late scene on no motion | `scene.hallway_off` |

Result:

- **17:00 to 23:00:** motion triggers *Hallway Evening*, and 90 seconds after the last motion, *Hallway Off*.
- **23:00 to 07:00:** motion triggers *Hallway Night Dim* instead, and the area switches off 90 seconds after the last motion.
- **07:00 to 17:00:** motion is ignored.

## How it works

1. **Triggers:** a sensor changes to `on`, the window start time is reached, or the late period start time is reached.
2. **Conditions:** the current time is inside the active window, and at least one sensor is currently detecting.
3. **On action:** the late scene is activated if the late period is active and a late scene is set. Otherwise the normal scene is used.
4. **Wait:** the automation waits for all sensors to be clear for the configured delay. If motion returns during the delay, the wait starts again.
5. **Off action:** the late no-motion scene, normal no-motion scene or fallback turn-off is applied, in that order of preference.

### Behaviour notes

- **The off action is not restricted by the time window.** If the last motion happens just before the window ends, the lights still switch to their off state afterwards, so they are never left on.
- **The scene tier is chosen once, when the automation triggers.** The on-scene and the later off-scene always come from the same tier, even if the late period or the window ends during the delay.
- **Mode is `restart`.** New motion re-applies the on-scene and resets the delay.

## Limitations

- **No manual override handling.** If someone changes the lights by hand while a sensor is still detecting, the blueprint won't reapply the scene until the sensor next changes from clear to detecting.
- **No light-level (lux) condition.** The automation is purely time-based. If you want it to depend on how dark it is, add a lux condition to the generated automation or ask for it as a feature.
- **Two scene tiers only:** normal and late.

## Troubleshooting

- **Lights never turn on:** check the current time is inside the active window, and that the sensor's state is actually `on` in **Developer Tools → States**.
- **Late scene never used:** make sure a late scene is set, and that the current time is between the late period start and the window end.
- **Lights don't turn off:** set a "no motion" scene or fallback targets. If neither is set, nothing happens when the area clears.
- **Blueprint won't load:** check the Home Assistant version and look for details in **Settings → System → Logs**.
