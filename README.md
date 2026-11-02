# MASTER README --- AI-Driven IoT Crop Monitoring and Tomato Disease Classification

> **Purpose:** Engineering handover and MVP runbook. This document
> records the implementation discussed in this project. Verify current
> local filenames, IP addresses and firmware before running commands. Do
> not treat planned features as completed.

## 1. Project summary

The system combines two independent data paths:

-   **Environmental telemetry:** DHT22, soil-moisture sensor and rain
    sensor connect to an ESP32-WROOM-32. The ESP32 sends JSON readings
    over Wi-Fi to a Flask API. Flask stores readings in SQLite and
    serves the dashboard.
-   **Leaf imaging and AI:** An ESP32-CAM captures a tomato-leaf image
    and exposes it through HTTP. Flask requests and stores the image.
    TensorFlow/Keras MobileNetV2 runs on the host computer, not on the
    ESP32-CAM, and produces ten-class predictions.

The MVP goal is one repeatable workflow: show real sensor readings,
capture a leaf image, run the existing model and display the actual
prediction. If automatic inference-to-dashboard integration is not yet
implemented, disclose the manual inference step.

## 2. Current known project state

Based on files and test outputs shared during development:

-   Sensor data has reached Flask and been stored in SQLite.
-   The dashboard has displayed ESP32-originated readings.
-   `POST /api/camera/capture` previously returned `success: True`, a
    filename and an image URL; the user confirmed the image opened.
-   PlantVillage tomato data was split into train, validation and test
    manifests.
-   TensorFlow 2.20.0 detected the RTX 3050 GPU in Ubuntu 22.04 / WSL2.
-   MobileNetV2 baseline and fine-tuning experiments reported 86.83% and
    92.15% held-out PlantVillage test accuracy.
-   A separate inference script generated top-three predictions for
    images.

**Must still be verified on the current checkout:** the current
ESP32-CAM firmware boots successfully, the camera capture route works
with its current IP, and the camera → Flask → model → dashboard path
runs automatically with one action.

## 3. Architecture

``` text
ENVIRONMENTAL PATH
DHT22 -----------\
Soil moisture ----> ESP32-WROOM-32 -- Wi-Fi / HTTP POST --> Flask Backend
Rain sensor -----/                                      |
                                                        v
                                                   SQLite Database
                                                        |
                                                        v
                                                   Web Dashboard

IMAGE / AI PATH
Tomato leaf --> ESP32-CAM --> Wi-Fi / HTTP image --> Flask Backend
                                                     |
                                                     v
                                                Saved image
                                                     |
                                                     v
                                      TensorFlow / MobileNetV2
                                                     |
                                                     v
                                      Top-3 class scores
                                                     |
                                                     v
                                                Dashboard
```

## 4. Hardware and wiring

  Component                 Function                            Interface
  ------------------------- ----------------------------------- ------------------
  ESP32-WROOM-32 / DevKit   Sensor controller                   GPIO, ADC, Wi-Fi
  DHT22                     Temperature and relative humidity   Digital
  Soil-moisture sensor      Soil measurement                    Analog
  Rain sensor module        Rain detection                      Digital
  ESP32-CAM                 Leaf image acquisition              Wi-Fi
  Camera sensor             Image capture                       Camera interface
  Laptop                    Flask, SQLite and AI inference      Wi-Fi / USB

Current sensor wiring reference:

  Signal                                    ESP32 pin
  ------------------ --------------------------------
  DHT22 DATA                                   GPIO 4
  Soil moisture AO                            GPIO 34
  Rain sensor DO                              GPIO 27
  Optional LDR AO      GPIO 35 (not required for MVP)

Use the actual circuit and board documentation as final authority. ESP32
GPIO inputs are not 5-V tolerant. Check sensor module output voltage and
use a common ground.

**Backend conventions:** - `soil_moisture` is the raw ADC value
(0--4095), not a percentage. - Current calibration constants in the
backend are `SOIL_DRY_ADC = 3200` and `SOIL_WET_ADC = 1200`. Calibrate
against the actual sensor and soil. - Rain convention: `rain = 0` means
rain detected; `rain = 1` means no rain. Verify the actual module's
active logic. - LDR is optional and not part of the core API.

