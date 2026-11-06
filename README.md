# HomelabRadar

An mmWave presence sensor for Home Assistant, built with ESPHome. An HLK-LD2450 radar tracks up to 3 people. A 0.96" OLED shows them live while someone is in the room, and two push buttons control it.

## Hardware
- Seeed Studio XIAO ESP32S3, with a 2.4 GHz U.FL antenna (the board has no onboard antenna)
- HLK-LD2450 24 GHz radar
- 0.96" SSD1306 OLED, 128×64, I2C
- 2 momentary push buttons

## Wiring
![Wiring diagram: XIAO ESP32S3 to the LD2450 radar, SSD1306 OLED and two push buttons](RadarDiagram.png)

The XIAO is drawn from the component side with USB-C at the top. To edit the diagram, open `docs/wiring.excalidraw` at https://draw.lab.richter.codes, then use Export image → PNG to replace `RadarDiagram.png`.

| XIAO pin | Connects to |
|---|---|
| 5V | LD2450 5V |
| GND | LD2450 GND, OLED GND, both buttons |
| D7 (GPIO44) | LD2450 TX |
| D6 (GPIO43) | LD2450 RX |
| 3V3 | OLED VCC |
| D9 (GPIO8) | OLED SDA |
| D10 (GPIO9) | OLED SCL |
| D0 (GPIO1) | Top button (other leg to GND; uses the internal pull-up) |
| D1 (GPIO2) | Bottom button (other leg to GND; uses the internal pull-up) |

The LD2450 runs on 5 V, but its UART uses 3.3 V logic, so it connects directly. Power the OLED from 3V3, not 5V. Many modules pull SDA/SCL up to their VCC.

Mount the radar with its long edge vertical and the antenna side (no connector) facing the room, 1.5–2 m high. It covers ±60° horizontally and up to 6 m.

## Display
The screen is off unless someone is tracked, which limits OLED burn-in. It turns on as soon as the radar sees someone and off 30 s after the room is clear (`screen_timeout_ms`), returning to the Map each time. Contrast is 60% and the picture shifts by 1 px each minute for the same reason. Everything is drawn toward the centre because the recessed opening hides the panel's edges.

| Button | Does |
|---|---|
| Top | Next view. Wakes the screen if it's off. |
| Bottom | Screen off now; it stays off until the room has been clear for 30 s. Wakes the screen if it's off. |
| Both, release after 5 s | Reboot. The screen shows a countdown; release sooner to cancel. |
| Both, hold 15 s | Wi-Fi setup (see below). In setup mode, the same hold cancels it. |

If the buttons come out reversed, swap `top_pin` and `bottom_pin` at the top of `radar.yaml`.

## Changing Wi-Fi
1. Hold both buttons for 15 s. The board reboots and shows "Wi-Fi setup".
2. After about 5 s, join the **Radar Setup** network from a phone. Its password is `fallback_ap_password` in `secrets.yaml`.
3. The setup page should open on its own; otherwise go to `http://192.168.4.1`. Pick the network and enter its password.

The board joins the new network and keeps using it after reboots and power cuts. Setup mode survives a reboot until new details are saved.

To cancel, hold both buttons for 15 s again while it shows "Wi-Fi setup". That clears the saved Wi-Fi, and the board goes back to the network built into the firmware (`wifi_ssid` in `secrets.yaml`). Separately, whenever the board can't reach its network for 90 s, it opens "Radar Setup" by itself.

Wi-Fi saved this way is tied to the exact firmware build. Flashing a changed `radar.yaml`, or a new ESPHome version, drops it and the board goes back to the built-in network. Update `secrets.yaml` before flashing if the network has changed for good.

| View | Shows |
|---|---|
| Map | Top-down cone with 5/10/15/20 ft rings and a dot per person; count and nearest distance below |
| Status | "1 person" / "2 people" / "Clear" in large text, plus nearest distance |
| Info | IP address, Wi-Fi signal and uptime |

Left and right are drawn as seen by someone facing the radar. Set `x_sign: "1"` to use the radar's own view instead.

## Behavior
- **Radar settings at boot:** the firmware turns the LD2450's Bluetooth off, since any nearby phone could otherwise reconfigure it. It also sets a filter zone that ignores anything within ~2 ft of the radar.
- **Home Assistant entities:** Presence, Target Count, Nearest Distance (ft, at most once a second), Wi-Fi Signal, a Restart button, a Radar Bluetooth switch and Presence Timeout.
- **Raw positions stay on the device.** X/Y positions update ~10×/s for the display and aren't sent to Home Assistant.

## Build and flash
```sh
brew install esphome
cp secrets.yaml.example secrets.yaml   # fill in Wi-Fi and generate the keys
esphome run radar.yaml                  # first time over USB, then over Wi-Fi
```

The default LD2450 sensor filters in ESPHome 2026.10 throttle updates to once a second. This config overrides them so the display keeps up with the radar's ~10 Hz frames.
