# 🚪 GATEKEEPER AI
### An Agentic AI-Driven Detection and Response System

**Manalo, Guian Jaundell R.**

Gatekeeper AI is a smart door notification system built around an ESP32, a PIR motion sensor, a 16×2 I2C LCD, and a local AI agent. When someone approaches the door, the ESP32 pings a local Python agent, which uses a locally-hosted LLM (via LM Studio) to check the owner's weekly schedule and generate a short, friendly status message. That message is shown on the LCD and simultaneously pushed to the owner as a Telegram notification — no need to answer the door in person.

The project was built to cut down on interruptions during online classes, studying, and other focused work, while still giving visitors a clear, real-time answer about availability.

---

## ✨ Features

| Feature | Description |
|---|---|
| **PIR Motion Detection** | ESP32 watches for visitors and triggers the agent on motion |
| **Agentic Schedule Checking** | The LLM calls tools (`get_current_time`, `check_calendar`, `check_research_mode`) to reason about availability rather than following a fixed script |
| **AI-Generated LCD Messages** | Produces short, friendly two-line messages sized for a 16×2 display |
| **Auto Line-Wrap + Scrolling LCD** | Long messages are split into lines and auto-scrolled on the display |
| **Telegram Notifications** | Sends a real-time alert to the owner every time a visitor is detected |
| **Conversational Schedule Manager** | A terminal chat lets the owner add, remove, update, or clear schedule events in natural language |
| **Persistent Schedule Storage** | Schedule is saved to and loaded from `schedule.json` |
| **Manual Do-Not-Disturb Mode** | `/research-mode` endpoint toggles a manual override flag the AI can check |
| **Buzzer Alerts (optional)** | Short beeps on motion detection and when a message is ready |
| **Cooldown State Machine** | ESP32 runs an Idle → Displaying → Cooldown loop to prevent repeat triggers |

---

## 🔩 Hardware Components

- ESP32 development board
- PIR motion sensor
- 16×2 I2C LCD
- Buzzer *(optional)*
- Breadboard + jumper wires
- 5V–12V DC power supply

### Pin Configuration

| Component | ESP32 Pin |
|---|---|
| PIR Sensor | GPIO 13 |
| Buzzer *(optional)* | GPIO 14 |
| LCD SDA | GPIO 21 |
| LCD SCL | GPIO 22 |

---

## 🗂️ Project Structure

```
project/
├── src/
│   └── gatekeeper_esp32.ino    # ESP32 firmware — PIR sensing, LCD display
│   ├── gatekeeper_agent.py         # Python agent — Flask server + LLM tool-calling + Telegram connection
├── schedule.json               # Auto-generated weekly schedule (created on first save)
└── README.md
```

The firmware lives in `src/` following the standard PlatformIO project layout. If you're using the Arduino IDE instead of PlatformIO, just open `src/gatekeeper_esp32.ino` directly.

---

## ⚙️ Requirements

### ESP32 Firmware Environment

Developed in **VS Code with the PlatformIO extension**. If you'd rather not use PlatformIO, the firmware is plain Arduino-framework code — just grab `gatekeeper_esp32.ino` from the `src` folder and open it directly in the Arduino IDE instead.

Required libraries either way:

- `WiFi.h`
- `HTTPClient.h`
- `ArduinoJson`
- `Wire.h`
- `LiquidCrystal_I2C`

### Python Packages

```bash
pip install flask openai requests
```

### External Services

- **LM Studio** — running locally at `http://localhost:1234/v1`, with a tool-calling-capable model loaded (developed against `qwen/qwen3-4b-2507`)
- **Telegram Bot** — a bot token and chat ID for push notifications

---

## 🔧 Configuration

### `gatekeeper_agent.py`

Update the constants near the top of the file:

```python
LM_STUDIO_URL    = "http://localhost:1234/v1"   # LM Studio local API endpoint
LM_MODEL         = "qwen/qwen3-4b-2507"         # Model name loaded in LM Studio
TELEGRAM_TOKEN   = "Your_Token"                 # Telegram bot token
TELEGRAM_CHAT_ID = "TChat_ID"                   # Telegram chat ID to notify
SCHEDULE_FILE    = "schedule.json"              # Where the weekly schedule is persisted
```

If no `schedule.json` exists yet, the agent falls back to a built-in default weekly schedule and will create the file the first time an event is added, updated, or removed.

### `gatekeeper_esp32.ino`

```cpp
const char* WIFI_SSID     = "Wifi_Name";
const char* WIFI_PASSWORD = "Wifi_Password";
const char* AGENT_HOST    = "http://192.168.1.6:5050/check";  // IP of the machine running the Python agent

#define BUZZER_PIN  14   // Set to -1 if no buzzer is connected
```

Set `AGENT_HOST` to the local IP address (and port `5050`) of the computer running `gatekeeper_agent.py`.