## 5. Project directories

Windows project root used during development:

``` text
D:\VSCODE\PROjects\AI-Driven-IoT-System-for-Smart-Crop-Monitoring-and-Disease-Detection
```

WSL mount:

``` text
/mnt/d/VSCODE/PROjects/AI-Driven-IoT-System-for-Smart-Crop-Monitoring-and-Disease-Detection
```

Known project paths include:

``` text
project-root/
├── README.md
├── run.bat
├── backend/
│   ├── app.py
│   ├── sensor_data.py
│   ├── database.py             # verify in local checkout
│   ├── .env                    # local config/secrets; do not commit
│   └── .env.example
├── frontend/
│   ├── index.html
│   ├── app.js
│   └── captures/
├── hardware/
│   ├── crop_monitor/
│   │   └── crop_monitor.ino
│   └── esp32_cam/
│       └── esp32_cam.ino       # if present in current checkout
├── ai_model/
└── venv/                       # Windows Python environment
```

AI data and model files were stored separately in WSL:

``` text
~/crop_ai_data/
├── PlantVillage-Dataset/
├── prepare_dataset.py
├── splits/
│   ├── train.csv
│   ├── validation.csv
│   └── test.csv
└── models/
    ├── class_names.json
    ├── tomato_mobilenetv2_best.keras
    ├── tomato_mobilenetv2_finetuned_best.keras
    ├── tomato_mobilenetv2_finetuned_final.keras
    └── finetuning_history.csv
```

Some paths and filenames may differ; inspect the actual directory before
running a script.

## 6. Environment setup

### 6.1 Windows / PowerShell

Check WSL:

``` powershell
wsl --list --verbose
```

Launch Ubuntu 22.04:

``` powershell
wsl -d Ubuntu-22.04
```

### 6.2 Ubuntu / WSL2

Check the environment:

``` bash
lsb_release -a
python3 --version
nvidia-smi
```

The established model environment used Ubuntu 22.04.5 LTS and Python
3.10.12.

Activate TensorFlow environment:

``` bash
source ~/crop-tf/bin/activate
```

Verify TensorFlow:

``` bash
python -c "import tensorflow as tf; print('TensorFlow:', tf.__version__); print('GPU:', tf.config.list_physical_devices('GPU'))"
```

Previously verified: TensorFlow 2.20.0 detected an NVIDIA RTX 3050
Laptop GPU. Do not reinstall TensorFlow if this environment still works.

Optional GPU smoke test:

``` bash
python - <<'PY'
import tensorflow as tf
for gpu in tf.config.list_physical_devices("GPU"):
    tf.config.experimental.set_memory_growth(gpu, True)
with tf.device("/GPU:0"):
    a = tf.random.normal([2000, 2000])
    b = tf.random.normal([2000, 2000])
    c = tf.matmul(a, b)
print("Device:", c.device)
print("Shape:", c.shape)
PY
```

## 7. Dataset preparation

Dataset source: https://github.com/spMohanty/PlantVillage-Dataset

Ten tomato classes: 1. Bacterial spot 2. Early blight 3. Late blight 4.
Leaf mold 5. Septoria leaf spot 6. Two-spotted spider mites 7. Target
spot 8. Tomato yellow leaf curl virus 9. Tomato mosaic virus 10. Healthy

Run the preparation script:

``` bash
source ~/crop-tf/bin/activate
python ~/crop_ai_data/prepare_dataset.py
```

Previously reported result:

``` text
Train:      12,697 images
Validation:  2,723 images
Test:        2,726 images
Invalid images removed: 0
Exact duplicates removed: 14
Total usable images: 18,146
```

Verify split counts:

``` bash
python - <<'PY'
import pandas as pd
from pathlib import Path
base = Path.home() / "crop_ai_data/splits"
for name in ["train", "validation", "test"]:
    df = pd.read_csv(base / f"{name}.csv")
    print(name, len(df))
    print(df["label"].value_counts().sort_index())
PY
```

