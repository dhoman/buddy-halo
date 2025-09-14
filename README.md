Bobblehead Notifier (Pi Zero 2 W)

Tiny LAN device that reacts to webhooks (e.g., Claude Code events) with:

a single RGB LED (status color/pattern),

OLED text (SSD1306),

custom MP3/WAV sounds,

optional listen window via USB mic (voice-activity timed).

1) Hardware (quick checklist)

Raspberry Pi Zero 2 W + 16–32GB microSD + 5V/2.5A PSU

USB hub (OTG) for USB speaker + USB mic

OLED 128×64 SSD1306 (I²C) → 3V3, GND, SDA=GPIO2, SCL=GPIO3

Single WS2812 “NeoPixel” LED (5mm diffused)

Data: Pi GPIO → 330–500Ω resistor → 74AHCT125 level shifter → LED DIN

Power: LED 5V & GND + 1000µF cap across 5V/GND near LED

2) Directory layout
bobblehead/
├─ app/
│  ├─ __init__.py
│  ├─ main.py                 # Flask app: /evt, /healthz, /config
│  ├─ hw_led.py               # rpi_ws281x single-LED helpers
│  ├─ hw_display.py           # SSD1306 text/graphics helpers
│  ├─ hw_audio.py             # play_mp3(), volume control
│  ├─ hw_mic.py               # optional: VAD (listen window)
│  ├─ events.py               # event routing + state machine
│  └─ config.py               # load .env / YAML
├─ clips/
│  ├─ run_complete.mp3
│  ├─ permission_required.mp3
│  ├─ waiting.mp3
│  └─ error.mp3
├─ config/
│  ├─ config.yaml
│  └─ .env.example
├─ service/
│  └─ bobblehead.service      # systemd unit
├─ scripts/
│  ├─ demo_send.py            # send sample webhooks to the device
│  └─ install.sh              # apt/pip + systemd setup
├─ README.md
└─ requirements.txt

3) Software stack

OS: Raspberry Pi OS Lite (32-bit)

Python runtime: 3.11+ recommended

Frameworks:

Web: Flask (or FastAPI)

LED: rpi_ws281x / adafruit-circuitpython-neopixel (choose one; rpi_ws281x is common)

Display: luma.oled (SSD1306 I²C)

Audio: mpg123 (CLI) or pygame; recommend mpg123 for simplicity

Mic/VAD (optional): sounddevice (or pyaudio)

4) Install (apt/pip)

Apt packages:

sudo apt update
sudo apt install -y git python3-pip python3-venv mpg123 \
  i2c-tools python3-dev libopenblas-dev # (sounddevice deps)


Enable I²C + audio:

sudo raspi-config  # Interface Options -> I2C: Enable; finish & reboot


Project setup:

git clone <your-repo> bobblehead && cd bobblehead
python3 -m venv .venv
source .venv/bin/activate

# requirements.txt (example)
# Flask
# luma.oled
# rpi-ws281x
# sounddevice
# python-dotenv
# pyyaml
pip install -r requirements.txt


If you use adafruit-circuitpython-neopixel instead of rpi-ws281x, install that and remove rpi-ws281x.

5) Configuration

config/config.yaml (example):

server:
  host: 0.0.0.0
  port: 8080
  bearer_token: "CHANGE_ME"

display:
  enabled: true
  i2c_bus: 1
  width: 128
  height: 64

led:
  enabled: true
  gpio: 18       # PWM-capable pin preferred with rpi_ws281x
  brightness: 0.2

audio:
  enabled: true
  player: "mpg123"   # or "pygame"
  volume: 85         # 0..100 (you can map to amixer/alsa)

mic:
  enabled: false
  device_name: ""    # arecord -l to discover
  rms_threshold: 0.02
  min_listen_sec: 5
  silence_timeout_sec: 10

