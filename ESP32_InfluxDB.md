Awesome! Using **InfluxDB** with **Grafana** is a powerful way to store and visualize time-series data like temperature and humidity. Since you’re already familiar with Python and Raspberry Pi/Arduino, this is a great next step into more advanced IoT + data stack territory.

---

Here’s how to integrate **InfluxDB** + **Grafana** into your ESP32 sensor project — GitHub-style, so you can add it to your repository!

---

## 📈 InfluxDB + Grafana Setup for ESP32 Sensor Network

This section expands the existing project by introducing a **time-series database (InfluxDB)** and a **real-time dashboard (Grafana)** for persistent storage and rich visualization of sensor data.

---

## 🧱 Architecture Overview

```plaintext
ESP32 Nodes ---> Flask Server ---> InfluxDB ---> Grafana
    [POST JSON]         [Python API]    [time-series DB]  [dashboard UI]
```

---

## 🧰 Tech Stack

| Component               | Purpose                                 |
| ----------------------- | --------------------------------------- |
| **ESP32**               | Reads and sends sensor data             |
| **Flask API**           | Receives data and writes to InfluxDB    |
| **InfluxDB**            | Stores time-series data                 |
| **Grafana**             | Visualizes InfluxDB data                |
| **Docker** *(optional)* | Easy deployment of InfluxDB and Grafana |

---

## 🐍 Updated Flask Server with InfluxDB Support

### 1. Install Required Python Packages

```bash
pip install flask influxdb-client
```

### 2. `app.py` — Submit to InfluxDB

```python
from flask import Flask, request
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS
import os

# === CONFIG ===
INFLUX_URL = "http://localhost:8086"
INFLUX_TOKEN = "your_influxdb_token"
INFLUX_ORG = "your_org"
INFLUX_BUCKET = "sensor_data"

app = Flask(__name__)

# === InfluxDB Client ===
client = InfluxDBClient(url=INFLUX_URL, token=INFLUX_TOKEN, org=INFLUX_ORG)
write_api = client.write_api(write_options=SYNCHRONOUS)

@app.route('/submit', methods=['POST'])
def submit_data():
    data = request.json
    node_id = data.get('node_id')
    temp = data.get('temperature')
    hum = data.get('humidity')

    point = (
        Point("environment")
        .tag("node", node_id)
        .field("temperature", temp)
        .field("humidity", hum)
    )

    write_api.write(bucket=INFLUX_BUCKET, org=INFLUX_ORG, record=point)
    return {'status': 'written to influxdb'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## 📦 ESP32 Firmware (unchanged)

The ESP32 continues to `POST` to `/submit`, as before — no changes needed unless you want to add timestamps on the ESP32 side (InfluxDB can auto-timestamp data).

---

## 🐳 Run InfluxDB + Grafana Using Docker (Recommended)

### 1. `docker-compose.yml`

```yaml
version: '3.7'

services:
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    ports:
      - "8086:8086"
    volumes:
      - influxdb-data:/var/lib/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=admin123
      - DOCKER_INFLUXDB_INIT_ORG=my-org
      - DOCKER_INFLUXDB_INIT_BUCKET=sensor_data
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=my-secret-token

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - influxdb

volumes:
  influxdb-data:
  grafana-data:
```

### 2. Run the Stack

```bash
docker-compose up -d
```

* InfluxDB → `http://localhost:8086`
* Grafana → `http://localhost:3000` (default user/pass: `admin` / `admin`)

---

## 📊 Configure Grafana

### 1. Add Data Source

* Go to **[http://localhost:3000](http://localhost:3000)**
* Login: `admin / admin`
* Add **InfluxDB** data source

  * URL: `http://influxdb:8086`
  * Auth Token: `my-secret-token`
  * Org: `my-org`
  * Bucket: `sensor_data`

### 2. Create Dashboard

* New Panel → Query `temperature` or `humidity` from `environment` measurement
* Use `node` as a tag to split values by location (e.g., `room1`, `kitchen`)
* Example Flux query:

```flux
from(bucket: "sensor_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "environment")
  |> filter(fn: (r) => r._field == "temperature")
  |> group(columns: ["node"])
```

* Add multiple panels to monitor each metric and location.

---

## 🔁 Recommended Update Intervals

| Interval | Use Case                       |
| -------- | ------------------------------ |
| 1 min    | Near real-time home automation |
| 5 min    | Energy-efficient monitoring    |
| 30+ min  | Outdoor or less dynamic data   |

---

## 📘 Example Repository Additions

```
esp32-temp-monitor/
├── /server/
│   ├── app.py
│   ├── influx_config.env
│   └── requirements.txt
├── docker-compose.yml
```

---

## ✅ TODOs

* [ ] Add authentication to Flask `/submit` endpoint
* [ ] Secure InfluxDB with HTTPS
* [ ] Set up Grafana alerts for high temp/humidity
* [ ] Auto-deploy with systemd or Docker on a Raspberry Pi

---

## 📜 License

MIT

---

Let me know if you want:

* A **prebuilt sample dashboard JSON** for Grafana import
* How to **persist InfluxDB and Grafana** on Raspberry Pi
* A **MicroPython version** of the ESP32 firmware

Ready to proceed with wiring diagrams or the full code export?