Do not regenerate split manifests after training unless intentionally
creating a new experiment. Preserve the manifests used to produce
reported metrics.

## 8. MobileNetV2 model training

The model uses ImageNet-pretrained MobileNetV2 as a feature extractor.
Its original classification head is replaced with a ten-class tomato
head. The input resolution is 224 × 224 × 3. The classification head
uses global average pooling, dropout (0.30), a dense layer and softmax
output.

### 8.1 Initial transfer learning

Documented configuration: - Adam optimizer, learning rate 0.001 - Sparse
categorical cross-entropy - Batch size 16 - Maximum 15 epochs - Seed
42 - Balanced class weights - Validation monitoring, checkpointing and
early stopping - Pretrained backbone frozen during initial head training

Expected checkpoint:

``` text
~/crop_ai_data/models/tomato_mobilenetv2_best.keras
```

Reported baseline test accuracy: **86.83%**.

### 8.2 Fine-tuning

The supplied `fine_tune.py` loads the best baseline checkpoint,
unfreezes the MobileNetV2 backbone, freezes all but the final 30
backbone layers and keeps BatchNormalization layers frozen.

Configuration: - Adam learning rate: `1e-5` - Maximum epochs: 12 - Batch
size: 16 - Image size: 224 × 224 - Early stopping patience: 4 -
ReduceLROnPlateau - Balanced class weights

Locate the script if necessary:

``` bash
find ~/crop_ai_data -maxdepth 3 -type f -name "*.py"
```

Run it using its actual filename/location, for example:

``` bash
python ~/crop_ai_data/fine_tune.py
```

Expected artifacts:

``` text
tomato_mobilenetv2_finetuned_best.keras
tomato_mobilenetv2_finetuned_final.keras
finetuning_history.csv
```

Do not retrain merely for the demonstration if the validated checkpoint
already exists.

### 8.3 Reported results

  Metric                         Baseline   Fine-tuned
  ------------------- ------------------- ------------
  Test accuracy                    86.83%       92.15%
  Macro precision       Not retained here       91.13%
  Macro recall          Not retained here       90.51%
  Macro F1-score        Not retained here       90.40%
  Weighted F1-score     Not retained here       92.28%

These metrics are from the held-out PlantVillage test subset, not an
independently labelled field dataset.

## 9. Model inference

The inference pipeline must: 1. Load the fine-tuned Keras model. 2. Read
the captured image. 3. Convert it to RGB. 4. Resize to 224 × 224. 5.
Apply exactly the same MobileNetV2 preprocessing used in training. 6.
Produce ten class scores. 7. Map output indices using
`class_names.json`. 8. Sort and display the top three predictions.

Check model artifacts:

``` bash
ls -lh ~/crop_ai_data/models/
cat ~/crop_ai_data/models/class_names.json
```

Locate the inference script:

``` bash
find ~/crop_ai_data -maxdepth 4 -type f \( -iname "*predict*.py" -o -iname "*infer*.py" \)
```

Use the script's actual arguments; inspect the script rather than
guessing command-line flags.

Softmax scores are model output scores, not automatically calibrated
probabilities of disease. Do not present a high score as a confirmed
diagnosis.

## 10. ESP32 sensor node

The sensor firmware must send JSON to:

``` text
http://<LAPTOP_LAN_IP>:5000/api/sensor-data
```

Example payload:

``` json
{
  "temperature": 27.6,
  "humidity": 71.2,
  "soil_moisture": 2400,
  "rain": 1
}
```

Important: send raw soil ADC in `soil_moisture`. The earlier incorrect
endpoint `/api/esp32-data` returned 404. The current backend route is
`/api/sensor-data`.

Find the laptop's current LAN IPv4 address in PowerShell:

``` powershell
ipconfig
```

Use that address in ESP32 firmware. The laptop and ESP32 must be on the
same network. Do not use `127.0.0.1` in firmware.

## 11. ESP32-CAM

For the near-term MVP, preserve the existing known-working
low-resolution camera firmware. Do not replace it with a new sketch
simply to increase resolution.

The earlier custom camera implementation printed URLs in this form:

``` text
Camera homepage: http://<CAMERA_IP>/
Capture:         http://<CAMERA_IP>/capture
Stream:          http://<CAMERA_IP>:81/stream
```

These paths must be confirmed against the currently flashed firmware.
Read the actual Serial Monitor output at 115200 baud after reset.

Test direct capture in PowerShell:

``` powershell
Invoke-WebRequest -Uri "http://<CAMERA_IP>/capture" -OutFile "camera_test.jpg"
```

Open the image and check focus, framing, lighting and orientation.

### Camera format warning

A previous attempt produced:

``` text
JPEG format is not supported on this sensor
Camera initialization failed: 0x106
```

The earlier working code used `PIXFORMAT_RGB565`. The exact sensor and
currently flashed firmware must be checked before changing formats.

Espressif's official camera driver explains sensor-specific pixel-format
support and warns that RGB/YUV can strain ESP32 memory, especially while
Wi-Fi is active. For the MVP, retain the known-working low-resolution
configuration.

Official reference: https://github.com/espressif/esp32-camera

## 12. Flask backend

Known routes in the uploaded `backend/app.py`:

  Method   Endpoint                Purpose
  -------- ----------------------- ------------------------------------
  GET      `/`                     Health check
  GET      `/dashboard`            Dashboard
  GET      `/dashboard/<path>`     Frontend assets and captured files
  GET      `/api/sensor-data`      Latest sensor reading
  POST     `/api/sensor-data`      Receive sensor reading
  GET      `/api/history`          Sensor history
  POST     `/api/chat`             Chatbot
  POST     `/api/camera/capture`   Request and save camera image

Camera capture configuration uses `CAMERA_IP` from `backend/.env`. The
historical default `10.5.135.90` may be stale; use the current camera IP
from Serial Monitor.

The camera route requests:

``` text
http://<CAMERA_IP>/capture
```

It checks the JPEG signature, saves the file under `frontend/captures/`,
and returns an image URL such as:

``` text
/dashboard/captures/<filename>.jpg
```

### 12.1 Start backend on Windows

PowerShell:

``` powershell
cd "D:\VSCODE\PROjects\AI-Driven-IoT-System-for-Smart-Crop-Monitoring-and-Disease-Detection"
```

Activate the Windows virtual environment:

``` powershell
.\venv\Scripts\Activate.ps1
```

If the requirements file exists:

``` powershell
pip install -r backend\requirements.txt
```

Run:

``` powershell
python backend\app.py
```

Use the project's existing `run.bat` only after checking what it
launches.

Open:

``` text
http://127.0.0.1:5000/
http://127.0.0.1:5000/dashboard
```

Verify health:

``` powershell
Invoke-RestMethod -Uri "http://127.0.0.1:5000/"
```

Verify sensor data:

``` powershell
Invoke-RestMethod -Uri "http://127.0.0.1:5000/api/sensor-data"
```

Verify camera capture:

``` powershell
Invoke-RestMethod -Uri "http://127.0.0.1:5000/api/camera/capture" -Method POST
```

Previously observed success returned a filename, an
`/dashboard/captures/...jpg` URL and `success: True`. Re-test on the
current network.

## 13. Dashboard acceptance criteria

Open:

``` text
http://127.0.0.1:5000/dashboard
```

Check: - Dashboard loads without blank screen. - Real ESP32 sensor
values appear. - Source indicates ESP32 when real readings arrive. -
Camera capture button requests a fresh image. - Captured image URL
opens. - Predictions are displayed only when actual inference
succeeds. - Errors are shown clearly; do not show stale or fabricated
results. - Dashboard still works after browser refresh.

## 14. MVP integration: camera → Flask → TensorFlow → dashboard

This is the most important integration to verify.

Target flow: 1. User clicks **Capture and Analyse**. 2. Flask requests
an image from ESP32-CAM. 3. Flask validates and saves the image. 4.
Host-side TensorFlow inference loads the saved image. 5. The model
returns ten class scores. 6. Flask maps indices through
`class_names.json` and returns the top three. 7. Dashboard displays the
actual image and predictions.