mapping:            # event -> behavior
  permission_required:
    led: "red_pulse"
    sound: "permission_required.mp3"
    display: "Permission needed:\n{summary}"
    listen_window: true
  run_complete:
    led: "green_pulse"
    sound: "run_complete.mp3"
    display: "Run complete: {task_id}"
  waiting:
    led: "amber_blink"
    sound: "waiting.mp3"
    display: "Waiting…"
  error:
    led: "magenta_flash"
    sound: "error.mp3"
    display: "Error!\n{summary}"

idle:
  led: "blue_breathe"
  display: "Idle"


.env.example:

BH_BEARER_TOKEN=CHANGE_ME
BH_CONFIG=config/config.yaml

6) Flask endpoints (behavior outline)

POST /evt

Auth: Authorization: Bearer <token>

Body (JSON):

{
  "source": "claude_code",
  "event": "permission_required",
  "task_id": "abc123",
  "summary": "Tool use requested: openai.files.create(...)",
  "meta": {}
}


Action: route with events.py: set LED state, display text (with wrapping), play clip, optionally open listen window (mic VAD).

GET /healthz → 200 OK with json {uptime, last_event_ts, devices: {display, led, audio, mic}}

POST /config/reload → reload YAML without restart (optional)

(Optional) GET /ui → minimal test page (buttons to trigger sample events)

7) LED states (single NeoPixel)

red_pulse, green_pulse, amber_blink, magenta_flash, blue_breathe, off

Implement animation ticks in a small loop or via non-blocking scheduler.

Reliability tips (Pi + NeoPixel):

Use 74AHCT125 level shifter + series resistor (330–500Ω).

Put 1000µF cap across LED 5V/GND.

Prefer GPIO18 (PWM) with rpi_ws281x for smooth timing.

8) Audio

Place MP3s in clips/.

Playback: subprocess.run(["mpg123", "-q", "clips/run_complete.mp3"])

Volume: either pre-gain your clips or expose a config value and call amixer (set PCM).

9) Mic (optional “listen window”)

On certain events (e.g., permission_required):

Start capture for 5s minimum, then continue until 10s of silence (RMS under threshold), whichever comes first.

You can log a boolean “heard activity” → emit a follow-up event or just use it as confirmation.

STT can be added later (offload to a server).

10) systemd service

service/bobblehead.service:

[Unit]
Description=Bobblehead Notifier
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/bobblehead
Environment=PYTHONUNBUFFERED=1
EnvironmentFile=/home/pi/bobblehead/config/.env
ExecStart=/home/pi/bobblehead/.venv/bin/python -m app.main
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target


Install & enable:

sudo cp service/bobblehead.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now bobblehead
sudo systemctl status bobblehead

11) Local testing

Send a sample event from your PC:

curl -X POST http://<pi-ip>:8080/evt \
  -H "Authorization: Bearer CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{"source":"claude_code","event":"permission_required","task_id":"abc123","summary":"Tool use requested: openai.files.create(...)"}'


You should see: LED red/pulsing, “Permission needed…” on OLED, and the permission sound.

12) Security & networking

Keep the service LAN-only; block WAN.

Use a random, long bearer token.

If you need to receive from the internet, run a Cloudflare Worker / n8n that validates upstream secrets and relays to the Pi (never expose the Pi directly).

Optional: MQTT (Mosquitto) → device subscribes to claude/events/#.

13) Starter features (weekend)

Event router (permission_required / run_complete / waiting / error)

LED state machine (solid, pulse, blink, breathe)

SSD1306 text with wrapping + spinner glyph

MP3 playback per event

Config via YAML + .env token

/healthz endpoint

systemd autostart

14) Endgame features (week+)

Local Web UI (test events, upload sounds, set idle color/brightness/volume)

Profiles/Themes (swap sound packs & color maps)

Metrics (Prometheus text endpoint: events counts, mic sessions, uptime)

Queue & retry (write incoming events to disk; backoff on failures)

OTA-ish updates (pull from Git tag on boot if version changed)

STT/TTS (optional) (offload STT; return text as a new event)

Multi-device fan-out via MQTT or relay

15) Nice-to-haves

Acknowledge/Mute button (GPIO) to stop sound and clear red state

Night mode (dim LED/display by schedule)

Self-test on boot (brief LED flash + sine beep + I²C ping)


