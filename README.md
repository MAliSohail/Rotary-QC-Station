# Rotary Defect Inspection Station

A Flask-based inspection station that rotates a part through four camera angles and classifies each view automatically using a trained image classification model. Built as part of a team project (Advanced Embedded Systems, HSHL). The flagship demo use case is mold detection on a food production line, but the turntable/camera/classifier pipeline works with any item and any visual defect the model is trained to catch.

## How it works

1. An operator places a part on the turntable and presses the Arduino button.
2. The Arduino notifies the Raspberry Pi over HTTP.
3. The Pi drives the Arduino turntable to four fixed angles (0, 60, 120, 180 degrees).
4. At each angle the Pi sends a capture command to the ESP32-CAM over MQTT.
5. The ESP32-CAM takes a photo and uploads the original JPEG to the Pi over HTTP.
6. The Pi runs the image through a TFLite classification model and records a PASS or REJECT label with a confidence score for that view.
7. Once all four views are captured and classified, the Pi computes the final result. Any rejected view produces a final REJECT; four passed views produce a final PASS.
8. Hardware or communication failures at any stage produce a SYSTEM_ERROR result instead.
9. Every inspection is saved in its own folder with a `result.json` containing the full record.

The dashboard shows live status, the four captured images, and each view's classification and confidence as the inspection runs.

## My contribution

- ESP32-CAM firmware (`firmware/esp32_mqtt`) — image capture, MQTT command handling
- Arduino turntable firmware (`firmware/arduino_http`) — rotation control over HTTP
- Pi-side coordination layer (`services/camera_mqtt.py`, `services/arduino.py`) — MQTT/HTTP messaging, command acknowledgement, timeout and failure handling
- Flask dashboard (`app.py`, `templates/`, `static/`)
- Integrating the trained classifier into the pipeline (`services/classifier.py`)

The classification model itself (`model.tflite`) was trained by a teammate as part of the wider team project. The full team repo, including SysML documentation and a manual-inspection variant, is here: [link to team repo].

## Project structure

```text
rotary-defect-inspection-station/
├── app.py                      # Flask routes and inspection sequence
├── config.py                   # IP addresses, ports, topics, timeouts, model settings
├── config.env.example          # Template — copy to config.env and fill in
├── model.tflite                 # Trained image classification model
├── requirements.txt
├── setup.sh
├── start.sh
├── services/
│   ├── arduino.py               # Arduino HTTP client
│   ├── camera_mqtt.py           # MQTT capture command handling
│   ├── classifier.py            # TFLite inference and image preprocessing
│   └── storage.py               # Inspection folders and result.json
├── templates/
│   └── dashboard.html
├── static/
│   ├── dashboard.css
│   └── dashboard.js
├── firmware/
│   ├── arduino_http/
│   │   ├── arduino_http.ino
│   │   └── secrets.h            # Wi-Fi + Pi address — fill in your own
│   └── esp32_mqtt/
│       ├── esp32_mqtt.ino
│       └── secrets.h            # Wi-Fi + Pi/MQTT address — fill in your own
└── data/
    └── inspections/              # Created at runtime, one folder per inspection
```

## Installation

### Raspberry Pi (coordinator)

```bash
git clone https://github.com/MAliSohail/Rotary-Defect-Inspection-Station.git
cd Rotary-Defect-Inspection-Station
chmod +x setup.sh start.sh
./setup.sh
cp config.env.example config.env
nano config.env   # set the Arduino's IP address
```

Confirm the MQTT broker (Mosquitto) is running on the Pi:

```bash
sudo systemctl status mosquitto --no-pager
```

Start the server:

```bash
./start.sh
```

Open the dashboard from any device on the same network:

```text
http://<PI_IP>:5000
```

### Firmware (ESP32-CAM and Arduino)

For each sketch under `firmware/`:

1. Fill in your Wi-Fi credentials and the Pi's IP address in `secrets.h`.
2. Upload through the Arduino IDE.

## Classification model

The station uses a TFLite model (`model.tflite`) exported from Teachable Machine. Input is a 224x224 RGB image; output is a two-class softmax score (PASS, REJECT).

Preprocessing before inference:

1. Center-crop the captured JPEG to a square.
2. Resize to 224x224 using nearest-neighbor interpolation.
3. Scale pixel values to the range -1 to 1.

To use a different model, replace `model.tflite` and update `CLASSIFIER_MODEL_PATH`, `CLASSIFIER_LABELS`, and `CLASSIFIER_INPUT_SIZE` in `config.py` to match.

## Main API routes

```text
GET  /
GET  /api/health
GET  /api/status
POST /api/inspection/start
POST /api/inspection/<id>/image
GET  /api/inspection/<id>/image/<view>
```

## Inspection output

```text
data/inspections/QC-YYYYMMDD-HHMMSS-xxx/
├── view_1_000deg.jpg
├── view_2_060deg.jpg
├── view_3_120deg.jpg
├── view_4_180deg.jpg
└── result.json
```

Each view in `result.json` records its classification label, confidence, and raw scores from the model.

## Known limitations

This is a working prototype, not a certified inspection system — the model would need training on real samples of the target item before it could be trusted beyond testing and demonstration. It inspects one item at a time (no conveyor integration or batching), and it does not currently resume an in-progress inspection if the Pi restarts mid-run.