The buzzer is optional — if you don't have one wired up, set `BUZZER_PIN` to `-1` and the firmware will skip all beep calls automatically.

---

## 🚀 Running the Project

### 1. Upload the ESP32 Firmware

Set your Wi-Fi credentials and `AGENT_HOST` in `src/gatekeeper_esp32.ino`, then upload it to the ESP32 using either:
- **VS Code + PlatformIO** — open the project folder and use the PlatformIO Upload button, or
- **Arduino IDE** — open `src/gatekeeper_esp32.ino` directly and upload as usual (no PlatformIO required).

### 2. Start LM Studio

Load a tool-calling-capable model in LM Studio and make sure its local server is running on `http://localhost:1234`.

### 3. Run the Python Agent

```bash
python gatekeeper_agent.py
```

This starts two things at once:
- A **Flask server** on `http://0.0.0.0:5050` that the ESP32 calls on motion detection
- An interactive **schedule chat** in the terminal for managing your weekly schedule conversationally

### 4. Test the System

Trigger the PIR sensor to simulate a visitor. Confirm the LCD shows the AI-generated message and that a Telegram notification arrives.

---

## 💬 Schedule Chat (Terminal)

Once the agent is running, the terminal drops into a conversational schedule manager. Example inputs:

```
I will go outside from 1pm to 3pm today
Modify my valorant session to end at 4pm
Clear today and add: outside until 5pm
Show my full schedule
Remove Thursday software design class
```

Type `exit` to quit the chat (the Flask server keeps running independently until the process ends).

---

## 🌐 Flask Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/check` | POST | Called by the ESP32 on motion detection; runs the door agent and returns the LCD message |
| `/health` | GET | Returns current time, calendar status, and schedule count |
| `/schedule` | GET | Returns the full current weekly schedule as JSON |
| `/research-mode` | POST | Toggles manual do-not-disturb mode (`{"active": true/false}`) |

---

## 🧠 How the Door Agent Works

1. The PIR sensor detects motion near the door.
2. The ESP32 shows a temporary "Checking schedule" message and sends a `POST /check` request to the Flask agent.
3. The agent runs an LLM conversation with tool access. The model calls `get_current_time`, then `check_calendar` (and optionally `check_research_mode`) to determine availability.
4. The LLM composes a short, LCD-friendly message (max 32 characters, two 16-character lines), for example:
   - `"In online class. Back at 10:00 AM"`
   - `"Free right now! Please knock :)"`
5. The message is returned to the ESP32 as JSON and displayed on the LCD, auto-scrolling if it spans more than two lines.
6. At the same time, the agent sends a Telegram notification containing the visitor alert and the displayed message.
7. The ESP32 holds the message for a display period, then enters a cooldown state before returning to idle and waiting for the next visitor.

---

## 🔁 ESP32 State Machine

```
STATE_IDLE
   │  PIR motion detected
   ▼
STATE_DISPLAYING  →  shows AI message, scrolls if needed, beeps buzzer
   │  display duration elapsed
   ▼
STATE_COOLDOWN    →  short countdown before re-arming
   │  cooldown elapsed
   ▼
STATE_IDLE
```

Timing constants (`gatekeeper_esp32.ino`):

| Constant | Value | Purpose |
|---|---|---|
| `DISPLAY_DURATION_MS` | 20000 | How long the AI message stays on the LCD |
| `SCROLL_INTERVAL_MS` | 2500 | Time between vertical scroll steps for long messages |
| `COOLDOWN_MS` | 10000 | Cooldown period before the sensor can trigger again |

---

## ⚠️ Notes & Limitations

- The LLM model must support **tool/function calling** for the door agent to work correctly.
- `AGENT_HOST` in the firmware must point to the correct local IP and port (`5050`) of the machine running the Python agent — update it if your network changes.
- Telegram notifications are silently skipped if `TELEGRAM_TOKEN` is left as the placeholder value.
- The schedule chat runs in the same terminal process as the Flask server; closing the chat with `exit` does not stop the Flask thread, but closing the whole process does.
- LCD messages are capped at 64 characters by the agent before being sent to the ESP32.
- The buzzer is fully optional — the circuit and firmware both work fine without one as long as `BUZZER_PIN` is set to `-1`.

---

## 🔐 Security Reminder

`TELEGRAM_TOKEN` and `TELEGRAM_CHAT_ID` are sensitive credentials. Avoid committing real values to a public repository — consider moving them to environment variables or a local config file excluded via `.gitignore`:

```
# .gitignore
schedule.json
.env
```

---

## 📄 Academic Context

This project was submitted as the Final Examination for **Feedback and Control Systems**, College of Engineering and Architecture, University of Batangas – Lipa Campus (May 2026).

## 📄 License

This project is for personal and academic use. Modify and extend freely.