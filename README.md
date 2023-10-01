# DHT_InfluxDB

This is a simple DHT sensor data logger with InfluxDB.

## Policy
Data duration: `1 year`
```sql
CREATE RETENTION POLICY sensor_data_rp ON sensor_data DURATION 365d REPLICATION 1 DEFAULT
```
## Daemon

Create a systemd service unit file with a .service extension.
```ini
[Unit]
Description=Sensor Data Monitoring Service
After=network.target

[Service]
ExecStart=/usr/bin/python3 /path/to/your/script.py
WorkingDirectory=/path/to/your/script/directory
Restart=always
User=user  # Replace with your username
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```
### Reload systemd
```bash
sudo systemctl daemon-reload
```
### Enable and start
```bash
sudo systemctl enable sensor_data.service
sudo systemctl start sensor_data.service
```
## Code
```python
import Adafruit_DHT
from influxdb import InfluxDBClient
import time

DHT_SENSOR = Adafruit_DHT.DHT11
DHT_PIN = 4       # GPIO pin for data
DLT_EACH = 518400 # There are 518400 "5 seconds" in a month
COUNTER = 0

# InfluxDB configuration
INFLUXDB_HOST = 'localhost'
INFLUXDB_PORT = 8086
INFLUXDB_DATABASE = 'sensor_data'
INFLUXDB_RETENTION_POLICY = 'sensor_data_rp'

client = InfluxDBClient(host=INFLUXDB_HOST, port=INFLUXDB_PORT)
client.switch_database(INFLUXDB_DATABASE)

while True:
    h, t = Adafruit_DHT.read(DHT_SENSOR, DHT_PIN)
    if h is not None and t is not None:
        json_body = [
            {
                "measurement": "temperature_humidity",
                "tags": {},
                "time": time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime()),
                "fields": {
                    "temperature": t,
                    "humidity": h
                }
            }
        ]
        client.write_points(json_body, retention_policy=INFLUXDB_RETENTION_POLICY)
        print("Temp={0:0.1f}C Humidity={1:0.1f}%".format(t, h))
        if COUNTER % DLT_EACH == 0:
            client.query('DELETE FROM temperature_humidity WHERE time < now() - 365d', database=INFLUXDB_DATABASE)
            print("Year old data deleted.")
        COUNTER += 1
        time.sleep(5)
```
