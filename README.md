# IoT Cold Chain Monitoring System

ESP32 + DHT sensor + OLED + LED  ->  Wi-Fi  ->  Node.js/Express  ->  SQLite  ->  Web dashboard

## 1. Hardware wiring
| Part | ESP32 pin |
|---|---|
| DHT11/DHT22 DATA | GPIO4 (VCC 3V3, GND) |
| OLED SDA | GPIO21 |
| OLED SCL | GPIO22 (VCC 3V3, GND) |
| Red LED | GPIO5 via 220 ohm resistor to GND |
| Buzzer (optional) | GPIO18 to GND |

## 2. Run the server
Install Node.js 18+ then:
```
cd coldchain
npm install
cp .env.example .env              # edit API_KEY etc. (Windows: copy .env.example .env)
npm start
```
Open http://localhost:3000  (database is created automatically in `data/coldchain.db`)

Optional Telegram alerts: set `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` in `.env`.

## 3. Flash the ESP32
1. Arduino IDE -> install ESP32 board support.
2. Library Manager: DHT sensor library, Adafruit Unified Sensor, Adafruit GFX, Adafruit SSD1306, ArduinoJson (v7).
3. Open `firmware/cold_chain/cold_chain.ino` and edit WIFI_SSID, WIFI_PASS, SERVER_URL (your PC's local IP, e.g. http://192.168.1.50:3000/api/readings), API_KEY (same as server), and DHTTYPE.
4. Upload. Serial Monitor at 115200 shows status.

PC and ESP32 must be on the same Wi-Fi. Allow port 3000 in your firewall.


## Project structure
```
coldchain/
  backend/
    server.js              # entry point
    src/config.js db.js alerts.js notify.js
    src/middleware/auth.js # x-api-key (device), x-admin-key (admin)
    src/routes/            # readings, alerts, settings, devices, stats
  public/index.html        # dashboard
  firmware/cold_chain/     # ESP32 code
  data/                    # SQLite DB (auto-created)
  .env.example
```

## API reference (base: /api)
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | /readings | x-api-key | ESP32 sends `{device,temp,hum}`; returns `alert` + current limits |
| POST | /readings/batch | x-api-key | `{device, readings:[{temp,hum,ts?}]}` up to 500, for offline buffered data |
| GET | /readings?hours=1&device= | - | Chart data (downsampled to 600 points) |
| GET | /latest?device= | - | Latest reading + settings |
| GET | /export.csv?device= | - | CSV download |
| DELETE | /readings?olderThanDays=30 | x-admin-key | Delete old data (needs `ADMIN_KEY` in .env) |
| GET | /alerts?limit=&device=&unacked=1 | - | Alert history |
| POST | /alerts/:id/ack , /alerts/ack-all | - | Acknowledge alerts |
| GET/POST | /settings | - | `{temp_min,temp_max,hum_max}` |
| GET | /devices , /devices/:id | - | Devices with online/offline + last values |
| GET | /stats?hours=24&device= | - | min/max/avg, out-of-range count, compliance % |
| GET | /health | - | Server health check |

Set `RETENTION_DAYS` in `.env` to auto-delete old readings.

## 4. Test without hardware
```
curl -X POST http://localhost:3000/api/readings -H "Content-Type: application/json" -H "x-api-key: mysecret" -d '{"temp":4.5,"hum":55}'
curl -X POST http://localhost:3000/api/readings -H "Content-Type: application/json" -H "x-api-key: mysecret" -d '{"temp":11,"hum":55}'   # triggers alert
```

## How alerts work
- Default safe range 2-8 C, humidity max 70% (change on the dashboard).
- The ESP32 also checks limits locally, so LED/buzzer still work if Wi-Fi or the server is down.
- Server logs each alert (max one per type every 5 min) and optionally sends Telegram.

## Notes
- Dashboard settings are not password protected; keep this on your local network.
- For deployment on the internet, use HTTPS and a strong API_KEY.