A proposed response format (not proof that the current backend already
implements it):

``` json
{
  "success": true,
  "image_url": "/dashboard/captures/example.jpg",
  "predictions": [
    {"class": "Late_blight", "score": 0.82},
    {"class": "Early_blight", "score": 0.11},
    {"class": "Bacterial_spot", "score": 0.04}
  ]
}
```

Integration rules: - Load the model once when the inference service
starts, not on every request. - Match preprocessing and class order
exactly. - Handle missing model files and invalid images. - Keep the
existing camera capture route working. - Never report success if
inference failed.

**Manual MVP fallback:** if automatic inference integration is unstable,
capture the image through Flask and run the existing inference script
manually in WSL. Explain honestly that inference is a separate host-side
step.

## 15. Three-minute demonstration

  -----------------------------------------------------------------------
  Time                                Action
  ----------------------------------- -----------------------------------
  0:00--0:30                          Explain problem and architecture

  0:30--1:00                          Show real sensor readings

  1:00--1:30                          Capture a tomato leaf using
                                      ESP32-CAM

  1:30--2:00                          Run MobileNetV2 inference

  2:00--2:30                          Explain top-three predictions and
                                      scores

  2:30--3:00                          Explain PlantVillage results and
                                      field-validation limitations
  -----------------------------------------------------------------------

Before the demonstration: - Connect laptop and ESP32 boards to the same
network. - Start sensor node and camera. - Confirm camera IP and update
`backend/.env`. - Start Flask. - Open dashboard. - Test one sensor
update, one camera capture and one prediction. - Keep a known-good
captured leaf image and inference output as backup. - Label prerecorded
results as prerecorded.

## 16. Troubleshooting

### Sensor ESP32 cannot reach Flask

-   Check same Wi-Fi network.
-   Check laptop IPv4 address.
-   Use laptop LAN IP in firmware, not localhost.
-   Ensure Flask listens on `0.0.0.0` for LAN access.
-   Check Windows Firewall.
-   Confirm endpoint is `/api/sensor-data`.

### Sensor POST returns 404

The correct route is `/api/sensor-data`, not `/api/esp32-data`.

### Camera capture fails

-   Check camera power and boot logs.
-   Confirm Wi-Fi connection and current IP.
-   Test `http://<CAMERA_IP>/capture` directly.
-   Confirm firmware actually registers `/capture`.
-   Check laptop-to-camera network reachability.
-   If backend expects JPEG, confirm returned bytes begin with
    `FF D8 FF`.

### Dashboard image is broken

-   Inspect capture JSON response.
-   Open the returned image URL in browser.
-   Confirm file exists under `frontend/captures/`.
-   Inspect browser console for 404/path errors.

### TensorFlow GPU is missing

Inside activated WSL environment:

