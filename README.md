# Classroom
[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/EbtZGzoI)
[![Open in Codespaces](https://classroom.github.com/assets/launch-codespace-2972f46106e565e64193e422d61a12cf1da4916b45550586e14ef0a7c637dd04.svg)](https://classroom.github.com/open-in-codespaces?assignment_repo_id=23647334)

# wokwi-stitch-webserver

## Elabore un WebServer Micropython y/o CPP en Raspberry PicoW Pico2W via Wokwi

**Nombre:** Angel Cassiel Neyra Mendez

---

## OBJETIVO

Interface objetivo-ficticia sobre un monitoreo del mundo real, que intervenga esta UIX de html-css-js dentro de MicroPython. El tema es abierto y no debe repetirse en el grupo.

---

## DESCRIPCIÓN

**AquaSense Live** es un sistema de monitoreo oceánico en tiempo real simulado, corriendo en una Raspberry Pi Pico W mediante MicroPython. Expone un servidor HTTP que sirve una interfaz web con estética de terminal futurista, mostrando datos de sensores marinos actualizados cada segundo.

### Sensores simulados:
| Sensor       | Rango           | Unidad |
|--------------|-----------------|--------|
| Temperatura  | 23.0 – 27.0     | °C     |
| Salinidad    | 33.8 – 34.6     | ppt    |
| pH           | 7.9 – 8.3       | pH     |
| Flujo bomba  | 12.0 – 13.5     | L/min  |
| Turbidez     | 0.38 – 0.55     | NTU    |
| Latencia     | 30 – 80         | ms     |

---

## CÓDIGO DEL RASPBERRY PI PICO W
- main.py
```python
import network
import socket
import time
import random
import ujson

# ── WiFi ──────────────────────────────────────────────
SSID = "Wokwi-GUEST"
PASSWORD = ""

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(SSID, PASSWORD)
print("Conectando WiFi...")
for _ in range(15):
    if wlan.isconnected():
        break
    time.sleep(1)
    print(".")
print("IP:", wlan.ifconfig()[0])


# ── Sensor simulado ───────────────────────────────────
temp_base = 24.9

def leer_sensor():
    global temp_base
    temp_base += random.uniform(-0.2, 0.2)
    temp_base = max(23.0, min(27.0, temp_base))
    return {
        "temperatura": round(temp_base, 1),
        "salinidad":   round(random.uniform(33.8, 34.6), 1),
        "ph":          round(random.uniform(7.9,  8.3),  1),
        "flujo":       round(random.uniform(12.0, 13.5), 1),
        "turbidez":    round(random.uniform(0.38, 0.55), 2),
        "latencia":    random.randint(30, 80)
    }


# ── HTML ──────────────────────────────────────────────
HTML = """<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>AquaSense Live</title>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700&family=Share+Tech+Mono&display=swap');
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #050e1a; color: #00f0ff;
      font-family: 'Share Tech Mono', monospace;
      display: flex; justify-content: center;
      min-height: 100vh; align-items: flex-start; padding: 20px;
    }
    .card { background: #071828; border: 1px solid #0a3a55; border-radius: 12px; width: 360px; overflow: hidden; }
    .header { display: flex; justify-content: space-between; align-items: center; padding: 10px 16px; border-bottom: 1px solid #0a3a55; }
    .title { font-family: 'Orbitron', sans-serif; font-size: 11px; letter-spacing: 2px; }
    .dot { width: 8px; height: 8px; border-radius: 50%; background: #00ff88; display: inline-block; animation: blink 1.5s infinite; }
    @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: .2; } }
    .times { display: flex; justify-content: space-between; padding: 8px 16px; font-size: 10px; color: #4a8fa8; background: #071828; border-bottom: 1px solid #0a1e2e; }
    .times span { color: #00c8e8; }
    .temp-section { padding: 16px; background: #050e1a; }
    .sensor-label { font-size: 9px; letter-spacing: 3px; color: #4a8fa8; }
    .feels { font-size: 10px; color: #4a8fa8; margin: 4px 0 8px; }
    .temp-big { font-family: 'Orbitron', sans-serif; font-size: 52px; font-weight: 700; line-height: 1; }
    .wave-wrap { height: 60px; overflow: hidden; margin-top: 8px; }
    .wave-wrap svg { width: 100%; height: 60px; }
    .metrics { display: grid; grid-template-columns: 1fr 1fr; gap: 1px; background: #0a2a40; }
    .metric { background: #071828; padding: 10px 16px; }
    .metric-label { font-size: 8px; letter-spacing: 2px; color: #4a8fa8; }
    .metric-val { font-size: 20px; color: #00f0ff; }
    .metric-unit { font-size: 9px; color: #4a8fa8; }
    .log { padding: 12px 16px; background: #050e1a; }
    .log-header { display: flex; justify-content: space-between; font-size: 9px; letter-spacing: 2px; color: #4a8fa8; margin-bottom: 8px; }
    .syncing { color: #00ff88; animation: blink 1.5s infinite; }
    .log-row { display: grid; grid-template-columns: 40px 1fr 64px; font-size: 10px; padding: 3px 0; border-bottom: 1px solid #0a1e2e; }
    .log-t { color: #4a8fa8; }
    .log-v { color: #00c8e8; text-align: center; }
    .warn { color: #ffaa00; }
    .nom  { color: #4a8fa8; }
    .sysbar { display: flex; justify-content: space-between; padding: 6px 16px; font-size: 9px; background: #050e1a; border-top: 1px solid #0a1e2e; }
    .ok { color: #00ff88; }
    .footer { display: flex; justify-content: space-around; padding: 10px; background: #071828; border-top: 1px solid #0a2a40; }
    .nav { display: flex; flex-direction: column; align-items: center; gap: 3px; font-size: 8px; color: #4a8fa8; }
    .nav.active { color: #00f0ff; }
  </style>
</head>
<body>
  <div class="card">

    <div class="header">
      <span class="title">&#9786; AQUASENSE LIVE</span>
      <span><span class="dot"></span> ONLINE</span>
    </div>

    <div class="times">
      <div>SESSION <span id="start">--</span></div>
      <div>TIEMPO  <span id="clock">--:--:--</span></div>
    </div>

    <div class="temp-section">
      <div class="sensor-label">TEMPERATURA DEL AGUA</div>
      <div class="feels">Sensación Térmica: <span id="feels">--</span>°C</div>
      <div class="temp-big" id="tempBig">--.--°C</div>
      <div class="wave-wrap">
        <svg viewBox="0 0 340 60" preserveAspectRatio="none">
          <path id="waveLine" fill="none" stroke="#00f0ff" stroke-width="2" opacity=".8"/>
          <path id="waveFill" fill="#00f0ff" opacity=".06"/>
        </svg>
      </div>
    </div>

    <div class="metrics">
      <div class="metric">
        <div class="metric-label">SALINIDAD</div>
        <span class="metric-val" id="sal">--</span>
        <span class="metric-unit">ppt</span>
      </div>
      <div class="metric">
        <div class="metric-label">NIVEL pH</div>
        <span class="metric-val" id="ph">--</span>
        <span class="metric-unit">pH</span>
      </div>
    </div>

    <div class="log">
      <div class="log-header">
        <span>TELEMETRÍA EN VIVO</span>
        <span class="syncing">&#9679; SYNC</span>
      </div>
      <div id="logRows"></div>
    </div>

    <div class="sysbar">
      <span>SISTEMA: <span class="ok">ESTABLE</span></span>
      <span>LATENCIA: <span id="lat">--</span>ms</span>
    </div>

    <div class="metrics">
      <div class="metric">
        <div class="metric-label">FLUJO BOMBA</div>
        <span class="metric-val" id="flow">--</span>
        <span class="metric-unit">L/MIN</span>
      </div>
      <div class="metric">
        <div class="metric-label">TURBIDEZ</div>
        <span class="metric-val" id="turb">--</span>
        <span class="metric-unit">NTU</span>
      </div>
    </div>

    <div class="footer">
      <div class="nav active">&#9685; MONITOR</div>
      <div class="nav">&#9196; HISTORY</div>
      <div class="nav">&#9888; ALERTS</div>
      <div class="nav">&#9881; SYSTEM</div>
    </div>

  </div>

  <script>
    const hist = [];
    let started = false;

    function fmtTime(d) { return d.toTimeString().slice(0, 8); }

    function updateClock() {
      document.getElementById('clock').textContent = fmtTime(new Date());
    }

    function buildWave() {
      if (hist.length < 2) return;
      const W = 340, H = 60;
      const mn = Math.min(...hist) - 0.5;
      const mx = Math.max(...hist) + 0.5;
      const pts = hist.map((v, i) => {
        const x = W * (i / Math.max(hist.length - 1, 1));
        const y = H - ((v - mn) / (mx - mn)) * (H - 10) - 5;
        return x.toFixed(1) + ',' + y.toFixed(1);
      });
      const d = 'M' + pts.join('L');
      document.getElementById('waveLine').setAttribute('d', d);
      document.getElementById('waveFill').setAttribute('d', d + 'L' + W + ',' + H + 'L0,' + H + 'Z');
    }

    function buildLog() {
      const rows = document.getElementById('logRows');
      rows.innerHTML = '';
      hist.slice(-5).reverse().forEach((v, i) => {
        const w = v > 24.7;
        rows.innerHTML += `<div class="log-row">
          <span class="log-t">T-${i}s</span>
          <span class="log-v">${v.toFixed(1)}°C</span>
          <span class="${w ? 'warn' : 'nom'}">${w ? 'WARN' : 'NOMINAL'}</span>
        </div>`;
      });
    }

    async function fetchData() {
      try {
        const r = await fetch('/data');
        const d = await r.json();
        if (!started) {
          document.getElementById('start').textContent = new Date().toLocaleTimeString();
          started = true;
        }
        hist.push(d.temperatura);
        if (hist.length > 20) hist.shift();
        document.getElementById('tempBig').textContent = d.temperatura.toFixed(1) + '°C';
        document.getElementById('feels').textContent   = (d.temperatura + 1.1).toFixed(1);
        document.getElementById('sal').textContent     = d.salinidad;
        document.getElementById('ph').textContent      = d.ph;
        document.getElementById('flow').textContent    = d.flujo;
        document.getElementById('turb').textContent    = d.turbidez;
        document.getElementById('lat').textContent     = d.latencia;
        buildWave();
        buildLog();
      } catch (e) {
        console.log('fetch error', e);
      }
    }

    updateClock();
    setInterval(updateClock, 1000);
    fetchData();
    setInterval(fetchData, 1000);
  </script>
</body>
</html>"""


# ── Servidor HTTP ─────────────────────────────────────
addr = socket.getaddrinfo("0.0.0.0", 80)[0][-1]
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(addr)
s.listen(5)
print("Servidor en http://", wlan.ifconfig()[0])

while True:
    conn, addr = s.accept()
    request = conn.recv(1024).decode()

    if "GET /data" in request:
        datos = leer_sensor()
        body  = ujson.dumps(datos)
        response = (
            "HTTP/1.1 200 OK\r\n"
            "Content-Type: application/json\r\n"
            "Access-Control-Allow-Origin: *\r\n"
            "Content-Length: " + str(len(body)) + "\r\n"
            "\r\n" + body
        )
    else:
        response = (
            "HTTP/1.1 200 OK\r\n"
            "Content-Type: text/html\r\n"
            "Content-Length: " + str(len(HTML)) + "\r\n"
            "\r\n" + HTML
        )

    conn.send(response)
    conn.close()
```

---

## CAPTURAS DE PANTALLA

## WOKWI
![AquaSense UI](https://github.com/user-attachments/assets/6de730e9-9221-4323-ba7d-46edbc986eb5)

# STITCH 
![AquaSense Wokwi](https://github.com/user-attachments/assets/0e9341c5-6c5d-4bcb-b156-d0cffac9316c)

---

## TECNOLOGÍAS USADAS

- **MicroPython** — lenguaje para la Raspberry Pi Pico W
- **Wokwi** — simulador online de microcontroladores
- **Stitch** — interfaz web embebida


