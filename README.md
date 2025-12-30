# UNO Q Pin Toggle

> Real-time GPIO control interface for Arduino UNO Q

![UNO Q Pin Toggle Web Interface](assets/docs_assets/web-ui-screen.png)

---

##  Overview

The **UNO Q Pin Toggle** application provides a modern, interactive web interface for controlling every pin on the Arduino UNO Q board. Toggle digital pins, analog pins, and RGB LED channels in real-time through an intuitive browser-based dashboard.

###  Key Features

- **Real-time Control** — Instant pin state updates via WebSocket communication
- **Interactive Board View** — Visual representation of the Arduino UNO Q with clickable toggle switches
- **RGB LED Control Panel** — Dedicated panel showing active LED channels with visual feedback
- **Activity Log** — Track all pin state changes with timestamps
- **Pin Statistics** — Monitor active pins at a glance
- **Quick Actions** — Bulk operations to turn all pins ON or OFF
- **Connection Status** — Live indicator showing WebSocket connection state

---

##  Brick Used

This example uses the following Brick:

| Brick | Description |
|-------|-------------|
| `web_ui` | Creates a web interface to display the board pins control dashboard |

---

##  Hardware and Software Requirements

### Hardware

| Component | Quantity |
|-----------|----------|
| Arduino UNO Q | 1 |
| USB-C® to USB-A Cable | 1 |

### Software

- Arduino App Lab

> **💡 Tip:** You can run this application using your Arduino UNO Q as a Single Board Computer (SBC). I use this adapter that I got on Amazon: [USB-C hub](https://a.co/d/i1lxgC6) and just add your own mouse, keyboard, and monitor.

---

##  How to Use

1. **Run the App** — Launch the application from Arduino App Lab
2. **Open in Browser** — Navigate to `<UNO-Q-IP-ADDRESS>:7000` in your web browser
3. **Control Pins** — Click on any toggle switch to change pin states
4. **Monitor LEDs** — Watch the RGB LED Control panel to see which LED channels are active

---

##  How It Works

###  Backend (`main.py`)

The Python backend handles all communication between the web interface and the microcontroller:

- **Pin Mapping** — Defines and maps all Arduino UNO Q board pin names
- **Secure Communication** — Helper functions manage frontend and MCU communication
- **REST API** — Exposes `/states` endpoint for retrieving current pin states
- **WebSocket Events** — Real-time bidirectional communication with the frontend

###  Frontend (`index.html` + `app.js`)

The web interface provides an intuitive control experience:

- **Socket.IO Connection** — Real-time communication with the backend
- **Board Visualization** — UNO Q illustration with toggle switches on each pin
- **RGB LED Indicators** — Visual feedback showing active LED channels with glow effects
- **Dynamic Scaling** — Responsive layout that scales switches with the board image
- **Activity Logging** — Records all pin state changes with timestamps

---

##  Understanding the Code

Once the application is running, access it from your web browser at `<UNO-Q-IP-ADDRESS>:7000`. The system performs the following operations:

### Serving the Web UI

The UI is hosted by the `WebUI` Brick and communicates with the backend via WebSocket (Socket.IO) and a REST API.

```python
from arduino.app_bricks.web_ui import WebUI

...

ui = WebUI()
ui.on_message("pin_toggle", on_pin_toggle)           # WebSocket event
ui.expose_api("GET", "/states", on_get_states)       # REST: current states
```

| Endpoint | Type | Description |
|----------|------|-------------|
| `pin_toggle` | WebSocket | Receives toggle requests from the browser |
| `GET /states` | REST | Returns the logical ON/OFF state of every pin |

### Processing Toggle Requests

When the browser flips a switch, it emits a `pin_toggle` message. The backend:

1. **Validates** and parses the payload (accepts dict/JSON/bytes)
2. **Updates** the logical state (`pin_states[name] = logical`)
3. **Converts** to the hardware level (respecting `active_low`)
4. **Calls** the MCU via the Bridge
5. **Broadcasts** the new logical state to all connected clients

```python
def on_pin_toggle(sid, message):
    try:
        data = _ensure_dict(message)
        name = data.get("name")
        if name not in PIN_NAMES:
            raise ValueError(f"Unknown Pin '{name}'")

        # Logical state from UI ("on"/"off", 1/0, true/false -> bool)
        logical = _normalize_state(data.get("state"))
        pin_states[name] = logical

        # Apply active-low for the MCU call
        state_for_hw = _state_for_hw(name, logical)

        Bridge.call("set_pin_by_name", name, state_for_hw)

        print(f"[{_iso_now()}] [{sid}] {name} -> logical={'ON' if logical else 'OFF'} hw={state_for_hw}")
        ui.send_message("pin_state_update", {
            "name": name,
            "state": logical,           # broadcast logical state to clients
            "timestamp": _iso_now()
        })

    except Exception as e:
        ui.send_message("error", f"Pin toggle error: {e}")
```

### MCU Communication via RouterBridge

The firmware exposes a boolean RPC through `Arduino_RouterBridge`. The backend calls it by pin name and state.

```cpp
#include <Arduino_RouterBridge.h>

// Generic setter: name + bool
void set_pin_by_name(String name, bool s) {
  int idx = findIndex(name.c_str());
  if (idx < 0) return;              // unknown name
  digitalWrite(kPins[idx].pin, s ? HIGH : LOW);   // Python already applied active_low
}

void setup() {
    for (auto &e : kPins) pinMode(e.pin, OUTPUT);  // set modes for all pins

    ...
    
    Bridge.begin();
    Bridge.provide("set_pin_by_name", set_pin_by_name);
}

void loop() {}
```

**Key implementation details:**

- The sketch maps device-tree indexes to Arduino pin numbers
- All relevant pins are set to `OUTPUT` mode
- RGB channels default to `HIGH` in `setup()` to keep active-low LEDs **off** at boot
- Each `Bridge.provide("<name>", <fn>)` pairs with the Python call for simple boolean pin control

---