``` bash
nvidia-smi
python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

Keep Windows backend Python and WSL TensorFlow Python environments
separate.

### Unexpected predictions

Check model checkpoint, class mapping, RGB conversion, image resize,
preprocessing, focus, lighting and framing. Camera images may differ
significantly from PlantVillage images.

## 17. Final acceptance checklist

### Hardware

-   [ ] Sensor ESP32 boots.
-   [ ] DHT22 values are plausible.
-   [ ] Soil ADC changes with probe conditions.
-   [ ] Rain sensor logic verified.
-   [ ] ESP32-CAM boots.
-   [ ] Captured image is usable.
-   [ ] Camera IP and URL recorded.

### Backend

-   [ ] Flask health check passes.
-   [ ] Sensor POST succeeds.
-   [ ] Latest sensor GET returns actual data.
-   [ ] SQLite stores readings.
-   [ ] Camera capture route succeeds.
-   [ ] Image is served by Flask.
-   [ ] Inference route tested if implemented.

### AI

-   [ ] TensorFlow environment activates.
-   [ ] Fine-tuned model loads.
-   [ ] Class mapping loads.
-   [ ] Known test image predicts.
-   [ ] Camera image can be passed to inference.
-   [ ] Top-three results are sorted correctly.
-   [ ] Limitations are explained.

### Demonstration

-   [ ] Dashboard loads.
-   [ ] Live sensor data visible.
-   [ ] Camera image visible.
-   [ ] Real model output available.
-   [ ] Backup image and inference output ready.
-   [ ] Demonstration repeatable.

## 18. Roadmap

  -----------------------------------------------------------------------
  Phase                   Scope                   Status
  ----------------------- ----------------------- -----------------------
  Phase 0                 Dashboard structure,    Completed earlier
                          authentication and      
                          dummy data              

  Phase 1                 ESP32 environmental     Working / demonstrated
                          sensing                 

  Phase 2                 Camera acquisition and  Previously
                          backend storage         demonstrated; verify
                                                  current firmware

  Phase 3A                PlantVillage            Completed
                          preparation and         
                          baseline model          

  Phase 3B                Fine-tuning and         Completed
                          held-out evaluation     

  MVP integration         Camera → model →        Verify / complete
                          dashboard               

  MVP validation          Repeatable end-to-end   Pending
                          tests                   

  Competition upgrade     Field dataset,          Future work
                          Grad-CAM, robustness,   
                          enclosure and polish    
  -----------------------------------------------------------------------

## 19. Competition-level future work

After the MVP: 1. Collect independently labelled tomato field images. 2.
Evaluate cross-dataset generalization, including PlantDoc where class
mapping permits. 3. Add Grad-CAM visual explanations. 4. Add blur,
brightness and leaf-presence checks. 5. Add uncertainty-aware rejection.
6. Calibrate confidence and evaluate reliability. 7. Add sensor-history
charts. 8. Add offline buffering during Wi-Fi outages. 9. Design a
protective enclosure and sensor mounts. 10. Measure latency, memory,
power and communication reliability. 11. Package reproducible setup
instructions and a demonstration video.

These are future extensions and must not be claimed as already
implemented.

## 20. Research results and responsible claims

Current reported results: - 18,146 usable PlantVillage tomato images. -
Ten classes. - 12,697 training, 2,723 validation and 2,726 test
images. - Baseline accuracy: 86.83%. - Fine-tuned accuracy: 92.15%. -
Macro F1-score: 90.40%. - Weighted F1-score: 92.28%.

These are held-out PlantVillage results, not field accuracy. Camera
tests establish functional image capture and preliminary inference only
unless independently labelled field evaluation is performed.

Environmental readings provide context; they do not independently
diagnose disease. Model outputs are decision-support predictions, not
confirmed diagnoses.

## 21. Security and safety

-   Keep Wi-Fi passwords and API keys out of source control.
-   Keep real credentials in `.env`; commit only `.env.example`.
-   Do not expose Flask's development server to the public internet.
-   Use correctly rated power supplies and verify GPIO voltage levels.
-   Do not apply pesticides based solely on a model prediction.
-   Preserve dataset manifests, checkpoints and evaluation outputs.

## 22. Official references

-   Espressif ESP32 camera driver:
    https://github.com/espressif/esp32-camera
-   Arduino ESP32 CameraWebServer example:
    https://github.com/espressif/arduino-esp32/tree/master/libraries/ESP32/examples/Camera/CameraWebServer
-   TensorFlow transfer learning guide:
    https://www.tensorflow.org/tutorials/images/transfer_learning
-   PlantVillage dataset:
    https://github.com/spMohanty/PlantVillage-Dataset
-   TensorFlow MobileNetV2 API:
    https://www.tensorflow.org/api_docs/python/tf/keras/applications/MobileNetV2

------------------------------------------------------------------------

## 23. MVP definition of done

The MVP is complete when a user can: 1. Open the dashboard. 2. Observe
real environmental readings. 3. Request a tomato-leaf image from the
ESP32-CAM. 4. Run the trained MobileNetV2 model on the captured image.
5. View the actual ranked predictions beside the image.

The workflow must be repeatable. If inference remains a separate manual
host-side step, document that limitation honestly.

**Engineering objective:** deliver a reliable, demonstrable prototype
with reproducible results, rather than claiming a fully validated
commercial disease-diagnosis system.
